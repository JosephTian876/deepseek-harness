# Agent Note: 手动排序把未保存的 Session 排在前面

Status: implemented

[English](2026-11-02-manual-order-new-sessions.md) | 中文

## Problem

浏览器的「手动排序」是一串保存下来的 Session id 序列，在用户拖拽行、置顶行或确定新 Session 的位置时写入。它是一份快照：此后创建的 Session——来自其他浏览器、其他标签页或一次 fork——都不在其中。`reconcileManualOrder()` 过去把这些缺失成员排在整段保存序列之后，普通成员按最近更新时间排列，因此它们只在所有已保存 Session 之后才渲染。这在实践中等于不可见：保留长历史的 Workspace 在展开到五行的折叠线之前只显示最前面的行，于是在其他设备创建的 Session 会落到数百行之后，看起来像是丢了。报告中的现象正是：每次新开客户端时，一个仍在使用的 Session 出现在很久以前 Session 的位置上。

## Decision

**没有保存位置的成员排在保存序列之前，越新越靠前。** 输出顺序依次是：按置顶数组顺序排列的缺失置顶成员、按最近更新时间排列的缺失普通成员、保存的相对位置、最后是缺失的归档成员。已保存成员保留各自的相对顺序和槽位；已在保存序列中的成员即使同时被置顶也留在原槽位。补齐只在内存中完成：推导顺序绝不写入顺序，因此保存序列仍只记录拖拽、置顶或新 Session 位置写入所保存的内容，而这些写入继续记录完整结果。

## Alternatives considered

**在成员缺失时把「手动排序」切到「最近更新」。** 否决：这会丢弃用户已经搭建的布局，并在未经同意的情况下更换所选模式。

**在读取时持久化补齐后的顺序。** 否决：一次读取就会写入浏览器本地存储，与显式保存的顺序相矛盾，并会让某个客户端的补齐对所有读取方永久生效。

**继续把缺失成员追加在保存块之后。** 否决：这正是被报告的缺陷——在长历史下新行始终藏在折叠线之后、够不到。

## Consequences

- 由其他浏览器、其他标签页或一次 fork 创建的 Session 会在每个新开客户端的「手动排序」顶部出现，折叠线内的可见区域已经能看到它。
- 拖拽和置顶写入仍然记录完整顺序，因此用户可以把新行拖到想要的位置，且该选择会保留。
- 补齐生效时行会移动一次：没有保存位置的既有 Session 会从末尾跳到保存块之前。
- 缺失的归档成员仍排在列表最后，缺失的置顶成员仍排在最前；已保存成员的相对顺序和槽位不变。

## Testing

- `packages/client/ui-workspace/tests/tree.client.spec.ts` —— `leads a stale saved order with Sessions created after it was saved` 覆盖较新 Session 按最近更新时间领先、保存的相对顺序、新置顶成员领先、新归档成员留在最后、没有摘要的成员以及幂等性；既有的补齐、fork 和 Ungrouped 用例断言新的顺序。
- `packages/client/ui-workspace/tests/workspace-browser.client.spec.tsx` —— `leads Manual with Sessions its saved order never recorded (created in another tab or browser)` 是渲染层面的回归用例：它固定可见行的顺序与五行折叠线、已保存的归档成员保留槽位而未被记录的归档成员仍排在最后、已在打开的标签页中到达的 Session 落在未记录块的最前、展开后按保存的相对顺序显示保存块、未被记录的当前空白会话仍然领先，以及保存序列从不被改写。
- `pnpm exec vitest run packages/client/ui-workspace/tests/tree.client.spec.ts --coverage --coverage.include='packages/client/ui-workspace/src/client/tree.ts'` 保持 `src/client/tree.ts` 的分支、语句、函数和行覆盖率均为 100%。
