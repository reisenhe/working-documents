# Claude 与 Git Worktree 结合使用指南

## 什么是 Git Worktree

Git Worktree 是 Git 提供的一项功能，允许在同一仓库中同时检出多个分支。每个 worktree 都是一个独立的工作目录，拥有自己的文件副本，但共享同一个 Git 对象数据库。

## 为什么需要 Worktree

- **并行开发**：可以在不切换分支的情况下同时处理多个任务
- **保留现场**：当你需要临时切换到其他任务时，可以将当前工作状态保存为一个新的 worktree
- **独立验证**：在不同 worktree 中可以独立运行测试、构建，互不干扰
- **Claude Code 结合**：配合 Claude Code 的 worktree 模式，可以在不同任务间无缝切换，每个 worktree 都有独立的会话上下文

## 如何在 Claude 中使用 Worktree

### 1. 创建 Worktree

```bash
# 基于当前分支创建新的 worktree
git worktree add ../worktree-feature -b feature/new-feature

# 或基于已有分支创建
git worktree add ../worktree-fix -b fix/issue-123
```

### 2. 在 Claude 中切换 Worktree

Claude Code 支持在进入 worktree 时自动切换分支上下文：

```bash
# 使用 EnterWorktree 工具进入已存在的 worktree
# 或创建新的 worktree 并自动进入

# 切换到已有的 worktree
# 输入 worktree 的路径即可
```

### 3. Worktree 管理命令

```bash
# 查看所有 worktree
git worktree list

# 查看 worktree 状态
git worktree prune

# 移除不再需要的 worktree
git worktree remove ../worktree-feature
```

### 4. Claude 中的 Worktree 模式

当使用 `EnterWorktree` 工具进入 worktree 时：
- 会自动切换到对应分支
- 会创建一个独立的工作目录
- 当前的 Claude 会话上下文保持不变，但指向新的工作目录

### 5. 最佳实践

- 为每个主要任务或功能创建独立的 worktree
- 使用描述性的 worktree 名称（如 `feature/xxx`、`fix/xxx`）
- 完成任务后及时清理不再需要的 worktree
- 在独立的 worktree 中测试和验证，避免污染主分支

## 注意事项

- 同一个分支不能被多个 worktree 同时检出
- worktree 之间共享对象数据库，磁盘占用较小
- 删除 worktree 不会影响其他 worktree 或仓库历史