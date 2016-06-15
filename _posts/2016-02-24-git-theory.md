---
layout: post
title: Git Intro
author: JamesW
categories: Git
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content 
{:toc}

Version Control System (VCS) 已经有很长一段历史了, 之前一直都是群雄逐鹿.
Git的出现改变了这种状况, 虽然还有些公司仍然在用其他VCS工具, 比如Perforce,
不过完全是不得已而为之. 老的项目可能因为这种那种原因不得不继续使用以前的工具,
对于新的项目, 绝大多数人都会认真考虑Git, 甚至写博客都开始使用Github Pages了.

## Basics

如果只是使用Github Pages来写博客的话, 确实只要知道几个简单命令就可以开始了,
但是想要发挥Git更好的作用, 了解add/commit几个命令是远远不够的.

### Objects

* Blob - 文件的内容. 比如一个.c文件, 里面的代码就是一个Blob. 就像文件系统里,
	文件需要inode来索引, Blob同样需要解决引用的问题, Git会把每个Blob都计算
	出一个SHA-1值, 这个SHA-1就可以认为是该Blob的inode. 由于SHA-1的冲突概率
	非常之小, 以至于可以忽略不计, 所以可以认为SHA-1唯一地对应于一个Blob.
	这个概念同样在Dedupe系统中使用, 只是用处有所不同.

		$ git cat-file -p aac8b4d0018c75cb461d0abaee66c48242f231c6
		content in foo

* Tree - 文件的目录结构. 跟文件系统一样, Git里面的文件是通过Tree组织在一起的.
	Tree里面可以包含文件和子目录. Tree也有对应的SHA-1值. (正常的文件系统是否
	能够借鉴这种方法?)

		$ git cat-file -p master^{tree} 
		100644 blob aac8b4d0018c75cb461d0abaee66c48242f231c6    foo
		040000 tree e1565d7eb2da807b3b27ef38e76549765dd24f15    subdir


* Commit - 对应每一次修改. Commit可以类比于文件系统的Snapshot, 每一个Commit
	都会生成一个Snapshot. Snapshot可以由一颗指向Root的Tree来表示. 当然Commit
	除了指向Tree外, 还包含一些额外的信息比如Author/Committer/Timestamp/Message
	等等, 当然如果不是第一个Commit的话, 还有Parent指针, 通过这个指针可以遍历
	Git的历史记录, 如果是Merge的话, 一个Commit会有2个甚至更多的Parents.
	顺便说一下, Snapshot能极大地提升访问速度, 这意味着任意版本
	的文件都是静态的完整的, 而不需要通过Patch去合成. 虽然这看起来会额外地占用
	一些磁盘空间, 但是Git也提供了Dedupe和压缩的功能. 这同样使用了跟存储领域相似
	的技术, 也就是在Dedupe/Snapshot的时候, 让以前的Snapshot指向最新的Snapshot,
	这样有利于新Commit的访问速度.

		$ git cat-file -p HEAD
		tree 1984939ca6a5cfd4114ecbb32a904de46c5bbbed
		parent 65f0a280addfc4e9b1e29a8df458dce7872083e2
		author Wu Fei <at.wufei@qq.com> 1456323698 +0800
		committer Wu Fei <at.wufei@qq.com> 1456323698 +0800

		add 2nd

有了这3种Objects, Git就可以轻易地表示各个版本.

### States

要想正确使用Git, 理解这些概念是必须的.

* Working Directory - 工作目录, 也就是我们正常看到的, Checkout出来用于编辑的目录.
	只在这里修改过的文件的状态就是Modified.
* Staging Area - 也叫Index, 中间状态的一个目录, 表示下一次Commit的改变.
	通过add等命令stage了的文件会放在这里.
* Git Directory - Git在本地的仓库, 所有已经Commit的内容都放在这里.

<img src="https://github.com/progit/progit2/blob/master/book/02-git-basics/images/lifecycle.png?raw=true" alt="Drawing" style="width: 600px;"/>

## Operations

### Basic

掌握了Git的基本概念之后, 就很容易掌握Git的基本操作. 

首先需要有一个Git本地的仓库, 这可以通过初始化一个本地仓库

	$ git init

或者Clone一个远程的仓库, 比如Clone我的Github Pages:

	$ git clone ssh://git@github.com/atwufei/atwufei.github.io

Git支持很多协议, 这里使用ssh是为了免去每次Commit时输入用户名密码, 当然这需要
先把公钥放到Github服务器上.

有了本地仓库后, 就可以Edit/Commit你的修改了.

	$ vim something
	$ git add something
	$ git commit -m "Add something"

这样就创建了本地的一个Commit, 但是这个Commit还没有同步到Remote Repository上.
如果在Remote上有写权限, 可以通过Push把本地的修改Checkin到Remote上.

	$ git push

当然Remote可能会拒绝这个Push操作, 因为已经有别人先Push过了, 这时候master(假设
在这个Branch上)和remote/master不一致. Git Server并不会主动帮你去Merge, Server
只会接受Fast-forward的Merge, 这个时候就需要

	$ git pull

这个操作会更新remote/master, 同时会对把它Merge进本地master. 如果Merge没有冲突,
那么这个时候就可以再次Push.  只要这个时间窗口没有人再Push了, 这个时候就能成功了,
因为Local的Branch HEAD是Remote Branch的子孙, 这样就可以形成一个Fast-forward Merge了.

### Branch

不到长城非好汉, 没用过Branch也就谈不上熟悉Git. Branch在Git是非常轻量级, 它就是
一个可移动的指针, 指向某一个Commit. 当有新Commit的时候, Branch就移动到这个Commit上.

在本地创建一个Branch, 或者同时创建和Checkout:

	$ git branch
	$ git checkout -b

也可以查看Local或者Remote的Branches:

	$ git branch -a
	* editing
	  master
	  remotes/origin/HEAD -> origin/master
	  remotes/origin/editing
	  remotes/origin/master

这是我Github Pages的Branches, Pages会把master Branch的内容处理然后当成博客呈现出来.
因为我不想show一篇没完的博客, 我会先把未完成的文章都Commit到editing Branch, 然后push
editing到Pages上去. 这样别人就不会看到临时未完成的博客, 而同时又能够把它存到Pages上去.

	$ vim _posts
	$ git add _posts
	$ git commit -m "Add new one"
	$ git push origin editing

当博客写完了, 我可以先把editing Merge到master, 然后在把master Push到Remote Server上,
这样就把博客发布了.

#### Merge/Rebase

一般来说, 把一个Branch的Commits搬到另一个Branch上主要有2种方法: Merge和Rebase.
比如我们想把topic的内容合并到当前的Branch master上

                     A---B---C topic
                    /
               D---E---F---G master

如果使用Merge的方法, 我们会得到这样的结果

	$ git merge topic
                     A---B---C topic
                    /         \
               D---E---F---G---H master

如果使用的是Rebase的方法

	$ git rebase master topic
                             A'--B'--C' topic
                            /
               D---E---F---G master

可以看到Rebase后的History更加干净, 变成线性了, 也没有额外的Merge Commit.
这两种方法没有好坏之分, 全看个人喜好. 需要注意的是, Rebase是修改了历史的,
如果之前的历史已经被其他人/Commit引用了, 那会带来不必要的麻烦. 对于Merge,
因为不存在修改Commit, 也就没有这个问题.

由于Rebase能够修改记录, 所以还能实现很多其他的功能.

	* --onto 选项
	* -i 选项

当然并不是只有Rebase才能修改历史, 比如git commit --amend就能修改最近的一个Commit.

### Reset

刚接触Git Reset可能觉得不是很容易理解, 其实很简单:

	1. MOVE HEAD, --soft
	2. UPDATING THE INDEX, --mixed
	3. UPDATING THE WORKING DIRECTORY, --hard

当然如果后面指定了具体的路径, 显然Step 1已经不可能执行了, 这时只会有后面2步.

### Revision

[Answer in stackoverflow](http://stackoverflow.com/questions/23303549/what-are-commit-ish-and-tree-ish-in-git)

[Manpage](https://www.kernel.org/pub/software/scm/git/docs/gitrevisions.html)

	----------------------------------------------------------------------
	|    Commit-ish/Tree-ish    |                Examples
	----------------------------------------------------------------------
	|  1. <sha1>                | dae86e1950b1277e545cee180551750029cfe735
	|  2. <describeOutput>      | v1.7.4.2-679-g3bee7fb
	|  3. <refname>             | master, heads/master, refs/heads/master
	|  4. <refname>@{<date>}    | master@{yesterday}, HEAD@{5 minutes ago}
	|  5. <refname>@{<n>}       | master@{1}
	|  6. @{<n>}                | @{1}
	|  7. @{-<n>}               | @{-1}
	|  8. <refname>@{upstream}  | master@{upstream}, @{u}
	|  9. <rev>^                | HEAD^, v1.5.1^0
	| 10. <rev>~<n>             | master~3
	| 11. <rev>^{<type>}        | v0.99.8^{commit}
	| 12. <rev>^{}              | v0.99.8^{}
	| 13. <rev>^{/<text>}       | HEAD^{/fix nasty bug}
	| 14. :/<text>              | :/fix nasty bug
	----------------------------------------------------------------------
	|       Tree-ish only       |                Examples
	----------------------------------------------------------------------
	| 15. <rev>:<path>          | HEAD:README.txt, master:sub-directory/
	----------------------------------------------------------------------
	|         Tree-ish?         |                Examples
	----------------------------------------------------------------------
	| 16. :<n>:<path>           | :0:README, :README
	----------------------------------------------------------------------

## End

Git还有很多有用的功能, 需要的时候可以很容易找到相应的文档, 这里主要介绍一下
基本概念, 有了这些知识, 就可以很容易理解其他的文档.
