---
layout: page
title: 在Eclipse中使用Git
---

## 说明
本文介绍如何从Gitlab上将repo拷贝到本地并将Eclipse的多个项目与Gitlab上repo进行同步。

## 创建新分支
首先，Gitlab上的repo会有master分支，假设这个分支上已经有了许多文件，如：

![img1](https://raw.githubusercontent.com/lsr1991/lsr1991.github.io/master/image/2015-06-20-Using-Git-in-Eclipse-1.png)

在repo`networkqualityanalyze`的master分支下已经有`web`目录和`.gitignore`文件。web目录可能是别人的Eclipse工程目录或者其他的什么。现在要先创建自己的分支，因为master分支可能是其他人创建的，不希望你的文件同步到这个分支上。创建分支如下：

![img2](https://raw.githubusercontent.com/lsr1991/lsr1991.github.io/master/image/2015-06-20-Using-Git-in-Eclipse-2.png)

其中，最右侧的下拉列表里New branch就是创建新分支，这个分支以自己的名字命名，并从master分支继承内容。

![img3](https://raw.githubusercontent.com/lsr1991/lsr1991.github.io/master/image/2015-06-20-Using-Git-in-Eclipse-3.png)

这里需要注意的是：在centos 6.6 + chrome环境下`create branch`按钮会失效，无法创建，Windows上则不会。

创建之后，自己的分支上就有了master分支上的所有内容。因为自己的分支已经跟master分支独立开，所以这些对自己没用的内容可以删除，但这是在不需要将自己的分支和master分支merge的情况下才能删的。

## 将创建的分支拷贝到本地
如果没有将本地账户的ssh key上传到Gitlab上，则需要点击项目首页右上角profile->SSH Keys中添加本地账户的ssh公钥，公钥的生成方法在[这里]。

图1：Profile标签

![img4](https://raw.githubusercontent.com/lsr1991/lsr1991.github.io/master/image/2015-06-20-Using-Git-in-Eclipse-4.png)

图2：SSH Keys标签

![img5](https://raw.githubusercontent.com/lsr1991/lsr1991.github.io/master/image/2015-06-20-Using-Git-in-Eclipse-5.png)

添加完ssh的公钥后，本地账户就可以免密码远程登录Gitlab服务器并将远程repo与本地repo进行同步了。

下面就可以将Gitlab上的分支linshurong拷贝到本地目录上。

执行以下命令：
```
git config user.name "林澍荣" user.email "linshurong@kuanguang.local"
```

上面命令用于配置当前本地账户使用git的用户名和email，当本地账户对任何本地git repo提交了commit，这个用户名和email都会被记录在这次commit中。当push到Gitlab上时，也可以看到关于用户名的记录，如：

![img6](https://raw.githubusercontent.com/lsr1991/lsr1991.github.io/master/image/2015-06-20-Using-Git-in-Eclipse-6.png)

这个配置是账户全局的，也就是说，执行这个命令后，git在本地账户主目录下的配置文件`~/.gitconfig`中的user.name和user.email参数会被修改。如果想要修改当前repo的git配置，可以去除选项`--global`，这样命令就修改当前repo目录下的`.git/config`文件；如果要修改系统的git配置，可以将选项`--global`替换成`--system`，这样命令就修改/etc/gitconfig文件中的参数。`git config`命令一次只会修改一个文件，不会同时修改多个文件，也就是说，修改`~/.gitconfig`文件时并不会把所有repo的`.git/config`文件也修改。这部分内容参考git的帮助文档，执行命令`man git-config`可以查看。

接着执行下面命令：
```
git clone git@gitlab.local:application/networkqualityanalyze.git -b linshurong
```

这样就会将Gitlab上的linshurong分支拷贝到本地了。

![img7](https://raw.githubusercontent.com/lsr1991/lsr1991.github.io/master/image/2015-06-20-Using-Git-in-Eclipse-7.png)

## Eclipse共享项目
这里的Eclipse版本是Kelper。

有时候，一个研发中的项目可能会有多个工程（比如maven工程+scala工程），而这些工程要一起同步到Gitlab分支下，因此需要先为这些工程所属的项目创建一个目录。如上图中的`kafka-spark-streaming-qos`就是一个项目目录：
```
[lsr@lsr1991 kafka-spark-streaming-qos]$ ls
kafka-producer spark-streaming-qos
```

它里面包含了两个工程`kafka-producer`（maven工程）和`spark-streaming-qos`（scala工程）。

这两个工程同步方法如下（这里讲的是同步已有工程，没有工程的话可以先在Eclipse中创建一个）：

1. Eclipse->Window->Show View->other->Git->Git Repositories->OK，打开Eclipse识别到的Git Repo。
2. 如下图所示，1处是添加本地repo到这个视图中，这样Eclipse才能识别这个repo（知道它的位置）并当工程需要同步到这个repo时Eclipse可以找到。2处是复制远程repo并添加到这个视图中，这个功能与之前`将创建分支拷贝到本地`一步的功能相同，可以使用上面的方法，也可以使用这种方法。
![img8](https://raw.githubusercontent.com/lsr1991/lsr1991.github.io/master/image/2015-06-20-Using-Git-in-Eclipse-8.png)
3. 右击工程->Team->Share Project
4. 选择Git->Next
5. 如下图所示，Repository的下拉列表中是Git repositories视图中已经有的repo，选择要同步工程的repo，然后Working directory会自动填上，接着选择要存放工程的路径`Path within repository`（Eclipse会将整个工程移动到这个路径下，而不是复制），这里假设repo是`/home/lsr/test/.git`，那么Working directory就是`/home/lsr/test`，并假设要存放工程的路径是`/home/lsr/test/web-qoe`，而你要同步的工程名称是ProjectA，则在Target Location中会显示`/home/lsr/test/web-qoe/ProjectA`。**注意**，如果路径已是一个Eclipse工程的目录（这里假设web-qoe是一个Eclipse工程目录，它包含`.project`文件），那么会提示类似下面的错误：
```
Can not move project ProjectA to target location /home/lsr/test/web-qoe/ProjectA, as this location overlaps with location /home/lsr/test/web-qoe, which contains a .project file
```

解决这个问题的办法是将存放工程的路径设为`/home/lsr/test`，因为该目录中没有`.project`文件。假如在repo的主目录下已经有`.project`文件，则可以在主目录下新创建一个工程目录，把主目录下的所有已有工程文件和`.project`文件一并移到这个新创建的目录下，再选择repo主目录作为存放新工程的路径。

![img9](https://raw.githubusercontent.com/lsr1991/lsr1991.github.io/master/image/2015-06-20-Using-Git-in-Eclipse-9.png)
6. 配置好Git repo之后，Eclipse->Window->Show View->Navigator，该视图会显示所有工程的文件系统中的目录和文件（Package Explorer视图只会显示部分文件）。如果这个视图中被同步的工程目录下有文件的右下角有?号，说明该文件没被添加到Git repo的索引中，选中这些文件右击后Team->Add to Index可以把它们添加到Git的索引，该工程除了`bin`目录以外的所有文件都可以被添加到本地repo的索引中（这是因为在repo主目录中的`.gitignore`文件中指定了`bin`目录不作为同步对象，如果还需要排除其他目录或文件，可以在`.gitignore`中添加）。然后就可以右击工程->Team->commit提交目前为止对repo的更改了。commit时可以同时选择push到远程分支。