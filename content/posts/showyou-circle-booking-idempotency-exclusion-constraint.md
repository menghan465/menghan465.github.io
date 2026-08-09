+++
date = '2026-08-09T21:00:00+08:00'
draft = false
title = '宿优圈预订一致性：幂等键、Serializable 事务与 PostgreSQL 日期排他约束'
description = '以宿优圈的民宿预订为例，说明如何用服务端重算价格、幂等键、Serializable 事务和 PostgreSQL EXCLUDE 约束处理重复提交与日期重叠，并明确当前单资源模型的边界。'
tags = ['PostgreSQL', '事务', '幂等性', 'EXCLUDE', '预订系统', 'Next.js']
categories = ['后端开发', '数据库']
slug = 'showyou-circle-booking-idempotency-exclusion-constraint'
+++

预订系统里，“按钮点一次只创建一条订单”和“两个请求不能同时占用同一房源”是两个问题。前者是幂等性，后者是资源冲突控制。它们经常被放在一起讨论，但不能用一个唯一索引解决全部问题。

宿优圈当前的实现使用了幂等键、Serializable 事务和 PostgreSQL `EXCLUDE` 排他约束。这里先纠正一个容易混淆的说法：代码并不是手写 `SELECT ... FOR UPDATE` 的“排他锁防超卖”，真正负责日期冲突裁决的是数据库排他约束。Serializable 是额外的事务一致性保护，不能替代日期约束。

## 先定义预订模型

当前模型把一个可预订的 `HomestayUnit` 当作一个具体资源。订单保存入住日期、退房日期、入住人数、房价快照和总价快照：

```text
Booking
  ├─ idempotencyKey       唯一幂等键
  ├─ unitId               具体房型/资源
  ├─ checkIn              PostgreSQL date
  ├─ checkOut             PostgreSQL date
  ├─ nights               服务端计算
  ├─ unitPriceSnapshot    下单时单价
  ├─ totalPriceSnapshot   下单时总价
  └─ status               PENDING / CONFIRMED / COMPLETED / CANCELLED
```

日期被当作 date-only 值处理，而不是带时区的时间戳。当前规则包括：入住不能早于当天，退房必须晚于入住，单次预订为 1 到 30 晚，入住人数为 1 到 12 人。日期解析和字段校验由 Zod schema 与领域函数共同完成，不能只依赖页面上的 `min`、`max` 属性。

## 第一层：客户端生成幂等键

预订表单在组件生命周期内生成一个 UUID：

```tsx
const [idempotencyKey] = useState(() => crypto.randomUUID());
```

这个 key 会作为隐藏字段随表单提交，提交按钮在 Server Action 执行期间被禁用。它可以覆盖几种常见情况：用户双击提交、浏览器重复发送请求、网络超时后客户端重试同一个请求。

但按钮禁用只是用户体验，不是并发安全措施。用户可以刷新页面、打开多个标签页，或者直接构造 HTTP 请求。因此最终判断必须在服务端和数据库完成。

当前实现还明确提醒：订单金额由服务端依据实时房价重新计算，前端展示的价格不能作为信任来源。这一点同时防止了简单的价格篡改和“页面显示价格与落单价格不一致却无法追溯”的问题。

## 第二层：服务端不信任 userId 和价格

Server Action 先通过认证上下文获得当前用户 ID，不接受表单中的 `userId`：

```text
当前会话
  → requireUserId()
  → 解析表单字段
  → createBooking(currentUserId, input)
```

预订 service 会先解析日期并计算晚数，再把读取实时房型、检查幂等键和创建订单放入一个 Serializable 事务：

```ts
const checkIn = parseDateOnly(input.checkIn);
const checkOut = parseDateOnly(input.checkOut);
const nights = assertBookableDates(checkIn, checkOut);

await prisma.$transaction(async (tx) => {
  const existing = await tx.booking.findUnique({
    where: { idempotencyKey: input.idempotencyKey },
  });

  if (existing && existing.userId !== userId) {
    throw new BookingDomainError('INVALID_STATE', '幂等请求不属于当前用户');
  }

  if (existing) return existing;

  const unit = await tx.homestayUnit.findUnique(/* 读取数据库状态 */);
  const unitPrice = new Prisma.Decimal(unit.pricePerNight);
  const totalPrice = unitPrice.mul(nights);

  return tx.booking.create(/* 写入服务端计算的快照 */);
}, {
  isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
  maxWait: 5_000,
  timeout: 10_000,
});
```

实际代码会先校验输入和日期，再进入最多 3 次的事务尝试。事务内依次完成：

1. 按 `idempotencyKey` 查找历史订单；
2. 如果 key 已存在但订单属于其他用户，拒绝请求；
3. 如果 key 属于当前用户，直接返回原订单；
4. 读取房型和民宿的实时状态；
5. 校验入住人数不超过房型容量；
6. 从数据库读取当前单价并计算晚数和总价；
7. 保存单价、总价和币种快照。

这里的价格快照很重要。房价可能在订单创建后发生变化，但历史订单仍应保留当时的计价依据。订单详情不能每次都用当前房价重新计算，否则会出现已创建订单金额漂移。

## 第三层：数据库唯一约束处理重复请求

`idempotencyKey` 在 Prisma schema 中是唯一字段，订单 `reference` 也具有唯一约束。两个相同 key 的请求几乎同时进入事务时，应用层的“先查再写”可能都看不到订单，最终由数据库唯一约束决定谁成功。

另一个请求会收到 Prisma `P2002`。service 捕获这个错误后，重新按幂等键查询订单，并且只有订单确实属于当前用户时才返回它：

```text
请求 A：查无订单 → create 成功
请求 B：查无订单 → create 触发 P2002
请求 B：按幂等键重查
  ├─ 属于当前用户 → 返回 A 创建的订单
  └─ 属于其他用户 → 不泄露订单
```

幂等性因此不是“前端只发一次”，而是“相同业务请求重复执行时，最终只产生一个可返回的业务结果”。key 的归属检查也避免了仅凭一个猜到的 key 读取其他用户订单。

有一个当前实现需要继续注意的边界：key 在表单组件生命周期内固定。如果成功后仍允许用户在同一个表单上继续创建另一笔预订，就必须在成功后重置 key；否则下一次提交会被正确地当成上一笔订单的重试。这不是数据库错误，而是幂等键生命周期设计问题。

## 日期冲突：不是先查可用再插入

迁移脚本为 PostgreSQL 增加了 `btree_gist` 支持，并在 `bookings` 表上建立排他约束：

```sql
ALTER TABLE "bookings"
  ADD CONSTRAINT "bookings_no_active_overlap"
  EXCLUDE USING gist (
    "unitId" WITH =,
    daterange("checkIn", "checkOut", '[)') WITH &&
  )
  WHERE ("status" IN ('PENDING', 'CONFIRMED'));
```

它表达的不是“查询结果为空时才允许插入”，而是一个数据库不变量：同一个 `unitId` 上，两个活动订单的日期范围不能重叠。

`daterange(checkIn, checkOut, '[)')` 使用左闭右开区间：入住日包含，退房日不包含。因此：

```text
订单 A：2026-08-10 ～ 2026-08-12
订单 B：2026-08-12 ～ 2026-08-14
```

两笔订单在 2026 年 8 月 12 日完成交接，不被认定为重叠；如果订单 B 从 2026-08-11 入住，则会触发排他约束。

约束只覆盖 `PENDING` 和 `CONFIRMED`。已取消订单不再占用房态，已完成订单也不再参与未来预订的冲突判断。数据库另外检查退房晚于入住、晚数大于零、入住人数大于零以及价格快照非负。

## 异常转换和事务重试

数据库抛出的错误不应该原样展示给用户。service 会识别排他约束名或 PostgreSQL `23P01`，转换成 `DATE_UNAVAILABLE`；页面可以据此提示“所选日期已被预订”，而不是展示 SQL 错误。

Serializable 事务可能因为并发访问出现序列化失败 `40001`，也可能出现死锁 `40P01`。当前实现最多重试两次，退避时间为 25ms、50ms：

```text
第一次失败 → 等待 25ms → 第二次事务
第二次失败 → 等待 50ms → 第三次事务
仍然失败   → 返回错误
```

重试只针对明确的瞬态错误，不能把所有异常都无限重试。数据库不可用、参数非法或日期冲突都不应该通过盲目重试来掩盖。

这套设计中，Serializable 提供的是事务级的并发一致性，`EXCLUDE` 提供的是日期重叠的最终数据库约束。即使应用层未来增加“检查可用性”的预查询，创建订单时仍必须保留排他约束；预查询只能改善用户体验，不能替代最终写入时的裁决。

## 取消订单也要防并发覆盖

取消流程先检查订单归属和当前状态，再使用带条件的 `updateMany`：

```ts
await prisma.booking.updateMany({
  where: {
    id: bookingId,
    userId,
    status: { in: ['PENDING', 'CONFIRMED'] },
  },
  data: {
    status: 'CANCELLED',
    cancelledAt: new Date(),
  },
});
```

更新数量不是 1，就认为订单状态已经被其他操作改变，要求客户端刷新后重试。这样不会因为“先读到可取消，再无条件更新”而覆盖并发确认、完成或再次取消的结果。

## 当前方案能保证什么，不能保证什么

在当前“一个 `unitId` 对应一个可独立占用资源”的模型下，这套实现可以做到：

- 同一个幂等 key 不重复创建订单；
- 不接受客户端传入的用户身份和价格作为最终依据；
- 同一资源的活动订单日期不能重叠；
- 取消订单只影响当前用户拥有且仍可取消的订单；
- 对部分并发事务失败进行有限重试。

但它不能直接解决“一个房型有 10 间库存”的聚合库存问题。若 `unitId` 代表房型而不是具体房间，单条排他约束会把整个房型当成只有一份库存，结果是过度保守而不是正确表达库存。支持库存数量需要单独的库存表、按日期扣减的库存模型，或把每个实际房间建模为独立资源并分配资源。

此外，当前项目尚未完成真实 PostgreSQL 并发压测，也没有在生产环境接入支付、超时释放、库存补偿、消息队列或后台人工确认。更准确的说法不是“完全杜绝超卖”，而是：在当前单资源预订模型下，由数据库拒绝活动订单的日期重叠，并通过幂等机制消化重复提交。

## 小结

预订一致性的实现可以压缩成三条边界：

```text
幂等键       → 解决同一请求重复执行
服务端重算   → 防止身份、日期、价格被客户端直接信任
EXCLUDE 约束 → 拒绝不同请求占用同一资源的重叠日期
```

Serializable 事务和有限重试让并发行为更可控，但没有改变这些基本职责。对一个大学项目来说，这种拆分比“加了一个锁所以不会超卖”更准确，也更容易在代码审查和面试追问中说明每个机制究竟解决什么问题。
