+++
date = '2026-08-07T00:00:00+08:00'
draft = false
title = '在 RPG Maker MV 测试游戏中实现可视化图层编辑器：全地图相机、旋转缩放与动态公式'
description = '为 RPG Maker MV 的 ULDS 地图图层系统增加测试模式可视化编辑器：直接在运行中的地图上放置和变换图片，处理全地图导航、平滑相机缩放、旋转后的八方向缩放、动态公式回读、撤销重做与地图 note 安全写回。'
tags = ['RPG Maker MV', 'JavaScript', 'PIXI.js', 'NW.js', '编辑器工具', '游戏开发']
categories = ['游戏开发']
slug = 'rpg-maker-mv-ulds-runtime-layer-editor'

+++

![image-20260807184624284](..\..\public\images\rpg-maker-mv-ulds-runtime-layer-editor\image-20260807184624284.png)

RPG Maker MV 的地图编辑器适合处理图块、事件和基础地图数据，但当地图中出现大量独立图片图层时，单靠参数和 JSON 很难继续维护。

我使用的 ULDS 图层系统会从地图 `note` 中读取多个 `<ulds>JSON</ulds>` 块，再把它们转换成 PIXI Sprite。一个图层可以配置图片路径、坐标、`z` 层级、透明度、旋转、缩放、混合模式、循环贴图和动态公式。

原始配置大致如下：

```json
{
  path: pictures,
  name: fog_01,
  x: this.rx(960),
  y: this.ry(720),
  z: 0.8,
  opacity: 180,
  scale.x: 1,
  scale.y: 1
}
```

这种格式可以运行，但存在几个直接影响制作效率的问题：

- 图片放置位置只能依靠手工填写坐标后反复测试；
- 图层数量增加后，很难判断同一 `z` 层级中有哪些对象；
- 图片旋转后，普通的水平和垂直缩放不再符合视觉方向；
- 动态公式能表达复杂行为，但不适合频繁手工编写；
- 修改最终仍要写回地图 JSON，需要避免覆盖外部修改或损坏地图文件。

因此我为 ULDS 增加了一个只在测试游戏中运行的可视化编辑器 `ULDS_Editor.js`。它不是 RPG Maker MV 自带编辑器的扩展窗口，也不是 Photoshop 的替代品，而是一个嵌入游戏运行时的地图图层制作工具。

最终实现的工作流是：

1. 在 RPG Maker MV 中启动测试游戏；
2. 按 `F10` 打开编辑器；
3. 从素材栏选择图片并直接放到当前地图；
4. 在游戏画面中拖动、旋转和缩放图片；
5. 使用全局导航移动到玩家当前视野以外的位置；
6. 编辑定位、裁切、动画帧和动态属性；
7. 通过 `Ctrl+S` 把结果写回当前地图的 `note`。

本文讨论的重点不是界面样式，而是这个运行时编辑器需要解决的坐标、渲染、变换和数据持久化问题。

## 为什么把编辑器放进测试游戏

一种更常见的方案是制作独立网页或桌面客户端，读取 RPG Maker MV 项目目录并编辑地图文件。独立工具的架构更清晰，也更容易使用现代前端框架，但它很难完整复现游戏实际运行时的渲染状态。

ULDS 图层的位置可能依赖：

- 当前地图滚动位置；
- RPG Maker MV 的循环地图规则；
- PIXI Sprite 的锚点、父节点和世界变换；
- 游戏开关 `$gameSwitches`；
- 游戏变量 `$gameVariables`；
- 玩家、事件或跟随者的实时坐标；
- 其他地图渲染插件产生的运行时状态。

如果在外部工具中重新实现这些规则，编辑器预览和游戏实际画面之间仍可能出现偏差。

运行时编辑器可以直接使用当前的 `$gameMap`、`Spriteset_Map` 和 ULDS Sprite，因此不需要再写一套近似渲染器。代价是它会依赖 RPG Maker MV、NW.js 和 PIXI 的内部结构，兼容范围比独立工具更窄。

编辑器首先限制自己的运行环境：

```javascript
if (!Utils.isOptionValid('test') || !Utils.isNwjs()) {
    return;
}
if (!window.ULDSRuntime) {
    console.error('ULDS_Editor requires ULDS.js to be enabled before it.');
    return;
}
```

这意味着正式发布版本不会加载编辑器，也不会获得本地文件写入能力。只有测试模式中的 NW.js 环境可以使用 Node.js 的 `fs` 和 `path` 模块修改地图文件。

## 编辑器和运行时之间的职责

我没有让 `ULDS_Editor.js` 重新实现一套图层渲染逻辑，而是在 `ULDS.js` 中暴露了一个较小的运行时接口：

```javascript
window.ULDSRuntime = {
    defaultSettings: cloneSettings(DEFAULT_SETTINGS),
    parse: parseSettings,
    serializeNote: serializeSettings,
    applySettings: applySettings,
    currentSpriteset: function() { /* ... */ },
    currentSprites: function() { /* ... */ },
    refreshCurrent: function(settingsList) { /* ... */ }
};
```

两部分的职责大致如下：

| 模块 | 主要职责 |
| --- | --- |
| `ULDS.js` | 解析地图 `note`、创建 Sprite、执行动态公式、把 Sprite 加入 Tilemap |
| `ULDS_Editor.js` | 维护编辑状态、生成 DOM 界面、处理鼠标交互、修改图层参数和保存地图 |

编辑器中的 `layers` 数组是当前编辑会话的数据源。参数发生变化时，编辑器调用 `runtime.applySettings(sprite, layer)` 更新对应 Sprite；当数组结构发生变化时，则调用 `runtime.refreshCurrent(layers)` 重建全部 ULDS 图层。

数据流可以概括为：

```text
地图 note 中的 <ulds> JSON
          ↓ parse
    Editor.layers
          ↓ applySettings / refreshCurrent
    PIXI / ULDS Sprite
          ↓ serializeNote
地图 MapXXX.json
```

界面本身使用 DOM 覆盖在游戏 Canvas 上。这样可以直接使用输入框、下拉框、滚动列表和原生拖放事件，而不需要用 RPG Maker 的 `Window_Base` 手工实现完整表单系统。

这种方式并不意味着 DOM 和 PIXI 使用同一套坐标。参数面板属于浏览器客户端坐标，图片属于 PIXI 世界坐标，地图位置又属于 RPG Maker 的图块坐标。后面的图片控制框必须在这些坐标之间进行转换。

## 四种图层定位方式

ULDS 的 `x` 和 `y` 不一定是普通数字。编辑器把常用位置规则整理成四种模式。

### 屏幕固定坐标

普通数值表示相对游戏屏幕的位置：

```json
{
  x: 320,
  y: 180
}
```

地图滚动时，图片仍停留在屏幕中的相同位置，适合遮罩、前景边框或固定效果。

### 地图固定坐标

地图坐标使用 ULDS 的 `rx()` 和 `ry()` 辅助函数：

```javascript
layer.x = 'this.rx(' + mapPixelX + ')';
layer.y = 'this.ry(' + mapPixelY + ')';
```

`rx()` 会根据 `$gameMap.adjustX()` 把地图像素位置换算为屏幕位置，因此图层会随地图滚动。

### 不同滚动速度的视差坐标

给 `rx()` 和 `ry()` 传入第二个参数后，可以改变图层相对地图的滚动速度：

```javascript
layer.x = 'this.rx(960, 24)';
layer.y = 'this.ry(720, 24)';
```

它适合远景、雾层或移动速度不同的背景。这里的数值不是通用渲染引擎中的 Parallax2D 节点，而是 ULDS 对地图滚动换算比例的封装。

### 绑定玩家、事件或跟随者

绑定模式会生成读取运行时对象坐标的公式：

```javascript
var obj = $gameMap.event(eventId);
return obj ? (obj._realX + 0.5) * $gameMap.tileWidth() + offsetX : offsetX;
```

最终结果仍会经过 `rx()` 或 `ry()`，因此可以把光源、标记或附着特效绑定到角色和事件上。

## 全局导航不是移动玩家

最初的运行时编辑器只能编辑玩家附近的画面。要在地图另一端放置图片，就必须先控制玩家走过去，这会让地图装饰和游戏流程耦合在一起。

全局导航的做法不是传送玩家，而是直接修改 `$gameMap` 的显示位置：

```javascript
function centerMapAtPixel(mapX, mapY) {
    var displayX = mapX / $gameMap.tileWidth()
        - $gameMap.screenTileX() / 2;
    var displayY = mapY / $gameMap.tileHeight()
        - $gameMap.screenTileY() / 2;

    $gameMap.setDisplayPos(displayX, displayY);
}
```

导航区域使用一个独立的 Canvas 绘制：

- 整张地图的缩略范围；
- 当前视口矩形；
- 玩家位置；
- 可以换算成地图坐标的 ULDS 图层位置；
- 当前选中图层。

点击导航区域会把游戏视口移动到目标位置，拖动则连续移动。按住 `Shift` 点击时，会把当前待放置素材直接放到对应的地图像素位置。

横向和纵向滑块则直接控制 `$gameMap.displayX()` 与 `$gameMap.displayY()`，用于比缩略图更精确地移动视口。

这种实现解决了“玩家必须先走到目标位置”的问题，但编辑器关闭后仍会恢复打开前的相机位置，避免测试游戏流程被编辑操作永久改变。

## 缩小画面时为什么会出现空白区域

如果只是对 `Spriteset_Map._baseSprite` 设置缩放：

```javascript
spriteset._baseSprite.scale.set(0.5, 0.5);
```

屏幕上的地图确实会缩小，但 Tilemap 原本只准备了正常视口附近的图块。缩小到 50% 后，理论上应该看到两倍宽和两倍高的地图范围，实际 Tilemap 却没有为这些区域创建足够的渲染层，于是画面边缘会出现空白或图块缺失。

编辑器会根据目标缩放计算所需的世界尺寸：

```javascript
var worldWidth = Math.ceil(Graphics.width / zoom);
var worldHeight = Math.ceil(Graphics.height / zoom);
var requiredWidth = worldWidth + margin * 2;
var requiredHeight = worldHeight + margin * 2;
```

随后扩展 Tilemap 的 `_width`、`_height` 和 `_margin`，并重新创建渲染层。循环贴图和 Parallax 的可见尺寸也需要同步调整。

这不是单纯的 CSS 缩放，而是同时改变：

1. PIXI 地图根节点的视觉缩放；
2. RPG Maker 地图显示位置；
3. Tilemap 实际准备的渲染范围；
4. 循环贴图和远景的视口尺寸。

## 缩放过程中保持观察中心

如果只改变缩放倍率，地图会默认围绕渲染根节点的原点缩放，观察位置会向左上角偏移。

编辑器在缩放前记录当前视口中心对应的地图坐标：

```javascript
function currentCameraFocus() {
    return {
        x: $gameMap.displayX()
            + Graphics.width / $gameMap.tileWidth() / viewZoom / 2,
        y: $gameMap.displayY()
            + Graphics.height / $gameMap.tileHeight() / viewZoom / 2
    };
}
```

应用新倍率后，再反推新的地图显示起点：

```javascript
$gameMap.setDisplayPos(
    focus.x - Graphics.width / $gameMap.tileWidth() / zoom / 2,
    focus.y - Graphics.height / $gameMap.tileHeight() / zoom / 2
);
```

这样缩放前后的地图中心保持不变。

为了减少滑块缩放时的突变，我没有直接对倍率做固定步长线性插值，而是对倍率的对数进行指数平滑：

```javascript
var easing = 1 - Math.exp(-elapsed / 85);
var zoom = Math.exp(
    Math.log(viewZoom)
    + (Math.log(viewZoomTarget) - Math.log(viewZoom)) * easing
);
```

对数空间插值更接近“按比例变化”的缩放。比如从 25% 到 50% 和从 100% 到 200% 都是倍率翻倍，视觉速度会比普通加法插值更一致。

同时，编辑状态下会暂时关闭 Tilemap 的像素取整，减少连续缩放时由整数坐标取整产生的跳动。

## 按 z 值自动组织图层

ULDS 使用数值 `z` 决定图层在 Tilemap 中的大致层级。编辑器没有再建立一套与运行时无关的文件夹系统，而是直接把相同数值 `z` 的图层收归到同一个组。

例如：

```text
z = 0.5
├── ground_shadow
├── grass_overlay
└── puddle

z = 0.8
├── fog_01
└── light_mask
```

图层组可以按升序或降序显示。把图层拖入另一个数值组时，编辑器会直接修改该图层的 `z`，而不是只调整界面上的展示位置。

动态 `z` 公式无法稳定归入单一数值组，因此会作为特殊情况显示。这里没有强行把动态层级伪装成固定分组。

## 旋转后的控制框为什么容易出错

普通图片在没有旋转时，可以使用轴对齐矩形表示选择框。向右拖动控制点增加宽度，向下拖动增加高度即可。

图片旋转后，屏幕水平方向不再等于图片自身的宽度方向。继续使用鼠标的 `deltaX` 修改宽度，会出现两个问题：

1. 控制点移动方向和图片边缘方向不一致；
2. 改变尺寸后，锚点和图片位置发生偏移。

PIXI 的 `sprite.getBounds()` 返回世界空间中的轴对齐包围盒。它可以用于粗略命中，但不能直接作为旋转控制框，因为旋转后的矩形会被扩大成一个没有方向信息的外接矩形。

因此编辑器需要从 Sprite 的世界变换矩阵恢复选择框方向。

## 从 PIXI 世界矩阵计算旋转选择框

PIXI 的二维世界矩阵可以表示为：

```text
| a  c  tx |
| b  d  ty |
| 0  0   1 |
```

其中：

- `(a, b)` 表示局部 X 轴在世界空间中的方向和缩放；
- `(c, d)` 表示局部 Y 轴在世界空间中的方向和缩放；
- `(tx, ty)` 表示 Sprite 原点的世界位置。

编辑器先取得 Sprite 的局部矩形，再用 `worldTransform` 计算其左上角和两条边轴：

```javascript
var originX = transform.a * local.x
    + transform.c * local.y
    + transform.tx;
var originY = transform.b * local.x
    + transform.d * local.y
    + transform.ty;

var axisXX = transform.a * local.width;
var axisXY = transform.b * local.width;
var axisYX = transform.c * local.height;
var axisYY = transform.d * local.height;

var width = Math.sqrt(axisXX * axisXX + axisXY * axisXY);
var height = Math.sqrt(axisYX * axisYX + axisYY * axisYY);
var angle = Math.atan2(axisXY, axisXX);
```

DOM 选择框使用同样的宽度、高度和角度：

```javascript
selection.style.left = geometry.left + 'px';
selection.style.top = geometry.top + 'px';
selection.style.width = geometry.width + 'px';
selection.style.height = geometry.height + 'px';
selection.style.transform = 'rotate(' + geometry.angle + 'rad)';
```

这样八个缩放控制点会跟随图片方向，而不是停留在屏幕水平和垂直方向。

## 把鼠标位移投影到图片局部轴

开始缩放时，编辑器记录：

- 鼠标起点；
- 当前选择框宽高；
- 当前旋转角；
- 原始 `scale.x` 和 `scale.y`；
- 图片局部宽高；
- 锚点；
- 原始地图位置。

拖动过程中，先取得鼠标在屏幕上的位移：

```javascript
var pointerDeltaX = event.clientX - startClientX;
var pointerDeltaY = event.clientY - startClientY;
```

然后使用旋转矩阵的逆方向，把它投影到图片自身的 X/Y 轴：

```javascript
var cos = Math.cos(screenAngle);
var sin = Math.sin(screenAngle);

var localDeltaX = pointerDeltaX * cos + pointerDeltaY * sin;
var localDeltaY = -pointerDeltaX * sin + pointerDeltaY * cos;
```

此后，东、西、南、北控制点分别只影响对应的局部轴；角点同时影响两个轴。

```javascript
var ratioX = east
    ? (startWidth + localDeltaX) / startWidth
    : west
        ? (startWidth - localDeltaX) / startWidth
        : 1;

var ratioY = south
    ? (startHeight + localDeltaY) / startHeight
    : north
        ? (startHeight - localDeltaY) / startHeight
        : 1;
```

按住 `Shift` 拖动角点时，两个轴使用同一个倍率，实现等比缩放。

## 为什么缩放后还要修正图片位置

如果 Sprite 锚点位于中心，拖动右侧控制点增加宽度时，图片会同时向左和向右扩展。对于编辑器交互来说，用户通常希望左侧保持不动，只有右侧跟随鼠标。

因此缩放不能只修改 `scale.x` 和 `scale.y`，还要根据锚点计算位置补偿。

局部坐标中的补偿量类似：

```javascript
var localOffsetX = localWidth
    * (nextScaleX - startScaleX)
    * anchorX;
```

然后再把这个局部偏移旋转回地图坐标：

```javascript
var offsetX = localOffsetX * Math.cos(rotation)
    - localOffsetY * Math.sin(rotation);
var offsetY = localOffsetX * Math.sin(rotation)
    + localOffsetY * Math.cos(rotation);
```

最终同时更新图片的地图位置与缩放值。

这也是此前“图片旋转后不能继续改变形状”问题的根源：如果只依赖轴对齐包围盒，缩放方向和位置补偿都会在图片旋转后失效。

当前画布控制点只允许静态、正数缩放和静态旋转。镜像图层或由公式动态控制缩放、旋转的图层仍要通过参数栏修改，因为拖拽操作无法稳定决定应该如何重写用户公式。

## 旋转交互和角度连续性

旋转手柄以 Sprite 的世界原点作为中心。鼠标按下时记录初始指针角度：

```javascript
var pointerAngle = Math.atan2(
    event.clientY - pivotY,
    event.clientX - pivotX
);
```

移动时不能直接使用“当前角度减初始角度”。当角度越过 `-π` 和 `π` 边界时，直接相减会突然跳过接近一整圈。

编辑器使用下面的方式把每一帧角度差规范到连续范围：

```javascript
var delta = Math.atan2(
    Math.sin(pointerAngle - lastPointerAngle),
    Math.cos(pointerAngle - lastPointerAngle)
);
accumulated += delta;
```

按住 `Shift` 时再把结果吸附到 15°：

```javascript
var snap = Math.PI / 12;
rotation = Math.round(rotation / snap) * snap;
```

这种方式处理的是编辑器交互的连续性，不是物理旋转或动画插值。

## 把动态公式包装成可回读配置

ULDS 的一个特点是字符串属性会被编译成每帧执行的 JavaScript：

```javascript
this._updater = new Function('t', 's', 'v', code);
```

其中：

- `t` 是 Sprite 已更新的帧数；
- `s` 是 `$gameSwitches`；
- `v` 是 `$gameVariables`。

例如透明度可以由时间驱动：

```javascript
180 + 60 * Math.sin(t / 120 * Math.PI * 2)
```

如果编辑器只负责生成公式，那么用户关闭再打开编辑器后，右侧表单无法知道这个公式原本来自“周期 120、中心值 180、振幅 60”的配置。

为了解决这个问题，编辑器会把结构化配置嵌入生成的公式：

```javascript
(function () {
    var uldsEditorDynamic = {
        property: opacity,
        type: time,
        waveform: sin,
        period: 120,
        amplitude: 60,
        center: 180
    };
    return 180 + 60 * Math.sin(t / 120 * Math.PI * 2);
})()
```

重新打开参数面板时，编辑器通过标记 `uldsEditorDynamic` 提取 JSON，并恢复成表单字段。

相同思路也用于：

- 玩家、事件和跟随者绑定；
- 循环贴图滚动速度；
- 像素或图块裁切；
- 图集动画帧参数。

这是一种轻量级的双向协议：运行时只关心表达式返回值，编辑器则额外读取表达式中的配置标记。

它并不是完整的 JavaScript 抽象语法树解析器。用户手写的任意公式可以原样保存和执行，但无法保证重新还原成结构化表单。编辑器只能可靠回读自己生成的公式，以及少数能够识别的原生格式。

## 图集动画仍然是 frame 公式

动画帧没有在运行时新增专用动画组件，而是生成一个随 `t` 变化的 `frame` 属性：

```javascript
var cycle = pingPong && count > 1
    ? (count - 1) * 2
    : count;
var step = Math.floor((t - 1) / interval) % cycle;
var index = pingPong && step >= count
    ? cycle - step
    : step;

return {
    x: startX + (vertical ? 0 : index * width),
    y: startY + (vertical ? index * height : 0),
    w: width,
    h: height
};
```

编辑器提供的帧数、间隔、帧宽高、起点、纵向排列和往返播放，最终都转换成这个表达式。

优点是复用了 ULDS 已有的动态属性机制；缺点是动画、静态裁切和循环贴图共享部分底层字段，不能随意叠加。界面需要在这些模式冲突时阻止操作或要求用户确认替换。

## 撤销重做使用完整快照

编辑器没有建立细粒度 Command 类，而是直接序列化全部图层：

```javascript
function snapshot() {
    return JSON.stringify(this.layers);
}
```

每次完成一次有效操作后，把新快照加入历史：

```javascript
this.history = this.history.slice(0, this.historyIndex + 1);
this.history.push(snapshot);

if (this.history.length > 100) {
    this.history.shift();
}
```

撤销时解析目标快照并重建当前图层。

这种实现适合当前规模，原因是单张 RPG Maker 地图中的 ULDS 图层通常不会达到非常大的数量。它也让拖拽、旋转、删除、复制和参数编辑共享同一套历史机制。

代价是每个历史状态都保存完整数组。图层数量或公式长度显著增加后，序列化成本和内存占用会线性增长。更大型的编辑器应该使用命令记录、结构共享或增量补丁，而不是继续扩大快照上限。

## 写回地图前检查外部修改

编辑器打开时会保存当前地图 `note` 的原始文本。保存时重新从磁盘读取 `MapXXX.json`，并检查两个条件：

1. 磁盘中的 `<ulds>` JSON 仍然可以解析；
2. 磁盘 `note` 和打开编辑器时保存的版本一致。

如果 RPG Maker 编辑器或其他工具在此期间修改了地图，保存会被拒绝：

```javascript
if (diskNote !== this.originalNote) {
    throw new Error(
        '地图 note 在编辑器打开后被外部修改，请关闭并重新打开编辑器。'
    );
}
```

这样做不能合并两个工具的并发改动，但至少避免静默覆盖。

实际写入流程为：

```text
读取并解析 MapXXX.json
        ↓
移除旧 <ulds> 块并序列化当前图层
        ↓
复制 MapXXX.json.ulds.bak
        ↓
写入 MapXXX.json.ulds.tmp
        ↓
覆盖原地图文件
        ↓
刷新 $dataMap metadata 与当前 ULDS Sprite
```

临时文件可以减少 JSON 序列化过程中直接破坏原文件的风险，备份则提供一次手工恢复机会。

不过当前实现最后使用的是临时文件复制覆盖，而不是同一文件系统中的原子重命名，因此不能把它描述成严格的事务保存。固定的 `.ulds.bak` 也只保留最近一次备份，不是版本历史。

## 右键菜单和快捷操作不是主要难点

编辑器还包含素材搜索、鼠标滚轮列表、侧栏隐藏、全局预览、图层折叠、右键菜单、复制、删除、翻转、旋转 90° 和重置变换等功能。

这些功能对实际使用有帮助，但实现上主要是 DOM 事件和状态更新，不是这个项目中最值得展开的技术部分。把文章写成完整功能清单，会掩盖坐标变换、相机渲染范围和公式回读这些真正需要推导的内容。

## 当前实现没有解决什么

### 1. 文件结构仍然过于集中

`ULDS_Editor.js` 超过 3000 行，CSS、DOM 模板、编辑器状态、相机逻辑、变换算法、公式生成和文件保存主要集中在一个 `Editor` 对象中。

这降低了早期实现成本，但继续增加功能后，修改一个模块容易影响其他交互。更合理的后续拆分至少应包括：

- `EditorState`：图层、选择和历史；
- `StageTransformController`：拖动、缩放和旋转；
- `MapCameraController`：导航与缩放；
- `FormulaCodec`：公式生成和回读；
- `MapRepository`：地图读取、校验和写入；
- 独立的样式文件与 DOM 视图层。

### 2. 相机实现依赖引擎私有字段

为了让 25% 缩放真正显示更大地图范围，当前实现会访问 `_baseSprite`、`_tilemap`、`_width`、`_height`、`_margin` 和 `_createLayers()`。

这些字段在 RPG Maker MV 当前运行时中可用，但不是稳定的公共插件 API。其他 Tilemap、分辨率或相机插件也可能修改同一批对象。它适合作为指定项目的内部工具，不应该未经兼容测试就宣称适用于所有 MV 项目。

### 3. 图片命中仍是包围盒级别

选择图片时使用 `sprite.getBounds().contains()`。对于透明区域很多或旋转角度较大的图片，鼠标可能点击到外接矩形的空白部分仍然选中图层。

更精确的方案可以把鼠标点转换到 Sprite 局部坐标后检查局部矩形，或者进一步读取纹理 Alpha 做像素级命中。但后者会增加纹理读取和性能成本。

### 4. 动态公式不是安全沙箱

ULDS 使用 `new Function` 执行地图中的表达式。编辑器只在本地测试模式运行，并假设项目文件可信。

如果公式来自不可信输入，它可以访问游戏和浏览器运行环境，因此不能把这套机制直接用于用户上传内容或联网脚本平台。

### 5. 缺少针对编辑器的自动化测试

当前可以进行 JavaScript 语法检查和人工 Playtest，但没有覆盖以下行为的自动化测试：

- 公式生成后能否完整回读；
- 旋转角度跨越 `π` 时是否连续；
- 不同锚点下缩放补偿是否正确；
- 循环地图中的全局导航边界；
- 保存冲突和临时文件清理；
- 与不同 Tilemap 实现的兼容性。

其中公式编解码适合直接写单元测试；PIXI 变换和地图相机则更适合增加可重复的集成测试场景。

## 什么时候适合使用这种方案

运行时可视化编辑适合以下情况：

- 项目已经存在一套可以运行的图片图层格式；
- 图层效果依赖实际地图、角色、变量或其他运行时状态；
- 工具主要服务于一个确定的 RPG Maker MV 项目；
- 希望继续兼容现有地图 `note`，不迁移数据格式；
- 可以接受编辑工具只在 Playtest 中使用。

如果目标是制作面向多个项目发布的通用编辑器，或者需要多人协作、资源数据库、版本合并和插件扩展，那么独立桌面工具会比继续扩大一个运行时 DOM 覆盖层更合适。

## 复盘：真正困难的是坐标和状态边界

这个编辑器表面上是在游戏画面上增加素材栏、参数栏和图层列表，但实际工作量主要集中在几条边界上：

- DOM 客户端坐标与 PIXI 世界坐标之间的转换；
- 地图坐标、屏幕坐标和不同滚动速度之间的转换；
- 相机缩放和 Tilemap 实际渲染范围之间的同步；
- 旋转后的局部轴与鼠标屏幕位移之间的转换；
- 可视化表单和可执行动态公式之间的双向转换；
- 内存中的编辑状态和磁盘地图文件之间的同步。

素材列表、按钮和右键菜单决定工具是否方便使用，但这些边界决定工具是否能够稳定地给出正确结果。

当前实现已经能够服务于项目内的 ULDS 图层制作，但它仍然是一个与 RPG Maker MV 内部结构紧密耦合的专用工具，而不是通用图形编辑器。后续如果继续维护，优先级应该是拆分模块、补充公式与变换测试，以及减少对 Tilemap 私有字段的直接依赖，而不是继续增加更多按钮。
