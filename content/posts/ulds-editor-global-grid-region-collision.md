+++
date = '2026-08-08T03:50:00+08:00'
draft = false
title = '把 ULDS Editor 的碰撞从图层提升到地图：全局网格、多碰撞体与 Region 安全烘焙'
description = '记录 ULDS Editor 如何把图片局部碰撞重构为地图级 Region 编辑：常驻全局网格、多碰撞体分组、拖动绘制、旧数据迁移、RLE 恢复记录与 YEP_RegionRestrictions 对接。'
tags = ['RPG Maker MV', 'JavaScript', 'PIXI.js', 'NW.js', 'YEP', '碰撞系统', '编辑器工具']
categories = ['游戏开发', '工具开发']
slug = 'ulds-editor-global-grid-region-collision'

------

![ULDS Editor 可视化图层编辑界面](/images/ulds-editor-global-grid-region-collision/8c59160b387191a77b9dc90bc0e3bf53.png)

上一篇文章记录了我如何把 ULDS 的 JSON 图层配置改造成运行在 RPG Maker MV Playtest 中的可视化编辑器，重点是全地图相机、图层组织、旋转缩放、动态公式和地图文件写回。本文继续记录后续增加的地图网格与碰撞编辑功能。

上一篇文章： [在 RPG Maker MV 测试游戏中实现可视化图层编辑器](/posts/rpg-maker-mv-ulds-runtime-layer-editor/)

这次功能迭代最初看起来只是“给图片增加碰撞体”，但真正需要确定的是碰撞数据应该属于谁：ULDS 图片图层，还是 RPG Maker 地图本身。

最终实现没有把碰撞继续绑定在某个 ULDS 图层上，而是把它提升为独立的地图级数据。ULDS 负责场景图片，地图碰撞编辑器负责 Region。两者共享同一个运行时视口和全局网格，但不再共享数据生命周期。

## 为什么图片局部网格不适合 Region 碰撞

第一版方案把每张 ULDS 图片按 48 像素划分为局部网格，再把选中的图片格经过位移、锚点、缩放和旋转转换到地图空间。

这个方案的问题不是矩阵计算错误，而是存在两次离散化：

```text
图片局部格
    ↓ Sprite 变换
地图空间中的旋转四边形
    ↓ 与地图图块求交
RPG Maker Region 格
```

当图片没有对齐地图格，或者图片经过缩放和旋转后，一个局部格可能同时覆盖多个地图格。最终写入 Region 时，碰撞范围会比用户看到的局部选择更大。

即使把“可选择范围”限制在图片矩形内部，也没有解决根本问题。RPG Maker MV 的 Region 最终仍然只能表达地图格，图片局部格只是一个额外的中间坐标系统。

因此碰撞编辑应当直接在目标坐标系统中完成：

```text
用户选择地图格 x, y
    ↓
直接写入 Region 层 x, y
```

这条路径没有第二次栅格化，也不会因为图片偏移而自动扩张到相邻格。

## 全局网格属于地图视口，不属于图片

碰撞坐标改成地图格后，全局网格也不再依赖当前选中的 ULDS 图层。只要按 `F10` 打开编辑器，网格就会始终显示，用于对齐图片、估算距离和检查地图坐标。

进入碰撞模式只改变交互含义：

- 普通模式中，左键仍然用于选择、放置和移动 ULDS 图层；
- 碰撞模式中，左键用于给当前碰撞体添加或擦除地图格；
- 中键或 `Space + 左键` 始终用于移动编辑视口；
- `F9` 全局预览仍然隐藏网格和编辑器界面，用于检查最终画面。

网格位置从地图显示坐标计算：

```javascript
function screenPoint(tileX, tileY) {
    return {
        x: $gameMap.adjustX(tileX)
            * $gameMap.tileWidth()
            * viewZoom,
        y: $gameMap.adjustY(tileY)
            * $gameMap.tileHeight()
            * viewZoom
    };
}
```

这里不能只使用 `tileX * tileWidth`，因为编辑器可以独立移动相机并缩放地图视口。`adjustX()`、`adjustY()` 和 `viewZoom` 必须同时参与换算。

为了避免大地图每帧绘制全部网格线，实际代码只计算当前视口附近的行列范围：

```javascript
var visibleColumns = Graphics.width / tileWidth / viewZoom;
var startX = Math.floor($gameMap.displayX()) - 1;
var endX = Math.ceil($gameMap.displayX() + visibleColumns) + 1;
```

多出的 1 格用于覆盖视口边缘的抗锯齿和小数相机位置。

## 固定颜色加描边比自动反色稳定

网格曾尝试根据图片范围自动在白色和黑色之间切换。这个方案在浅色和深色素材上都能获得局部对比，但图片边缘会出现网格颜色跳变；旋转图片和透明图片较多时，整体视觉也不稳定。

最终改为更常见的双层描边：先画粗黑线，再画细的浅色线。

```javascript
function strokeGrid(style, width) {
    context.save();
    context.globalAlpha = 1;
    context.globalCompositeOperation = 'source-over';
    context.beginPath();
    appendGridPath();
    context.strokeStyle = style;
    context.lineWidth = width;
    context.stroke();
    context.restore();
}

if (collisionEditMode) {
    strokeGrid('rgba(0,0,0,.72)', 4);
    strokeGrid('rgba(220,240,248,.88)', 2);
} else {
    strokeGrid('rgba(0,0,0,.45)', 3);
    strokeGrid('rgba(220,240,248,.75)', 1);
}
```

两次描边使用独立路径和固定的 `source-over` 合成，避免前一次绘制状态影响后一次。普通模式保持较细，碰撞模式加粗，但不再根据图片逐段反色。

## 覆盖层必须裁剪在中央地图视口

游戏 Canvas 位于整个窗口下方，而 ULDS Editor 的素材栏和属性栏是 DOM 覆盖层。如果网格 Canvas 直接覆盖游戏 Canvas，它会在视觉上穿过左右面板。

解决方式不是缩小地图 Canvas，而是在网格 Canvas 中按照中央舞台和游戏 Canvas 的交集建立裁剪区域：

```javascript
var clipLeft = Math.max(canvasRect.left, stageRect.left);
var clipTop = Math.max(canvasRect.top, stageRect.top);
var clipRight = Math.min(canvasRect.right, stageRect.right);
var clipBottom = Math.min(canvasRect.bottom, stageRect.bottom);

context.beginPath();
context.rect(clipX, clipY, clipWidth, clipHeight);
context.clip();
```

这样素材栏、参数栏、顶部工具栏和底部状态栏都不会被网格遮挡。侧栏隐藏后，中央舞台范围变大，裁剪范围也会自动更新。

## 地图级碰撞数据结构

ULDS 图层仍然保存在多个 `<ulds>JSON</ulds>` 块中，地图碰撞则使用独立的 note 区块。简化后的结构如下：

```text
<ULDS Editor Map Collision>
{
  "enabled": true,
  "activeBodyId": "body-wall",
  "bodies": [
    {
      "id": "body-wall",
      "name": "建筑阻挡",
      "enabled": true,
      "target": "all",
      "regionId": 252,
      "color": "#dc5b4c",
      "cells": ["12,8", "13,8", "12,9"]
    }
  ]
}
</ULDS Editor Map Collision>
```

每个碰撞体保存：

- 稳定 ID 和显示名称；
- 编辑器中的区分颜色；
- 是否启用；
- 阻挡目标；
- 对应 Region ID；
- 选中的地图格坐标。

碰撞体与图片没有引用关系。删除、隐藏或替换某个 ULDS 图层，不会同时删除地图碰撞。

## 多碰撞体如何映射到 YEP RegionRestrictions

当前碰撞体支持三种阻挡目标：

| target | 生成的 YEP 规则 | 默认 Region |
| --- | --- | --- |
| `player` | `Player Restrict Region` | 250 |
| `event` | `Event Restrict Region` | 251 |
| `all` | `All Restrict Region` | 252 |

保存地图时，编辑器会根据当前启用的碰撞体生成地图备注：

```text
<ULDS Editor Collision Regions>
<Player Restrict Region: 250>
<Event Restrict Region: 251>
<All Restrict Region: 252>
</ULDS Editor Collision Regions>
```

YEP 的作用不是提供像素级物理碰撞，而是把 Region ID 插入玩家和事件的地图通行判断。原版 Tileset 通行仍然负责基础墙壁和地形，Region 更适合地图特有的空气墙、NPC 活动边界和剧情限制。

需要明确的是，一张 RPG Maker 地图的每个格子只能保存一个 Region ID。因此多个碰撞体可以拥有不同颜色和用途，但如果它们选择了同一个地图格，列表中靠后的碰撞体会覆盖靠前的碰撞体。编辑器提供前移和后移操作来控制这个顺序。

## 选中格直接写入 Region 数据层

RPG Maker MV 的地图数据是一维数组。Region 位于第 5 层，其索引计算为：

```javascript
var dataIndex = (5 * mapHeight + tileY) * mapWidth + tileX;
diskMap.data[dataIndex] = regionId;
```

地图碰撞编辑器保存的 `cells` 已经是 `tileX,tileY`，因此写入过程不再需要 Sprite 矩阵、图片锚点或多边形求交。

鼠标点击也直接换算成地图格：

```javascript
var mapX = $gameMap.displayX() * tileWidth
    + canvasPoint.x / viewZoom;
var mapY = $gameMap.displayY() * tileHeight
    + canvasPoint.y / viewZoom;

var tileX = Math.floor(mapX / tileWidth);
var tileY = Math.floor(mapY / tileHeight);
```

拖动绘制时，编辑器会在前一个格和当前格之间插值，补齐快速移动可能跳过的中间格。一次拖动只在鼠标释放时提交一条历史记录，而不是把每个 `mousemove` 都压入撤销栈。

## 为什么保存恢复记录，而不是直接清空 Region

如果移动或删除碰撞体后直接把旧格设为 0，可能会破坏地图原本手工绘制的 Region。编辑器需要记住每个写入格此前的值。

每批烘焙记录包含目标 Region 和压缩后的恢复范围：

```json
{
  "regionId": 252,
  "ranges": [
    [12913, 3, 0],
    [12963, 2, 7]
  ]
}
```

数组含义是：

```text
[起始 dataIndex, 连续长度, 原 Region ID]
```

保存新结果前，编辑器按相反顺序恢复旧批次，再按当前碰撞体顺序重新写入。只有磁盘中的值仍等于上一次写入的 Region ID 时才执行恢复，减少覆盖外部修改的风险。

连续格使用简单的游程编码压缩。否则一个覆盖数千格的碰撞体会把完整恢复列表写入地图 note，导致高级 JSON、撤销快照和地图文件迅速膨胀。

## 旧图层碰撞如何迁移

在确定地图级架构之前，碰撞数据曾保存在图层的 `_uldsEditorCollision` 字段中。直接删除这些字段会丢失已经编辑的范围。

迁移时优先读取旧碰撞体的 `baked.ranges`：

1. 从 Region 层的 `dataIndex` 减去前五层偏移；
2. 根据地图宽度还原 `tileX` 和 `tileY`；
3. 生成新的地图级 `cells`；
4. 保留原名称、颜色、阻挡目标和 Region ID；
5. 删除图层中的旧碰撞字段；
6. 下一次保存时先恢复旧 Region，再按地图级碰撞重新烘焙。

使用已烘焙的地图索引迁移，比重新读取图片局部格并再次求交更可靠，因为它代表游戏实际使用过的 Region 结果。

如果旧碰撞既没有地图格坐标，也没有烘焙恢复记录，编辑器不会猜测转换结果，而是保留旧字段并拒绝保存，要求先处理无法迁移的数据。

## 撤销快照同时包含图层和地图碰撞

原来的撤销快照只包含 ULDS 图层数组。碰撞独立后，快照改为：

```javascript
function snapshot() {
    return JSON.stringify({
        layers: layers,
        mapCollision: mapCollision
    });
}
```

这样图层移动、碰撞格绘制、碰撞体排序和 Region 设置都进入同一个历史时间线。编辑相机位置不属于地图内容，不进入快照。`activeBodyId` 会作为编辑器元数据保存，便于下次打开时恢复当前碰撞体，但它不参与 Region 烘焙结果。

## 地图文件写回与并发修改

保存流程现在同时处理三类内容：

```text
MapXXX.json
├─ 原有地图和事件数据
├─ <ulds> 图层块
├─ <ULDS Editor Map Collision> 碰撞 JSON
└─ YEP Restrict Region 标签
```

实际流程为：

```text
重新读取磁盘地图
        ↓
检查 note 是否被外部修改
        ↓
恢复旧图层碰撞与旧地图碰撞写入的 Region
        ↓
烘焙当前地图碰撞
        ↓
分别序列化 ULDS、地图碰撞和 YEP 标签
        ↓
生成 .ulds.bak
        ↓
写入 .ulds.tmp 并覆盖地图
```

开发过程中实际遇到过 RPG Maker 编辑器重新保存地图后覆盖外部 `note` 的情况。固定备份保留了上一份 ULDS 数据，但它不能替代版本控制，也不能自动合并两个编辑器的并发修改。

因此当前策略是检测到 `note` 变化就拒绝静默覆盖。需要恢复备份时，应先比较当前地图和 `.ulds.bak` 中的事件、图块与 Region 差异，而不是直接复制整个文件。

## 验证范围

这次修改执行了 JavaScript 语法检查，并使用 Node.js `vm` 构造最小 RPG Maker 运行环境进行回归检查，覆盖：

- 没有选中 ULDS 图层时绘制地图碰撞；
- 普通模式仍然显示全局网格；
- 网格只裁剪在中央地图视口；
- 点击和拖动生成正确的地图格坐标；
- 添加、擦除、撤销和重做；
- 多碰撞体重叠时的优先级；
- 旧 Region 的反向恢复；
- 地图碰撞 JSON 的序列化和重新解析；
- `Player`、`Event` 和 `All` Restrict 的 YEP 通行结果；
- 普通模式和碰撞模式的双层描边颜色、线宽与合成方式。

这些检查验证的是坐标、数据和保存算法，不等同于不同 MV 插件组合下的完整兼容测试。最终仍需要在真实 Playtest 中检查 Tilemap、相机插件、循环地图和不同分辨率。

## 当前限制

地图级碰撞系统仍然有明确边界：

- 碰撞精度固定为地图格，不支持像素、多边形或斜面碰撞；
- 每格只有一个 Region ID，重叠碰撞体不能同时保存多个规则；
- `Event Restrict` 默认影响所有事件，不能直接区分某个 NPC；
- YEP 的 Allow Region 可以覆盖原版通行规则，配置错误可能造成穿墙；
- 网格和碰撞界面仍然集中在体积较大的 `ULDS_Editor.js` 中；
- 固定 `.ulds.bak` 只保留最近一次备份；
- 编辑器依赖 RPG Maker MV、NW.js、PIXI 和当前地图内部结构。

## 复盘：在目标坐标系中编辑最终数据

这次重构最重要的结论不是增加了多少按钮，而是应该尽量在最终数据所属的坐标系中进行编辑。

ULDS 图片适合使用像素坐标、锚点和 Sprite 变换，因为最终结果是 PIXI 图层。YEP Region 适合直接使用地图格坐标，因为最终结果是地图 Region。把两种数据强行绑定，会增加转换、生命周期和保存恢复的复杂度。

全局网格仍然可以帮助 ULDS 图片对齐地图，但碰撞数据不再属于图片。这样两套工具共享视觉参照，却保持独立的数据模型和保存边界。
