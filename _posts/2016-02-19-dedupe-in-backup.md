---
layout: post
title: Dedupe in Backup
categories: Backup
---

* content 
{:toc}

## WHAT
通过识别数据中相同的部分，并只保留其中的一份拷贝，这个过程就是去重(Dedupe)。往往
Dedupe和压缩一起使用，而且都能起到减少数据长度从而提高磁盘利用率，这是两种不一样的技术。

自从Data Domain把Dedupe技术在Backup领域发扬光大之后，Dedupe就成为了Backup领域的标配。
通常Backup的数据都有大量的重复，所有Backup解决方案都必须包括这样那样的Dedupe，而且
现在越来越多的Primary Storage也都引入了Dedupe的功能。

各个Backup厂商的Dedupe技术五花八门，区别无外乎来自以下几个方面。

## WHERE
Backup的过程其实就是把数据从Source端搬到Target端。比如你的笔记上安装一个Avamer Client，
Backup的时候这个Client就会把硬盘上的文件复制到Avamer的后端。

### Target Dedupe

Dedupe的过程在Target端完成。很多情况会把一台NAS服务器作为后端，当然亦可以是Tape
或者VTL之类的。最具代表性的产品就是Data Domain的DDR。Backup软件只要把数据写到DDR
上就可以了，其本身并不需要任何Dedupe相关的处理，所有的Dedupe操作都是在Target上
完成。这样的好处是用户可以使用原来的任何Backup软件也能够获得Dedupe的好处。
		
### Source Dedupe

与Target Dedupe不一样，Dedupe是在Source端完成的。一般来说，这个过程会涉及到与Target
端的一些Metadata的交互，比如Source端会去询问这些数据的Fingerprint是否在Target存在。
如果存在的话，Source端就不需要传输这些数据了。当然Source端也是可能做一些Cache的，这样
即使是这些Metadata的交互都可以节省了，这样做不只是节省了网络带宽，同时也能够Offload
Target端Fingerprint的查找，毕竟绝大多的使用场景是多个Source Backup到一个Target上，通过
分布计算能够有效减少Target的负担。

带来的好处有：

* 有效减少Source和Target端之间的通信，当他们是通过WAN连接起来的特别有用，throughput可能
  会有质的提升。
* 有效减少Server端的计算等资源。对于Dedupe Appliance，计算资源是非常宝贵的，
  Dedupe和压缩都需要大量的计算。
* 在Source端有个Agent后，可以想象会有更多的可能性，比如说实时的Virtual Synthetics，
  如果只是单纯的NAS Target是不太容易实现的。

当然也有其劣势：

* 增加了Source端的负荷，当Source端不能够提供足够的资源的时候，会影响其他软件的运行。
* 如果之前的Backup软件不支持Dedupe的话，需要重新购买新的Backup软件。

Source Dedupe正得到越来越多的青睐。

## WHEN

对于Source Dedupe来说，并不会存在When的问题，因为Dedupe在Source端就做了。对于Target Dedupe
来说，却可以有不同的时间点。

### Post-process vs. Inline

* Post-process是一种很自然而且相对比较容易实现的方法。当数据到达Target的时候，会首先放置在
  Landing Area上，这个时候数据并没有Dedupe，而是存放的原始数据，在将来的某个时候再进行Dedupe。

* Inline指的是在数据写到磁盘之前就已经做完Dedupe。

### Pros and Cons

* Post-process需要额外的存储空间作为Landing Area，而且需要注意的是原始数据可能是Dedupe后数据
  的很多倍，这种情况时会消耗大量的I/O带宽。
* Post-process的瓶颈更可能在于磁盘的带宽因为它需要写更多的数据。对于Inline，瓶颈更可能来源于
  计算相关的资源。当然还会有很多其他的因素，比如说Fingerprint的查找等等。
* Post-process往往可以推迟到某个合适的时间点，而Inline Dedupe却需要及时的处理Ingest进来的数据。
  从这个角度看，Inline Dedupe似乎更有挑战性。往往Inline Dedupe都需要使用到像NVRAM这种设备来
  提高性能。
* Post-process因为有Landing Area，虽然它消耗了大量的存储空间和I/O带宽，但它也不是一无是处的。
  特别现在越来越多Restore需求，如果想要保证Restore的RTO，最好的情况就是能够直接读回原始数据。
  而对于Inline Dedupe，所有的数据都是Dedupe过的，从而需要Rehydrate，这可能会延长Restore的时间。
  以前做Inline Dedupe似乎觉得比Post-process高级那么一点点，但所谓的优势，不过是找到更适合自己
  的场景。

## HOW

几乎所有的Backup产商都需要实现Dedupe能够，但是各有各的不同。

### Fingerprint

几乎所有的Dedupe系统都会实现Fingerprint。当新数据到来时，为了查找系统是否已经保存过该数据，简单
的字符串匹配什么的肯定是不能满足需求的，必须实现至少一种快速检索的方法。

Fingerprint能够描述数据的特征，往往原始数据的长度会是Fingerprint的很多倍，比如说一个24B的SHA1
值来代表8KB的数据。不同要求的Fingerprint需要不同的算法。

* Fingerprint只用来描述相同性，而不用来描述相似性。也就是说，不同的数据不应该产生相同的Fingerprint，
  即使是相似的数据，它们的Fingerprint不能相同或者是毫不相干。SHA1就是这样的一种算法，即使相似的数据，
  产生的结果却有很大的变化。而且对于已知的SHA1值，你很难再构造出另外一个数据。很多系统都是简单的认为
  数据和SHA1是一对一映射的，虽然理论上是不成立的，但是实际使用上往往不会有问题，因为两个不同数据段
  对应到同一个SHA1概率实在太低。
  
* Fingerprint用来描述相似性。对于相似的数据，该算法需要产生相似的Fingerprint。最简单的算法，比如说
  抽出数据的第0, 10, 20, ...个字节。这种算法可能应用于粗力度的选择，比如在Scale-out的环境下，选择把
  数据存到哪个Node上时，就可能使用这种方法。

### Fixed Length

这是最简单直接的Dedupe方法。对于一个数据流，我们把它分成一份一份相同大小的Segment，比如说4KB，
然后把这些Segment跟已有的数据比较，如果已经存在就不再需要重复存储，只需要通过指针来引用。

这种方法的好处就是简单，在某些Filesystem和Primary Storage有应用。当然它的劣势就是Dedupe Ratio
不会太高。所以如果一个Dedupe系统只是实现了这种方法，那是没有竞争力的。

### Variable Length
