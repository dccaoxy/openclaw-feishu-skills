# MiniMax TTS API 参考

## 端点

```
POST https://api.minimaxi.com/v1/t2a_v2
```

## 请求头

| Header | Value |
|--------|-------|
| Authorization | Bearer \<API_KEY\> |
| Content-Type | application/json |

## 请求体

```json
{
  "model": "speech-2.8-hd",
  "text": "要转换的文字",
  "voice_setting": {
    "voice_id": "female-tianmei",
    "speed": 1.0,
    "vol": 1.0,
    "pitch": 0
  },
  "audio_setting": {
    "sample_rate": 32000,
    "bitrate": 128000,
    "format": "mp3"
  }
}
```

## 支持模型

| 模型 | 说明 |
|------|------|
| speech-2.8-hd | 最新 HD 模型（推荐） |
| speech-2.6-hd | HD 模型 |
| speech-02-hd | ❌ Max-极速版不支持 |

## 音色列表

| 音色 | voice_id |
|------|----------|
| 甜美女性 | female-tianmei |
| 御姐 | female-yujie |
| 青年男性 | male-qn-qingse |
| 主持人 | presenter_male |
| 男性有声书 | audiobook_male_1 |

## 响应格式

```json
{
  "data": {
    "audio": "<hex encoded audio>",
    "status": 2
  },
  "extra_info": {
    "audio_length": 2808,
    "audio_sample_rate": 32000,
    "audio_size": 46644,
    "word_count": 17
  },
  "base_resp": {
    "status_code": 0,
    "status_msg": "success"
  }
}
```

## 情感标签

在文字中使用：

```
(laughs) - 笑声
(sighs) - 叹气
(coughs) - 咳嗽
```

## 停顿控制

```
<#0.5#> - 0.5秒停顿
<#1.0#> - 1秒停顿
```
