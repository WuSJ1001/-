#git #github 

[还不会使用 GitHub ？ GitHub 教程来了！万字图文详解 - 知乎](https://zhuanlan.zhihu.com/p/369486197)
# 1.创建本地文件夹

# 2.本地初始化

	在该目录下打开Git Bash
	输入`git init`，进行初始化
# 3.连接

输入一下命令
```
git remote add origin git@github.com:yourName/repositoryname.git
git remote add origin https://github.com/yourName/repositoryname.git
```
yourName是用户名，repositoryname是仓库名字
# 4.从远程仓库拉取文件

git pull origin "分支名"
# 5.查看工作目录状态

git status
# 6.提交更改，添加备注信息
```
git commit -m "备注信息"
注意：若第5步的信息中有以下情况：
1.Untraked Files
使用git add .解决该问题
2.Changes not staged for commit
使用 git commit -am "备注信息" 解决
```

# 7.将本地文件push到远程仓库

```
git push "远程分支名" "本地分支名"
```
第一次push需要将本地分支和远程分支绑定在一起，在远程分支名和本地分支名加`-u`表示设置上游分支
push常用五个参数：
1. **-u** (--set-upstream)：第一次推新分支时用，绑定远程分支。
    
2. **-f** (--force)：强推！覆盖远程代码（**慎用！**尽量用 --force-with-lease 代替）。
    
3. **-d** (--delete)：删除远程分支。
    
4. **--tags**：把本地打的版本标签（Tag）推送到远程。
    
5. **--no-verify**：代码检查过不去，但我确信没问题非要推送时用。

# 8.利用 SSH 完成 Git 与 GitHub 的绑定

### 第一步：生成 SSH key
输入`ssh-keygen -t rsa`命令，表示我们指定 RSA 算法生成密钥，然后敲三次回车键，期间不需要输入密码，之后就就会生成两个文件，分别为id_rsa和id_rsa.pub，即密钥id_rsa和公钥id_rsa.pub. 对于这两个文件，其都为隐藏文件，默认生成在以下目录：

Linux 系统：~/.ssh

Mac 系统：~/.ssh

Windows 系统：C:\Documents and Settings\username\\.ssh

密钥和公钥生成之后，我们要做的事情就是把公钥id_rsa.pub的内容添加到 GitHub，这样我们本地的密钥id_rsa和 GitHub 上的公钥id_rsa.pub才可以进行匹配，授权成功后，就可以向 GitHub 提交代码啦！

### 第二步：添加 SSH key
进入我们的 GitHub 主页，先点击右上角所示的倒三角▽图标，然后再点击Settins，进行设置页面；点击我们的头像亦可直接进入设置页面：

进入Settings页面后，再点击SSH and GPG Keys进入此子界面，然后点击New SSH key按钮：

我们只需要将公钥id_rsa.pub的内容粘贴到Key处的位置（Titles的内容不填写也没事），然后点击Add SSH key 即可。

### 第三步：验证绑定是否成功
在我们添加完SSH key之后，也没有明确的通知告诉我们绑定成功啊！不过我们可以通过在 Git Bash 中输入ssh -T git@github.com进行测试：
![](https://pic2.zhimg.com/v2-aa1938b4970cd5ccddd82406f58aee83_1440w.png)
如上图所示，此结果即为Git 与 GitHub 绑定成功的标志。

# 9.通过 Git 将代码提交到 GitHub
