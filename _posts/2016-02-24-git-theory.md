---
layout: post
title: Git - 入门到进阶
author: JamesW
categories: Git
---

* content 
{:toc}

Version Control System (VCS) 已经有很长一段历史了, 之前一直都是群雄逐鹿.
Git的出现改变了这种状况, 虽然还有些公司仍然在用其他VCS工具, 比如Perforce,
不过完全是不得已而为之. 老的项目可能因为这种那种原因不得不继续使用以前的工具,
对于新的项目, 绝大多数人都会认真考虑Git, 甚至写博客都开始使用Github Pages了.

## 基本概念

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
	Git的历史记录. 顺便说一下, Snapshot能极大地提升访问速度, 这意味着任意版本
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

![](https://github.com/progit/progit2/blob/master/book/02-git-basics/images/lifecycle.png?raw=true)

### 版本号

git log -S
git log --since
$ git remote show origin
HEAD (Git branching)
In Git, HEAD is a pointer to the local branch you’re currently on
$ git log --decorate --oneline
fast-forward
multiple parents
git rebase -i Make a list of the commits which are about to be rebased. Let the user edit that list before rebasing

