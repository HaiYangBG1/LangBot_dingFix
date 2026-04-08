# LangBot 自定义补丁记录

> 本文件记录对 LangBot 开源项目所做的本地修改，方便项目升级后快速复刻。

---

## Patch #1：钉钉语音消息使用 recognition 转写文字

- **修改日期**：2026-04-08
- **首次应用版本**：v4.9.5
- **修改人**：本地开发

### 问题描述

钉钉用户发送语音消息后，LangBot 返回 `Request failed`。

**根因**：LangBot 将语音消息打包为 `file_base64` 类型的 ContentElement，直接透传给 LLM API（qwen-max、gemma-4、gemini-3 等），这些模型的 OpenAI 兼容接口均不支持 `file_base64` 类型，返回 400 错误：

```
Invalid value: file_base64. Supported values are: 'text', 'image_url', 'video_url' and 'video'.
```

**关键发现**：钉钉发送给机器人的语音消息回调 JSON 中，`content` 字段已包含 `recognition`（由 Qwen 提供的语音转文字结果），但 LangBot 代码完全忽略了这个字段，只取了 `downloadCode` 去下载音频二进制。

钉钉语音消息原始数据结构：
```json
{
    "msgtype": "audio",
    "content": {
        "downloadCode": "xxx",
        "duration": "3000",
        "recognition": "你好，可以听到我说话吗?"
    }
}
```

---

### 第一步：检查当前版本是否已原生支持

升级 LangBot 后，**先执行以下检查**，确认官方是否已修复此问题。如果已支持，则无需打补丁。

#### 检查点 1：api.py 中是否提取了 recognition

打开 `src/langbot/libs/dingtalk_api/api.py`，搜索 `message_type == 'audio'`，查看该分支代码：

- ✅ **已支持**：代码中包含 `recognition` 关键字，且将其赋值给了 `message_data['Content']` 或类似字段
- ❌ **未支持**：代码仅使用 `downloadCode` 下载音频，没有提取 `recognition`

**未支持的典型代码特征**（仅有 downloadCode 逻辑）：
```python
elif incoming_message.message_type == 'audio':
    message_data['Audio'] = await self.get_audio_url(incoming_message.to_dict()['content']['downloadCode'])
    message_data['Type'] = 'audio'
```

#### 检查点 2：dingtalk.py 中是否优先使用转写文字

打开 `src/langbot/pkg/platform/sources/dingtalk.py`，搜索 `event.audio`，查看消息转换逻辑：

- ✅ **已支持**：在 `event.audio` 的处理中有对 `recognition` / `event.content` 的判断，优先使用文字
- ❌ **未支持**：直接 `Voice(base64=event.audio)` 无任何文字优先逻辑

**未支持的典型代码特征**：
```python
if event.audio:
    yiri_msg_list.append(platform_message.Voice(base64=event.audio))
```

#### 结论

| 检查点 1 | 检查点 2 | 操作 |
|:--------:|:--------:|------|
| ✅ | ✅ | **停止！官方已支持，无需打补丁** |
| ❌ | ❌ | 继续执行下方补丁 |
| ❌ | ✅ 或 ✅ | ❌ | 仅修改未支持的文件 |

---

### 第二步：应用补丁（仅在检查未通过时执行）

#### 文件 1：`src/langbot/libs/dingtalk_api/api.py`

**位置**：`get_message()` 方法中 `incoming_message.message_type == 'audio'` 分支

**修改前**：
```python
elif incoming_message.message_type == 'audio':
    message_data['Audio'] = await self.get_audio_url(incoming_message.to_dict()['content']['downloadCode'])

    message_data['Type'] = 'audio'
```

**修改后**：
```python
elif incoming_message.message_type == 'audio':
    raw_content = incoming_message.to_dict().get('content', {})
    # 兼容处理：如果 content 仍为 JSON 字符串则进行解析
    if isinstance(raw_content, str):
        try:
            raw_content = json.loads(raw_content)
        except (json.JSONDecodeError, TypeError):
            raw_content = {}

    if self.logger:
        await self.logger.info(f'DingTalk audio raw content: {json.dumps(raw_content, ensure_ascii=False)}')

    # 提取钉钉自带的语音转写文字（Powered by Qwen）
    recognition = raw_content.get('recognition', '')
    if recognition:
        message_data['Content'] = recognition

    download_code = raw_content.get('downloadCode')
    if download_code:
        message_data['Audio'] = await self.get_audio_url(download_code)

    message_data['Type'] = 'audio'
```

---

#### 文件 2：`src/langbot/pkg/platform/sources/dingtalk.py`

**位置**：`DingTalkMessageConverter.target2yiri()` 方法

**修改前**：
```python
        else:
            # 回退到原有简单逻辑
            if event.content:
                text_content = event.content.replace('@' + bot_name, '')
                yiri_msg_list.append(platform_message.Plain(text=text_content))
            if event.picture:
                yiri_msg_list.append(platform_message.Image(base64=event.picture))

            # 处理其他类型消息（文件、音频等）
        if event.file:
            yiri_msg_list.append(platform_message.File(url=event.file, name=event.name))
        if event.audio:
            yiri_msg_list.append(platform_message.Voice(base64=event.audio))
```

**修改后**：
```python
        else:
            # 回退到原有简单逻辑
            # 对于音频消息，content 来自 recognition 转写文字，在下方音频处理块中统一处理
            if event.content and event.type != 'audio':
                text_content = event.content.replace('@' + bot_name, '')
                yiri_msg_list.append(platform_message.Plain(text=text_content))
            if event.picture:
                yiri_msg_list.append(platform_message.Image(base64=event.picture))

            # 处理其他类型消息（文件、音频等）
        if event.file:
            yiri_msg_list.append(platform_message.File(url=event.file, name=event.name))
        if event.audio:
            # 优先使用钉钉自带的语音转写文字（recognition字段）
            if event.content and event.type == 'audio':
                yiri_msg_list.append(platform_message.Plain(text=event.content))
            else:
                yiri_msg_list.append(platform_message.Voice(base64=event.audio))
```

### 修改逻辑说明

```
修改前的数据流：
  钉钉语音 → downloadCode → 下载音频二进制 → Voice(base64=...) → file_base64 → LLM API → ❌ 400 Error

修改后的数据流：
  钉钉语音 → recognition 字段 → Plain(text=...) → text → LLM API → ✅ 正常回复
```

1. **api.py**：从钉钉回调的 `content` 中提取 `recognition`（转写文字），存入 `message_data['Content']`
2. **dingtalk.py**：
   - 通用内容处理块中排除 audio 类型（避免重复添加文字）
   - 音频处理块中优先用 `recognition` 文字，无转写时才回退到音频二进制

### 验证方法

1. 重启 LangBot
2. 发送语音消息到钉钉机器人
3. 查看日志 `data/logs/langbot-*.log`，搜索 `DingTalk audio raw content`
4. 确认日志中的 JSON 包含 `recognition` 字段且有转写文字
5. 确认机器人正常回复（而非 `Request failed`）
