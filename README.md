# LingChat 角色制作速通（AI 助手 Skill 包）

> 在 AI 助手的辅助下，基于LingChat软件，快速制作「拥有自训练日语语音模型的全语音可互动 AI 角色」（专为FGO角色制作）——
> 静态立绘 + 20 情绪差分 + 独立训练的语音模型。

## 这是什么

两个配套的 AI 助手 Skill（技能包），装进支持 Skill 的 AI 助手（DeepseekHarness / codex / claude / Trae 等，需能读写文件 + 联网）即可全程指导制作角色：

| 包 | 内容 | 适用 |
|---|---|---|
| `lingchat-role-maker`（速通版） | 端到端制作「能说话的角色」：前置检查 → 资源清单 → Mooncell 语音/资料抓取 → 差分获取 → SBV2 训练 → LingChat 搭建 → TTS 接入 → 验证清单 | 第一次做角色 |
| `lingchat-role-maker-pro`（进阶版） | 语音全量补全 / BGM 人声分离补充语料 / 立绘补全 / 语速调优 / 本地 TTS / 手机端 / 完整踩坑大全（#1~#43）/ 视觉模型可选配 / 非 FGO 迁移 | 速通版跑通后按需加载 |

## 使用说明（安装）

给你的 AI Agent 发送如下内容：

> 帮我下载 https://github.com/CF-612/lingchat-role-maker 上的 skill 到本地，并加载这个 skill。只下载 skill，不克隆整个仓库。

Agent 会只拉取本仓库 `skills/` 下的两个 skill 文件夹（`lingchat-role-maker` 与 `lingchat-role-maker-pro`）放到本地 skill 目录并加载。加载完成后，对 AI 说出你想做的角色名（如「梵高」），它会按流程带你走完整个制作过程。

**前置条件**：梯子（部分下载站需科学上网）、能写文件的 AI 助手、至少一个 LLM API key

**不适合**：不想自己训练语音模型（只想用现成 TTS）的用户

## 流程速览

```
确认需求 → Agent 爬资料/语音 与 用户找差分 并行 → 差分先到手
→ 资料与数据集就绪后交用户去语音训练（数小时）
→ 训练期间搭完角色（settings.yml + avatar + 人设）
→ 训练完接 TTS（默认外部 API）→ 验证清单收尾
```

## 参考资源

- LingChat 开源项目：https://github.com/SlimeBoyOwO/LingChat
- 语音训练整合包（Style-Bert-VITS2-FULL）：https://www.modelscope.cn/models/lingchat-research-studio/Style-Bert-VITS2-FULL
- 语音/资料源（Mooncell）：https://fgo.wiki/
- 表情差分合成工具（来自b站up主阿良良睦历）：https://pan.baidu.com/s/1m6tv52y5FJfz-bgk7OHDfw?pwd=fgo1
https://pan.quark.cn/s/a6162b75285b?pwd=E11Z

## 版权与许可

- Skill 文档与模板：**CC BY-NC-SA 4.0**（署名-非商业-相同方式共享），详见 [LICENSE](LICENSE)；
- 角色素材（语音/立绘/差分/训练模型）版权归 TYPE-MOON 等权利方，**仅限个人自用，禁止公开分发**——本仓库不包含任何 FGO 素材。
