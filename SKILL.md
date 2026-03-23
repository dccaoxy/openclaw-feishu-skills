---
name: feishu-tts
description: 使用 MiniMax TTS 生成语音并通过飞书发送原生音频消息。适用场景：用户请求语音回复、TTS 语音合成、飞书音频消息发送。当用户说"发语音"、"语音回复"、"TTS"时激活。
---

# Feishu TTS Skill

将文字转为语音，并通过飞书发送原生音频消息（非文件附件）。

## 完整流程

```
文字 → MiniMax TTS API → MP3 → ffmpeg转OPUS → ffprobe获取时长 → 上传飞书(带duration) → 发送audio消息
```

## Step 1: 生成语音

```bash
API_KEY="<MiniMax API Key>"
TEXT="要转换的文字"
MODEL="speech-2.8-hd"  # Max-极速版支持
VOICE_ID="female-tianmei"  # 可选音色

curl -s -X POST "https://api.minimaxi.com/v1/t2a_v2" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"$MODEL\",\"text\":\"$TEXT\",\"voice_setting\":{\"voice_id\":\"$VOICE_ID\"},\"audio_setting\":{\"format\":\"mp3\"}}" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
audio_hex = data['data']['audio']
audio_bytes = bytes.fromhex(audio_hex)
with open('/tmp/tts_output.mp3', 'wb') as f:
    f.write(audio_bytes)
print('MP3 saved, size:', len(audio_bytes))
"
```

## Step 2: 转换格式并获取时长

```bash
# 转换为 OPUS 格式
ffmpeg -i /tmp/tts_output.mp3 -c:a libopus -b:a 128k /tmp/tts_output.opus -y

# 获取时长（秒转毫秒）
DURATION_MS=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 /tmp/tts_output.opus | python3 -c "import sys; print(int(float(sys.stdin.read().strip()) * 1000))")

echo "Duration: $DURATION_MS ms"
```

## Step 3: 获取飞书 Access Token

```bash
APP_ID="<飞书 App ID>"
APP_SECRET="<飞书 App Secret>"

TOKEN=$(curl -s -X POST "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d "{\"app_id\": \"$APP_ID\", \"app_secret\": \"$APP_SECRET\"}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('tenant_access_token',''))")

echo "Token: $TOKEN"
```

## Step 4: 上传音频文件

**关键：上传时必须带 `duration` 参数（毫秒），否则飞书显示时长为0**

```bash
FILE_KEY=$(curl -s -X POST "https://open.feishu.cn/open-apis/im/v1/files" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file_type=opus" \
  -F "file_name=tts_output.opus" \
  -F "duration=$DURATION_MS" \
  -F "file=@/tmp/tts_output.opus" \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('data',{}).get('file_key',''))")

echo "File key: $FILE_KEY"
```

## Step 5: 发送原生音频消息

```bash
CHAT_ID="<飞书会话ID>"

curl -s -X POST "https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"receive_id\": \"$CHAT_ID\",
    \"msg_type\": \"audio\",
    \"content\": \"{\\\"file_key\\\": \\\"$FILE_KEY\\\"}\"
  }"
```

## 音色选择 (voice_id)

| 音色 | voice_id |
|------|----------|
| 甜美女性 | female-tianmei |
| 御姐 | female-yujie |
| 青年男性 | male-qn-qingse |
| 主持人 | presenter_male |
| 男性有声书 | audiobook_male_1 |

## 可用模型

| 套餐 | 支持模型 |
|------|----------|
| Max-极速版 | speech-2.8-hd, speech-2.6-hd, speech-02-hd |
| **注意** | speech-02-hd 不支持，需用 speech-2.8-hd |

## 快速脚本

将以下内容保存为 `send_feishu_tts.sh`：

```bash
#!/bin/bash
# send_feishu_tts.sh <文字> [输出路径]

TEXT="$1"
OUTPUT_DIR="${2:-/tmp}"

API_KEY="<MiniMax API Key>"
APP_ID="<飞书 App ID>"
APP_SECRET="<飞书 App Secret>"
CHAT_ID="<飞书会话ID>"

MP3_FILE="$OUTPUT_DIR/tts_$$.mp3"
OPUS_FILE="$OUTPUT_DIR/tts_$$.opus"

# 1. 生成语音
curl -s -X POST "https://api.minimaxi.com/v1/t2a_v2" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"speech-2.8-hd\",\"text\":\"$TEXT\",\"voice_setting\":{\"voice_id\":\"female-tianmei\"},\"audio_setting\":{\"format\":\"mp3\"}}" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
audio_hex = data['data']['audio']
audio_bytes = bytes.fromhex(audio_hex)
with open('$MP3_FILE', 'wb') as f: f.write(audio_bytes)
"

# 2. 转换格式
ffmpeg -i "$MP3_FILE" -c:a libopus -b:a 128k "$OPUS_FILE" -y

# 3. 获取时长
DURATION_MS=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "$OPUS_FILE" | python3 -c "import sys; print(int(float(sys.stdin.read().strip())*1000))")

# 4. 获取 Token
TOKEN=$(curl -s -X POST "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d "{\"app_id\":\"$APP_ID\",\"app_secret\":\"$APP_SECRET\"}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('tenant_access_token',''))")

# 5. 上传
FILE_KEY=$(curl -s -X POST "https://open.feishu.cn/open-apis/im/v1/files" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file_type=opus" \
  -F "file_name=tts.opus" \
  -F "duration=$DURATION_MS" \
  -F "file=@$OPUS_FILE" \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('data',{}).get('file_key',''))")

# 6. 发送
curl -s -X POST "https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"receive_id\":\"$CHAT_ID\",\"msg_type\":\"audio\",\"content\":\"{\\\"file_key\\\":\\\"$FILE_KEY\\\"}\"}"

# 清理
rm -f "$MP3_FILE" "$OPUS_FILE"
```

## 故障排查

| 问题 | 原因 | 解决 |
|------|------|------|
| 语音显示0秒 | 上传时没加 duration 参数 | 必须加 `duration=<毫秒>` |
| API 返回 2061 | 模型名称错误 | 用 `speech-2.8-hd` 而非 `speech-02-hd` |
| Token 获取失败 | App ID/Secret 错误 | 检查飞书应用凭证 |
| MP3 生成失败 | API Key 无效 | 检查 MiniMax Token Plan |

## References

- MiniMax TTS API: `references/minimax-tts.md`
- Feishu Audio API: `references/feishu-audio.md`
