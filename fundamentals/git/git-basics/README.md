## Git的基本使用

> ### *Yoo, I'm Obstacle19 👋*
>
> - *🍻 Junior at 🇨🇳 [HNU], BSc in Computer Science*
> - *⚡ C++ / Python / Matlab.*
> - *✍️ Writer at [CSDN](https://blog.csdn.net/obstacle19?type=blog)*

### 一、建立本地仓库

新建一个文件夹：

```
mkdir github
```

`PowerShell`进入`github`文件夹目录下，执行命令将该文件夹初始化为一个仓库：

```
git init
```

此时在该目录下输入`ls`命令，发现为空；使用`ls -a`命令，发现路径下存在隐藏的`.git`文件夹：

```
ls -a
```

### 二、环境配置

```
git config [--global] user.name "[name]"
```

```
git config [--global] user.email "[email address]"
```

> `git config` 命令用于配置 Git 的设置，这些设置可以控制 Git 的行为。当你使用 `--global` 选项时，这些设置会被应用到你的全局 Git 配置文件中（通常是 `~/.gitconfig`），这意味着它们会影响你所有的 Git 仓库。如果你不使用 `--global` 选项，那么这些设置将仅应用于当前仓库（这些设置会被保存在当前仓库的 `.git/config` 文件中）。
>
> - `git config [--global] user.name "[name]"`：这个命令用于设置你的用户名。当你提交更改到 Git 仓库时，这个用户名会被记录在提交历史中。使用 `--global` 选项会将这个设置应用到你的所有 Git 仓库，如果不使用 `--global`，则仅对当前仓库有效。请将 `[name]` 替换为你的实际用户名。
> - `git config [--global] user.email "[email address]"`：这个命令用于设置你的电子邮件地址。与用户名相似，这个电子邮件地址也会被记录在提交历史中。使用 `--global` 选项可以将这个设置应用到你的所有 Git 仓库，如果不使用 `--global`，则仅对当前仓库有效。请将 `[email address]` 替换为你的实际电子邮件地址。
>
> 例如：
>
> ```
> git config --global user.email "718839296@qq.com"
> ```
>
> ```
> git config --global user.name "ShaoShuai"
> ```

### 三、增加 / 删除文件

#### 3.1 添加目录下所有文件到暂存区

```
git add .
```

 添加当前目录的**所有文件**到暂存区。

### 四、远程操作

#### 4.1 显示所有远程仓库

```
git remote -v
```

#### 4.2 显示某个远程仓库的信息

```
git remote show [remote]
```

> 例如：
>
> ```
> git remote show origin
> ```
>
> 显示结果如下：
>
> ```
> remote origin
> Fetch URL: https://github.com/Obstacle19/ns3-datacenter.git
> Push  URL: https://github.com/Obstacle19/ns3-datacenter.git
> HEAD branch: master
> Remote branches:
>   buffer              new (next fetch will store in remotes/origin)
>   main                new (next fetch will store in remotes/origin)
>   master              tracked
>   migrating-waf-cmake new (next fetch will store in remotes/origin)
>   patch-1             new (next fetch will store in remotes/origin)
>   queueing            new (next fetch will store in remotes/origin)
>   vpi                 new (next fetch will store in remotes/origin)
> Local ref configured for 'git push':
>   master pushes to master (up to date)
> ```

#### 4.3 增加远程仓库

```
git remote add [remote] [url]
```

> 例如：
>
> ```
> git remote add origin https://github.com/Obstacle19/ns3-datacenter.git
> ```
>
> - `git remote add`：这是Git命令的一部分，用于添加一个新的远程仓库到你的本地仓库中。远程仓库通常是一个在线仓库，如`GitHub`、`GitLab`或`Bitbucket`等，它允许你在多个用户之间共享你的代码库。
> - `origin`：这是你为这个远程仓库指定的名称。在Git中，你可以为每个远程仓库指定一个简短的名称（如`origin`、`upstream`、`github`等），以便在后续的命令中引用它。这里的`origin`是一个自定义名称，你可以根据需要将其更改为任何你喜欢的名称。
> - `https://github.com/Obstacle19/ns3-datacenter.git`：这是远程仓库的URL，指向了GitHub上的一个具体仓库。这个URL告诉Git，当你想从远程仓库拉取（pull）或推送（push）更改时，应该去哪里找。在这个例子中，URL指向的是`Obstacle19`用户在`GitHub`上创建的名为`ns3-datacenter`的仓库。

#### 4.4 删除远程仓库

```
git remote rm [remote]
```

> 例如：
>
> ```
> git remote rm origin
> ```
>
> 用于删除当前仓库中名为 `origin` 的远程仓库引用

#### 4.5 获取远程仓库URL

```
git remote get-url [remote]
```

> 例如：
>
> ```
> git remote get-url origin
> ```
>
> 结果为`https://github.com/Obstacle19/ns3-datacenter.git`

#### 4.6 修改远程仓库URL

```
git remote set-url [remote] [remote-url]
```

> 例如：
>
> ```
> git remote set-url origin https://github.com/Obstacle19/ns3-datacenter.git
> ```
>
> 用于更改 Git 仓库中名为 `origin` 的远程仓库的 URL。

#### 4.7 PULL

```
git remote add [remote] [url]
```

增加远程仓库。

> 例如：
>
> ```
> git remote add origin https://github.com/Obstacle19/ns3-datacenter.git
> ```
>

```
git pull [remote] [branch]
```

取回远程仓库的变化，并与本地分支合并。

> 例如：
>
> ```
> git pull origin master
> ```
>
> `git pull origin master` 这段代码在 Git 中用于从名为 `origin` 的远程仓库中拉取（pull）名为 `master` 的分支的最新更改，并将这些更改合并到当前所在的本地分支中。
>
> ​	如果出现`fatal: unable to access 'https://github.com/Obstacle19/ns3-datacenter.git/': SSL certificate problem: unable to get local issuer certificate`报错，说明 Git 在尝试通过 HTTPS 连接到 GitHub 时，无法验证 SSL 证书的有效性。
>
> 这时可以临时禁用SSL证书验证（不推荐，仅用于测试）：
>
> ```
> git config --global http.sslVerify false
> ```
>
> > 如果你只是想快速绕过这个问题进行测试，可以临时禁用 SSL 证书验证。但这会降低安全性，因此不建议在生产环境中使用。
>
> ```
> git config --global http.sslVerify true
> ```
>
> > 在完成测试后重新启用 SSL 验证
>
> 再次尝试`git pull origin master`命令就可以成功拉取远程仓库了。

#### 4.8 PUSH

```
git add .
```

首先添加当前目录的**所有文件**到**暂存区**。

```
git commit -m [message]
```

然后提交**暂存区**到**仓库区**。

> 例如：
>
> ```
> git commit -m "drew pictures"
> ```
>
> `git commit -m "drawed pictures"` 这条命令用于将暂存区（staging area）的改动提交到本地仓库中，并附带一条消息 "drew pictures" 来描述这次提交的内容。
>
> 命令的详细解释如下：
>
> - `git commit`：这是 Git 中用于提交更改的命令。它告诉 Git 将暂存区的所有改动永久保存到仓库中。
> - `-m` 或 `--message`：这是一个选项，用于直接在命令行中提供提交信息，而不是在 Git 编辑器中编写。这样做可以更快地完成提交。
> - `"drew pictures"`：这是你为这次提交附带的消息。它应该简洁明了地描述你的更改。

```
git push [remote] [branch]
```

上传本地指定分支到远程仓库。

> 例如：
>
> ```
> git push origin master
> ```
>
> 如果报告错误`fatal: unable to access 'https://github.com/Obstacle19/PowerTCP-RAW.git/': GnuTLS recv error (-110): The TLS connection was non-properly terminated.`
>
> 那么回到根目录，使用
>
> ```
> ssh-keygen -t rsa
> ```
>
> 生成 RSA 类型的 SSH 密钥对。SSH（Secure Shell）是一种加密的网络协议，用于安全地访问远程计算机。SSH 密钥对包括一个公钥和一个私钥，它们用于在客户端和服务器之间建立安全的连接，而无需在每次连接时输入密码。
>
> ```
> cd .ssh
> ```
>
> ```
> cat id_rsa.pub
> ```
>
> 复制密钥，输入到github上面。
>
> github上添加秘钥完毕后，可以使用
>
> ```
>ssh -T git@github.com
> ```
> 
> 命令，验证你的 SSH 密钥是否已经被 GitHub 账户接受，而无需实际登录或执行任何操作。
>
> 当你运行这个命令时，如果一切设置正确，你会看到类似这样的响应：
>
> ```
>Hi username! You've successfully authenticated, but GitHub does not provide shell access.
> ```

...

### 五、查看历史信息

#### 5.1 显示有变更的文件

```
git status
```

