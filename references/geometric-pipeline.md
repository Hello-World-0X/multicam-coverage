# 几何管线（路线 A）— 工具链实现方案

路线 C（提示词）之外的升级路线：深度 → 位姿 → 反投影 → 渲染 → 定向补洞。
**透视由数学保证，生成模型只负责补「确定不了」的部分。**
与 prompt-strategy §1 链路 e 对应——那份机位表在这里同样有效，是三路线共享资产。

适用：单个关键镜头透视要求极高、同场景要出几十个机位、或反复遇到
模型不服从机位描述的场合。想省事就别上，路线 C 够用时别自找麻烦。

## 1. 总体架构（阶段 + 数据契约）

```
[0] 机位表（skill Step 2 产物）
      参数式列 ──转录──► camera_spec.yaml        ← 输入契约
      ↓
[1] 深度估计      ref.png            → depth.png + scale.json
      ↓
[2] 位姿          camera_spec.yaml   → pose.json   （见技巧 2：不用解 SfM）
      ↓
[3] 反投影/融合   depth + pose       → cloud.ply   （或内存里直接进 [4]）
      ↓
[4] 新机位渲染    cloud + target_pose → render_rgb.png
                                        + render_depth.png
                                        + hole_mask.png     ← 哪里是空的
      ↓
[5] 补洞          render_rgb + hole_mask + 画面式提示词 → filled.png
      ↓
[6] 结构锁定细化   filled + render_depth(ControlNet) → final.png
      ↓
[7] 质检          18 项自检 + provenance 图（哪像素可信）
```

**工程原则**：阶段之间只传文件/标准结构（png/ply/json），每阶段一个 CLI 入口
（`depth.py / pose.py / render.py / fill.py`），实现可整体替换——换深度模型
不动别的阶段。

## 2. 每阶段选型

| 阶段 | 首选 | 备选 | 关键说明 |
|---|---|---|---|
| ① 深度 | Depth Anything V2、Marigold | Metric3D v2 / DepthPro（要绝对尺度） | 前两者输出**相对深度**（尺度未知）→ 技巧 1 |
| ② 位姿 | **机位表直给**（技巧 2） | VGGT / COLMAP / DUSt3R 用于校验或双图融合 | 正反打夹角小，SfM 解算深度不精，别迷信 |
| ③ 3D 化 | numpy 反投影点云（最简） | Open3D TSDF 网格 / 少样本 3DGS（SparseGS 类） | 两张图别一上来就 3DGS——约束不足，训练出的是幻觉 |
| ④ 渲染 | Open3D 离屏点 splat（z-buffer） | gsplat、pyrender | 点云渲染必有空洞——特性不是 bug，洞就是未见区 |
| ⑤ 补洞 | FLUX Fill / SDXL inpaint | LaMa（小洞快） | 只在 hole_mask 内生成，其余像素保留 |
| ⑥ 结构锁定 | ControlNet(depth+normal) | 低 denoise（0.3–0.45）img2img | 防补洞时几何漂移的关键一步 |

⚠️ 此领域迭代极快（VGGT、3DGS 都是 2024–2025 的产物），动手前搜当期 SOTA：
**架构照本文不变，选型可换**。弱纹理深度不可信时接 manual-constraints.md。

## 3. 两个决定成败的技巧

### 技巧 1：尺度锚——拿人当尺子

单目深度只有**相对**深度（差一个尺度因子），机位表写的是「高度 1.35m」这种
**绝对**量。桥接：

```yaml
# scale.json
scale_anchor:
  method: person_height
  A_height_m: 1.65        # 假设值，不确定就问用户（uncertain 流程）
  pixel_height_in_ref: 820
  → 把深度图归一化到「A 站高 = 1.65m」
```

有了尺度，机位表的米数才生效。用 Metric3D/DepthPro 公制深度可跳过锚定，
但精度别期待过高。

### 技巧 2：位姿不用解 SfM——机位表就是位姿源

Step 1 反推已给参考图机位，Step 2 机位表给目标机位的同一组参数——
**以参考图机位为原点，目标位姿从参数式列直接转录**：

```yaml
# camera_spec.yaml —— 由机位表参数式列自动生成
scene_scale: {anchor: person_A, height_m: 1.65}
source_view:
  image: A.jpg
  camera: {height_m: 1.35, focal_mm: 35, azimuth_deg: 0, tilt_deg: 0}
target_view:
  id: M3
  camera: {height_m: 1.50, focal_mm: 35, azimuth_deg: 40, tilt_deg: -5}
```

SfM（VGGT/COLMAP）降级为**校验工具**：解出的相对位姿与机位表差太多 →
反推或机位表有问题，报用户。省掉工具链最脆的一环。

## 4. 三个实战陷阱（不处理就是玩具）

1. **掠射角伪影**：从侧面看，反投影点拉成「面条」。对策：目标视线与表面法向
   夹角过大（如 >70°）的区域**并入 hole_mask**——宁可标洞让模型编，
   别让错误几何混进渲染
2. **双参考融合**：正反打两图各反投影一次、按 pose 合并，可见区立刻变大
   （两视角互补）。但深度不一致处会「重影」——按深度方差合并，
   方差大的并入 hole_mask
3. **补洞 = 幻觉区，显式留痕**：产出 **provenance 图**（绿=输入派生、
   红=模型编的）随图交付。这是路线 A 独有优势——**知道每张图哪里可信**；
   18 项自检 D 组（恒定物）重点查红色区

## 5. 三种实现形态

| | ComfyUI 工作流 | Python CLI 管线 | **Agent + API（推荐）** |
|---|---|---|---|
| 组成 | 深度/ControlNet 节点 + 外挂反投影脚本 | 一个 Python 包（open3d+diffusers） | agent 出机位表，小脚本反投影渲染，API 干深度/补洞 |
| 上手 | 最快（可视化） | 中 | 快（贴合 agent 批量出图链路） |
| 批量/可测 | 弱 | 强（可 CI） | 中 |
| GPU | 本地要 | 本地要 | **免**（fal/replicate 等出深度和 inpaint） |
| 适合 | 起步验证算法 | 生产化批处理 | agent + 生图 API 的现有架构 |

Agent + API 形态几乎不改现有架构：skill 机位表本来就是 Step 2 产物，
工具链只是把 Step 3–4 的「提示词生成」换成「反投影渲染 + 定向补洞」，QC 照旧。

## 6. MVP 分期（按性价比排序）

| 分期 | 内容 | 产出体感 |
|---|---|---|
| **P0（1–2 天）** | 单参考图 + 手动 camera_spec → 深度 → warp → inpaint。不渲点云也行（目标机位离参考机位不远时） | **透视正确但有洞的图长什么样、洞在哪**——最重要的一手体感 |
| **P1（约 1 周）** | 尺度锚、双参考融合、ControlNet 结构锁定、掠射角掩码、provenance 图、机位表→camera_spec 转换脚本 | 可用工具 |
| **P2（2–4 周）** | 批处理 CLI（整表进 N 张出）、少样本 3DGS 替换点云、QC 半自动（provenance + D 组对比）、失败重试与抽卡预算统计 | 生产化 |

## 7. 与 skill 的对接（零缝）

```
multicam-coverage skill                    几何管线
─────────────────────                      ──────────────
Step 1 analysis（K/R/t 反推）    ──校验──►  [2] pose
Step 2 机位表·参数式列           ──转录──►  camera_spec.yaml（输入契约）
Step 2 机位表·画面式列           ──用作──►  [5] 补洞提示词
Step 4 锁定块                    ──用作──►  [5] 补洞的场景一致性描述
Step 5 18 项自检                 ──照用──►  [7] QC（+ provenance 新增项）
uncertain / 弱纹理               ──升级──►  manual-constraints.md 标注入口
```

**换到路线 A 不用重写 skill，只换 Step 3–4 的执行器**。机位表被设计成
双列、路线无关，就是为了让它成为三路线共享资产。

## 8. 诚实的边界

- 渲染是精确数学，**误差全部来自 [1][2] 的估计**——深度/位姿是真值则新视角
  像素级完美（所以建模才完全准确，见 multiview-geometry.md §6）
- 弱纹理/反射/透明的深度是「大结构对、细节是猜的」——猜的部分进 hole_mask
- 路线 A 的价值不是完美，是**把误差从结构性（人物翻转/道具换位）降级为
  局部性（这块墙歪 20 像素）**
- 连这个精度都不接受 → 只剩建模一条路（给真值深度，误差归零）。
  两张照片反推 3D 本来就是病态问题，「大结构对 + 不确定处显式留痕」
  已是不建模条件下的最优解
