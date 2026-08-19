# voice_models 完整 20 字段（填 settings.yml 防漏）

> 来源：梵高 v1.0 成品 settings.yml 实测。字段**必须 20 个齐全**（速通版 4.3/4.4.1 纪律；pro 第 6 章 #24），顺序照抄下面，不要省。
> `'0'`（带引号）= 字符串；`0`（不带引号）= 整数；`null` = 空值。

```yaml
voice_models:
  sva_speaker_id: '0'                    # SVA 引擎说话人 id（字符串）
  sbv2_name: Van Gogh                    # 外部 SBV2 服务模型名（= 训练模型名）
  sbv2_speaker_id: '0'                   # 外部 SBV2 说话人 id（字符串）
  bv2_speaker_id: '0'                    # BV2 引擎说话人 id（字符串）
  sbv2api_name: Van Gogh                 # sbv2api 引擎模型名
  sbv2api_speaker_id: '0'                # sbv2api 说话人 id（字符串）
  gsv_voice_text: ''                     # GPT-SoVITS 备用文本（未用留空）
  gsv_voice_filename: ''                 # GPT-SoVITS 参考音频文件名（未用留空）
  gsv_gpt_model_name: ''                 # GPT-SoVITS GPT 模型名（未用留空）
  gsv_sovits_model_name: ''              # GPT-SoVITS SoVITS 模型名（未用留空）
  aivis_model_uuid: ''                   # Aivis 引擎模型 uuid（未用留空）
  opentts_voice: null                    # OpenTTS 音色（未用 null）
  fish_s2_voice: null                    # Fish-S2 音色（未用 null）
  sbv2_local_voice_id: van-gogh          # 本地 SBV2 语音 id（kebab-case ASCII 小写，见 pro 第 5 章）
  sbv2_local_speaker_id: 0               # 本地 SBV2 说话人 id（整数）
  sbv2_local_style_id: null              # 本地 SBV2 风格 id（整数或 null）
  sbv2_local_length_scale: null          # 本地 TTS 语速（null=默认 1.0，填 0.85 变快，见 pro 第 5 章）
  sbv2_local_sdp_ratio: null             # 本地 TTS SDP 混合比（null=默认）
  sbv2_local_cloud_fallback_model: null  # 本地 TTS 云端备用模型（未用 null）
  sbv2_local_cloud_fallback_speaker_id: null  # 云端备用说话人 id（未用 null）
```

## 常用组合

| 使用场景 | 关键字段 |
|---|---|
| 外部 API TTS（VoiceSpeechMaker 5000） | `tts_type: sbv2` + `sbv2_name` / `sbv2_speaker_id` |
| 内置本地 TTS（电脑/手机通用） | `tts_type: localsbv2api` + `sbv2_local_voice_id` + `sbv2_local_speaker_id` |
| 本地 TTS + 语速调整 | 上面 + `sbv2_local_length_scale: 0.85` |
| 迁移旧角色到本地 TTS | 只改 `tts_type` 和 `sbv2_local_voice_id`，其余字段不动（5.0） |
