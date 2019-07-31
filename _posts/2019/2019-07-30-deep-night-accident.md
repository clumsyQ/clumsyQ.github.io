---
layout: post
title: 深夜生产事故，人工多线程来救场！
category: it
excerpt: 什么才是优秀的程序员？
---

​有一个读者问我：你认为一个程序员具备什么样的能力，才算得上是厉害的程序员？

我答：拥有解决问题的能力的程序员。

这个回答貌似有点抽象，不要紧看下面的文章你会慢慢有所了解。
 
## 一、解决问题的能力

很多年前，当我还是一个小菜鸟的时候，我的领导经常告诉我，解决问题的时候，不要局限于技术本身，并且形象的给我举了一个例子。

有一次两个程序员一直讨论，如何判断两台服务器之间是否网络正常，争争吵吵了很久。旁边的一个测试说，Ping 一下不就知道了吗？就这样他们用 Java 代码实现了 Ping 操作解决了这个问题。

多年以后，虽然我知道有更优雅的方式来解决这个问题，但是我仍然觉得之前的那个测试人员很聪明。后面我们持续打过一年交道，她能力真的很强，在小公司相当于产品经理+测试的职能。

需要给大家说明的是：解决问题的能力和技术能力是两个能力区间，我见过很多程序员源码玩得一溜，生产出现问题的时候仍然不知道如何去解决问题。

生产出现问题的时候，是考验一个程序员的最高水准，在面对高强度高压力下，动作不变形，能够冷静思考、分析、解决问题，能达到这个水平的程序员，这在古代可以拜为上将军。

我一直非常喜欢能够快速解决问题的程序员，我也乐于在各种生产出现问题的时候，第一时间去研究去分析。说一句不厚道的话，好程序员都是在解决问题中锻炼出来的，特别是生产环境出现问题时，能够站出来的程序员。

## 二、给大家分享一个深夜技术故事

### 01. 老平台和新平台

公司有一个老系统，一个新系统。

老系统使用了很多年，早已经超出了它能支持的极限，最早在2013年上线这套系统的时候，预估每天的交易量在一两个亿，实际上现在每天已经跑出了 40 亿的交易量。

从2013-2017年，技术团队做了很多努力，老系统使用的是 Oracle 数据库，为支持最大的交易量，读写分离分库分表+各种最强硬件搞上；系统拆分、重构、优化了很多次，仍然满足不了公司日益增长的交易量。

说实话，团队能把一个老系统整成这个样子，也确实不易。最初的架构设计不合理，缝缝补补终究解决不了大问题。研发新平台成为公司发展必须要做的一件事情，新平台一期设计可以支撑日百亿的交易量，最重要的是支持后期日千亿的扩展。

经过兄弟们的艰苦奋站，新平台终于上线了。新平台上线是成功的一小半，后面的数据迁移才是最最重点的事情。
运行了好几年的老系统，使用的是传统的垂直架构，（架构演进可以参考这篇文章：[从架构演进的角度聊聊Spring Cloud都做了些什么？](http://www.ityouknow.com/springcloud/2017/11/02/framework-and-springcloud.html)），各种业务、政策、活动、风控都揉在了一起。

新平台使用的微服务架构，光微服务就搞了上百个，数据库 Mysql HA。两个系统在架构设计上隔了一代，在设计时为了兼容老系统的部分功能，还做了部分沉余设计，反正这两个系统就不是一个时代的产物。

迁移的要求是，从老平台迁移到新平台的时候，不能影响到商户的正常交易。打个比喻就相当于，你开着车在高速公路上跑，在行驶的过程中换掉车轮，而这个过程还得让坐车的你还不能有任何感觉。

于是我们研发了一套迁移系统，本来计划着一批一批的迁移，给新平台一次切一两个亿的交易量，慢慢看看效果再根据节奏来走，但是突然来的一次政策（活动）打乱了这个节奏。

### 02. 新政策带来的变化

对于第三方支付公司来讲，经常会随着市场环境推出一些新政策（活动），有些政策比较简单，但大多数政策很复杂，往往需要很大的开发量。

当时新平台已经切换了一段时间，大家慢慢对新平台有了一定的信心，就决定在这个新政策在新平台实施。计划在执行政策的当天晚上，把其中一个老平台上剩余的商户全部迁移到新平台。

方案定下来之后，各部门开始各司其职，运营中心对外发通知，我们要在元旦的时候搞个大动作，可能会有什么样的变化；营销中心负责联系各代理进行分批培训；商务部门开始出公司的红头文件，下发各分公司。

我们提前和客服、运营部门做好沟通，可能会面临哪些问题提前做预案；公众号、公司官网、App、邮件对外通知政策变化，公布开始执行日期；产品中心负责政策落地需求梳理，研发中心开发新政策确定的方案。

最最最重要的是，要确保元旦晚上可以把剩余几百万的商户，一次性平稳的迁移到新平台。

### 03. 半夜开始迁移

迁移程序之前已经执行了很多次，所以大家对这块相对比较放心，但仍然和主要负责迁移的同事确认了好多次，开发环境提前两周必须测试完毕，UAT环境需要在迁移一周前测试完毕，研发和测试双验证。

直到距离迁移还有三天的时候，我还专门找到负责迁移的那名程序员了解进展，问有没有在生产上进行过模拟测试。确认没有问题后，根据主负责人反馈的时间预估三四个小时可以迁移完毕，这样凌晨 1：00 开始，凌晨 4：00-5：00 之间可以迁移完毕。

在真正执行迁移的前一天，又拉着各部门做了一次沟通会，大家一起讨论可能出现的各种情况，以及各部门需要留守的人员。开完会之后，大家感觉都还不错，静等晚上这一场大战！

当天晚上留下来十几名开发和两名测试，以及一些其它部门的同事，大概二十几人左右，12点之前大家说说笑笑，打打游戏静等凌晨 1 刻迁移，因为刚好是元旦，办公室一片过节的感觉。

**时间过得很快，凌晨1点的北京，窗外星光点点，办公室内一片紧张。**

十几名同事都围在了主要负责迁移的这名程序员旁边，能明显感觉到这名程序员很有压力（哈哈，我估计这种事情放谁身上都会有压力）。不过他还是熟练的按照之前多次测试的那样，核查了多遍数据之后，点击迁移按钮。

首先在生产环境迁移一个代理商，看看数据是否正确，执行完毕后相关人员开始核验数据。运维人员核查日志，开发人员确认相关节点正常、数据库工程师核对迁移数据；测试人员在运营平台查询数据核验、测试 Pos 刷卡测试，一切正常！

试了两个代理商都没有问题，下面就准备 All In 了，剩下几百万商户，上千个代理商就计划一把梭了。负责迁移的程序员，将所有代理商编号，配置到执行程序中，点击了执行按钮，生产跟踪了一下日志，一切正常。

留下几个人监控数据，其他人就散了，等迁移完成后再进行后续工作。我也回到了工位，点起了一根烟，想着今晚还比较顺利。

### 04. 突发事故

凌晨的夜晚比较困，当我点起第三根烟的时候，负责迁移的这位程序员，急匆匆的跑过来找我了。

“强哥，出现问题了！”

心中一惊，猛吸一口烟，把烟掐灭，忙问到：“出现啥问题？”

原来这位程序员在迁移程序执行后，就一直在跟踪迁移的进展，发现过了半小时才迁移了10万商户，老平台总共几百万商户，按照这个速度，全部执行完需要几天后。

**这个事可大了！**

如果在上午8：00 之前不搞定这个事情，那就完全是重大事故了。

先不说怎么处理新老平台数据割裂，如果公司政策推迟执行，怎么在这么短的时间内把信息通知到几百万商户、几千个代理商，就是一个不可能完成的工作量。

可以想象第二天都会出现什么样的状况，客服400电话被打爆、运营人员沟通到吐血，因政策推迟执行可能导致的公司损失，针对代理商的补偿行为...

如果这个问题我们没有在一个小时内解决掉，就需要立刻上报公司副总经理，然后估计连夜公司所有的管理层，都需要来公司开会商量后续处理方案。

大脑中虽然闪过迁移失败后的严重后果，但眼前还需要压下所有的想法，先分析到底是哪里出现了问题，有没有什么样的降级方案或者补救方法。

**分析原因：**

经过查询日志、核对数据基本查明了原因，开发人员在生产测试的时候，都使用的是中小型代理商进行的测试；但忽略了公司不同代理商规模之间差异极大，最大的核心代理商一家的数据，可能占平台整体交易量的5%-6%。

所以根据中小型代理商评估的时间肯定是不准确的，事已至此先不说谁的问题。如何快速解决问题才是接下来的关键，大家都一起想解决方案，有什么办法可以让迁移速度更快一点。

**补救方案：**

比如先同步核心数据，其它内容后续再进行处理，先保障第二天的交易；比如可不可以全部使用人工导表来处理，数据库工程师听到这个方案的时候，差点哭晕过去，上千多张表，关系极为复杂；其它各种各样的方案..

在大家七嘴八舌讨论优化方案的时候，才发现迁移程序的主流程没有使用多线程来迁移。

迁移程序提供了一个界面，每次迁移的时候开发人员会在页面填写需要迁移的代理商编号，后台接收到页面传递的参数后，开始 for 循环执行迁移。
虽然代理商下的商户使用了多线程迁移，但是迁移代理商的主程序入口，却没有使用多线程，因此大家想是否把代理商这块也用多线来加快迁移速度。

### 05. 人工多线程救场

大家讨论之后，觉得多线程来迁移代理商应该是目前比较好的一个方案，但是如果让现场写，没有经过测试直接就生产执行，风险还是比较大。

那还有什么不用改程序就可以实现这种代理商并发迁移的效果吗？确实有！

大家知道我们平时开发的 Web 应用，前台的每一次请求到后端就会分配一个 Servlet 来处理响应，这个  Servlet  其实就是一个独立的线程。那么每次多打开几个页面，同时执行迁移请求不就实现了多线程迁移代理商的效果吗？

说干就干，把之前的迁移程序停掉之后，选择十几个代理商进行多线程迁移测试，同时打开了4个页面，每个页面输入不同的代理商，开始迁移测试，测试后发现一切正常。

开始加大测试量，使用几十个代理商，在不同的页面输入后，先后点击了迁移程序，在第二次并发迁移的过程突然发现不时的会报一些错误。

停止迁移程序，开始寻找原因，根据报错的原因发现是出现共享数据了。

我们知道 Servlet 是线程不安全的，当出现多线程访问的时候，如果有全局共享变量就会出现线程安全问题。

这个问题好解决，使用 ThreadLocal 来修饰就行，ThreadLocal 为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。

这个问题解决之后就继续打开多个页面执行，但同一个 Tomcat 并行超过 6 个线程的时候，机器负载就会比较高，因为每个线程内还会再次调起另外的线程池来处理商户、业务员的迁移逻辑。

于是就立刻安排运维人员，在生产环境找十台服务器，在这十台服务器上都部署上迁移的主调度程序。为了防止开发人员手抖出现问题，我让运维给我开了权限。

于是在我的电脑上（我使用了多个屏幕），分别打开了十台服务器上的迁移程序页面，把所有需要迁移的代理商按照每次十五个分组，每次在一个页面输入一组代理商来迁移，如此循环依次在每台服务器开始迁移代理商。

当我循环执行了6次的时候，数据库工程师检测到明显数据的迁移速度加快，就这样我用了两个小时，在页面把所有的代理商分别进行了迁移。

大概到凌晨 4 点的时候，我的工作基本上搞完了，剩下的就让程序慢慢跑了；凌晨 5 点的时候，大部分商户数据都已经迁移过来，只剩下两台服务器还在继续跑；到了凌晨 6 点的时候，十台服务器的迁移程序全部跑完。

安排把所有相关数据一一进行了核对后，大家长长的舒了一口。

早上 7 点一起下来吃早餐的时候，大家还在说，感觉昨天晚上差点就过不去了。开玩笑说，如果凌晨2、3点的时候，给我们老板打电话，老板会是什么样的感觉。

那时候想，出现这么大的事故，老板把我们开除了都是小事，如何收场才是我们最关心的。工作丢了可以再找，事情不管怎样都是需要我们这批人来解决处理的。

上午 9 点打开交易后，又陆陆续续出现了一些小问题，但都是小面积的、不影响交易的问题，整体范围可控，晚上迁移的这帮人，几乎都坚持到了下午没有太大问题了才回去睡觉。

**事后回想时，大家都一种劫后余生的感觉。**

## 三、事件回顾

事后我们开复盘会的时候，总结了这里面疏漏的很多点，但这些都不是本文的重点。我们还是回到文章的开头，什么是厉害的程序员？

大家可以看到这个问题并不是特别的复杂，处理时需要的技术手段也比较简单，但最关键是解决了当时最最紧迫的问题。所以说技术没有什么高低之分，学习技术的本质也是为了解决各种各样的问题，不要对技术迷之自信，能用起来才是最好。

技术人要学会享受压力，因为压力就是动力，压力就是让你去成长的，越早遇到成长得越快。人在高压高强度的环境中，哪怕很简单的动作可能都会变形，从而有可能引发更大的二次事故。

在高强度、高压力的环境下稳定保持一颗冷静分析的心，只有你自己沉静下来才能真正的发现问题解决问题。很多技术人，出现问题时你看他在忙，其实是没有思路在那瞎操作。

冷静下来，仔细分析整个链条，设想哪个地方可能会出现问题，然后通过查询日志或者相关命令，一步一步去排查、验证问题的根源在哪里，只有真正找到了问题的根源，你解决的时候才可以胸有成竹。

当天晚上留守迁移的程序员，都是我司最核心的一批程序员，但是谁有能力上谁有能力下，在这一晚上很容易就能发现，优秀的程序员就像金子一样，关键的时刻它会发光的。

很多人遇到问题就自然的就会后退几步，有的人遇到问题就喜欢冲上去。不管你平时源码研究得多牛逼，不管你的 PPT 写得有多好，公司需要的是遇到问题的时候，有人能够顶上去把问题解决掉。

凡是能够在关键时刻顶上去的程序员，基本上后面都很容易走上管理岗位。人和人其实就是在不断磨合中建立信任的，领导在选择提拔员工时，主要考虑就是能不能放心把事情交给你。

所以大家平时研究技术的时候，不要走偏，源码、设计模式这些东西是应该研究，但更应该考虑的是研究后如何去应用，多专注一些实战型的知识，这些东西关键时刻可以救你的命（职场）。

## 四、如何做一名有能力的程序员

那么作为一名程序员，如何培养自己解决问题的能力呢？**实践！实践！实践！**平时的技术学习只是一种强力输入，如果不进行实践，这些能力就会很快流失了。

那如何实践呢，多做项目，如果公司的项目用不到此技术，可以自己业余时间写写代码自己去调试一番；另外同事出现问题的时候多去帮忙解决问题，公司出现问题的时候，主动去帮忙解决问题；解决各种各样的问题，是提升能力的最快方式。

实践完成之后，最好还能复盘总结一番，把总结的内容作为日志或者博客记录下来。记录下来的内容就会成为你的一个知识宝库，以后遇到类似问题的时候，检索一下即可解决，如此不断丰富自己解决问题的经验。

**最后，愿你成为一名真正的技术大拿！**









