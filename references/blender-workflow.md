# Blender 工作流 — 建模路线（完全准确）操作手册

定位：建模 + 渲染是三条路线之上的「天花板」——深度是真值、未见区在建模时
就造全了。本文件是它的操作手册，与 `geometric-pipeline.md`（不建模的近似
路线）成对。核心工作：**把机位表参数式列翻译成 Blender 相机对象，批量渲静帧**。

## 1. 机位表 → Blender 映射

| 机位表参数式 | Blender | 说明 |
|---|---|---|
| 焦距 35mm / 50mm | `cam.data.lens = 35` | **Blender 默认 sensor_width = 36mm = 全画幅**，焦距数值与机位表 1:1，FOV 表直接适用 |
| 机位高度 1.35m | `cam.location.z = 1.35` | Scene Properties → Unit: Metric，数值即米 |
| 方位角（B 外侧后方 30°） | 绕被摄者展开 `location.x/y` | 见 §3 脚本 `cam_behind` |
| 距离 | 相机到被摄者水平距离 | 决定人物占画比例 |
| 俯仰 | look-at 目标高度 | 平视→胸口 1.2–1.4m；仰拍→脸部以上 |
| 过肩前景 | 相机摆前景人物背后，真实模型挡进画面 | 物理遮挡自动正确，不用写「右下 15%」 |
| 景深 | `cam.data.dof`（focus_distance / fstop） | 过肩镜常配浅景深，每机位可独立设 |

机位表「画面式」列在这里退化成交付说明——内容由场景物理决定。

## 2. 三种摆机位方式

1. **交互**：摆个大概位置 → 视图 `Ctrl+Alt+Numpad 0` 对齐相机 → 微调
2. **look-at 脚本（推荐）**：写位置 + 目标点，`to_track_quat('-Z','Y')` 自动算旋转
3. **时间轴标记**：每机位一台相机，时间轴选中相机按 `Ctrl+B` 绑定 marker，
   渲动画时逐帧换相机（视频交付物用，见 §8）

## 3. 批量渲染（机位表当输入）

```python
import bpy, math
from mathutils import Vector

def make_cam(name, location, look_at, lens=35):
    cam_data = bpy.data.cameras.new(name)
    cam_data.lens = lens
    cam_data.sensor_width = 36.0          # 全画幅：与机位表焦距 1:1
    cam = bpy.data.objects.new(name, cam_data)
    bpy.context.collection.objects.link(cam)
    cam.location = Vector(location)
    direction = Vector(look_at) - cam.location
    cam.rotation_euler = direction.to_track_quat('-Z', 'Y').to_euler()
    return cam

def cam_behind(name, fg, subject, azimuth_deg, dist, height, lens, look_at):
    """站在 fg 背后（相对 subject 远侧）偏开 azimuth 度 → 过 fg 肩拍 subject"""
    away = (fg - subject).normalized()
    a = math.radians(azimuth_deg)
    rot = Vector((away.x*math.cos(a) - away.y*math.sin(a),
                  away.x*math.sin(a) + away.y*math.cos(a), 0.0))
    loc = fg + rot * dist
    loc.z = height
    return make_cam(name, loc, look_at, lens)

# 场景假设：A 站原点、B 站 (3,0,0)，轴线 = X 轴
A, B = Vector((0, 0, 0)), Vector((3, 0, 0))

jobs = [
    # 机位表每一行 → 一个相机（参数式列直接转录）
    cam_behind("M1_过B肩拍A", fg=B, subject=A, azimuth_deg=30, dist=2.2,
               height=1.35, lens=35, look_at=Vector((0, 0, 1.35))),
    cam_behind("M2_过A肩拍B", fg=A, subject=B, azimuth_deg=-30, dist=2.2,
               height=1.35, lens=35, look_at=Vector((3, 0, 1.2))),
    make_cam("M3_总机位", (1.5, -2.5, 1.5), (1.5, 0, 1.2), 35),
]

scene = bpy.context.scene
scene.render.engine = 'CYCLES'            # 预览阶段用 'BLENDER_EEVEE_NEXT'
scene.render.image_settings.file_format = 'PNG'

# 对接富化/合成时输出真值通道（免费的路线 A 遗产）
scene.view_layers[0].use_pass_z = True
scene.view_layers[0].use_pass_normal = True
# 想连通道一起存：file_format = 'OPEN_EXR_MULTILAYER'

for cam in jobs:
    scene.camera = cam
    scene.render.filepath = f"//renders/场景名_{cam.name}.png"   # 命名照 skill 约定
    bpy.ops.render.render(write_still=True)
```

无头批跑（agent / CI 集成的关键一行）：

```bash
blender -b scene.blend -P render_cams.py
```

## 4. 引擎选择与流程

| | EEVEE Next | Cycles |
|---|---|---|
| 速度 | 实时级 | 慢（路径追踪） |
| 用途 | **全表先跑一遍**看构图/遮挡 | 最终出片 |
| 精度 | 光栅化近似 | 物理正确（光影/景深/反射） |

流程：EEVEE 全表预览（几秒）→ 确认机位表无误 → Cycles 出正片。

## 5. 白送的红利（建模路线相对生成路线）

1. **光照一致是物理事实**——所有机位共享同一套灯，锁定块最麻烦的项自动过
2. **尺度天然绝对**——「人当尺子」的尺度锚不需要，1.35m 就是 1.35m
3. **轴线规则变成摆位纪律**——模型不会自己越轴
4. **真值深度/法向免费**——勾 pass 输出，等于路线 A 最贵的「深度估计」环节的完美替代
5. **改机位边际成本≈0**——改一行参数重渲

## 6. 富化（写实化）的正确位置

渲染图常有「CG 感」（皮肤太滑、布料太干净、缺微纹理）。**只让模型管质感，
不让它管几何**：

```
Blender 静帧 + depth/normal 通道
        ↓
ControlNet(depth+normal) + img2img（低 denoise 0.3–0.45）
        ↓
写实化多机位图 ✓ 几何仍由渲染锁死
```

每张独立富化、独立重试（接 skill 修正协议），provenance 记录「纯渲染 / 富化过」。

## 7. 多模态模型的两个岗位（不是「富化后截图」）

| 岗位 | 干什么 | 价值 |
|---|---|---|
| **审片（QC）** | 喂多机位图跑 18 项自检：左右关系/地平线/恒定物/遮挡顺序 | 人工核对→半自动 |
| **质感提示词** | 看渲染图写出「缺什么质感」的 img2img 提示词 | 富化有的放矢 |

## 8. 为什么静帧必须直接渲染（禁止从渲染视频里截图）

「渲一段视频 → 多模态富化 → 截图」对静帧需求是**错误流程**，逐条：

1. **位姿量化误差**：视频帧率把连续机位路径离散化（24fps = 每帧一个时间格），
   机位表要的精确 (R,t,K) 只在「碰巧对齐的帧」出现，一般取不到设计值。
   静帧渲染 = 参数精确命中
2. **运动模糊**：移动镜头每帧带快门拖影（180° 快门下可达数十像素的涂抹），
   静帧渲染零模糊（除非刻意开）
3. **编码损失**：YUV 4:2:0 色度半采样、8/10-bit、帧间压缩（截到 P/B 帧是
   重建画质）；窗框等**透视判断依赖的高对比边缘**最吃压缩 ringing。
   静帧 PNG/EXR 全质量全色深
4. **分辨率上限**：视频受码率/格式限制，静帧可 8K/EXR
5. **重渲粒度**：改一个机位（如 50→85mm），静帧只重渲一张；视频要重渲序列、
   重剪、重编码
6. **provenance**：静帧有机位名/引擎/参数可追溯；视频第 517 帧没有身份
7. **渲染通道**：depth/normal 随静帧多层 EXR 自然输出；视频流要另存整段通道
   序列（翻倍存储）或直接丢失
8. **生成富化毁真值**：视频扩散「富化」= 逐帧重新想象像素，几何真值被扔掉
   （透视漂移/身份漂移回来），再叠加压缩损失——两头都占

**停驻段变通**（如果非要从视频拿）：机位时间轴上每机位停 0.5–1s，从停驻段
截帧 ≈ 精确机位。但这只是增加了排程负担，1–3、5–7 条照样吃亏——
**有 Blender 工程在手，静帧直接渲永远是上策**。

## 9. 与 skill 的对接

```
skill Step 2 机位表·参数式列  ──转录──►  jobs 列表 / render_cams.py
skill Step 5 文件命名约定      ──照用──►  renders/场景名_M1_*.png
skill Step 5 18 项自检         ──大半自动──► 物理事实保证，只剩内容/表演类检查
manual-constraints            ──不需要──► 深度是真值
geometric-pipeline            ──被取代──► 仅富化段（ControlNet）沿用其思路
```

## 10. 决策速查

| 交付物 | 流程 |
|---|---|
| 多机位静帧 | **直接渲静帧**（EEVEE 预览 → Cycles 出片） |
| 静帧 + 写实感 | 静帧 + depth/normal ControlNet 富化 |
| 视频成片 | 时间轴排运镜 → 渲视频 → V2V 富化**整段** |
| 静帧 + 视频都要 | 时间轴机位停驻；**静帧另渲**，不从富化视频截图 |
| 自动化 QC | 多模态模型跑 18 项审片（§7） |
