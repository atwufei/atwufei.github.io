---
layout: post
title: Dedupe in Backup
author: JamesW
categories: Backup
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content 
{:toc}

## WHAT
通过识别数据中相同的部分, 并只保留其中的一份拷贝, 这个过程就是去重(Dedupe). 往往
Dedupe和压缩一起使用, 而且都能起到减少数据长度从而提高磁盘利用率, 这是两种不一样的技术.

自从Data Domain把Dedupe技术在Backup领域发扬光大之后, Dedupe就成为了Backup领域的标配.
通常Backup的数据都有大量的重复, 所有Backup解决方案都必须包括这样那样的Dedupe, 而且
现在越来越多的Primary Storage也都引入了Dedupe的功能.

各个Backup厂商的Dedupe技术五花八门, 区别无外乎来自以下几个方面.

## WHERE
Backup的过程其实就是把数据从Source端搬到Target端. 比如你的笔记上安装一个Avamer Client, 
Backup的时候这个Client就会把硬盘上的文件复制到Avamer的后端.

### Target Dedupe

Dedupe的过程在Target端完成. 很多情况会把一台NAS服务器作为后端, 当然亦可以是Tape
或者VTL之类的.最具代表性的产品就是Data Domain的DDR. Backup软件只要把数据写到DDR
上就可以了, 其本身并不需要任何Dedupe相关的处理, 所有的Dedupe操作都是在Target上
完成.这样的好处是用户可以使用原来的任何Backup软件也能够获得Dedupe的好处.
		
### Source Dedupe

与Target Dedupe不一样, Dedupe是在Source端完成的. 一般来说, 这个过程会涉及到与Target
端的一些Metadata的交互, 比如Source端会去询问这些数据的Fingerprint是否在Target存在.
如果存在的话, Source端就不需要传输这些数据了. 当然Source端也是可能做一些Cache的, 这样
即使是这些Metadata的交互都可以节省了, 这样做不只是节省了网络带宽, 同时也能够Offload
Target端Fingerprint的查找, 毕竟绝大多的使用场景是多个Source Backup到一个Target上, 通过
分布计算能够有效减少Target的负担.

带来的好处有：

* 有效减少Source和Target端之间的通信, 当他们是通过WAN连接起来的特别有用, throughput可能
  会有质的提升.
* 有效减少Server端的计算等资源. 对于Dedupe Appliance, 计算资源是非常宝贵的, 
  Dedupe和压缩都需要大量的计算. 可以一定程度上认为这是一个Scale-out架构.
* 在Source端有个Agent后, 可以想象会有更多的可能性, 比如说实时的Virtual Synthetics, 
  如果只是单纯的NAS Target是不太容易实现的.

当然也有其劣势：

* 增加了Source端的负荷, 当Source端不能够提供足够的资源的时候, 会影响其他软件的运行.
* 如果之前的Backup软件不支持Dedupe的话, 需要重新购买新的Backup软件.

Source Dedupe正得到越来越多的青睐.

## WHEN

对于Source Dedupe来说, 并不会存在When的问题, 因为Dedupe在Source端就做了.对于Target Dedupe
来说, 却可以有不同的时间点.

### Post-process vs. Inline

* Post-process是一种很自然而且相对比较容易实现的方法.当数据到达Target的时候, 会首先放置在
  Landing Area上, 这个时候数据并没有Dedupe, 而是存放的原始数据, 在将来的某个时候再进行Dedupe.

* Inline指的是在数据写到磁盘之前就已经做完Dedupe.

### Pros and Cons

* Post-process需要额外的存储空间作为Landing Area, 而且需要注意的是原始数据可能是Dedupe后数据
  的很多倍, 这种情况时会消耗大量的I/O带宽.
* Post-process的瓶颈更可能在于磁盘的带宽因为它需要写更多的数据.对于Inline, 瓶颈更可能来源于
  计算相关的资源.当然还会有很多其他的因素, 比如说Fingerprint的查找等等.
* Post-process往往可以推迟到某个合适的时间点, 而Inline Dedupe却需要及时的处理Ingest进来的数据.
  从这个角度看, Inline Dedupe似乎更有挑战性.往往Inline Dedupe都需要使用到像NVRAM这种设备来
  提高性能.
* Post-process因为有Landing Area, 虽然它消耗了大量的存储空间和I/O带宽, 但它也不是一无是处的.
  特别现在越来越多Restore需求, 如果想要保证Restore的RTO, 最好的情况就是能够直接读回原始数据.
  而对于Inline Dedupe, 所有的数据都是Dedupe过的, 从而需要Rehydrate, 这可能会延长Restore的时间.
  以前做Inline Dedupe似乎觉得比Post-process高级那么一点点, 但所谓的优势, 不过是找到更适合自己
  的场景.

## HOW

几乎所有的Backup产商都需要实现Dedupe能够, 但是各有各的不同.

### Fingerprint

几乎所有的Dedupe系统都会实现Fingerprint. 当新数据到来时, 为了查找系统是否已经保存过该数据, 简单
的字符串匹配什么的肯定是不能满足需求的, 必须实现至少一种快速检索的方法.

Fingerprint能够描述数据的特征, 往往原始数据的长度会是Fingerprint的很多倍, 比如说一个20B的SHA1
值来代表8KB的数据.不同要求的Fingerprint需要不同的算法.

* Fingerprint只用来描述相同性, 而不用来描述相似性. 也就是说, 不同的数据不应该产生相同的Fingerprint, 
  即使是相似的数据, 它们的Fingerprint不能相同或者是毫不相干. SHA1就是这样的一种算法, 即使相似的数据, 
  产生的结果却有很大的变化. 而且对于已知的SHA1值, 你很难再构造出另外一个数据. 很多系统都是简单的认为
  数据和SHA1是一对一映射的, 虽然理论上是不成立的, 但是实际使用上往往不会有问题, 因为两个不同数据段
  对应到同一个SHA1概率实在太低.

* Fingerprint用来描述相似性. 对于相似的数据, 该算法需要产生相似的Fingerprint. 最简单的算法, 比如说
  抽出数据的第0, 10, 20, ...个字节.这种算法可能应用于粗力度的选择, 比如在Scale-out的环境下, 选择把
  数据存到哪个Node上时, 就可能使用这种方法.

### File level

这种方式比较原始, 可以通过文件名, 也可以对文件做个Hash, 简单.


### Fixed Length

这是最简单的Sub-file Dedupe方法. 对于一个数据流, 我们把它分成一份一份相同大小的Segment, 比如说4KB, 
然后把这些Segment跟已有的数据比较, 如果已经存在就不再需要重复存储, 只需要通过指针来引用.

这种方法的好处就是简单, 在某些Filesystem和Primary Storage有应用. 当然它的劣势就是Dedupe Ratio
可能不会太高, 它比较适合数据是Appended而不是Inserted. 所以如果一个Dedupe系统只是实现了这种方法,
那是没有竞争力的.

### Variable Length

这可能是最主流的一种Dedupe算法了. 上面提到Fixed Length并不能很好的Dedupe数据, 举个例子, 在文件的
开始位置插入了一个字节, 这会使得Fixed Length完全失效, 因为所有的Segment都不再相同了. 而如果使用
Variable Length的话, 就有可能发现这个问题, 并且通过切片（Anchor）算法把第一个4K+1B作为一个Segment, 
而后的所有4KB数据还像之前一样分片, 从而获得高Dedupe Ratio.

这种算法的主要挑战就是怎么切片, 切片算法必须做到快和准.

* 性能是切片算法的一个重要指标, 往往一个Backup Window需要备份大量的数据, 如果切片算法不够快的话就满足不了要求.
* 切片的位置决定了Dedupe Ratm://github.com/atwufei/atwufei.github.iotio, 如果在不该切的地方切了, Dedupe的目的就没有达到.

Rabin算法应该是应用最广的切片算法. 想实现一个高效的算法，一个基本思想就是要充分利用已完成的计算.
人生也是一样, 随风飘荡地做着布朗运动, 不能利用之前的经验教训, 成功就是偶然的.
Rabin算法也没有两样, r(s[0..m])可以O(1)时间计算出r(s[1..m+1]). 虽然说我们可以使用Rabin算法的结果
直接作为Fingerprint, 但是这个不可约多项式需要保密, 否则就可能受到攻击. 所以一般情况都是通过Rabin
来切片, 然后使用SHA1之类计算Fingerprint.

解决了切片效率算法的问题, 基本上就实现了Variable Length Dedupe. 当然因为这个变长, 在存储的时候
也会带来麻烦.

### Others

除了这些常用的算法, 也会有一些新的算法出现, 特别是当一些新的公司进入Backup领域.

#### (WD) Arkeia Progressive Dedupe

在2010左右, 该公司发布了使用该算法的产品, 号称Dedupe Ratio比其他产商都好, 当然知道现在也没有真实数据.
直到现在, 其市场占有率都不值一提. 虽然技术不是市场的唯一决定因素, 如果真的能比其他产商更高的Dedupe Ratio,
Backup领域还是有机会的. 这里有一篇该算法的[Whitepaper](http://s.nsit.com/fr01/fr/content/shop/arkeia/arkeia-whitepaper-dedupe-technology-2011.pdf),
虽然没有详细的实现步骤, 我觉得它的主要问题是高估了Variable Length算法的复杂性, 又低估了其有效性.

#### ExaGrid Zone Level Dedupe

ExaGrid提供的是Scale-out的Appliance. 对于Scale-out的系统, 其中的一个挑战就是Global Dedupe.
虽然现在网络带宽越来越快, 但是Latency还是跟本地系统没法比, 特别是现在的趋势是使用SSD存Metadata.
又想Dedupe又要避免网络访问, 最好的办法就是把相似的数据放在同一个Node上. 那么问题来了, 怎么判断
Ingest Data跟哪个Node上的数据是相似的呢? 具体算法不得而知, 但是就像上面提到过的, 相似的数据需要
有相似的Signature/Fingerprint.

## End

Dedupe是Backup领域的一个永恒主题. Dedupe的理论和实践这么多年来幷没有大的提升, 但是随着新的应用
场景比如VM环境, 新的硬件出现比如SSD, Scale-out的需求, Restore的需求, copy management, integrated,
converged, 这事还远没有完.

_TODO: 加入图片_
