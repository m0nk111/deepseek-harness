# Agent Note: Pending steering bubbles gain a remove action

Status: implemented

[English](2026-08-23-web-pending-steering-remove-action.md) | 中文

## 问题

轮次运行期间，通过输入框（或某条排队行的 Steer 操作）发出的插话消息，会以宿主权威的待处理 steering 气泡渲染在会话流末尾。QueueDock 里的排队行提供编辑、删除与插话发送操作，但待处理 steering 气泡只渲染复制。一旦插话进入 next-step 窗口、运行中的轮次尚未接纳它，就没有办法撤消——用户只能任其落地，再作为持久历史处理。

宿主面早已支持该操作：`conversation.updateQueue` 配合 `{ kind: 'remove' }` 与处理排队项一样定位 `next-step` 单次入队项，将其从待处理集合中退役。缺口纯粹在 Web 呈现层。

## 决策

`PendingSteeringBubble` 现在在复制旁渲染一个删除（垃圾桶）操作。点击会以 `{ kind: 'remove' }` 调用会话级 `updateQueue` 动词，由宿主通过聊天视图的注入面接线。拒绝（插话已被接纳、其行已离开待处理集合）会被静默吞掉：权威的 `session/queue` 快照会自行校调气泡，因此没有需要呈现的内容。

该操作只存在于待处理、接纳前的气泡上。持久的 user 或 steering 气泡（由 `user/message` 渲染）保留复制与时钟，不带删除操作，这与已发送消息不可删除的既有规则一致。

新增 locale 键（`queue.removeSteering`）使该操作与排队行删除文案（"删除排队消息"）区分开来，因为两个表面针对不同的对象。

## 考虑的替代方案

- **复用排队行文案。** 排队行的文案描述的是排队消息，而待处理气泡是插话；对象不同，文案也应不同。
- **拒绝时提示而非吞掉。** 权威快照在接纳时就会移除该行，为良性收敛呈现错误只会给一个自行化解的状态添噪声。
- **把删除扩展到持久气泡。** 已发送消息是持久模型历史；删除它们需要并不存在的宿主删除语义。该入口只保留在它所对应的易失待处理单次入队项上。

## 后果

轮次中途的插话可以在到达模型之前被撤消，与用户对排队消息已有的控制对齐。Web 视图新增一个注入动词（`updateQueue`）并传给待处理气泡；无需改动宿主或会话日志。

## 测试

聊天视图组件测试新增一条待处理插话，断言点击删除操作会以该单次入队项 id 和 `{ kind: 'remove' }` 调用 `updateQueue`，而持久 user 气泡永远不会渲染该操作。`apply-inject` 测试断言注入的聊天视图 `updateQueue` 到达会话面。两处 `steering` web e2e 的 `mid-steer` 金样重新生成以包含新按钮；`settled` 金样不变，因为持久气泡不带删除操作。
