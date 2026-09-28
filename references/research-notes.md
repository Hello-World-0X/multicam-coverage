# 研究笔记 — 本 skill 的证据基础

改动方法论前先看这里：每条核心策略都有出处。
标 ⭐ 的是「实践与研究互相印证」的关键结论。

## 1. 经典剪辑体系（影视规则的原始定义）

来源：Wikipedia [180-degree rule](https://en.wikipedia.org/wiki/180-degree_rule) ·
[Continuity editing](https://en.wikipedia.org/wiki/Continuity_editing) ·
[Eyeline match](https://en.wikipedia.org/wiki/Eyeline_match) ·
[Over-the-shoulder shot](https://en.wikipedia.org/wiki/Over-the-shoulder_shot) ·
[Shot reverse shot](https://en.wikipedia.org/wiki/Shot_reverse_shot)，
源头是 Bordwell & Thompson《Film Art》与 Mascelli《The Five C's of Cinematography》。

### ⭐ 180° 规则的准确定义

- 轴线 = 两个角色之间的假想连线（也可是运动路径：走出画左，下镜必入画右）
- 越轴（jumping the line / crossing the line）会**翻转人物左右次序，使观众失去方向感**
- 实证研究（180-degree rule 条目引）：越轴**损害空间表征与物体位置记忆**，
  但不影响叙事理解、观众愉悦度基本不变——
  → 这解释了为什么**AI 生成图越轴看起来「怪」但说不出哪里怪**：
  它破坏的是空间认知，不是内容正确性。自检清单 A 组因此是第一优先级

### 合法越轴的三种手法（用户要越轴镜头时的正确写法）

1. **弧形移机**：一个连续镜头内让摄影机绕过轴线，观众能看到过程
2. **轴线上的缓冲镜**（buffer shot）：插入一个沿轴线方向的镜头再切到另一侧
3. **重建定位**：用总机位重新交代空间后，允许在新一侧展开

> 对本 skill 的含义：用户要越轴机位时，输出里要**同时给缓冲镜/总机位**，
> 不能只给孤立的越轴图。这是「提示后果」之外的具体解法。

### 30° 规则（新增约束）

来源：Continuity editing 条目。同一主体的相邻两个镜头，机位夹角必须 ≥30°
（或景别显著不同），否则产生跳剪感。

> 对本 skill 的含义：**机位表里的机位两两夹角要 ≥30° 或景别跳两档**。
> 若用户要求的两个机位夹角过近（如 20° 两个中景），提示改景别（一个改特写），
> 否则成组看会像同镜位抽了两张卡。写进 camera-geometry §2 的机位排布规则。

### ⭐ 视线匹配的正式规范（Eyeline match 条目）

两个演员的匹配特写应满足：
**相同焦距 + 相同机位高度 + 相同距离，画外演员位于镜头等距的相反两侧**
→ A 看画外右，B 看画外左。

> 这是「正反打两镜参数对称」的原始出处。反推时两张参考图的机位高度/
> 焦距不一致就是素材本身不规范，报告用户。

### OTS 的三层深度结构（Over-the-shoulder 条目）

标准过肩镜 = **三个景深层**：前景（肩+常有后脑）、中景（对方面孔，合焦主体）、
背景（虚化）。对面部对焦，前景背景浅景深。

> 机位表「画面式」的第一句先写前景层，就是照这个结构。
> 另：OTS 配对镜要「match」——两人保持相同屏幕侧 + 视线匹配。

## 2. 图像模型为什么不服从相机指令（研究证据）

### ⭐⭐ 写「24mm」没用——这有论文实锤

> "if one asks the generator to synthesize a specific camera setting such as
> creating different fields of view using a 24mm lens versus a 70mm lens, the
> generator will not be able to interpret and generate scene-consistent images."
> — [Generative Photography](https://arxiv.org/abs/2412.02168) (2024)

直接文生图模型**无法把焦距参数翻译成一致的透视**。他们的方案是专门训练
「差分相机内参学习」才能做到 24mm/70mm 场景一致切换。

> 对本 skill 的含义：**焦距写两遍（参数 + 视觉效果）不是啰嗦，是必须**。
> 以及：纯提示词路线的天花板就这么高，要求高时引导用户走「草图定构图」或
> 带相机控制的链路（prompt-strategy §1 链路 e）。

### ⭐ 提示词工程路线的天花板

PreciseCam（[arXiv:2501.12910](https://arxiv.org/abs/2501.12910)）：用
**4 个相机参数（1 内参 + 3 外参）**显式条件化，就**超过传统提示词工程**。
说明自然语言描述机位的精度有限，参数化条件才是精确控制的方向。

> 对本 skill 的含义：机位表的「参数式」列不是装饰——如果用户的模型/插件
> 支持相机参数（ComfyUI 相机节点、PreciseCam 类工具），把参数式列直接喂给
> 它们，提示词只管画面内容（prompt-strategy §7 已列，此处是依据）。

### 文本-视觉隐空间里没有三维相机结构

[Camera Control via Viewpoint Tokens](https://arxiv.org/abs/2604.19954) (2026)：
「Current text-to-image models struggle to provide precise camera control using
natural language alone」，需专门学习「视点 token」才能把三维相机结构注入
文本-视觉隐空间。

> 对本 skill 的含义：「描述画面结果而非相机运动」（prompt-strategy §2）是有
> 理论依据的——模型的隐空间里根本没有「相机向左转 45°」这个概念，
> 但有「右肩入画」这类画面统计。

## 3. 多视角一致生成的技术现状（为什么用「校验迭代」兜底）

| 工作 | 思路 | 对本 skill 的启示 |
|---|---|---|
| [MV-Adapter](https://arxiv.org/abs/2412.03632) (2024) | 即插即用 adapter 给 T2I 加多视角一致能力，统一条件编码器吃相机参数+几何 | 真一致性要**架构级支持**（adapter/相机条件），不是提示词能凑的 |
| [MVDiffusion++](https://arxiv.org/abs/2402.12712) (2024) | 稠密多视角扩散做单/稀疏视图重建 | 多图联合生成 > 逐张独立生成（呼应「锚点先行」） |
| [Free3D](https://arxiv.org/abs/2312.04551) (2023) | 无显式 3D 表示的多视角一致 | 一致性可以来自跨视角注意力，不必真重建 3D |
| [ShoulderShot](https://arxiv.org/abs/2508.07597) (2025) | 专门做**过肩对话视频**：双镜生成 + 循环视频，点名挑战=角色一致性+空间连续性 | 正反打的空间连续性**是公认的开放问题**，2025 年才有专门框架；本 skill 的保守策略（锚点+校验）是务实解 |

> ⭐ 横向结论：**没有银弹**。学术界做这事要么改模型架构，要么联合生成多视角。
> 用通用图像模型 + 提示词时，「机位图 + 锚点先行 + 逐张校验」就是能做到的上限。
> 改 skill 时不要承诺「一次生成 100% 一致」——把成本（抽卡预算）如实告诉用户。

## 4. 附：实测 FOV 数值（35mm 全画幅，36×24mm，对角 43.3mm）

来源：[Angle of view](https://en.wikipedia.org/wiki/Angle_of_view)。
公式 α = 2·arctan(d / 2f)，d 取画幅尺寸。

| 焦距 | 对角 FOV | 水平 FOV | 透视感受 |
|---|---|---|---|
| 16mm | 107.1° | 95.1° | 强畸变、近大远小夸张 |
| 24mm | 84.1° | 73.7° | 明显广角 |
| 35mm | 63.4° | 54.4° | 轻微广角、环境交代多 |
| 50mm | 46.8° | 39.6° | 接近人眼 |
| 85mm | 28.6° | 23.9° | 背景压缩 |
| 200mm | 12.3° | 10.3° | 强压缩 |

同一景别的「肩:头」大小比参考（过肩镜快捷判焦距）：
广角下前景肩可近 1.5–2 倍于对方面孔，50mm 约 1.1–1.3 倍，长焦趋近 1。

**透视规律**（Angle of view 条目）：
- 焦距越短，近大远小与透视畸变越强；越长，距离被「压缩」
- **保持主体占画面不变而换焦距 = 同时改变机位距离**，前景/主体的相对大小随之改变
  → 这就是为什么机位表必须同时写「焦距 + 距离/占比」，单写一个锁不住透视
- 机位不垂直于主体时广角畸变更明显（仰拍建筑后倾）

## 5. 检索记录（复现用）

- Wikipedia：180-degree rule / Continuity editing / Eyeline match /
  Over-the-shoulder shot / Shot reverse shot / Angle of view
- arXiv API：`"camera control" "text-to-image"`、`"multi-view consistent" "image generation"`、
  `"shot reverse shot"` 等 query，取 2023–2026 论文
- 检索日期：2026-09-28
