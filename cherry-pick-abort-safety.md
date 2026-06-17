# Cherry-Pick Sequencer 中的 abort-safety 深度解析

> 本文纠正之前对 `abort-safety` 的误解（不是简单的"跨 worktree 防护"），从 Git 内部机制角度分析其真实含义、与 `sequencer/head` 的本质差异，以及为什么成功应用部分提交后它会更新。

---

## 一、核心结论先给

`.git/sequencer/` 目录下有两个容易混淆的"HEAD 相关"文件：

| 文件 | 存什么 | 何时变化 | 作用 |
|------|--------|---------|------|
| **`sequencer/head`** | cherry-pick **启动时**的 HEAD commit hash（40 位） | **全程不变** | abort 时要**回退到的目标点** |
| **`sequencer/abort-safety`** | **当前预期**的 HEAD commit hash（40 位） | **每成功应用一个提交就更新一次** | abort 前的**安全校验标记** |

一句话概括：
> **head 是"我们要回到哪里"，abort-safety 是"我们现在应该在哪里"。**

两者配合实现"安全中止"：只有当前实际 HEAD 与 abort-safety 一致时，才允许 abort 并回退到 head。

---

## 二、为什么需要两个文件？——事务式安全模型

可以把批量 cherry-pick 看作一个**事务**：
- 起点：`head`（事务开始前的状态）
- 当前进度：每成功应用一个提交，状态前进一步
- 中止：`--abort` = 回滚事务，回到起点

但 Git 不是数据库，用户可以随时手动操作（`git reset`、`git checkout` 等）改变 HEAD。如果用户在 cherry-pick 过程中手动移动了 HEAD，再执行 `--abort`，Git 就无法保证能正确回滚。

**`abort-safety` 就是这个事务的"状态校验点"**：

1. 每次成功应用一个提交 → 更新 abort-safety 为新的 HEAD
2. 用户执行 `--abort` 时 → Git 先比较当前 HEAD 与 abort-safety
   - 一致 → 状态连续，安全，可以回滚到 head
   - 不一致 → 用户可能手动改过 HEAD，状态不连续，拒绝 abort，报错

这就是为什么叫 **abort-safety**——它是 abort 操作的"安全前提"。

---

## 三、完整生命周期：3 提交 Cherry-Pick 的状态演变

设定：当前分支 HEAD = X，要 cherry-pick A → B → C（从旧到新），B 冲突。

### 阶段 0：执行前

```
.git/sequencer/  （不存在）
HEAD → X
```

### 阶段 1：启动 cherry-pick，正在应用 A

```
.git/
 ├── CHERRY_PICK_HEAD    → <hash of A>
 └── sequencer/
      ├── todo           → pick A / pick B / pick C
      ├── head           → <hash of X>       （启动时的 HEAD，固定不变）
      └── abort-safety   → <hash of X>       （初始等于 head）

HEAD → X
```

此时 head 和 abort-safety 都是 X，因为还没成功应用任何提交。

### 阶段 2：A 应用成功，开始应用 B

```
.git/
 ├── CHERRY_PICK_HEAD    → <hash of B>       （开始处理 B）
 └── sequencer/
      ├── todo           → pick B / pick C   （A 已从 todo 移除）
      ├── head           → <hash of X>       （**不变**，仍是起点）
      └── abort-safety   → <hash of A'>      （**更新了！** A' 是 A 的 cherry-pick 副本）

HEAD → A'
```

**关键点**：
- A 成功应用后，HEAD 前进到 A'
- `head` 仍然是 X（回退目标不变）
- `abort-safety` 更新为 A'（当前预期的 HEAD 位置）

如果此时执行 abort：
1. Git 检查 HEAD == abort-safety → A' == A' → ✓ 安全
2. Git 重置 HEAD 到 head → X
3. 清理 sequencer 目录 → 完成回滚

### 阶段 3：B 冲突，暂停

```
.git/
 ├── CHERRY_PICK_HEAD    → <hash of B>       （冲突的就是 B）
 ├── MERGE_MSG           → B 的提交信息
 └── sequencer/
      ├── todo           → pick B / pick C   （B 留在首行）
      ├── head           → <hash of X>       （不变）
      └── abort-safety   → <hash of A'>      （不变，B 没成功）

HEAD → A'
```

**注意**：B 冲突时，abort-safety 保持为 A'（最后一个成功的提交），不会更新。

此时 abort 仍然安全：
- HEAD == abort-safety → A' == A' → ✓
- 回退到 head → X

### 阶段 4：用户解决冲突后 continue，B 和 C 都成功

```
.git/
 ├── CHERRY_PICK_HEAD    → （被删除）
 ├── MERGE_MSG           → （被删除）
 └── sequencer/          → （整个目录被删除）

HEAD → C'
```

全部完成，sequencer 目录被整体删除。

---

## 四、Abort-Safety 的校验时机与失败场景

### 4.1 正常 abort（校验通过）

```
用户执行 git cherry-pick --abort

  1. 读取 abort-safety → A'
  2. 读取当前 HEAD     → A'
  3. 两者一致 ✓ → 继续
  4. 读取 head         → X
  5. 重置 index 和工作区到 X
  6. HEAD 指向 X
  7. 删除 sequencer 目录、CHERRY_PICK_HEAD、MERGE_MSG
  8. 完成
```

### 4.2 异常 abort（校验失败）

假设冲突期间用户手动执行了 `git reset --hard HEAD~1`（回退了一个提交）：

```
用户手动 git reset → HEAD 从 A' 退到 X

用户再执行 git cherry-pick --abort

  1. 读取 abort-safety → A'
  2. 读取当前 HEAD     → X
  3. 两者不一致 ✗ → 报错拒绝
  4. 错误信息类似："fatal: Unable to read current commit.  Cannot abort."
     或 "error: your local changes would be overwritten by abort."
```

**为什么要拒绝？** 因为 Git 无法确定当前状态下 abort 是否安全。用户手动改了 HEAD，可能还有其他变化，盲目回滚可能丢失数据。

---

## 五、与 Sequencer 其他文件的关系

### 5.1 四文件分工总览

```
sequencer/
 ├── todo           ← 剩余待处理的源提交列表（"要做什么"）
 ├── head           ← 起始 HEAD（"回退到哪里"）
 └── abort-safety   ← 当前预期 HEAD（"现在应该在哪"）

CHERRY_PICK_HEAD    ← 当前正在处理的源提交（"正在做哪个"）
MERGE_MSG           ← 当前源提交的信息（"提交信息是什么"）
```

可以用"施工队"类比来记忆：

| 文件 | 类比 |
|------|------|
| `todo` | 施工清单：要把 A、B、C 三座建筑搬过来 |
| `head` | 施工前的原始空地位置 |
| `abort-safety` | 施工队当前的安全站位点（每盖好一座就前进一步） |
| `CHERRY_PICK_HEAD` | 正在搬的那座建筑的编号 |
| `MERGE_MSG` | 正在搬的那座建筑的设计图纸 |

### 5.2 与 `CHERRY_PICK_HEAD` 的关系

| | CHERRY_PICK_HEAD | sequencer/abort-safety |
|---|---|---|
| 指向 | 源提交（被 cherry-pick 的那个） | 目标分支上的新提交 |
| 更新时机 | 切换到下一个源提交时 | 成功应用一个源提交后 |
| 冲突时 | 指向冲突的源提交（B） | 指向最后一个成功的目标提交（A'） |
| 数量关系 | todo 中每一条 pick 对应一个 | todo 中每成功一条，前进一个 |

**冲突时的关键差异**：
- CHERRY_PICK_HEAD = B（源提交，"冲突的是哪个"）
- abort-safety = A'（目标分支上的提交，"最后一个安全点"）

### 5.3 与 `sequencer/todo` 的关系

todo 首行 = 当前正在处理 / 冲突的源提交 → 与 CHERRY_PICK_HEAD 对应。
每成功应用一条 todo，abort-safety 就前进一步。

如果 todo 有 N 条，则 abort-safety 会更新 N 次（全部成功的话），最终 HEAD 前进 N 个提交。

---

## 六、单提交 Cherry-Pick：没有 sequencer，怎么校验？

单提交 cherry-pick 不创建 `sequencer/` 目录，自然也没有 `abort-safety`。

那 `git cherry-pick --abort` 怎么判断是否安全？

答案：**单提交时直接用 CHERRY_PICK_HEAD 做轻量校验**。Git 检查：
1. `CHERRY_PICK_HEAD` 是否存在
2. 工作区和 index 是否处于预期的冲突状态

如果满足，就执行 abort：重置到 cherry-pick 前的状态。

这也解释了为什么批量 cherry-pick 需要 sequencer 而单提交不需要：
- 单提交只有一个状态，用 CHERRY_PICK_HEAD 足够描述
- 批量提交有"进度"概念——已应用 N 个、还剩 M 个 → 需要 sequencer 来记录完整状态

---

## 七、Lazygit 代码中的体现

Lazygit **不直接读写** `abort-safety` 文件——这是 Git 内部机制。但 lazygit 的行为处处依赖它：

### 7.1 调用 `--abort` 时隐式依赖

`pkg/gui/controllers/helpers/merge_and_rebase_helper.go` 中 `genericMergeCommand` 调用 `git cherry-pick --abort`：

```go
func (self *MergeAndRebaseHelper) genericMergeCommand(command string) error {
    // ...
    result := self.c.Git().Rebase.GenericMergeOrRebaseAction(commandType, command)
    // commandType="cherry-pick", command="abort"
    if err := self.CheckMergeOrRebase(result); err != nil {
        return err  // 如果 abort-safety 校验失败，Git 返回错误，在这里被捕获
    }
    return nil
}
```

如果 abort-safety 校验失败（用户手动改过 HEAD），Git 会返回非零退出码和错误信息，lazygit 通过 `CheckMergeOrRebase` 将错误呈现给用户。

### 7.2 冲突提交显示逻辑

`pkg/commands/git_commands/commit_loader.go` 中 `getHydratedSequencerCommits` 用 todo 文件来确定冲突提交：

```go
commits := self.getSequencerCommits(hashPool) // 读 sequencer/todo
if len(commits) > 0 {
    commits[len(commits)-1].Status = models.StatusConflicted
    // todo 最后一项（显示在最顶）= 当前冲突的提交
}
```

这与 abort-safety 的位置是**错开一位**的：
- todo 首行 = 正在处理但没成功的源提交 → 显示为"冲突"
- abort-safety = 上一个成功应用的目标提交 → 也就是当前 HEAD 的位置

### 7.3 Auto-Stash 与 abort-safety 的交互

`pkg/gui/controllers/helpers/cherry_pick_helper.go` 中 `Paste` 方法在 cherry-pick 前自动 stash 脏文件：

```go
mustStash := IsWorkingTreeDirtyExceptSubmodules(self.c.Model().Files, ...)
if mustStash {
    self.c.Git().Stash.Push(self.c.Tr.AutoStashForCherryPicking)
}

// ...执行 cherry-pick...

// 成功后 pop
isInCherryPick, _ := self.c.Git().Status.IsInCherryPick()
if !isInCherryPick {
    if mustStash {
        self.c.Git().Stash.Pop(0)
    }
}
```

**注意冲突时的行为**：如果 cherry-pick 冲突暂停（仍在 cherry-pick 状态），stash **不会被 pop**。为什么？

因为此时工作区处于冲突合并状态，如果 pop stash 会在冲突之上再叠改动，后果不可控。等用户解决冲突、continue 完成后，也不会自动 pop——因为 continue 走的是另一条代码路径（`merge_and_rebase_helper.go` 的 `genericMergeCommand`），不经过 `Paste` 方法的收尾逻辑。

这与 abort-safety 的"状态连续性"设计理念一致：**只有状态连续且可预期时，才执行恢复操作**。

---

## 八、常见误区澄清

### 误区 1：abort-safety 是 worktree 标识

**不准确**。虽然 Git 确实有跨 worktree 防护，但那是通过其他机制实现的。abort-safety 的核心作用是**校验 HEAD 的连续性**，防止用户手动移动 HEAD 后误 abort。

（补充：在某些 Git 版本或配置下，abort-safety 可能包含 worktree 相关信息，但其首要和本质的功能是 HEAD 连续性校验。）

### 误区 2：head 和 abort-safety 是一回事

**完全不同**。
- `head` 是起点（固定不变）
- `abort-safety` 是当前安全点（随进度更新）
- 两者配合实现"从安全点回退到起点"的 abort 操作

### 误区 3：cherry-pick sequencer 有 done 文件

**没有**。done 文件是 rebase 的（在 `rebase-merge/done`）。cherry-pick sequencer 不需要 done——已成功应用的提交直接"消失"（从 todo 移除，HEAD 前进），因为 cherry-pick 是线性推进的，不需要记录每一步的动作类型。

rebase 有 done 是因为 rebase 可以有多种动作（pick、drop、reword、fixup、squash...），需要完整记录已执行了什么。

### 误区 4：abort 就是回到 abort-safety

**不对**。abort 是回到 `head`（起点）。`abort-safety` 只是**校验前提**，不是回退目标。

---

## 九、总结：一个比喻

想象你在走楼梯，从 1 楼（X）出发，要爬到 4 楼（A'→B'→C'）。

- `head` = 1 楼（你出发的地方，掉下去要回到底层）
- `abort-safety` = 你当前站的那层台阶（每爬一层更新一次）
- `todo` = 剩下还要爬的楼层
- `CHERRY_PICK_HEAD` = 你正在抬脚上的那层台阶

正常情况：
- 你爬到 2 楼 → abort-safety = 2 楼
- 你爬到 3 楼 → abort-safety = 3 楼
- 你从 3 楼坠下来（冲突）→ abort-safety 还是 2 楼（最后一个安全点）

按"回到底层"按钮（`--abort`）：
1. 先确认你确实站在 abort-safety 那层 → 是 → 可以跳
2. 直接跳到 1 楼（head）
3. 收好楼梯（清理 sequencer）

如果你从 2 楼偷偷坐电梯跑到了别的地方（手动改了 HEAD），再按"回到底层"按钮：
1. 发现你不在 abort-safety 那层 → 不对，不能乱跳
2. 报错拒绝

这就是 `head` + `abort-safety` 的双保险机制。
