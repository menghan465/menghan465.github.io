+++
date = '2026-07-30T02:53:13+08:00'
draft = false
title = '不重写 2D 玩法，如何在 Godot 中嵌入 3D 火车：SubViewport、坐标投影与屏幕空间碰撞'
description = 'GMTK Game Jam 2026 提交版本《黑猩猩与电车问题》的技术复盘：在约 97 小时的提交约束下保留 2D 规则层，用 SubViewport 嵌入 3D 火车与轨道，并将 3D 坐标投影回屏幕完成目标绘制和近似碰撞。'
tags = ['Godot', 'GDScript', 'Game Jam', 'SubViewport', '2D/3D', '游戏开发']
categories = ['游戏开发']
slug = 'godot-subviewport-2d-3d-train'

[cover]
alt = '《黑猩猩与电车问题》的三轨跑酷画面'
caption = '《黑猩猩与电车问题》游戏画面'
relative = false
hidden = false
+++

[《黑猩猩与电车问题（The Chimpanzee and the Trolley Problem）》](https://menghan465.itch.io/the-chimpanzee-and-the-trolley-problem) 是我为 **GMTK Game Jam 2026** 制作并提交的参赛项目。比赛提供了约 97 小时的提交窗口。本文讨论的技术方案属于截止时间前提交并锁定的版本，不是赛后重构。

它的规则层是一款三轨跑酷游戏：玩家控制电车换道，目标沿轨道接近，撞击后进入击飞状态；两分钟回合结束后，玩家可以在商店购买下一轮生效的道具。

在提交窗口内，项目进行过一次中途的视觉改造：**玩法已经按 2D 坐标运行，我却希望把电车、轨道和沿线环境换成 3D，同时不重写现有的目标生成、碰撞、HUD 和回合逻辑。**

我最后采用了一个混合方案：

- 3D 火车、轨道和环境放进 `SubViewport`；
- `SubViewport` 的纹理显示在 2D 主场景的 `TextureRect` 上；
- 猩猩、命中特效、HUD 和商店继续由 `Node2D` 管理；
- 通过 `Camera3D.unproject_position()` 把 3D 轨道位置投影回 2D 屏幕；
- 机车车头的 3D 投影再被转换成一个屏幕空间碰撞框。

这不是 3D 物理系统，也不是自定义渲染管线。它是一种面向 Game Jam 开发的范围控制：尽量保留已经能工作的规则层，只替换视觉层。

## 为什么不把整个游戏改成 3D

当时已经存在的 2D 系统包括：

- 三条轨道及换道逻辑；
- 目标生成与移动；
- 猩猩的 Sprite 动画和击飞效果；
- 手工命中判定；
- 两分钟回合、商店和三种道具；
- HUD、音频和最佳成绩存档。

如果全部改成 3D，就需要同时处理：

1. 目标节点和运动方式重写；
2. `Area3D` 或刚体碰撞重新标定；
3. 3D 模型尺寸、相机 FOV 和玩法速度重新匹配；
4. HUD 与世界对象的遮挡关系；
5. 已经调好的 2D 击飞动画和商店流程回归测试。

对长期项目来说，全 3D 可能更统一；但在约 97 小时的提交窗口里，已有 2D 闭环能够运行时，全面重写带来的回归风险大于收益。这里的判断不是“混合架构永远优于全 3D”，而是“在剩余时间有限、现有规则已经可用的条件下，不应为了形式统一扩大改动范围”。我的目标不是建立一套通用框架，而是在不破坏现有闭环的前提下完成一次可控的视觉升级。

| 方案 | 优点 | 主要代价 |
| --- | --- | --- |
| 全部重写成 3D | 坐标和物理系统统一 | 改动范围最大，原有玩法需要全面回归 |
| 预渲染 3D 背景 | 实现快 | 电车换道、轨道滚动和相机联动受限 |
| `SubViewport` 混合 2D/3D | 保留 2D 规则，3D 画面仍可实时变化 | 需要维护 2D、3D 和屏幕坐标之间的映射 |

最终我选择了第三种。

## 整体架构：规则层和视觉层分开

项目中的职责大致分成三层。

### 2D 玩法层

主场景仍然是 `Node2D`。`scripts/main.gd` 负责：

- `INTRO`、`PLAYING`、`SHOP` 等游戏状态；
- 三轨输入和车道切换；
- 目标数据、生成概率和击飞状态；
- 商店、道具、HUD、音效与存档；
- 最终的屏幕空间命中判定。

猩猩没有被改成 `Node3D`。它们仍然保存在 2D 逻辑的数据结构中，主要状态是车道、纵向进度、屏幕偏移和击飞速度。

### 3D 视觉层

`scripts/train_system_visual_3d.gd` 在运行时创建：

- 一个全画面 `SubViewport`；
- 一个作为 3D 根节点的 `TrainWorld`；
- `Environment`、平面地面、方向光和 `Camera3D`；
- 机车模型；
- 三条轨道，每条 15 段，共 45 个轨道实例；
- 两侧循环移动的树、花、栅栏和路灯。

### 坐标桥接层

桥接层不单独对应一个 Godot 节点，而是一组投影函数。它负责把：

```text
车道编号 + 目标进度
        ↓
3D 世界坐标 Vector3
        ↓ Camera3D.unproject_position()
2D 屏幕坐标 Vector2
```

同一个屏幕坐标既用于绘制猩猩，也用于命中判断。这样至少避免了“画面中的目标位置”和“碰撞系统认为的目标位置”各算一遍。

## 用 SubViewport 把 3D 世界嵌入 2D 主场景

`TrainSystemVisual3D.setup()` 首先加载机车和轨道资源。如果任一关键资源不存在，函数直接返回 `false`，让主场景决定是否启用 3D 视觉。

下面是删减后的核心代码：

```gdscript
func setup(view_size: Vector2i, target: TextureRect, camera_fov: float = 32.0) -> bool:
    var locomotive_scene := load(LOCOMOTIVE_PATH) as PackedScene
    var track_scene := load(TRACK_PATH) as PackedScene
    if locomotive_scene == null or track_scene == null:
        return false

    viewport = SubViewport.new()
    viewport.name = 'TrainSystemViewport'
    viewport.size = view_size
    viewport.render_target_update_mode = SubViewport.UPDATE_ALWAYS
    viewport.msaa_3d = Viewport.MSAA_DISABLED
    add_child(viewport)

    display_target = target
    display_target.texture = viewport.get_texture()
    display_target.visible = true

    var world_root := Node3D.new()
    world_root.name = 'TrainWorld'
    viewport.add_child(world_root)
    _build_environment(world_root, camera_fov)
```

这里的关键点不是创建 `SubViewport` 本身，而是它在场景中的位置：`TrainSystemVisual3D` 仍然挂在 2D 主场景下，但它内部维护了一个独立的 3D 世界。`SubViewport.get_texture()` 返回的纹理交给全屏 `TextureRect`，因此 3D 画面会像一张实时更新的背景图一样出现在 2D 场景中。

主场景随后继续在这张纹理上方绘制猩猩、撞击圆环和 UI。

### 轨道为什么看起来在无限延伸

电车本身只在三条轨道之间横向移动。前进感主要由轨道和装饰物向相机方向移动产生。

```gdscript
func update_visual(delta: float, normalized_lane_position: float, game_speed: float) -> void:
    train_instance.position.x = lerpf(
        -LANE_SPACING,
        LANE_SPACING,
        clampf((normalized_lane_position + 1.0) * 0.5, 0.0, 1.0)
    )

    var travel := game_speed * 0.022 * delta
    var wrap_distance := TRACK_COUNT_PER_LANE * TRACK_LENGTH

    for track_instance in track_instances:
        track_instance.position.z += travel
        if track_instance.position.z > TRACK_NEAR_Z + TRACK_LENGTH:
            track_instance.position.z -= wrap_distance
```

每条轨道由 15 段相同模型组成。某一段越过近端边界后，就被移动到整条轨道的远端。沿线装饰使用相同思路，只是循环距离不同。

`0.022` 不是现实单位换算，而是视觉速度系数。2D 规则层的 `game_speed` 使用屏幕移动速度，3D 世界使用 Z 轴距离，两者量纲并不相同。这也是混合方案需要接受的事实：它追求画面与操作感一致，而不是物理单位一致。

## 三套坐标如何互相对应

这个项目里同时存在三套坐标：

1. **2D 逻辑坐标**：目标用 `y` 表示从远处向玩家靠近的进度；
2. **3D 世界坐标**：轨道沿 Z 轴延伸，车道沿 X 轴分布；
3. **屏幕坐标**：最终用于 Sprite 绘制和命中检测的 `Vector2`。

### 先把 2D 进度映射到 3D 轨道

目标在 2D 逻辑中从 `y = -65` 移动到屏幕下方。绘制目标时，先把这个范围归一化成 `0..1`：

```gdscript
func _chimp_approach_position(chimp: Dictionary) -> Vector2:
    var y := float(chimp['y'])
    if use_train_system_visuals:
        var travel_progress := clampf(
            inverse_lerp(-65.0, VIEW_SIZE.y + 90.0, y),
            0.0,
            1.0
        )
        var projected_position := train_visual.get_target_screen_position(
            int(chimp['lane']),
            travel_progress
        )
        projected_position.x += float(chimp['offset_x'])
        return projected_position
```

`travel_progress` 只表示“目标从远端走到了多近”，不等于真实的物理时间或轨道里程。

### 再把 3D 世界点投影到屏幕

`TrainSystemVisual3D` 根据车道和进度构造一个 3D 点：

```gdscript
func get_target_screen_position(lane: int, progress: float) -> Vector2:
    if camera == null:
        return Vector2(640.0, 360.0)

    var world_z := lerpf(
        TARGET_FAR_Z,
        TARGET_NEAR_Z,
        clampf(progress, 0.0, 1.0)
    )

    return camera.unproject_position(
        Vector3(float(lane - 1) * LANE_SPACING, 0.72, world_z)
    )
```

三条车道对应 `lane = 0, 1, 2`，所以它们的世界 X 坐标分别是：

```text
-LANE_SPACING, 0, +LANE_SPACING
```

Z 坐标则在 `TARGET_FAR_Z` 和 `TARGET_NEAR_Z` 之间插值。最后，Godot 的 `Camera3D.unproject_position()` 把这个世界点转换成当前 Viewport 中的二维像素坐标。

这个 API 的命名容易让人误解。在这里，它做的是 **3D 世界位置到 2D Viewport 位置** 的转换。

这里还有一个成立条件：`SubViewport` 的尺寸与 2D 逻辑视口都是 1280×720，承载纹理的 `TextureRect` 也覆盖同一块区域。因此 `unproject_position()` 返回的像素坐标可以直接交给 `Node2D` 使用。

如果为了性能把 3D Viewport 降到 640×360，或者让 `TextureRect` 以保持比例的方式显示在带黑边的区域，就不能再直接复用该坐标。至少需要应用：

```text
screen_position = display_offset
                + viewport_position × (display_size / viewport_size)
```

这也是后续降低 `SubViewport` 分辨率时必须一起修改的部分，否则画面会缩放，2D 目标和命中框却仍停留在原坐标上。

### 这种映射为什么够用

因为目标本身就是 2D Sprite。只要以下两件事一致，玩家通常不会察觉目标并不真的存在于 3D 世界中：

- Sprite 的屏幕位置沿着 3D 轨道投影移动；
- 命中判断也使用同一个投影位置。

代价是映射依赖一组手工标定的常量，例如：

- `LANE_SPACING = 4.2`；
- `TARGET_FAR_Z = -112.0`；
- `TARGET_NEAR_Z = 25.0`；
- 相机 FOV 当前固定为 `32°`；
- 目标在 3D 世界中的高度固定为 `0.72`。

如果相机位置、FOV、机车缩放或轨道模型发生明显变化，这些值也需要重新校准。

## 从机车车头投影出屏幕空间碰撞框

目标虽然是 2D，但机车已经变成 3D。最直接的问题是：**什么时候算撞到车头？**

如果直接使用整辆机车在屏幕上的外接矩形，目标在接近车身中部时就可能提前触发。我的处理方式是只取机车前端的一个 3D 薄片，将它的四个角投影到屏幕。

核心过程如下：

```gdscript
func get_collision_rect(wide_bumper: bool) -> Rect2:
    var model_scale := train_instance.scale.x
    var half_width := 1.74 * model_scale * (1.45 if wide_bumper else 1.0)
    var front_z := train_instance.position.z - 11.9 * model_scale
    var bottom_y := train_instance.position.y + 0.15 * model_scale
    var top_y := train_instance.position.y + 2.45 * model_scale

    var points := [
        camera.unproject_position(Vector3(train_instance.position.x - half_width, bottom_y, front_z)),
        camera.unproject_position(Vector3(train_instance.position.x + half_width, bottom_y, front_z)),
        camera.unproject_position(Vector3(train_instance.position.x - half_width, top_y, front_z)),
        camera.unproject_position(Vector3(train_instance.position.x + half_width, top_y, front_z)),
    ]

    var minimum: Vector2 = points[0]
    var maximum: Vector2 = points[0]
    for point in points:
        minimum = minimum.min(point)
        maximum = maximum.max(point)

    var result := Rect2(minimum, maximum - minimum).grow(12.0)
    if result.size.y < 58.0:
        var center := result.get_center()
        result.size.y = 58.0
        result.position.y = center.y - result.size.y * 0.5
    return result
```

四个投影点得到后，取每个轴上的最小值和最大值，构造一个屏幕空间的轴对齐矩形。`grow(12.0)` 会为它增加一定容错范围；最小高度限制则避免远近透视变化把判定框压得过薄。

主逻辑中的命中判定仍然非常直接：

```gdscript
func _check_chimp_collisions() -> void:
    var hit_rect := _current_vehicle_hit_rect().grow(18.0)

    for index in range(chimpanzees.size()):
        var chimp := chimpanzees[index]
        if String(chimp['state']) != 'approach':
            continue

        var chimp_position := _chimp_approach_position(chimp)
        if hit_rect.has_point(chimp_position):
            _knock_chimp(index, chimp_position)
```

这里又额外 `grow(18.0)`，意味着最终判定是刻意偏宽松的。Game Jam 作品中，玩家感知到的“应该撞到了”通常比几何上的精确更重要。

### Area2D 在这里不是碰撞系统

场景中仍然有 `VehicleHitArea`、`CollisionShape2D` 和 `HitboxPreview`，但 `Area2D` 的碰撞层和遮罩都设为 0，也没有依赖 `area_entered` 等信号。

每帧只是把计算出的 `Rect2` 同步给 `CollisionShape2D` 和调试多边形：

```gdscript
func _update_vehicle_hit_area() -> void:
    var hit_rect := _current_vehicle_hit_rect()
    vehicle_hit_area.position = hit_rect.get_center()

    var rectangle_shape := vehicle_hit_shape.shape as RectangleShape2D
    if rectangle_shape != null:
        rectangle_shape.size = hit_rect.size
```

调试构建中可以按 `F3` 显示这个碰撞框。它的作用是校准和观察，不是让 Godot 物理服务器负责命中。

准确地说，这套方案是：

> **基于 3D 几何投影的 2D 屏幕空间命中近似。**

它不是 3D 物理碰撞。

## 为什么没有把猩猩也放进 SubViewport

把目标改成 `Sprite3D` 或带 Billboard 的 `Node3D`，表面上会让系统更统一，但会重新引入几个问题：

- 目标击飞要从 2D 抛物线改成 3D 运动；
- Sprite 与 3D 模型之间的遮挡顺序需要重新处理；
- 目标吸附道具要在三维坐标中重新调参；
- HUD 和命中特效仍然需要世界到屏幕的转换；
- 原来的生成和碰撞逻辑不能直接复用。

当前玩法没有高度差、坡道或立体障碍物。目标只需要沿三条视觉轨道接近玩家，所以保留 2D 目标是更便宜的选择。

如果以后加入桥梁、隧道、跳跃、上下层轨道或真实遮挡，这个前提就会失效。届时继续维持屏幕空间近似，复杂度反而可能超过一次完整的 3D 重构。

## 资源加载失败时回退到 2D

混合视觉模块通过 `setup()` 返回是否可用：

```gdscript
func _build_train_visual() -> void:
    train_visual = TrainSystemVisual3D.new()
    train_visual.name = 'TrainSystemVisual3D'
    add_child(train_visual)

    use_train_system_visuals = train_visual.setup(
        Vector2i(int(VIEW_SIZE.x), int(VIEW_SIZE.y)),
        train_viewport_display
    )

    if not use_train_system_visuals:
        train_viewport_display.visible = false
        train_visual.queue_free()
        train_visual = null
```

主场景的 `_draw()` 根据结果选择渲染路径：

```gdscript
func _draw() -> void:
    if not use_train_system_visuals:
        _draw_background()
        _draw_tracks()

    for chimp in chimpanzees:
        _draw_chimp(chimp)

    for effect in hit_effects:
        _draw_hit_effect(effect)

    if not use_train_system_visuals:
        _draw_vehicle()
```

如果机车或轨道的 `PackedScene` 没有正确导入，游戏仍然可以使用旧的程序化 2D 背景、轨道和车辆运行。材质与贴图走的是另一层回退：ShaderMaterial 加载失败时会尝试重建 `StandardMaterial3D`，它们本身不会触发整套 2D 降级。在提交时间受限的情况下，这至少避免了关键 3D 场景丢失后主场景完全无法运行。

它不能替代正式的资源校验和自动化构建，但在原型阶段是有价值的保护。

## 这个方案没有解决什么

这部分比“它能做什么”更重要。

### 1. 没有真实性能结论

当前 `SubViewport` 使用：

```gdscript
viewport.size = Vector2i(1280, 720)
viewport.render_target_update_mode = SubViewport.UPDATE_ALWAYS
```

同时场景里有 45 个轨道实例和多组沿线装饰，2D 主场景也会每帧重绘目标和特效。项目目前没有保存系统性的 Godot Profiler 数据，因此我不能据此声称它“高性能”或“适合低端设备”。

更合理的后续测试包括：

- 把 3D Viewport 分辨率降到 960×540 或 640×360，比较 GPU 帧时间；
- 用 `MultiMeshInstance3D` 代替 45 个独立轨道节点；
- 缓存每个目标一帧内的投影结果，避免绘制和碰撞重复计算；
- 分别记录 2D 回退模式和 3D 模式的 CPU、GPU 帧时间；
- 在集成显卡设备上验证，而不是只报告开发机 FPS。

在这些数据产生之前，以上都只是优化方向，不是优化成果。

### 2. 碰撞是近似值

当前命中框是四个投影点的轴对齐包围盒，并且只检测目标中心点。它没有处理：

- 目标 Sprite 的实际面积；
- 机车旋转后的投影多边形；
- 高速运动下的连续碰撞；
- 遮挡；
- 不同高度的目标；
- 真正的 3D 深度关系。

对于固定相机和三轨跑酷，这种近似足够容易控制；对于自由相机或复杂地形，它不可靠。

### 3. 投影参数仍然是硬编码

相机 FOV、目标远近位置、机车前端位置和模型比例都写在脚本常量中。更完整的版本应该把它们集中到一个 `Resource`，并提供可视化校准工具，而不是在多个函数里手工修改数值。

### 4. 规则层仍然过于集中

当前 `main.gd` 同时管理状态、生成、目标、碰撞、商店、音频、UI 和存档。它适合快速完成原型，不适合长期扩展。继续开发时，应先拆分目标生成器、车辆控制、商店和音频，而不是继续向同一个脚本添加功能。

## 什么时候适合采用这种混合方案

这套方案适合以下情况：

- 既有玩法和碰撞已经稳定运行在 2D；
- 3D 主要承担视觉表现，不负责复杂规则；
- 相机基本固定；
- 游戏对象数量有限；
- 玩家关心的是屏幕上的命中反馈，而不是严格物理一致性；
- 项目时间有限，重写风险较高。

以下情况则应谨慎使用：

- 相机可自由旋转或大范围移动；
- 游戏存在高度差和复杂遮挡；
- 玩法依赖刚体、关节、射线或真实深度；
- 大量对象需要频繁执行世界到屏幕投影；
- 计划做网络同步或确定性回放；
- 3D 视觉不再只是表现层，而是规则的一部分。

## 复盘：这次真正节省的是重写成本

从实现难度看，`SubViewport` 和 `Camera3D.unproject_position()` 都是 Godot 提供的标准能力。这个方案的价值不在于 API 有多复杂，而在于它明确划定了改造边界：

- 3D 模块只负责火车、轨道、相机和环境；
- 2D 模块继续负责玩法、目标、UI 和反馈；
- 坐标投影是两者之间唯一必要的桥梁；
- 关键 3D 场景不可用时，旧的 2D 路径仍然存在。

对约 97 小时内需要完成并上传的 Game Jam 提交版本来说，这比追求架构上的绝对统一更实际。我没有得到一个完整的 3D 游戏，也没有得到精确的物理系统；我得到的是一个能够保留原有玩法、完成视觉升级并按时提交的可玩版本。

项目页面：

- [The Chimpanzee and the Trolley Problem — itch.io](https://menghan465.itch.io/the-chimpanzee-and-the-trolley-problem)

主要实现文件：

- `scripts/main.gd`：2D 规则、投影调用、碰撞、商店与回退绘制；
- `scripts/train_system_visual_3d.gd`：`SubViewport`、3D 世界、轨道循环和投影函数；
- `shaders/train_surface_3d.gdshader`：机车、轨道和装饰物的三色风格化材质。



