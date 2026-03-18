# Git 嵌套子模块解除指南

当你的 Git 仓库中嵌套了另一个 Git 仓库（作为 submodule），想要解除嵌套关系时，按照以下步骤操作。

## 场景说明

- 主仓库：私人笔记仓库（如 Gitee）
- 嵌套仓库：公开仓库（如 GitHub，作为 submodule 存在）
- 目标：解除嵌套，让子目录变成普通文件夹

## 操作步骤

### 1. 确认子模块信息

```bash
# 查看子模块状态
git submodule status

# 查看 .gitmodules 文件内容
cat .gitmodules
```

输出示例：
```
[submodule "2"]
    path = 2
    url = git@github.com:username/repo.git
```

### 2. 清空子目录内容（如不需要保留文件）

```bash
# 进入子目录，删除所有文件
cd <子目录名>
rm -rf * .*

# 回到主目录
cd ..
```

> 如果需要保留文件，跳过此步骤。

### 3. 取消子模块注册

```bash
git submodule deinit -f <子目录名>
```

### 4. 删除子模块的 git 数据

```bash
rm -rf .git/modules/<子目录名>
```

### 5. 从 Git 索引中移除子模块（关键！）

```bash
git rm --cached <子目录名>
```

> 这一步最重要！submodule 在 git 中以特殊类型 `160000`（gitlink）存储在索引中，必须用 `--cached` 参数移除索引引用。

### 6. 删除 .gitmodules 文件

```bash
rm .gitmodules
```

或者手动编辑 `.gitmodules`，删除对应的子模块条目。

### 7. 提交更改

```bash
git add .gitmodules <子目录名>/
git commit -m "移除嵌套的子模块"
```

### 8. 推送到远程（可选）

```bash
git push
```

---

## 快速执行命令（假设子目录名为 `2`）

```bash
# 清空目录
cd 2 && rm -rf * .* && cd ..

# 取消注册
git submodule deinit -f 2

# 删除 git 数据
rm -rf .git/modules/2

# 从索引中移除 submodule（关键！）
git rm --cached 2

# 删除配置文件
rm .gitmodules

# 提交并推送
git commit -m "移除嵌套的 github 子模块"
git push
```

---

## 操作后验证

```bash
# 确认子模块已移除
git submodule status
# 应该无输出

# 确认 .gitmodules 不存在
cat .gitmodules
# 应该显示文件不存在
```

---

## 注意事项

| 需求 | 操作 |
|------|------|
| 不保留子目录文件 | 步骤 2 清空目录 |
| 保留子目录文件 | 跳过步骤 2 和步骤 5 |
| 保留子目录 git 历史 | 需要额外操作：进入子目录 `git remote remove origin`，使其成为独立仓库 |
| 彻底删除子目录 | 步骤 2 后执行 `rmdir <子目录名>` |

---

**创建时间**: 2026-03-18
