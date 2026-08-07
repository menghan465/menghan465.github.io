+++
date = '2026-07-29T20:00:00+08:00'
draft = false
title = '从节点图生成 GLSL：为 RPG Maker MZ 实现小型 Shader Graph 编译链路'
description = '记录一个面向 RPG Maker MZ 的 2D Shader Graph 原型，说明节点数据、类型检查、GLSL ES 1.00 生成、PixiJS 预览和 MZ 项目导出的实现方式与限制。'
tags = ['RPG Maker MZ', 'TypeScript', 'React', 'PixiJS', 'GLSL', '编译器']
categories = ['游戏开发', '图形技术']
slug = 'rmmz-shader-graph-compiler-pixijs'
+++

RPG Maker MZ 可以通过 PixiJS Filter 使用自定义 fragment shader，但直接在插件中手写 GLSL 会带来两个问题：参数和表达式不容易复用，非图形程序员也很难快速调整效果。为验证“可视化编辑—实时预览—导出到 RPG Maker MZ”这条链路，我做了一个小型 2D Shader Graph 原型。

## 项目要验证的链路

项目的核心流程是：

```text
节点图 JSON
    ↓
节点类型与连接检查
    ↓
GLSL ES 1.00 生成
    ↓
PixiJS 预览
    ↓
RPG Maker MZ 运行时插件 + .rmmzshader.json
```

编辑器使用 React 18、TypeScript、`@xyflow/react` 和 Vite。预览使用 RPG Maker MZ 所采用的 PixiJS 5.3.12。Shader 目标是 GLSL ES 1.00，因为这个版本更接近项目需要覆盖的 WebGL 运行环境。

## 节点图不是 shader 字符串

每个节点在图数据中拥有 ID、类型、位置、参数和输入连接。连接引用节点 ID 以及端口名，而不是引用数组下标。这样在移动节点、插入新节点或重新排列数据时，连接关系不会因为数组顺序变化而失效。

当前节点集合被限制在一个较小的子集：`Texture`、`UV`、`Time`、`Float`、`Color`、`Add`、`Multiply`、`Sine`、`Mix`、`UV Warp` 和 `Fragment Output`。它们足以验证纹理采样、时间动画、颜色运算和 UV 扰动，但不等于完整的材质编辑器。

## 类型检查和表达式生成

编译器目前处理 `float`、`vec2`、`vec3` 和 `vec4` 这几类值。节点连接时先检查输入端口与输出端口是否兼容；在必要场景下做有限的类型提升和构造转换。例如，`float` 可以用于构造同维度的向量，但不能把任意 `vec4` 静默连接到只接受标量的端口。

编译过程不是简单地把节点按画布位置从左到右拼接。编译器会从 `Fragment Output` 反向访问依赖节点，递归生成表达式，并为已经访问过的节点缓存结果。这样有两个好处：

- 没有被输出节点引用的节点不会生成无用代码；
- 一个节点被多个下游节点使用时，不会重复生成相同表达式。

如果图中没有 `Fragment Output`、输入端口缺失，或者依赖关系形成环，编译器会返回诊断信息而不是生成一段看似完整但无法工作的 shader。环检测尤其重要，因为节点图的连线操作很容易形成循环，而 GLSL 表达式本身没有办法表达这种数据依赖。

## uniform 和公开参数

`Float Parameter` 与 `Color Parameter` 的值不会直接写死在表达式中。编译器会为它们生成 uniform 声明和运行时参数清单，预览组件与 MZ 运行时使用同一份参数定义。这样编辑器中修改参数时，可以直接更新 PixiJS Filter，而不需要重新生成整段 shader。

时间节点使用运行时提供的时间 uniform。它只负责生成时间值，具体的速度、振幅和混合方式仍由节点图决定。对于水面扰动、呼吸、闪烁等简单效果，这种模式已经够用；对于需要粒子系统、骨骼或复杂光照的效果，当前模型并不适合。

## PixiJS 预览和真实运行时并不完全相同

编辑器预览使用 PixiJS 创建 Filter，并把生成的 fragment shader 和 uniform 传入 WebGL。预览的意义是快速检查节点连接、颜色结果和参数变化，但它不能保证所有 RPG Maker MZ 场景都完全一致。纹理尺寸、渲染分辨率、底层 spriteset 结构以及设备 WebGL 能力都可能影响最终结果。

当 WebGL 不可用时，编辑器会显示静态预览并明确标记当前环境不支持实时 shader。这个降级行为只针对编辑器；RPG Maker MZ 运行时仍然需要可用的 WebGL 才能执行自定义 Filter。

## 导出到 RPG Maker MZ

点击 `Export MZ` 时，工具会生成两类文件：

```text
js/plugins/RMMZ_<shader>_Runtime.js
data/shaders/<shader>.rmmzshader.json
```

资源 JSON 保存图和参数信息，运行时插件负责注册 shader、读取资源并向当前场景应用 Filter。插件命令提供了 `Apply to current scene`，脚本 API 也可以操作已注册的 shader，例如：

```js
RMMZShaderGraph['waterline-warp'].applyTo(sprite);
RMMZShaderGraph['waterline-warp'].setParameter('warpAmount', 0.06);
RMMZShaderGraph['waterline-warp'].remove();
```

为了避免误写其他目录，CLI 导出要求目标目录包含 `Game.rmmzproject`、`data/` 和 `js/plugins/`。所有计算出的输出路径都必须位于项目根目录之下；已有文件默认不覆盖，只有显式设置 `force`、`RMMZ_EXPORT_FORCE=1` 或传入 `--force` 时才允许覆盖。写入过程使用临时文件替换，避免导出中断留下半份资源。

运行时适配器默认操作当前场景的 `_spriteset._baseSprite`。这能验证导出闭环，但它是 RPG Maker MZ 的内部对象，版本升级时可能变化。因此生产版本还需要增加稳定的 Sprite/Picture 绑定层，不能把当前内部字段当成长期公共 API。

## 编辑器状态和恢复

节点移动在指针释放时记为一次历史操作，而不是把每个 pointer move 都写入撤销栈。参数编辑在控件失焦或交互结束时提交。选择状态、视口平移、预览播放和标签切换不属于文档变更，因此不会污染撤销记录。

当前支持 `Ctrl/Cmd+Z`、`Ctrl/Cmd+Shift+Z`、`Ctrl/Cmd+Y`、`Ctrl/Cmd+S` 和 `Delete`。撤销栈最多保存 100 个图快照。编辑器还会在文档处于 dirty 状态时写入短期本地恢复副本，保存后删除。导入操作同时接受可编辑图格式和包含嵌入图数据的运行时 manifest，便于从已经导出的资源恢复编辑内容。

## 验证结果

截至 2026 年 7 月 28 日，项目执行了：

```text
npm test
npm run build
```

测试覆盖编译器、历史记录和 MZ 项目导出，共有 3 个测试文件、10 项测试通过。TypeScript 检查和 Vite 生产构建也通过。构建产物约为 822 KB，并出现 PixiJS 等依赖导致的包体积警告。这是目前原型阶段的已知问题，不能因为构建成功就把体积问题忽略掉。

## 当前限制

这个 Shader Graph 目前只实现了单个 2D 图层的 fragment shader，具体限制包括：

- 只有一个 pass，没有完整顶点、光照或粒子 shader；
- 不支持节点循环；
- 不兼容 Godot Shader Language、`.tres` 或 `.res`；
- 不能直接导入 Godot VisualShader 图；
- 依赖 RPG Maker MZ 的内部 spriteset 结构；
- 没有实现 GLSL 编译器，只负责生成代码并交给 WebGL/PixiJS 编译；
- Vite 构建包体积偏大，仍需进一步拆分或按需加载。

因此，项目的实际价值在于把一条窄而完整的工程链路跑通，而不是提供一个可以替代 Godot、Unity 或专业材质编辑器的通用工具。后续如果继续开发，优先级应该是稳定的运行时绑定、更多诊断信息、资源生命周期管理和体积优化，而不是盲目增加节点数量。
