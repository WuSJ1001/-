#SSH #github #git

我们在往github上push项目的时候，如果走https的方式，每次都需要输入账号密码，非常麻烦。而采用ssh的方式，就不再需要输入，只需要在github自己账号下配置一个ssh key即可。

生成 SSH Key 并配置 GitHub/GitLab 详细教程

## 🟢 第 1 步：检查是否已有 SSH Key

在 **`Git Bash (Windows)`**、终端 **`(Linux/macOS)`** 运行以下命令：

```shell
ls -al ~/.ssh
```

🔹 可能的输出：

如果已有 **SSH Key（如 id_rsa 和 id_rsa.pub）**  
说明你已经生成过 **SSH Key**，可以跳到 第 4 步 直接添加到 **GitHub/GitLab**。  
**`如果 .ssh 目录不存在或没有 id_rsa 文件 说明你需要生成新的 SSH Key，请继续下一步`**。

## 🟢 第 2 步：生成新的 SSH Key

运行以下命令：

```shell
ssh-keygen -t rsa -b 4096 -C "你的邮箱"
```

示例：

```shell
ssh-keygen -t rsa -b 4096 -C "yaoyuzhuo6@gmail.com"
```

🔹 参数说明：

**`-t rsa`** ：使用 RSA 算法（GitHub/GitLab 推荐）  
**`-b 4096`** ：生成 4096 位密钥（比默认 2048 位更安全）  
**`-C`** "你的邮箱" ：添加注释（通常是你的 GitHub/GitLab 绑定邮箱）

💡 系统会询问以下问题：

1. **`Enter file in which to save the key`** (~/.ssh/id_rsa 默认回车)

```shell
Enter file in which to save the key (/home/your-user/.ssh/id_rsa):
```

- **直接回车 使用默认路径（推荐）**
- 如果已有密钥，可以换个名字，如 **`id_rsa_github`**

2. Enter passphrase（输入密码，可留空）

```shell
Enter passphrase (empty for no passphrase):
```

- **建议直接回车（否则每次使用 SSH 都要输入密码）**

生成成功后，会在 ~/.ssh/ 目录下创建两个文件：

```powershell
~/.ssh/id_rsa      # 私钥（不要分享）
~/.ssh/id_rsa.pub  # 公钥（需要添加到 GitHub）
```

## 🟢 第 3 步：启动 SSH Agent 并添加密钥（linuex）

1️⃣ 启动 SSH 代理（用于管理密钥）：

```shell
eval "$(ssh-agent -s)"
```

成功时会返回：

```shell
Agent pid 12345
```

2️⃣ 将私钥添加到 SSH 代理：

```shell
ssh-add ~/.ssh/id_rsa
```

如果你的私钥文件名不是 **`id_rsa`**，请修改为实际名称：

```shell
ssh-add ~/.ssh/id_rsa_github
```

💡 如果报错：

```shell
Could not open a connection to your authentication agent
```

请先运行：

```shell
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa
```

## 🟢 第 4 步：复制 SSH 公钥

运行：

```shell
cat ~/.ssh/id_rsa.pub
```

🔹 复制公钥的方法：

**`Windows Git Bash`**：

```shell
clip < ~/.ssh/id_rsa.pub
```

**`macOS`**：

```shell
pbcopy < ~/.ssh/id_rsa.pub
```

**`Linux（手动复制）`**：

```shell
cat ~/.ssh/id_rsa.pub
```

## 🟢 第 5 步：添加 SSH Key 到 GitHub/GitLab

🔹 GitHub：

1. **`打开 GitHub SSH 设置`**
2. **`点击 New SSH Key`**
3. **`Title：随便填（如 “My Laptop”）`**
4. **`Key：粘贴 id_rsa.pub 里的内容`**
5. **`点击 Add SSH Key`**

🔹 GitLab：

1. **`打开 Preferences -> SSH Keys`**
2. **`粘贴 id_rsa.pub 的内容`**
3. **`点击 Add Key`**

## 🟢 第 6 步：测试 SSH 连接

```shell
ssh -T git@github.com
```

如果成功，会看到：

```shell
Hi <your-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

🎉 SSH Key 配置成功！

## 🟢 第 7 步：设置 Git 使用 SSH 方式拉取代码

默认情况下，Git 可能还在使用 HTTPS，需要手动改为 SSH。

✅ 设置全局 Git 远程 URL 方式：

```shell
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

**`这样你 git clone 或 git push 就不会要求输入 GitHub 账号和密码了！`**

## 🟢 第 8 步：使用 SSH 克隆 GitHub/GitLab 仓库

💡 HTTPS 方式（需要输入密码）：

```shell
git clone https://github.com/your-username/repository.git
```

💡 SSH 方式（不需要输入密码）：

```shell
git clone git@github.com:your-username/repository.git
```

如果使用 GitLab：

```shell
git clone git@gitlab.com:your-username/repository.git
```









## 第五步：验证是否设置成功
ssh -T git@github.com

设置成功后，即可不需要账号密码clone和push代码

注意之后在clone仓库的时候要使用ssh的url，而不是https！

# 验证原理
SSH登录安全性由非对称加密保证，产生密钥时，一次产生两个密钥，一个公钥，一个私钥，在git中一般命名为id_rsa.pub, id_rsa。

那么如何使用生成的一个私钥一个公钥进行验证呢？

本地生成一个密钥对，其中公钥放到远程主机，私钥保存在本地
当本地主机需要登录远程主机时，本地主机向远程主机发送一个登录请求，远程收到消息后，随机生成一个字符串并用公钥加密，发回给本地。本地拿到该字符串，用存放在本地的私钥进行解密，再次发送到远程，远程比对该解密后的字符串与源字符串是否等同，如果等同则认证成功。

**通俗解释！！**
重点来了：一定要知道ssh key的配置是针对每台主机的！，比如我在某台主机上操作git和我的远程仓库，想要push时不输入账号密码，走ssh协议，就需要配置ssh key，放上去的key是当前主机的ssh公钥。那么如果我换了一台其他主机，想要实现无密登录，也就需要重新配置。

（1）为什么要配？
配了才能实现push代码的时候不需要反复输入自己的github账号密码，更方便
（2）每使用一台主机都要配？
是的，每使用一台新主机进行git远程操作，想要实现无密，都需要配置。并不是说每个账号配一次就够了，而是每一台主机都需要配。
（3）配了为啥就不用密码了？
因为配置的时候是把当前主机的公钥放到了你的github账号下，相当于当前主机和你的账号做了一个关联，你在这台主机上已经登录了你的账号，此时此刻github认为是该账号主人在操作这台主机，在配置ssh后就信任该主机了。所以下次在使用git的时候即使没有登录github，也能直接从本地push代码到远程了。当然这里不要混淆了，你不能随意push你的代码到任何仓库，你只能push到你自己的仓库或者其他你有权限的仓库！