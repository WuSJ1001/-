#git 

[还不会使用 GitHub ？ GitHub 教程来了！万字图文详解 - 知乎](https://zhuanlan.zhihu.com/p/369486197)
## 检查当前文件的状态
```
git status
```
## 跟踪新文件或暂存已修改的文件
```
git add <file>
```
## 查看已暂存和未暂存的修改
此命令比较的是工作目录中当前文件和暂存区域快照之间的差异。 也就是修改之后还没有暂存起来的变化内容。
```
git diff
```
若要查看已暂存的将要添加到下次提交里的内容，可以用 `git diff --staged` 命令。 这条命令将比对已暂存文件与最后一次提交的文件差异

## 提交更新(后面加`-m "注释信息"`可以将提交信息与命令放在同一行)
```
git commit
```
若要跳过暂存步骤直接将所有已经跟踪过的文件暂存起来一并提交，可以在后面加`-a`从而跳过`git add`步骤
## 移除文件
```
git rm <file>
```
#### 从 Git 索引中移除（不删除本地物理文件）

在终端执行：
```
git rm -r --cached <filename>
```

- --cached 参数的意思是：**只告诉 Git 把这个文件夹从管理名单里踢出去，但别动我电脑硬盘里的实际文件。**
## 移动文件
```
git mv <file_from> <file_to>
```
## 忽略文件，编写.gitignore文件的语法
注意文件编码格式要是UTF-8格式
忽略具体某个文件直接写入文件名，忽略某个文件夹就是文件夹名后加一个`/`
## 查看日志
```
git log
```

在命令行窗口的光标处，输入git log"命令，打印 Git 仓库提交日志

## 查看分支
```
git branch
```
创建新分支时在branch后跟新分支的名称
```
git branch <name>
```
## 删除分支
```
git branch -d <name>
```
## 切换分支
```
git checkout <name>
```
## 合并分支
```
git merge <name>
```
将name分支合并到master分支

## 添加标签
```
git tag <name>
```