# Agent Note: 恢复的日志保护其表面新获得的 system 消息

Status: implemented

[English](2026-10-15-resumed-log-system-prompt-head.md) | 中文

## Problem

一条 v4 存储会话的历史由「尚未记录 system 消息」的格式写入，随后被当前循环恢复。恢复时循环把已存事件作为 seed 重放，而 `SystemPromptProjection.systemNodes()` 在该表面上找不到存活的 `system/message`，于是该 step 的首次 system prompt 提交是 `append`（`packages/core/agent-loop/src/agent.ts`），落在重放出的 user、assistant、tool 节点之后。原生 V4 关系校验拒绝任何在表面已有节点后到达的 `system/message`，并要求受保护的 system head 必须是表面的第一个节点，因此它拒绝了整个产物：`SessionFormatError: system/message requires a protected first surface head`，对外表现为 `stored log is corrupt`。写入端产出了自己的读取端拒绝的日志，且此后每次追加都保持被拒绝状态。一条由旧写入端产出的三轮 COMSOL 工作会话（948 个事件，`~/.dsh/sessions/--F-COMSOL64--/session-1b68db67-…`）在 2026-10-14 正是这样失败的；会话层（`dsh-session` 的 `foldSurface`）接受同一条日志，只有原生 V4 关系校验拒绝它。存储产物本身没有任何畸形之处：表面的 system 节点只是不在第一个位置。

## Decision

`packages/session/session-format-v3-to-v4/src/relationships.ts` 接受加入已有历史的表面的 `system/message`。第一个加入表面的 `system/message` 成为受保护节点，无论它是追加在任意位置，还是在尚无受保护节点时由一次替换引入；`protectedHead` 记录该节点，既有的保护规则原样适用——只有恰好覆盖该节点的 `system/message` 才能替换它，compaction 永远不能遮蔽它。表面顺序不受其他影响，因此位于表面中部的 system 节点保留其记录位置，模型可见历史与写入时一致。取用无需改动：`SystemPromptProjection.systemNodes()` 按表面顺序读取存活节点，因此恢复后的循环按 seq 找到被追加的节点，并在渲染提示词下次变化时替换它。V2-to-V3 迁移保留自己的拒绝规则，因为该流程在迁移中合成 head，并拒绝首个 step 之前已有表面的 V2 日志，而不是重排时序；原生 V4 读取没有时序可改。`SESSION_FORMAT_VERSION` 保持 4：不改变任何存储字段、信封或 schema。

## Alternatives considered

**拒绝该日志，改为修写入端。** 否决：一旦已有更早历史，仅追加的日志就无法把 system prompt 放到表面位置 0，因此拒绝这次追加只会在「模型可见的 system prompt 没有对应日志事件」（破坏 model-visible ⟺ logged）与「恢复的会话在无提示词的情况下静默运行」之间二选一。

**接受 append，但继续拒绝引入首个 system 节点的替换。** 否决：这样两条路径会产出保护级别不同的 system 节点，而由替换引入的那个会让表面带着未受保护的 system 节点；在两条路径上都以第一个 system 消息确立受保护节点，才能维持同一个不变量。

**重写或迁移存储日志，而不是放宽读取。** 否决：读取流程不得重写存储产物，而且该产物本身完全有效，因此修正归属校验而非交给一个必须重编号历史的迁移。

**同时放宽保护规则，允许 compaction 遮蔽该节点。** 否决：保护正是让会话的 system prompt 在 compaction 之后仍可取回的东西；错的只是位置要求，不是保护本身。

## Consequences

读取严格变宽：此前能加载的日志仍能加载，而表面在更早节点之后才获得 system 消息的日志现在也能加载，且该消息受保护、不被 compaction 或任何非 system 替换遮蔽。由旧写入端写出的会话可以被读取并继续，其下一次提示词变化会按 seq 替换受保护节点，因此表面始终只有一个受保护的提示词节点。代价是存储产物现在可能把会话的 system prompt 呈现在表面顺序中的任意位置，包括夹在更早的对话节点之间；此前假定 system 消息位于派生消息之首的消费方必须改为按表面顺序读取。保护强度不变：compaction 与非 system 替换不得遮蔽的节点，就是表面最先获得的那个 system 消息。

## Testing

`packages/session/session-format-v3-to-v4/tests/relationships.spec.ts` 把此前被拒绝的用例（user 输入后跟 system 消息）改为被接受的用例：该日志可重新打开，其派生消息依次为 user、system；遮蔽该节点的 `compaction/prune` 抛出 `/protected system head/`，对该节点的非 system 替换抛出同样的错误，而恰好覆盖它的 system 替换被接受，由替换引入表面唯一 system 消息的情况也被接受。其余畸形关系用例仍然抛出，因此这次放宽没有削弱范围、引用与保护检查。`packages/session/session-persistence-jsonl/tests/current-event-admission.spec.ts` 存入一条表面为「user 输入后跟 system 消息」的日志，断言它可以按读与写打开、其后的追加不改动已有 generation 字节，且重新打开可取得全部事件。`packages/session/session-persistence-jsonl/tests/native-developer-admission.spec.ts` 继续拒绝 developer 替换受保护的 system 节点。
