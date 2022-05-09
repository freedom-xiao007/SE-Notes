# git 使用记录
***
git 在我看来是很重要的，怎么重要现在我的概念还不是太清楚，等我用一段时间以后补充
***

## 下面是一些git的使用心得

### 一、git 的初步使用基本配置步骤：
#### 1.git 的初始化：用户名和邮箱的配置（自己先申请一个git账号）
```
设置命令和方法如下：
```
```
git config --global user.name "这里填写你的用户名"
git config --global user.email "这里填写你的邮箱"
```
```
这样就配置好了用户
```
/home/nssas/enviroment/installFile/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic logs --from-beginning

#### 2.创建版本库
```
到你想要将其内容上传git的目录下，运行下面的命令：
```
```
git init
```
```
我来演示一下，我是window系统，安装的是 git for window ，通过下面的步骤完成创建：
```
```
比如我想让 D:\tmp\test 这个目录创建版本库，想进入 D:\tmp\test 目录下：
```
![init1](./photo/init1.jpg)
```
然后鼠标右键，弹出如下的鼠标菜单选项：
```
![init2](./photo/init2.jpg)
```
简单的就是选择 GitExt Create new repository 生成版本库就行···········
但为了装一波，你得学会命令行的操作啊！点击Git Bash Here，输入 git init 出现像下面一样
的界面：
```
![init3](./photo/init3.jpg)
```
成功后生成 .git 的隐藏文件：
```
![init4](./photo/init4.jpg)

#### 3.添加文件
```sh
使用下面的命令添加该目录下的所有文件, "." 代表当前目录：
git add .
```

#### 4.提交至本地版本库中
```sh
把添加的文件提交到本地的版本库中，使用下面的命令：
```

```sh
git commit
```

```sh
会出现下面的界面叫你输入描述信息，这个其实随便输一点就行了,然后保存退出，就提交成功了
```
![init5](./photo/init5.jpg)
![init6](./photo/init6.jpg)

#### 5.连接远程版本库
```sh
你得把本地版本库和远程版本库连接起来，让本地版本库的文件能提交到远程版本库上，像下面这样
```
```sh
git remote add origin 你的远程版本库地址
```

#### 6.提交到远程版本库
```sh
git push origin master
```

```sh
后面加force是强制覆盖
```

```sh
git push origin master --force
```

#### 7.合并其他分支的部分改动
```sh
# 切换到需要合并的分支，并拉取更新，让其处于最新状态
git checkout develop    
git pull

# 基于当前分支创建一个临时分支，用于合并
git checkout -b develop_temp

# 进行文件或文件夹合并
git checkout origin/被合并的分支 需要合并的文件和文件夹

# 将本次checkout内容提交到develop_temp上
git commit -am“提交信息”

# 切换到需要合并的分支并合并，如有冲突，也仅会是src/routes/test下的冲突
git checkout develop
git merge develop_temp
```

### 二、其他的使用
#### 不重复输入密码进行拉取和提交
- 在工程目录下输入命令即可：
```sh
git config --global credential.helper store
git config credential.helper store
```

#### 取消文件跟踪
```sh
git rm -r --cached . 　　//不删除本地文件

git rm -r --f . 　　//删除本地文件

git rm --cached readme1.txt    删除readme1.txt的跟踪，并保留在本地。

git rm --f readme1.txt    删除readme1.txt的跟踪，并且删除本地文件。
```

#### 直接复制文件夹后的认证问题
##### Windows
&ensp;&ensp;&ensp;&ensp;在直接复制git项目的目录时进行操作后显示认证失败，无法进行拉取之类的操作，需要运行下面的命令重置一下，重新输入自己的用户名和密码（提示，需要以管理员身份运行CMD）：

```sh
# 远程服务端的用户名和密码与当前系统中git保存的用户名和密码有冲突
git config –-system –-unset credential.helper
```

#### 无密码进行拉取和提交
- 在工程目录下输入命令即可：

```sh
git config --global credential.helper store
git config credential.helper store
```

### 切换分支
```sh
git checkout -b dev origin/dev
```

### 删除分支
```sh
git branch -d vo
```

### 版本回退
```sh
git reset --hard adsfkdsjfksjfksfjksdks
```

### 以改动后切换分支
```sh
git stash
git checkout branchname
git stash pop
```

### 删除提交后的文件
```sh
git rm xxx
```

### 复制代码
直接点击页面上的fork即可

### git修改远程仓库地址
```bash
git remote set-url origin [url]
```

### 强制覆盖远程仓库
```bash
git push origin master --force
```

### 初始仓库操作
```
# Git global setup
git config --global user.name "Administrator"
git config --global user.email "admin@example.com"

# Create a new repository
git clone http://f540892fafe8/root/enginelib.git
cd enginelib
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master

# Push an existing folder
cd existing_folder
git init
git remote add origin http://f540892fafe8/root/enginelib.git
git add .
git commit -m "Initial commit"
git push -u origin master

# Push an existing Git repository
cd existing_repo
git remote rename origin old-origin
git remote add origin http://f540892fafe8/root/enginelib.git
git push -u origin --all
git push -u origin --tags
```

### Commit合并
```sh
# 合并
git rebase -i idxxxxxxxxx

# 撤销
git rebase --abort
```

### github代理加速
```sh
# socks5协议，1080端口修改成自己的本地代理端口
git config --global http.http://github.com.proxy socks5://127.0.0.1:1080
git config --global https.https://github.com.proxy socks5://127.0.0.1:1080

# http协议，1081端口修改成自己的本地代理端口
git config --global http.http://github.com.proxy http://127.0.0.1:1080
git config --global https.https://github.com.proxy https://127.0.0.1:1080

# 取消代理
git config --global --unset http.http://github.com.proxy
git config --global --unset https.https://github.com.proxy
```

### forker工程拉取源工程代码
```shell script
set upxxx
git pull {remote-url} {branch}

git pull upstream master
```

### git clone 文件名太长
```shell script
git config --global core.longpaths true
```

### 合并策略
当代码编写完成，但已经落主线代码时，想要将自己的修改提交并作为最新的记录（常规合并是按照commit的时间），可以按照下面的方法操作：

```shell script
git rebase 主线 分支线
```

- [彻底掌握git(三)](https://segmentfault.com/a/1190000021471268)

### 抽取某次提交到其他分支

```shell
git cherry-pick 7fcb3defff
```

#### tag相关的操作
```sh
// 查看tag，列出所有tag，列出的tag是按字母排序的，和创建时间没关系。
$ git tag
v0.1
v1.3

//查看指定版本的tag，git tag -l “v1.4.2.**”
$ git tag -l 'v1.4.2.*'
v1.4.2.1
v1.4.2.2
v1.4.2.3
v1.4.2.4

//显示指定tag的信息
$ git show v1.4

//创建轻量级tag：这样创建的tag没有附带其他信息
git tag v1.0

//带信息的tag：-m后面带的就是注释信息，这样在日后查看的时候会很有用
git tag -a v1.0 -m 'first version'

//我们在执行 git push 的时候，tag是不会上传到服务器的，比如现在的github，创建 tag 后 git push ，在github网页上是看不到tag 的，为了共享这些tag，你必须这样：
git push origin v1.0
或者
//将所有tag 一次全部push到github上。
git push origin --tags

//删除本地tag
git tag -d v1.0

//删除github远端的指定tag
git push origin :refs/tags/v1.0.0

# 创建一个基于指定tag的分支
git checkout -b tset v0.1.0
```

#### 代理设置
```sh
git config --global https.proxy http://127.0.0.1:1080

git config --global https.proxy https://127.0.0.1:1080

git config --global --unset http.proxy

git config --global --unset https.proxy


npm config delete proxy

git config --global http.proxy 'socks5://127.0.0.1:1080'
git config --global https.proxy 'socks5://127.0.0.1:1080'
```

## 配置相关
### 国内云服务器访问Github
可以在hosts中进行配置

```shell
vi /etc/hosts

# 添加下面的内容
140.82.114.4 github.com
```

## 参考链接
- [http://mp.weixin.qq.com/s?__biz=MzA4NTQwNDcyMA==&mid=2650661735&idx=1&sn=9aceac07d272e9202d1b5294f857a5ff&scene=23&srcid=0527tuPapv7riaNZFSHQGe4w#rd](http://mp.weixin.qq.com/s?__biz=MzA4NTQwNDcyMA==&mid=2650661735&idx=1&sn=9aceac07d272e9202d1b5294f857a5ff&scene=23&srcid=0527tuPapv7riaNZFSHQGe4w#rd)
- [http://mp.weixin.qq.com/s?__biz=MzA4NTQwNDcyMA==&mid=2650661762&idx=1&sn=8282241cf7414030f4e1d315a173beb1&scene=23&srcid=0527o8THYaNT8wb7pcE8JS5G#rd](http://mp.weixin.qq.com/s?__biz=MzA4NTQwNDcyMA==&mid=2650661762&idx=1&sn=8282241cf7414030f4e1d315a173beb1&scene=23&srcid=0527o8THYaNT8wb7pcE8JS5G#rd)
- [http://www.admin10000.com/document/5374.html](http://www.admin10000.com/document/5374.html)
- [git忽略特定文件或目录](https://blog.csdn.net/huzhenwei/article/details/7426093)
- [git配置过程中fatal:拒绝合并无关的历史](https://blog.csdn.net/yamanda/article/details/79375698)
- [Git 合并指定文件或文件夹](https://juejin.im/post/5adff0d0f265da0b7f4434dc)
- [一招 git clone 加速](https://juejin.im/post/6844903862961176583)
- [【最新】解决Github网页上图片显示失败的问题](https://zhuanlan.zhihu.com/p/148370694)
- [Git Flow 的正确使用姿势](https://www.jianshu.com/p/41910dc6ef29)
- [GitHub Desktop 拉取 GitHub上 Tag 版本代码](https://www.cnblogs.com/zongsir/p/10292013.html)
- [git将某分支的某次提交合并到另一分支](https://blog.csdn.net/I_recluse/article/details/93619400)
- [Github 中Tag的使用](https://www.jianshu.com/p/36202c29e6ae)