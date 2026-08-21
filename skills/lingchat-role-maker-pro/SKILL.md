---
name: lingchat-role-maker-pro
description: LingChat 角色制作**进阶扩展**：前置速通版 lingchat-role-maker（已跑通「能说话的角色」）。按需加载：语音全量补全（灵衣/新皮肤/漏下载）、BGM 人声分离补充语料（demucs+VAD+whisper）、立绘补全/全身化（psd-tools）、语速与合成参数调优、本地 TTS 备选方案（.sbv2 转换/导入）、完整踩坑大全（#1~#43）、手机端部署（v0.4.7+ APK/局域网同步/MT管理器）、视觉模型可选配（角色看图/DeepSeek deepseek-v4-flash-vision-exp、阿里百炼官方推荐、智谱 4V-Flash 免费）、非 FGO 迁移思路。**不适合**：还没跑通速通版、只做 FGO 速通角色的用户。
---

# LingChat 角色制作进阶扩展（lingchat-role-maker-pro）

> 本 skill 是 **lingchat-role-maker（速通版）的进阶扩展**。**先跑通速通版**（至少到「角色能说话」），再按需加载本 skill 的对应章节。
> 本 skill **不重复速通版内容，只写增量**；引用「速通版 X.Y」指 lingchat-role-maker 的章节，引用「本章 X」指本 skill 的章节。

## 0. 概览与加载时机

| 章节 | 什么时候需要 | 前置 |
|---|---|---|
| 1 语音全量补全 | 想补灵衣/新皮肤/漏下载的语音（语音越多越像，30min+/200条+） | 速通版第 1 章数据集已建 |
| 2 补充语音流水线 | 官方语音不够，想加剧情语音合集（BGM 分离） | 本章 1 + 整合包 env |
| 3 立绘补全/全身化 | 角色移动露白穿帮，想从根上解决 | 速通版第 4 章角色文件已搭 |
| 4 语速调优 | 角色说话太快/太慢/太机械 | 速通版第 5 章 TTS 已通 |
| 5 本地 TTS 备选 | 嫌开黑窗口麻烦/想在手机用/想零外部依赖 | 速通版第 5 章 TTS 已通 |
| 6 完整踩坑大全 | 遇到速通版没覆盖的报错，或想系统排障 | — |
| 7 手机端 | 想把角色搬到手机玩 | 本章 5 或外部 API 已通 |
| 8 非FGO迁移 | 想给其他游戏做角色 | 速通版全流程跑通 |
| 9 进阶验证清单 | 做完进阶项逐项检查 | — |
| 11 视觉模型（可选） | 想让角色看图 / 用户问视觉模型 key 获取与配置 | — |

---

## 1. 语音全量补全（灵衣/新皮肤/漏下载）

### 1.1 何时需要

同一 CV 的新形态（灵衣、新皮肤、活动从者形态，如矿工梵高 SV428）语音**并入同一训练模型，不用新建模型**。速通版已覆盖「开场问角色名时主动问是否有其他形象」；以下场景做全量补全：

- 一开始用户要求不爬取全部语音，后修改为需要爬取全部语音；
- 需要补充新语音（灵衣/新皮肤/新活动形态上线）；
- 首次解析漏条目（语音页跨表编号覆盖，见 6.5 #41）。

### 1.2 补全步骤

1. **解析语音页全部条目**：按速通版 1.2 / 速通版 7.2 正则解析出该页所有 mp3 + 日文原文（如矿工页 83 条、本尊页 106 条）。
2. **对比现有 esd.list 找缺失**：esd 里是 `.wav`，wiki 是 `.mp3`，先 `name.replace('.wav','.mp3')` 再比对；缺失 = 解析出但 esd 没有的条目（含本尊页缺的也一起补）。
3. **MediaWiki API 批量查文件 URL 下载**：`https://fgo.wiki/api.php?action=query&titles=File:a.mp3|File:b.mp3&prop=imageinfo&iiprop=url&format=json`，titles 用 `|` 分隔**一次查 50 个**；返回 key 规范化见 6.5 #20；直链在 media.fgo.wiki，国内可直连（无需代理）。mp3 原件存「mp3_补下载/」备查。
4. **转 wav + 追加 esd.list**：ffmpeg 转 44100Hz/单声道/PCM16；追加前读现有 esd 文件名集合，跳过已存在（防重，见 6.9 #34）。
5. **双数据集同步**：工作区维护的数据集副本 + 整合包 `Data/<模型名>/` 训练集**两处都要同步**（esd.list 全量覆盖 + raw 增量复制），只同步一边会导致训练数据不齐（见 6.9 #34）。
6. **退路和最终手段**：多次、反复进行全量补全容易使语料库混乱、序号错误或其他问题，导致训练失败。这时建议直接重新一次性爬取完整语音，而不是再逐一比对哪条已存在哪条需要补，或者二分法寻找排除问题语音，直接全部重来，新建语料库。

完整脚本模板见 10.1，质量检查见速通版 7.4。

### 1.3 相关解析坑

- **语音页必须按 VoiceTable 表格切分解析，不能全局 kv 收集**（6.5 #41）：fgo.wiki 语音页**多个表格（战斗形象1 2 / 战斗形象3 / 召唤强化 / 个人空间等）各自从编号 1 重新编号**，全局 `|标题(\d+)=` 收集会把前面的条目全部覆盖（实战：阿尔托莉雅·卡斯特全量解析只出 34 条，按表格切分后 119 条）。正确做法：`re.split(r'\{\{#invoke:VoiceTable\|table\|表格标题=([^\n]+)', wiki)` 按表格切块，块内再按编号收集（保留表格标题做分类）；反向引用跨行正则仅作单表兜底（见速通版 1.2）。
- **MediaWiki 批量查 URL 个别文件漏掉 → allimages 前缀搜索兜底**（6.5 #42）：titles 批量查询（一次 50 个）偶尔个别文件查不到（常见于中文文件名，key 规范化后仍匹配不上；实战：本尊 119 个 mp3 漏 2 个生日语音）→ 下载完必须核对「清单文件数 vs 实际下载数」；缺失的用 `action=query&list=allimages&aiprefix=<SV编号前缀>&ailimit=N` 前缀搜索列出真实文件名+直链，逐个补下。

---

## 2. 补充语音流水线（BGM 人声分离）

### 2.1 训练数据量建议

SBV2 训练建议 **30 分钟+ 干净人声（200+ 条）**。实战教训：仅 182 条 FGO 语音（约 6~10 分钟，多为 0.5~3s 短句）训练效果明显不足（音色泛化差）。**官方语音不够时的补充来源**：B 站下载从者「情人节」「活动」等剧情语音合集视频 → 提取音频 → 人声分离 → 切分转录。**下载工具推荐 BiliDown**（开源 B 站下载器：https://github.com/MLChinoo/bilidown-reborn/releases/tag/1.2.6 ，或任意支持 B 站音视频下载的工具，先检查本机有没有）。

### 2.2 人声分离（htdemucs）

带 BGM 的语音必须人声分离，直接训练会污染音色：

- 安装：VoiceSpeechMaker 的 env pip 若报 `onnx` 文件被占用（WinError 5，WebUI/API 进程锁着 site-packages）→ 用 `pip install --no-deps demucs einops julius dora-search`（demucs 纯 torch，不碰 onnx）；`from demucs.pretrained import get_model` + **必须** `from demucs.apply import apply_model`。
- ⚠️ **htdemucs 必须立体声输入**：单声道报 `expected input to have 2 channels, but got 1 channels` → 加载后 `w.repeat(2,1)` 复制成双声道；分离后 vocals 用 `mean(0,keepdim=True)` 压回单声道保存。
- 模型 htdemucs 首次自动下载（HF 缓存；Windows symlink 警告可忽略）。
- 分离出的 vocals 轨重新走 VAD 切分 + whisper 转录 + esd.list。

### 2.3 VAD 切分与 whisper 转录

- **faster-whisper word_timestamps 不可靠（CPU int8）**：word 级时间戳错乱会切出 1.4s 短块且文本错位（丢失大量音频）→ 改用 **silero-vad**（整合包 `third_model/silero-vad/silero_vad.onnx` + `utils_vad.OnnxWrapper`，onnxruntime 离线加载）做精细 VAD 切分，再逐段 whisper 转录对齐。要点：min_silence 调大（0.6s+）可减少碎片化半句；VAD 能自动过滤 BGM 段；补充段文件名必须英文（如 supp_000.wav），中文文件名会踩路径坑。
- **faster-whisper 报「The current model is English-only but the language parameter is set to 'ja'」**：日语模型也会报此警告，转录结果实际是日语，**可忽略**。但 whisper 对 FGO 专有名词有转录误差（实战：「ダ・ヴィンチ」→「ドヴィンチ」、「お手製菓子」→「手製菓子葉」）→ **supp 段文本必须标注「whisper 转录，未经人工校对」**，重要语音让用户人工校对。

### 2.4 筛选标准（不合格/错误补充语音）

1. 转录**中英混杂乱码**（whisper 对噪声段的产物，如「我不破綻的自己的自己嫌悪…」）→ 删；
2. **听错的词**（语义可疑，如「膣内」）→ 删或试听确认；
3. **重复句**（同一台词出现多次，如「エヘヘ……」）→ 去重留最长/最清晰的一条；
4. **超长混乱段**（8s+ 但内容逻辑断裂）→ 删；
5. 太短（<0.5s）自动过滤。

**筛选清单先给用户确认再合并，不要直接全量入库**（参照但丁实战：87 条补充段最终 0 条进训练集——说明筛选环节决定补充语音能否用）。

### 2.5 合并纪律

- **esd.list 追加防重 + 双数据集同步**：追加前读现有文件名集合跳过已存在（否则产生重复训练数据）；补充段要同时写入**工作区数据集副本**和**整合包 `Data/<模型名>/` 训练集**（esd.list 全量覆盖 + raw 增量复制），只同步一边会让训练数据不齐。
- **系统 Python 缺重型库**（torch / demucs / librosa / onnxruntime / faster_whisper）：人声分离、VAD、whisper 转录**必须用整合包自带 `env\python.exe`**（依赖齐全），系统 Python 只适合轻量解析（urllib/re/ffmpeg 调用）。用错解释器最典型报错 `ModuleNotFoundError: No module named 'torch'`——先确认跑的哪个 python（`python -c "import torch"`），别在 torch 上浪费时间排查。

完整流水线脚本见 10.2。

---

## 3. 立绘补全 / 全身化

### 3.1 问题背景与两种解法

LingChat 角色上下移动时，如果立绘画布内容比显示区域小（如 FGO 半身差分 1024×768 横版，缩放比为1.0时，角色会刚好填满整个屏幕），角色移动会露出画布外的空白 → 穿帮。目前软件调整缩放比需要设置修改-退出软件-重进才能在肉眼上观察到角色缩放变化。桌宠可以单独调整缩放比例。两种解法：

- **方案一（最简单）**：调大缩放比例（settings.yml 的 `scale` / `scale_p`），让角色显示更大、盖住空白。缺点：缩放对所有服装**统一生效**，放太大会顶出边界，移动幅度大仍可能露白。
- **方案二（复杂但是美观）**：**立绘补图**——把半身立绘补成全身立绘，从根上消除空白，就是补完后记得自行调整缩放比。

### 3.2 补图完整流程

1. **合并所有服装的全部差分到一个 PSD**（或同类绘画文件格式）：手动在绘画软件里搭，或求助 Agent 用 psd-tools 检查/整理图层树。⚠️ CSP 的 .clip 是专有格式脚本读不了，**必须导出 .psd**（见 6.10 #38）。提前建议用户使用 psd 格式。
2. **分好类别**：每组服装 = 一张「补图」图层（下半身/身体补充，全组只有一张）+ 一个「差分」文件夹（多张表情差分，同一画布坐标对齐）。补图放在差分上方或下方都可以（素材已摆好，合成时按图层栈顺序即可）。
3. **扩大画布/图片**：给补图留出位置（画布从半身 1024×768 扩到全身尺寸如 1099×1429，所有服装共用同一画布）。
4. **找全身素材**：在 fandom（速通版 2.2）等资料站找该角色全身立绘，借助 AI 或手动绘制出透明背景的补图（下半身），补进 PSD。
5. **让 AI 合并并直接输出命名好的文件**：用 psd-tools 读 PSD，对每组「补图 + 每个差分」按图层栈顺序像素合成，按 avatar 目录现有文件名清单命名输出（脚本见 10.3）。

### 3.3 关键注意（血泪，决定补图成败）

- **输出必须保留全部透明底（固定整画布尺寸），不要「经典裁切」**：裁切到内容边界会让各服装尺寸不一、立绘在画面中的位置漂移。
- **所有版本的立绘位置必须一致**：因为软件的缩放比和位置微调（scale/offset）是对**所有服装统一生效**的。把所有服装放进同一个 PSD 画布、用同一坐标，输出固定整画布，天然保证各服装位置一致。
- 合成顺序按图层栈：补图在差分层上方 → 先画差分再贴补图；补图在下方 → 先贴补图再画差分。补图必须是透明背景（只含下半身像素），贴上去不遮差分。
- 命名：以现有 avatar 目录实际文件清单为准（速通版 4.2 铁律）；缺的差分（如某服装没有「平静」「调皮」）让用户指定替代或复用近似差分。

相关坑：#38（.clip 导出 .psd）、#39（保留透明底+位置一致）、#40（Mooncell 愚人节卡面带背景不可用）。

---

## 4. 语速与合成参数调优

### 4.1 语速是推理参数，不用重训

语速有问题 → **不需要重训！** 语速是**推理参数**（`length`/`length_scale`），不是模型参数，改一个数字即可。

**外部 API TTS（tts_type: sbv2，走 VoiceSpeechMaker `/voice`）的语速**：
- ⚠️ LingChat 的 sbv2 适配器把 `length` **硬编码为 1.0**（源码 `src-tauri/src/ai_service/tts/adapters/sbv2.rs`），界面语速滑条**不作用于**外部 SBV2 合成，配置文件里也没有对应字段。
- 改法：编辑 `VoiceSpeechMaker/scripts/server_fastapi.py`，在 `/voice` 的 `model.infer(...)` 调用前插入（`length` 越小越快，1.0=基准）：
  ```python
  # 语速修正：按模型名关键词强制更快（如但丁 0.85≈快15%）
  if model_name and 'Dante' in model_name and length >= 0.95:
      length = 0.85
  ```
- 微调：太快把 `0.85` 改大（0.9），想更快改小（0.8）；**改完重启 `.启用API服务.bat`** 生效。
- 想对所有模型生效就去掉模型名条件；想按角色区分就各自按 `model_name` 关键词判断。

**内置本地 TTS（tts_type: localsbv2api）的语速**：`settings.yml` 的 `voice_models.sbv2_local_length_scale`（null=默认 1.0），填 `0.85` 变快（改前先 read，见 6.7 #24）。

### 4.2 外部 API TTS 其他可调参数

（`/voice` 接口支持；LingChat 把部分硬编码了，直接改 server_fastapi.py 里的 query 或 infer 参数）

| 参数 | 含义 | 基准 | 调大效果 | 调小效果 |
|---|---|---|---|---|
| `length` | 语速（时长缩放） | 1.0 | 更慢 | **更快** |
| `sdp_ratio` | SDP/DP 混合比 | 0.2 | 音调起伏更大 | 更平稳 |
| `noise` | 采样噪声 | 0.6 | 随机性/自然度↑ | 更机械 |
| `noisew` | SDP 噪声 | 0.8 | 发音间隔变化↑ | 更均匀 |
| `split_interval` | 分段静音（秒） | 0.5 | 句间停顿更长 | 更紧凑 |
| `style_weight` | 风格强度 | 1.0 | 风格化更强 | 更中性 |

---

## 5. 本地 TTS 备选方案

### 5.0 内置本地 TTS vs 外部 API TTS（速通版默认外部 API，这里给备选）

| 维度 | 内置本地 TTS（角色 TTS 类型「本地 SBV2 API」） | 外部 API TTS（tts_type: sbv2 / sbv2api / gsv…） |
|---|---|---|
| 运行位置 | 模型与应用**同进程**，ONNX Runtime 直接推理 | 外部独立服务进程（VoiceSpeechMaker 5000、sbv2api 3000、simple-vits-api 23456…），HTTP 调用 |
| 外部依赖 | **零依赖**：装好模型即用，无需启动任何外部程序、无需 API | 必须先装外部引擎（SBV2 整合包，解压后约 12GB）+ 配 URL，服务窗口常开 |
| 模型格式 | 专用 `.sbv2`（单文件，内嵌风格向量）或 `.onnx` + `style_vectors.json` | 外部引擎自有格式（safetensors 训练产物直接加载，**无需转换**） |
| 语言 | 目前**只合成日语** | SBV2 日语为主；v0.5.0 起 GPT-SoVITS 中/日/英/韩、IndexTTS 情绪标签 |
| 推理硬件 | Windows 可选 CPU/GPU(DirectML)/指定显卡；macOS/Android **固定 CPU** | 由外部服务决定（电脑 GPU 服务端通常更快） |
| 手机支持 | ✅ 多平台兼容，官方为 Android ARM 做了性能分级 | ❌ 手机跑不动外部引擎 |
| 训练衔接 | 训练产物**必须先转换**（5.2 的官方工具） | 训练产物放进外部引擎 model_assets 直接用 |
| 容错 | 可配「云端备用模型」（全局开关关掉时自动走云端） | 服务挂了就没声音 |

**结论**：手机端**只能**用内置本地 TTS；电脑端两者都行，各有取舍，让用户自己选（取舍清单见下，不预设推荐）。

#### 日常使用体验差异（用户最直接的感受）

- **启动流程**：API 方案每次要**先手动开** VoiceSpeechMaker 的 `.启用API服务.bat`（黑窗口常驻，关了就没声音），再开 LingChat，两个窗口并存；本地方案**只开 LingChat 一个软件**，模型常驻应用内，开箱即出声
- **稳定性**：API 方案的语音服务窗口被误关、被杀进程、开机没启动 → 角色静音且无明显提示；本地方案无此问题
- **首次配置成本**：API 方案装引擎 + 配 URL 一步不少；本地方案 DeBERTa（278MB）+ 导入 .sbv2 一次到位，之后零维护
- **性能**：API 方案可吃 GPU（服务端）；本地方案 Windows 可选 DirectML GPU、Android/macOS 固定 CPU（手机较慢，有明显延迟，但可用）

#### 怎么帮用户选（把取舍摆出来，不替用户决定）

- **本地 TTS 的已知缺点（实测）**：① **感情标签识别弱**——情绪差分对应的语气/感情表现不如外部 API；② **合成语音的听感与训练时（API 推理）相比有变化**（音色/质感微差）。优点：只开 LingChat 一个软件、模型常驻、开箱即出声、零外部依赖。
- **外部 API 的取舍**：可吃 GPU、听感更接近训练产物、支持多语言与高级功能（GPT-SoVITS / IndexTTS 等）；缺点：要先开 `.启用API服务.bat` 黑窗口常驻、服务挂了就没声、首次配置引擎 + URL。
- 迁移已有角色（随时可切回）：`settings.yml` 改 `tts_type: localsbv2api` + `voice_models.sbv2_local_voice_id: <导入的 voice_id>`（如 `van-gogh`），其余字段不动（先 read 再改，见 6.7）——**试过再定，别先入为主**。

### 5.1 模型格式（LingChat 引擎 parse_sbv2file 逐字节核对过）

- `.sbv2` = **zstd 压缩的 tar 归档**，内含 `model.onnx` + `style_vectors.json`（可多一个 version.txt，引擎会忽略）→ 单文件、内嵌风格向量，**推荐**
- `.onnx` = 裸 ONNX + 同目录 `style_vectors.json`（JSON：`{"shape":[N,256],"data":[[...]]}`），缺 style_vectors 无法启用

### 5.2 官方转换工具（整合包自带，血泪验证）

`scripts/conversion/onnx_vioce_model_convert.py`
```powershell
# 前置：PYTHONPATH=整合包根目录；模型目录里要有 config.json + style_vectors.npy
$env:PYTHONPATH = '<整合包根目录>'   # 指向整合包安装位置
# 有 N 卡用 --device cuda（约 8 分钟/模型）；CPU 转 239MB 模型很慢（30min+）
& <整合包>\env\python.exe <整合包>\scripts\conversion\onnx_vioce_model_convert.py `
  --model "<整合包>\model_assets\<模型>\<模型>_eXXX_sXXX.safetensors" `
  --device cuda --mode sbv2api --precision fp32 --pack-sbv2 --force
```
- 一步完成：ONNX 导出 + onnxsim 简化 + style_vectors.json + **.sbv2 打包**（zstd level 19）
- 输出目录：`model_assets/<模型>/<模型名>_convert_output/<模型名>.sbv2`（约 220MB）+ `*.onnx`（约 238MB）+ `style_vectors_*.json`
- ⚠️ `model_assets` 通常在工作区外 → 沙箱会拦写输出（PermissionError: WinError 5 mkdir）→ 用 `danger-full-access` 重试；`PYTHONPATH` 不设会报 `ModuleNotFoundError: style_bert_vits2`
- zstandard 库整合包 env 已装；若手动打包：tarfile 写 tar → `ZstdCompressor(threads=-1, level=19).copy_stream` 压缩；解压验证必须用 `decompressobj`（streaming 帧无 content-size 头，`decompress()` 会报错）

### 5.3 导入 / 部署

- 应用内：`设置 → 高级设置 → 本地 TTS` → 打开「全局本地 TTS」→「本地导入」选 .sbv2（或 zip/7z 压缩包，内含 model.sbv2/model.onnx）；voice_id 自动从文件名生成（kebab-case ASCII 小写，如 `Van_Gogh.sbv2` → `van-gogh`）
- 手动部署（桌面端）：`.sbv2` 复制到 `<LingChat>/data/models/tts-local/voices/<voice_id>/model.sbv2`；DeBERTa 必须应用内下载（约 278MB → `assets/deberta/deberta.onnx` + `tokenizer.json`，不装引擎不启动）
- 角色绑定：`设置 → 角色 → 齿轮 → 语音设置` → TTS 类型「本地 SBV2 API」→ 本地语音 ID 选模型 → 保存即生效
- 手机端导入见第 7 章

### 5.4 验证 .sbv2（模板）

```python
import io, tarfile, json, zstandard
def verify(sbv2_path):
    with open(sbv2_path, 'rb') as f: compressed = f.read()
    dctx = zstandard.ZstdDecompressor()
    dobj = dctx.decompressobj()
    raw = dobj.decompress(compressed) + dobj.flush()   # ⚠️ 流式解压，decompress() 会报 content-size 错
    tar = tarfile.open(fileobj=io.BytesIO(raw), mode='r')
    names = tar.getnames()
    assert 'model.onnx' in names and 'style_vectors.json' in names
    sv = json.loads(tar.extractfile('style_vectors.json').read())
    assert sv['shape'][0] == len(sv['data']) and sv['shape'][1] == 256
    print('VERIFY OK', names)
```

---

## 6. 完整踩坑大全（#1~#43 全量，速通版只留速查，这里给全）

### 6.0 报错关键词速查（按症状定位条目，别通读）

| 症状 / 报错关键词 | 定位 |
|---|---|
| LingChat 启动闪退、`Failed to parse settings.yml`、文件头 `EF BB BF` | 6.1 #1（BOM） |
| 训练/运行报错且路径含中文 | 6.1 引言 + 速通版 0.5 |
| `Permission denied` / `WinError 5` 写工作区外目录 | 6.2 #4（沙箱权限） |
| 文件被占用（关软件也解不开） | 6.2 #5（多进程托盘退出） |
| curl `SEC_E_NO_CREDENTIALS` / Invoke-WebRequest / Node fetch 全挂 | 6.3 #6（Python urllib + 代理） |
| HuggingFace 连不上 | 6.3 #7（hf-mirror.com） |
| WebUI 模型列表没有新模型 | 6.3 #8（点「刷新」） |
| 手机连电脑/局域网同步超时 | 6.3 #9（先关梯子） |
| `Expected unicode escape` / `Bad character escape sequence` / 反斜杠被吞 | 6.4 #10（数组 join 构造脚本） |
| PowerShell 裸写报 `Expected ident` | 6.4 #11 |
| Add-Type 字符串引号错乱 | 6.4 #12 |
| Start-Process 路径含空格被拆碎 | 6.4 #13 |
| 语音解析不到（编号重复） | 6.5 #14（表格切分主方案，速通版 1.2） |
| esd.list 查不到（.mp3 vs .wav） | 6.5 #15（replace 后缀） |
| 差分→情绪识别不准 | 6.5 #16 + 6.8 #26（用户定映射） |
| 服装子目录缺图报错 | 6.5 #17（21 张放齐） |
| 看图片尺寸/编码（无 PIL） | 6.5 #18（ffprobe） |
| `ModuleNotFoundError: no module named 'torch'` / pyyaml / PIL | 6.5 #19 + 6.9 #31（用整合包 env python） |
| MediaWiki API 报 `no url` | 6.5 #20（title 规范化：`split(':',1)[1].replace(' ','_')`） |
| 语音表日文/中文为空（灵衣等） | 6.5 #21（whisper 兜底转录） |
| 文件被占用反复失败 | 6.6 #22（让用户关进程） |
| 半身立绘填满屏幕/露白 | 6.6 #23 + 本章 3（scale=1.0） |
| 覆盖了用户设置（数据库无备份） | 6.7 #24（改前先 read） |
| 台词/原文张冠李戴 | 6.7 #25（必须爬原文，禁止凭记忆） |
| 情绪图全错位 | 6.8 #26（Agent 只搭架子） |
| 音色泛化差、像不像 | 6.9 #27（训练数据量 30min+/200 条+） |
| htdemucs 报 `expected input to have 2 channels` | 6.9 #28（repeat(2,1)） |
| whisper 切出碎片短块、文本错位 | 6.9 #29（改 silero-VAD） |
| 系统 python 缺 torch/demucs/librosa | 6.9 #30/31（用整合包 env） |
| faster-whisper 报「English-only … 'ja'」 | 6.9 #32（可忽略） |
| 转录乱码/可疑词/重复句 | 6.9 #33（筛选标准） |
| 训练数据重复、两处数据不齐 | 6.9 #34（esd 防重 + 双数据集） |
| `ImportError: cannot import name 'AttributeProto' … 'onnx'` | 6.9 #35（onnx 半损坏修复） |
| 训练启动即崩（torch_cuda.dll 堆栈、无 traceback） | 6.9 #36（numpy 2.x → 1.26.4） |
| 固定 step 崩溃 `ScatterGatherKernel.cu:145` / 异常码 `0xc0000409` | 6.9 #37（config.json data 段污染） |
| `.clip` 脚本读不了图层 | 6.10 #38（先导出 .psd） |
| 立绘错位/露白（多服装） | 6.10 #39 + 本章 3（保留整画布透明底） |
| Mooncell 愚人节卡面带背景不可用 | 6.10 #40（愚人节立绘只能去 fandom 找透明底，两版本自己挑） |
| 语音页跨表编号覆盖、解析漏条目 | 6.5 #41（按 VoiceTable 表格切分） |
| MediaWiki 批量查 URL 个别文件漏掉 | 6.5 #42（allimages 前缀搜索兜底补下） |
| 预处理完成但训练没启动 | 6.9 #43（查 train_ms 进程确认） |

### 6.1 编码类（最坑）

- ⚠️ **中文路径坑**：语音模型安装目录、训练素材/数据集路径含中文会报错，必须全英文（拼音可）；**LingChat 角色文档（角色文件夹/情绪图/头像）命名中英文皆可**，两者别混淆。
1. **PowerShell Set-Content -Encoding UTF8 会加 BOM** → serde_yaml 解析失败 → **LingChat 启动闪退**（panic: Failed to parse settings.yml）。**所有写入 YAML/配置文件一律用 Python 写（utf-8 无 BOM）**，或 PowerShell 用 `[System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($false))`。诊断：检查文件头 3 字节是否 `EF BB BF`。
2. Python 打印中文到 GBK 控制台报 UnicodeEncodeError → 脚本开头 `sys.stdout.reconfigure(encoding='utf-8')` 或写文件用 read 读。
3. 下载的 HTML/wikitext 用 `utf-8`（errors='replace'）解码保存，读取统一 `-Encoding UTF8`。

### 6.2 沙箱权限类

4. **写工作区外目录（C:\Users\...、D:\SoftWare\...）被沙箱拒（Permission denied）** → 这不是文件锁！用 `sandbox_permissions: "danger-full-access"` + 一句话 justification 重试（用户批准后生效）。
5. **多进程应用关闭不干净**：LingChat 有主进程+子进程，只关一个仍锁文件 → 让用户从托盘彻底退出，再操作文件。

### 6.3 网络类

6. curl（schannel）HTTPS 报 `SEC_E_NO_CREDENTIALS (0x8009030e)`；Invoke-WebRequest 报「基础连接已经关闭」；Node fetch 沙箱无网——**这类报错不要反复重试，直接换 Python urllib + ProxyHandler（完整方案见速通版 0.8，已内联不依赖外部 skill）**；普通环境先试直连，别照搬沙箱结论。
7. HuggingFace 连不上 → 用 hf-mirror.com（国内可直连）；模型文件要手动放进 HF 缓存目录（snapshots 结构 + refs/main）。
8. 训练后 WebUI 模型列表不显示新模型 → 点「刷新」按钮（列表是启动时扫描的）。
9. **⚠️ 局域网连接/数据同步时必须关掉梯子（血泪）**：电脑开系统代理（Clash 等）时，手机连电脑的局域网 IP（如 `192.168.x.x`、`http://0.0.0.0:8765` 访问）可能被代理规则劫持 → 超时/连不上。处理：① 连接前关电脑梯子（或给局域网 IP 加直连规则）；② 手机端同样关 VPN/代理；③ 手机浏览器访问 `127.0.0.1`/局域网地址失败先排查代理，再排查防火墙（Windows 防火墙要放行 LingChat 后端端口）。

### 6.4 run_code / PowerShell 转义类

10. **run_code 的模板字符串（反引号）会吞反斜杠**：`\x00`、`\u` 等报 `Expected unicode escape`；String.raw 也可能被编译管道搞坏 → **写脚本一律用数组 join 构造**（每行一个数组元素），Python 路径用正斜杠、换行用 `chr(10)`、正则里 `\s`→`[ \t]`。
11. **别把 PowerShell 裸写在 run_code 里**（会报 `Bad character escape sequence`/`Expected ident`）→ 同样用数组 join。
12. PowerShell 里 C# Add-Type 字符串的引号：用 **PowerShell 单引号字符串**包 C# 代码（C# 里的双引号不用转义），JS 层用双引号字符串数组元素。
13. Start-Process 的 -ArgumentList 遇到**含空格路径**会拆碎 → 用 `& exe script` + `*> log` 重定向，或给路径加引号。

### 6.5 数据/解析类

14. fgo.wiki wikitext 多表格编号重复（每表从 1 重编号）→ **按 VoiceTable 表格切分后块内收集**（速通版 1.2 主方案 / 本章 #41）；反向引用跨行正则仅单表兜底。
15. esd.list 后缀不匹配（wiki 是 .mp3，查询 .wav）→ replace 后再查。
16. modlens/视觉模型对表情识别精度有限 → 差分→情绪映射**别指望全自动**，做好初步映射后让用户在软件里手动校准（见 6.8 #26）。
17. 服装子目录缺情绪文件会报错（平静除外）→ 每个服装目录放齐**该角色应有的全部情绪图 + 头像**（标准 20 情绪 + 头像 = 21 张；用户自定义情绪如闭眼/感动会更多，以参考角色实际清单为准，见速通版 references `avatar_情绪文件名清单.md`）。
18. ffprobe 可看图片尺寸/编码（无 PIL 时）：`ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of csv=p=0 img.png`。
19. 系统 python 可能没装 pyyaml/PIL → 用 VoiceSpeechMaker 的 `env\python.exe`（依赖齐全）。
20. **MediaWiki API 查文件 URL 的标题解析坑（fgo.wiki 灵衣语音等）**：`action=query&titles=File:xxx.mp3&prop=imageinfo&iiprop=url` 返回的 `page.title` 是**中文前缀「文件:」**（不是 File:），且**下划线被规范化为空格**（`SV437_11_开始1.mp3` → `文件:SV437 11 开始1.mp3`）。**不能用 `title[5:]` 切片**——「文件:」只占 3 个字符，`[5:]` 会切掉文件名首字符（`SV437`→`V437`）导致 key 永远匹配不上（报 `no url`）。正确取 key：`title.split(':', 1)[1].replace(' ', '_')`。症状「查到URL数=N 但全部 no url」优先怀疑 key 规范化，别怀疑批量查询（titles 用 `|` 分隔一次 50 个没问题）。
21. **灵衣/部分语音表只有 mp3 文件名、日文/中文为空**：fgo.wiki 灵衣语音表常缺日文/中文 → 兜底方案：MediaWiki API 查 `File:<名>.mp3` 的 url → 下载 → ffmpeg 转 44100Hz/单声道/PCM16 wav → 用 VoiceSpeechMaker 自带 `third_model/whisper/faster-whisper-ja-anime-v0.3` 转录日语（其 `env\python.exe` 已装 faster_whisper，CPU int8 即可跑）；转录文本标注「whisper 转录，未经人工校对」，中文翻译另行对照或暂缺。
41. **语音页必须按 VoiceTable 表格切分解析，不能全局 kv 收集（血泪）**：fgo.wiki 语音页**多个表格（战斗形象1 2 / 战斗形象3 / 召唤强化 / 个人空间等）各自从编号 1 重新编号**，全局 `|标题(\d+)=` 收集会把前面的条目全部覆盖（实战：阿尔托莉雅·卡斯特全量解析只出 34 条，按表格切分后 119 条）。正确做法：`re.split(r'\{\{#invoke:VoiceTable\|table\|表格标题=([^\n]+)', wiki)` 按表格切块，块内再按编号收集（保留表格标题做分类）；反向引用跨行正则仅作单表兜底（见速通版 1.2）。
42. **MediaWiki 批量查 URL 个别文件漏掉 → allimages 前缀搜索兜底**：titles 批量查询（一次 50 个）偶尔个别文件查不到（常见于中文文件名，key 规范化后仍匹配不上；实战：本尊 119 个 mp3 漏 2 个生日语音）→ 下载完必须核对「清单文件数 vs 实际下载数」；缺失的用 `action=query&list=allimages&aiprefix=<SV编号前缀>&ailimit=N` 前缀搜索列出真实文件名+直链，逐个补下。

### 6.6 用户协作类

22. 需要用户配合的（关软件、翻页、录坐标）要给出明确操作步骤；文件被占用先让用户关进程再操作，别反复重试。**全程用户交接用速通版 references `role_handoff_checklist_template.md` 清单逐项勾选**（已按阶段分好：哪些 Agent 代劳、哪些必须用户亲手、大文件下载归用户）。
23. 半身立绘完全可用（object-fit: contain + 底部对齐），示例是全身只是示例选择；显示大小用 scale/offset 调。**FGO 半身立绘 scale 默认 1.0**（官方默认 1.4 是全身立绘用的，半身用会填满屏幕，见速通版 4.3）。

### 6.7 配置文件改写纪律（血泪，最高优先级）

24. **改角色 settings.yml 前必须先 read 当前文件的最新内容**——用户可能已在软件里改过（user_name、clothes prompt、info、body_part 等），凭记忆重写整个文件会**覆盖丢失用户设置**（数据库无备份，无法恢复）。正确做法：① read 现有文件 → ② 严格对齐现有角色（如诺一钦灵）的**字段顺序和字段完整度**（voice_models 必须 20 字段齐全，别省）→ ③ **只替换用户点名要改的字段**（如 system_prompt/example），其余字段原样保留。没让改的绝对别动。
25. **语音/台词内容必须从原网站爬取，禁止凭记忆填写（最高优先级）**：示例语音（system_prompt_example / example_old）、人设里的台词引用，必须重新爬 fgo.wiki 语音页（`https://fgo.wiki/w/<角色>/语音?action=raw`）拿准确原文再填。凭记忆填日文原文极易张冠李戴（实战曾把三破对话2「太阳/月亮/水母」、三破喜欢的东西「我喜欢疼痛」的台词配错）。fgo.wiki 是国内站，**可直接 urllib 直连（无需代理）**；解析用表格切分主方案（见速通版 1.2）。填完后必须用 yaml.safe_load 验证，并检查块标量内每行缩进一致（缩进不一致的行会被 YAML 当结构解析报错）。**⚠️ 示例语音必须全部为角色原语音（原句摘抄），除非用户明确要求自编/改编**——Agent 不得自编台词冒充原语音（实战血泪：编造示例被用户一眼识破；正确做法：从语音页留档 txt 逐条摘抄原句，加【情绪】差分，中文+日文一一对应）。

### 6.8 差分与情绪映射纪律（血泪，最高优先级）

26. **差分→情绪映射必须由用户定，Agent 只搭架子，绝不硬编码猜测**：LingChat 的标准 20 个情绪槽文件名（兴奋/厌恶/伤心/害怕/害羞/平静/心动/惊讶/慌张/担心/无奈/生气/疑惑/紧张/自信/认真/调皮/羞耻/高兴/正常，见速通版 4.2）是**硬编码协议字段**（注意历史文档里的「哭泣/难为情」是旧叫法，以速通版 4.2 的「伤心/羞耻」及参考角色实际文件为准），但**哪张差分对应哪个情绪只有用户知道**——视觉模型/LLM 对动漫微表情识别精度极低（实战曾把整套差分的 index→情绪映射全部配错，「自信/调皮/疑惑」等命名全是瞎猜，用户一眼看穿）。正确分工：
   - ① Agent 只做**机械搭建**：建 avatar 目录、复制差分、按参考角色 avatar 目录的真实文件名（非硬编码情绪词）机械命名（`正常.png`/`高兴.png`……）、愚人节等单图服装复制全部命名、头像就位；缺图/多图如实标注，不要用同套图「补位」冒充缺失情绪（会导致情绪图错乱）。
   - ② 模型若有视觉能力，只可做**初步分组**（把差分按表情相似度分堆，附对照表标注「仅供参考，真实情绪由用户定」），并帮命名愚人节系列、搭好架子；**真实的情绪识别（哪张图配哪个情绪名）必须由用户自己决定**。
   - ③ **绝不**替用户定死 index→情绪的映射、**绝不**给文件起「自信/调皮」等猜测名、**绝不**擅自覆盖用户已调整过的 avatar 目录与 settings.yml。
   - ④ 用户明确说「别改我的文件」时，只更新 skill/文档，不碰角色文件。

### 6.9 训练类（含补充语音相关）

27. **训练数据量建议**：SBV2 训练建议 **30 分钟+ 干净人声（200+ 条）**（实战：仅 182 条 FGO 语音约 6~10 分钟，训练效果明显不足）。官方语音不够时走补充流水线——**完整方案见本章 2.1**（下载工具推荐 BiliDown，先检查本机有没有）。
28. **带 BGM 的语音必须人声分离（demucs/htdemucs）**，直接训练会污染音色 → 安装/用法/立体声坑见**本章 2.2**（要点：`pip install --no-deps demucs einops julius dora-search` 避开 onnx 文件占用；htdemucs 必须立体声输入 `w.repeat(2,1)`，vocals 压回单声道 `mean(0,keepdim=True)`）。
29. **faster-whisper word_timestamps 不可靠（CPU int8）**：word 级时间戳错乱会切出碎片短块且文本错位 → 改用 **silero-vad** 做精细切分，再逐段 whisper 转录对齐——**详见本章 2.3**（要点：`third_model/silero-vad/silero_vad.onnx` + `utils_vad.OnnxWrapper`；min_silence≥0.6s 减少碎片；补充段文件名必须英文）。
30. **补充语音完整流水线**：ffmpeg 转 wav → htdemucs 分离 → silero-VAD 切分 → whisper 转录 → 人工筛选 → 合并——**完整步骤与参数见本章 2.2~2.5，可复用脚本见本章 10.2**。
31. **系统 Python 缺重型库**（torch/demucs/librosa/onnxruntime/faster_whisper）→ 人声分离/VAD/whisper 转录**必须用整合包 `env\python.exe`**（依赖齐全），系统 Python 只适合轻量解析（urllib/re/ffmpeg）。用错解释器最典型报错 `ModuleNotFoundError: No module named 'torch'`——先确认跑的哪个 python（`python -c "import torch"`），别在 torch 上浪费时间排查（详见本章 2.2 开头）。
32. **faster-whisper 报「The current model is English-only but the language parameter is set to 'ja'」**：日语模型也会报此警告，转录结果实际是日语，**可忽略**；但 whisper 对 FGO 专有名词有转录误差（如「ダ・ヴィンチ」→「ドヴィンチ」）→ supp 段文本必须标注「whisper 转录，未经人工校对」，重要语音让用户人工校对（详见本章 2.3）。
33. **筛选标准（不合格/错误补充语音）**：中英混杂乱码 / 听错的词 / 重复句 / 超长混乱段（8s+）/ 过短（<0.5s）→ 删；**筛选清单先给用户确认再合并，不要直接全量入库**（但丁实战：87 条补充段最终 0 条进训练集）——**详见本章 2.4**。
34. **esd.list 追加防重 + 双数据集同步**：追加前读现有文件名集合跳过已存在（防重复训练数据）；补充段同时写入**工作区副本** + **整合包 `Data/<模型名>/`**（esd.list 全量覆盖 + raw 增量复制）——**详见本章 2.5**。
35. **WebUI 启动崩溃 `ImportError: cannot import name 'AttributeProto' from partially initialized module 'onnx'`**：根因——向 env pip 装新包（如 audio-separator）时更新 onnx 被运行中的 WebUI/API 进程打断（WinError 5 文件锁定），onnx 半损坏（`pip show onnx` 显示版本但 `import onnx` 报 circular import）。解决：① 停占用进程：`Get-CimInstance Win32_Process -Filter "Name='python.exe'"` 查 app.py / server_fastapi.py / pyopenjtalk worker → `Stop-Process -Id <pid> -Force`；② `pip install --force-reinstall onnx==<原版本>`（`pip show onnx` 查原版本，如 1.17.0）；③ ⚠️ **`--force-reinstall` 会连带卸载 onnxruntime、把 numpy 升到 2.x（与 numba 0.63 冲突，且 2.x 会让训练 C 层崩溃，见 36）→ 必须补 `pip install onnxruntime numpy==1.26.4`**（⚠️ 不能只写 `numpy<2.4`——那会装 2.3.5 照样崩，必须锁定 1.26.x）；④ 逐项验证 `import onnx / onnxruntime / numpy / numba / faster_whisper` 全部 OK 再启动；⑤ 重启 WebUI（`.启用全部功能.bat`）与 API 服务（`.启用API服务.bat`）。**预防**：向 env 装任何新包前先停 WebUI/API 服务；装完立即验证关键依赖，别让半更新状态留到下次启动才发现。
36. **训练启动即崩溃（只有 torch_cuda.dll/c10.dll 内存堆栈、无 Python Traceback）**：根因——numpy 2.x（如 2.3.5）与 SBV2 训练栈的 C 扩展 ABI 不兼容（常见于修 onnx 时 `--force-reinstall` 连带升级 numpy 后）。排查：① `python -c "import numpy; print(numpy.__version__)"` 确认版本，**SBV2 训练必须 numpy 1.x（1.26.4）**；② 修复：`pip install numpy==1.26.4` → 验证 `import numpy / torch / numba / onnx / onnxruntime / faster_whisper` 全 OK；③ torch CUDA 自检：`torch.cuda.is_available()` + 建个 cuda tensor 求和（排除 torch 环境本身问题）；④ **重启 WebUI/API**（旧进程内存里还是旧 numpy）；⑤ 手动前台跑一次 `train_ms_jp_extra.py` 观察是否进入 Epoch 循环来确认修复（崩溃是秒级、正常会推进 step）。**预防**：任何 `pip --force-reinstall` 后必查 numpy 版本；训练前先确认 `numpy.__version__` 是 1.x。
37. **训练启动后固定 step 崩溃（`ScatterGatherKernel.cu:145 Assertion "index out of bounds"` + 异常码 0xc0000409 / ucrtbase.dll，无 Python Traceback）**：根因——**训练 config.json 的 data 段被污染**（`training_files` / `validation_files` / `spk2id` 指向了其他模型，如复制/改模型时残留）。症状：训练在**某个固定 step（如 step 15）稳定崩溃**（特定数据点触发 index 越界）、同环境其他模型训练正常。排查：① 看 `Data/<模型>/config.json` 的 `data.training_files` 与 `data.spk2id`——必须指向**本模型自己的** `train.list` 和 speaker（实战：Van Gogh 的 config 残留 `Data\Dante Alighieri\train.list` + `spk2id: {Dante Alighieri: 0}` → 训练加载错数据/speaker → 崩溃）；② 训练日志 `total: N` 与数据量对比（数据量对不上说明加载错列表）；③ 修复：`training_files/validation_files` 改回 `Data\<本模型>\train.list`，`spk2id` 改回 `{<本模型名>: 0}`；④ 二分验证：同环境跑**另一个正常模型**（如但丁）确认环境没问题，再单独排查目标模型。**预防**：复制 config.json 换模型时检查 data 段路径；WebUI 训练后抽查 config 未串模型。
43. **预处理完成 ≠ 训练启动（必须确认，血泪）**：WebUI 预处理日志出现「All preprocess finished!」只代表 resample/text/bert/style 全完成，**训练要再点「开始训练」才启动**；点完确认训练真的在跑：`Get-CimInstance Win32_Process -Filter "Name='python.exe'"` 应能看到 `train_ms_jp_extra.py` 进程（只有 app.py / pyopenjtalk_worker 说明没在训练），或 `Data/<模型>/models/` 下出现新的 G_/D_ 权重（非底模）。**训练是纯后台数小时**（速通版 0.3.1 多线并行）：期间提醒用户配 key 测文字聊天、Agent 推进角色搭建，别干等。提醒用户：黑色终端窗口别点（点了暂停，按 Enter 恢复）。

### 6.10 立绘处理类（补图/全身化）

38. **CSP 的 .clip 文件无法被脚本解析**：`.clip` 是 Celsys 专有容器（文件头 `CSFCHUNK`，图层数据为私有二进制，可能加密），任何脚本都改不了内部图层 → 想对图层做自动化（合并/重命名/导出）**必须先在 CSP 里导出 .psd**；PSD 格式公开，可用 psd-tools（`pip install psd-tools pillow`，或手动下载 wheel 解压到本地 libs 目录再 `sys.path.insert` 引用，避开系统 Python 装包权限问题）解析图层树、`layer.composite()` 渲染单图层像素、`layer.offset` 取画布定位。提前建议用户使用 psd 格式。
39. **立绘合成必须保留透明底 + 位置一致**：输出**固定整画布**（各服装同一坐标），**不要裁切到内容边界**——否则软件对 scale/offset 统一缩放时各服装会错位/露白（详见本章 3）。验证透明底：PIL 检查四角像素 alpha 是否为 0、角色中心像素不透明。
40. **Mooncell 愚人节卡面带背景不透明，不可用（血泪）**：fgo.wiki 的 `<角色>愚人节卡面.png` 是**带背景**的卡面图，**不能直接做 LingChat 愚人节服装**——别自作聪明下载它复制重命名（实战：用户一眼看出没法用，自己重新下了透明底）。**愚人节立绘唯一来源：fandom**（https://fategrandorder.fandom.com/wiki，**需梯子**），且**有两个版本（一个清晰一个模糊，容易出错），建议让用户自己去挑**（Agent 不代爬代找，爬取也不保证成功）。用户提出做愚人节立绘时**立刻说清**：去 fandom 找透明底愚人节立绘（网址 + 梯子提醒），加进用户待办；拿到图后 PIL 验证四角 alpha=0（同 #39），webp 源文件用 PIL `convert('RGBA').save(png)` 转 png 保留透明。

---

## 7. 手机端使用（v0.4.7+ 官方手机版，内置本地 TTS）

### 7.1 版本事实（血泪：旧文档已过时）

- **v0.4.7（2026-07-16）起官方发布 Android APK 手机版**：软件内核全部重写，支持手机游玩、局域网数据同步、自动更新；v0.5.0（2026-08-14）内置本地 TTS 引擎正式完善（多平台：Windows/Linux/macOS/Android）
- 下载：GitHub Releases 页面 **setup.exe=Windows，apk=手机**，linux/macos 各自版本
- **Python 版已不再提供发行版（半归档）**——旧版 Android 部署（ZeroTermux + tmoe/proot-distro 装容器）整套过时，不要再用
- 手机版首次安装**进去没人物 → 退出来重新进**就正常
- 手机版数据目录：`/storage/emulated/0/Android/data/com.noiq.lingchat/files/`（Android 11+ 受限访问，改文件用 **MT管理器**）；本地 TTS 模型在 `.../files/models/tts-local/`
- 手机版已适配竖屏、触摸操作；Tauri 新版手机与电脑是**各自独立 App**（旧 FAQ「Tauri 不支持手机连电脑」指旧版浏览器方案，已过时）

### 7.2 手机端部署声音模型（核心）

手机**只能用内置本地 TTS**（外部 SBV2 API 引擎手机跑不动）：

1. 手机 App → `设置 → 高级设置 → 本地 TTS` → 打开「**全局本地 TTS**」开关
2. 「模型下载」区下载 **DeBERTa-v3-base**（约 278MB，自动附带分词器，必装）+ **Ling-v2**（约 249MB，官方日语模型，先跑通用）
3. 用**自己训练的模型**：按 5.2 把 safetensors 转成 `.sbv2`（官方工具 `onnx_vioce_model_convert.py --pack-sbv2`）→ 传到手机 → 放置/导入（见下「放置 .sbv2 的正确姿势」）
4. `设置 → 角色` → 齿轮 → 「语音设置」→ TTS 类型「**本地 SBV2 API**」→ 本地语音 ID 选模型 → 保存即生效
5. 注意：**Android 固定 CPU 推理**（无 GPU 选项，正常现象）；官方已做 ARM SOC 性能分级优化；内置 TTS 只合成日语（自训日配模型正好匹配，`voice_lang: ja`）

#### 把 .sbv2 弄到手机（首选：电脑转换 + 局域网同步拉取；备用：MT管理器手动放）

**✅ 首选（推荐，不用手动找位置放文件）**：
1. **电脑端**：按 5.2 把 safetensors 转成 `.sbv2` → 放到电脑的 `data/models/tts-local/voices/<voice_id>/model.sbv2`（voice_id 用 kebab-case，如 `van-gogh`；或直接在电脑端 App 里「本地导入」）
2. **手机端**：手机与电脑连**同一 WiFi** → 手机 App 用官方「**局域网数据同步**」（v0.4.7+ 功能，电脑和手机可随时同步数据）拉取电脑数据 → tts-local 模型自动同步到手机对应目录（`Android/data/com.noiq.lingchat/files/models/tts-local/voices/<id>/model.sbv2`）
3. 同步后回手机 App 的「本地 TTS」页刷新，「已安装语音」应能看到模型
- 优点：不用 MT管理器、不用手动找路径，目录由同步自动维护
- ⚠️ **同步前先关梯子**（见 6.3 #9）：电脑系统代理会劫持局域网流量导致同步失败；手机端也关 VPN/代理
- 若手机同步功能未覆盖 tts-local / 同步失败 / 没电脑可同步 → 退回下方备用方案

**备用：MT管理器手动放（血泪）**：
- ⚠️ **必须用 MT管理器（MT Manager）移动/复制文件，不能用手机自带的文件管理**：`.sbv2` 要放进 LingChat 应用私有目录 `Android/data/com.noiq.lingchat/`，**Android 11+ 默认文件管理器看不到这个目录**（受限存储），只有 MT管理器能浏览；MT管理器首次访问会弹「授权访问」确认即可
- **目标路径**：`/storage/emulated/0/Android/data/com.noiq.lingchat/files/models/tts-local/`
- 放置方式二选一：
  - **a) 推荐：先放到 tts-local 目录 → 打开 App 用「本地导入」**：在 MT管理器把 `xxx.sbv2` 复制到上面路径，回 App → `设置 → 高级设置 → 本地 TTS → 本地导入` → 从该位置选文件导入（应用自动生成 voice_id、建好 `voices/<id>/` 子目录，最稳）
  - **b) 手动部署：直接建子目录**：在 MT管理器里自建 `tts-local/voices/<voice_id>/` 并把 .sbv2 **改名为 `model.sbv2`** 放进去（voice_id 用 kebab-case ASCII 小写，如 `van-gogh`；目录结构同桌面端 `voices/<id>/model.sbv2`），放好后回 App 刷新「已安装语音」应能看到
- 无论哪种方式：**DeBERTa 仍须在 App 内下载**（278MB，`tts-local/assets/deberta/`，别手动放）；模型放好后「本地引擎」状态应为「已就绪」

### 7.3 手机与电脑的数据同步（把角色搬到手机）

- **局域网数据同步**（v0.4.7 新增）：手机和电脑同一 WiFi 下随时同步数据
- **角色导出/导入**（v0.5.0，PR #520）：角色导出为 zip/7z 压缩包再导入另一台设备
- ⚠️ **同步/连接前先关梯子**（见 6.3 #9）：电脑系统代理会劫持局域网流量导致连不上；手机端也关 VPN/代理

### 7.4 手机端排障速查

- 没人物 → 退出 App 重进
- 白屏/无响应 → 等待加载完成刷新
- 没声音 → ① TTS 类型是否「本地 SBV2 API」且本地语音 ID 已选 ② 全局本地 TTS 开关是否打开 ③ 本地引擎状态是否「已就绪」④ 语音语言是否与模型匹配（ja）
- 合成报 FP16/Gelu 错 → 用官方 FP32 模型（CPU 不支持部分 FP16 算子）
- 模型「缺 style_vectors」→ `.sbv2` 已内嵌不用管；`.onnx` 需补 `style_vectors.json`

---

## 8. 非 FGO 拓展迁移（附录·思路层，未实战验证，最后读）

> **最不重要的章节**：本 skill 只服务 FGO 角色，其余内容先读完、实战跑通后再回来看。
> 核心流程（语音准备 → SBV2 训练 → LingChat 角色搭建 → TTS → 手机端）与具体游戏无关，理论上可迁移到其他游戏角色（已设想：BA 蔚蓝档案、明日方舟）。**未实战，细节以实际为准**。

**可直接照抄**：语音处理与数据集结构（速通版第 1 章）、SBV2 训练（速通版第 2 章）、LingChat 目录结构与 settings.yml（速通版第 4 章）、TTS 接入（速通版第 5 章）、手机端（本章 7）、踩坑大全（本章 6）绝大多数通用（编码/沙箱/网络/numpy/config 污染/本地 TTS 等）。

**必须由用户提供**（Agent 不预设、不猜测）：
- 资料/语音爬取网站：需类似 fgo.wiki 的可解析结构；若无结构化接口，语音文本只能人工整理或标注「whisper 转录，未经人工校对」
- 差分获得途径：表情差分合成工具只服务 FGO，其他游戏需用户自己找（解包/社区工具等）
- 服装/立绘素材与命名约定

**必须改写的模板**（FGO 特有板块 → 目标游戏，速通版 4.4.2 映射）：

| FGO 板块 | BA 蔚蓝档案 | 明日方舟 |
|---|---|---|
| 【FGO 的从者设定】（资料1） | 学生档案（学校/社团/年龄/武器） | 干员档案（职业/种族/出身/专精） |
| 【史实生平】/【原典】（资料2） | 背景故事（剧透警告章节） | 干员背景（矿石病等设定） |
| 【主线物语的记录】 | 主线剧情经历 | 主线/故事集经历 |
| 【迦勒底的人际关系】 | 与其他学生的关系 | 与其他干员的关系 |
| 【不同灵基的差别】 | 换装/社团服的差异 | 皮肤/时装的差异 |
| 「御主」称呼与设定段 | 「老师」 | 「博士」 |
| clothes 服装 prompt（4.1） | 换装/泳装/社团服 | 皮肤/时装 |

**流程差异注意**：
- 速通版 0.4 资源地址表整体作废，换成用户提供的网站/工具
- 目标游戏若无现成差分 → 先走单图服装方案跑通（速通版 2.4 单图服装）
- 训练数据量目标不变（30min+/200 条+，6.9 #27）
- 版权同样仅限个人自用（速通版 0.6）

---

## 9. 进阶验证清单（做完全部进阶项逐项检查）

- [ ] 语音全量补全：语音页解析条目数 vs esd.list 覆盖缺失 0（本章 1）；补全后双数据集同步一致（6.9 #34）
- [ ] supp 补充段：已按筛选标准去重去错（6.9 #33）、文本标注「whisper 转录未经人工校对」、与主数据集无重名（本章 2）
- [ ] 立绘补全（本章 3）：输出保留整画布透明底（PIL 查四角 alpha=0）、各服装立绘位置一致（同一画布坐标，不裁切）、情绪文件名无历史后缀（_full 等已清理）
- [ ] 语速调优（本章 4）：外部 API 改 server_fastapi.py 后重启生效；本地 TTS 用 sbv2_local_length_scale
- [ ] 本地 TTS（本章 5）：自训练模型已转 `.sbv2`（5.2 官方工具）并导入本地 TTS（应用内导入或手动放 `data/models/tts-local/voices/<id>/model.sbv2`），角色 TTS 类型为「本地 SBV2 API」且本地语音 ID 已选 → 对话有声音
- [ ] 手机端（本章 7，v0.4.7+ 官方 APK）：DeBERTa 已下载、`.sbv2` 已导入、角色语音已绑定 → 手机对话有声音
- [ ] 视觉模型（可选，本章 11）：已按 11.1 配好视觉 key → 角色能描述/回应图片；不配不影响其他功能

---

## 10. 可复用脚本模板（进阶专用）

### 10.1 语音页缺失批量补全（解析 → 查 URL → 下载 → 转 wav → 追加 esd）

```python
# 前置：missing = {mp3文件名: 日文原文}（速通版 1.2/7.2 解析出但 esd.list 没有的条目）
# 依赖：urllib/re 即可，系统 Python 能跑；ffmpeg 需先检查本机（没有则下载 FFmpeg-Builds 解压）
import urllib.request, urllib.parse, json, os, subprocess

opener = urllib.request.build_opener()
opener.addheaders = [('User-Agent', 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0')]
FFMPEG = '<ffmpeg 可执行文件路径>'          # 先检查本机有没有 ffmpeg
MP3_DIR = '<数据集>/mp3_补下载'             # mp3 原件备查目录
RAW = '<数据集>/raw'
ESD = '<数据集>/esd.list'

def query_urls(files):                      # MediaWiki API 批量查直链，一次 50 个
    result = {}
    for i in range(0, len(files), 50):
        batch = files[i:i+50]
        titles = ['File:' + f for f in batch]
        url = 'https://fgo.wiki/api.php?action=query&format=json&prop=imageinfo&iiprop=url&titles=' + urllib.parse.quote('|'.join(titles))
        data = json.loads(opener.open(url, timeout=60).read().decode('utf-8'))
        for pid, p in data['query']['pages'].items():
            t = p.get('title', '')
            if ':' in t:
                key = t.split(':', 1)[1].replace(' ', '_')   # 关键坑，见 6.5 #20
                ii = p.get('imageinfo', [{}])[0]
                result[key] = ii.get('url')
    return result

os.makedirs(MP3_DIR, exist_ok=True)
existing = set(l.split('|')[0] for l in open(ESD, encoding='utf-8').read().splitlines() if '|' in l)
for fn, jp in missing.items():
    wav = fn.replace('.mp3', '.wav')
    if wav in existing:
        continue
    mp3 = os.path.join(MP3_DIR, fn)
    # 下载 mp3（urllib）→ ffmpeg 转 wav → append esd 行 wav|Van Gogh|JP|jp
    subprocess.run([FFMPEG, '-y', '-i', mp3, '-ar', '44100', '-ac', '1', '-c:a', 'pcm_s16le', os.path.join(RAW, wav)], capture_output=True)
    with open(ESD, 'a', encoding='utf-8') as f:
        f.write(wav + '|Van Gogh|JP|' + jp + chr(10))
```

### 10.2 补充语音完整流水线（分离 → VAD → 转录 → 筛选 → 合并）

```python
# 依赖：整合包 env python（torch/demucs/librosa/onnxruntime/faster_whisper 齐全），系统 Python 缺这些库
# 流程：ffmpeg 转 wav → htdemucs 分离 vocals → silero-VAD 切分 → whisper 逐段转录 → 筛选 → 合并
import os, sys, subprocess, glob, re
sys.path.insert(0, '<整合包>/third_model/silero-vad')   # OnnxWrapper 所在目录
import numpy as np, torch
from utils_vad import OnnxWrapper
from faster_whisper import WhisperModel

FFMPEG = '<ffmpeg 路径>'
WHISPER_MODEL = '<整合包>/third_model/whisper/faster-whisper-ja-anime-v0.3'
WORK = '<工作目录>'          # 中间产物：vocals/（分离）、candidates/（候选段）、candidates.list
SEP = WORK + '/vocals'; CAND = WORK + '/candidates'
os.makedirs(CAND, exist_ok=True)
vad = OnnxWrapper('<整合包>/third_model/silero-vad/silero_vad.onnx', force_onnx_cpu=True)
model = WhisperModel(WHISPER_MODEL, device='cpu', compute_type='int8')

def vad_segments(wav16k, sr=16000, threshold=0.4, min_speech=0.4, min_silence=0.6):
    import librosa
    audio, _ = librosa.load(wav16k, sr=sr, mono=True)
    frame = 512; probs = []; vad.reset_states()
    for i in range(0, len(audio), frame):
        chunk = audio[i:i+frame]
        if len(chunk) < frame: chunk = np.pad(chunk, (0, frame-len(chunk)))
        probs.append(float(vad(torch.from_numpy(chunk.astype('float32')).unsqueeze(0), sr)[0]))
    sp = [p >= threshold for p in probs]
    segs = []; in_s = False; start_f = 0
    for i, s in enumerate(sp):
        if s and not in_s: in_s = True; start_f = i
        elif not s and in_s: in_s = False; segs.append((start_f*frame/sr, i*frame/sr))
    if in_s: segs.append((start_f*frame/sr, len(sp)*frame/sr))
    merged = []
    for s in segs:
        if merged and s[0]-merged[-1][1] < min_silence: merged[-1] = (merged[-1][0], s[1])
        else: merged.append(s)
    return [s for s in merged if s[1]-s[0] >= min_speech]

# ① 分离（htdemucs，单声道先 repeat(2,1) 成双声道，vocals mean 压回单声道，见 6.9 #28）
# ② 每轨：ffmpeg 转 16k → vad_segments → 每段 ffmpeg 切出 → whisper 转录（beam_size=5, language='ja'）
#    过滤：dur<0.5s 或 >20s、转录空文本；命名 supp_NNN.wav；candidates.list 记 {name|来源|起始|时长|文本}
# ③ 筛选（见 6.9 #33）：乱码/听错/重复/超长混乱 → 生成 keep.list 给用户确认
# ④ 合并：复制候选 wav 到 raw + 追加 esd.list（防重）+ 双数据集同步（见 6.9 #34）
```

### 10.3 立绘补图合成（psd-tools）：合并 PSD 中每组「补图+差分」为完整立绘

```python
# 依赖：pip install psd-tools pillow（或手动下载 wheel 解压到本地 libs 目录，sys.path.insert 引用）
from psd_tools import PSDImage
from PIL import Image
import re, os

psd = PSDImage.open('<PSD路径>')
W, H = psd.width, psd.height          # 固定整画布尺寸
root = psd[0]                          # 顶层组（所有服装都在里面）
layers = list(root)

def render_onto(base, layer):          # 把图层渲染到画布（带 offset 定位）
    img = layer.composite().convert('RGBA')
    base.paste(img, layer.offset, img)

# 对每组：folder=差分文件夹, patch=补图图层
f_idx = layers.index(folder); p_idx = layers.index(patch)
patch_on_top = p_idx > f_idx           # 补图在上=先差分后补图；在下=先补图后差分
for diff in folder:                    # 每个差分合成一张
    base = Image.new('RGBA', (W, H), (0,0,0,0))
    if patch_on_top:
        render_onto(base, diff); render_onto(base, patch)
    else:
        render_onto(base, patch); render_onto(base, diff)
    base.save(os.path.join('<输出目录>', diff.name + '.png'))  # 保留整画布透明底，勿裁切
```

### 10.4 esd.list 质量检查（格式/重复/raw 对应，速通版引用）

```python
lines = [ln for ln in open('<数据集>/esd.list', encoding='utf-8').read().splitlines() if ln.strip()]
bad = []
for i, ln in enumerate(lines):
    parts = ln.split('|')
    if len(parts) != 4: bad.append((i+1, '段数=%d' % len(parts))); continue
    fn, char, lang, text = parts
    if not fn.endswith('.wav') or char != 'Van Gogh' or lang != 'JP' or not text.strip():
        bad.append((i+1, '字段异常'))
# 文件名去重：Counter(fn) 出现 >1 的
# raw 双向比对：esd 有但 raw 缺 / raw 有但 esd 缺（os.listdir 与 esd 文件名集合差集）
```

---

## 11. 视觉大模型（可选）：角色看图能力与 key 获取

### 11.0 什么时候需要（按需加载本章）

LingChat 的视觉大模型 key **可选配**（速通版 0.2/4.0）：不开也能用，只是角色少「图像理解」能力。以下情况加载本章：

- 用户想让角色**看懂自己发的图片**（如发画作、表情、照片，角色能描述/回应）；
- 用户询问「**视觉模型 key 怎么获取 / 配哪个**」；
- Agent 侧想用视觉能力辅助差分**初步分组**（仅初步分组，真实情绪映射仍归用户——速通版红线 3 / 6.5 #16）。

### 11.1 key 获取（三个方案，任选其一）

**① DeepSeek 官方视觉模型（2026-08 更新，有 DeepSeek key 即用）**
- 模型名：**`deepseek-v4-flash-vision-exp`**（DeepSeek 更新后的视觉模型）。
- 流程：已有 DeepSeek API key 的用户**无需额外注册任何平台**——LingChat「大模型管理」里视觉模型直接填模型名 `deepseek-v4-flash-vision-exp` 即可。
- 适合：已有 DeepSeek key 的用户（零额外注册成本，最省事）。

**② 阿里云百炼（官方推荐，付费按量）**
- 平台：https://bailian.console.aliyun.com/cn-beijing?tab=api#/api/?type=model&url=2712195
- 流程：登录阿里云 → 开通百炼（DashScope）→ 控制台「API-KEY 管理」创建 key → 在 LingChat 大模型管理里选视觉模型（qwen-vl 系列，如 qwen-vl-plus / qwen3-vl-plus 等，**以平台当前模型列表为准**）。
- 适合：已有阿里云账号、需要稳定高精度识别的用户。

**③ 智谱 GLM-4V-Flash（免费，作者自用）**
- 平台：https://bigmodel.cn/apikey/platform
- 智谱开放平台的免费视觉模型（作者实测自用，免费额度以平台当前政策为准），模型名 **GLM-4V-Flash**。
- 适合：想零成本体验角色看图能力的用户。

### 11.2 配置

- 入口与语言模型相同：LingChat「游戏配置 → 高级设置 → 大模型管理」（速通版 4.0），**视觉模型单独填一条**（模型名 / API 地址按所选平台提供的信息填；具体界面字段以 LingChat 当前版本为准）。
- key 安全纪律与语言模型一致：**由用户在软件内手输，Agent 不触碰、不写入任何脚本/文件、不向用户回显**（速通版 4.0）。

### 11.3 使用纪律（红线 3 延伸，别踩）

- 视觉模型**不能用于「差分→情绪映射」的定论**：对动漫微表情识别精度有限（6.5 #16 / 速通版 6.1 #14），只能做**初步分组**供用户参考（附对照表标注「仅供参考，真实情绪由用户定」），最终映射必须由用户定。
- Agent 侧默认**不用识图看差分**，按数字序号机械命名即可（速通版 2.0 步骤 5）；仅当用户明确要求识图比对时才用视觉工具。
