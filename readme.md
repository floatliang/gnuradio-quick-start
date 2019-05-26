# USRP及GNU Radio入门教程

主要包括环境搭建、一些基础知识文档、一个小的实战训练以及一些常用的参考网站。

### 环境搭建

USRP B210即插即用，只要有USB口就行，没什么好说的。

GNU Radio一般运行在linux系统上，这里推荐用ubuntu 14.04或者16.04，其他的版本也可以用，但可能会因为一些库的版本导致兼容问题。

主要需要装两个软件，一个UHD即USRP Hardware Driver，USRP硬件驱动，一个GNU Radio。前者提供USRP的驱动接口，后者可以调用这些接口控制USRP实时通信，也可以离线在电脑上模拟运行一些通信系统。

具体安装步骤见文件夹内：ubuntu14.04(13.10)安装UHD+gnuradio.docx

也可参考文件夹内“GNU Radio教程”第3章：GNU Radio安装

### 入门文档

关于GNU Radio最标准的文档当然是看官方wiki：https://wiki.gnuradio.org/index.php/What_is_GNU_Radio%3F 了解GNU Radio是什么，可以做什么。

以及基础Tutorials：https://wiki.gnuradio.org/index.php/Tutorials，Working with GRC介绍了图形界面的使用，Programming GNU Radio in Python & C++介绍了如何利用GNU Radio写自己的信号处理模块（block）。

先看文档走一遍编写新block的具体流程。

再回想GNU Radio block编写主要知识点：

* 有几种block：source信号源模块、sink接收端模块、sync进多少数据出多少数据，interpolators根据比例对输出数据插值，decimators根据比例减少输出数据。
* 输入输出管道的定义：gr::io_signature::make(1, 1, sizeof(gr_complex))，第一个1定义了输入或者输出最少有几个的管道数量，即有几路输入或者输出端口，体现在图形界面就是左边或者右边那几个小接口，第二个1定义了输入或者输出最多有几个管道，一般第一跟第二大小相同，第三个定义了输入或者输出管道数据的最小单位，这里一个数据大小是一个复数，64字节，对于sync模块来说，输入输出的最小单位数量是相同的，而不是字节数。
* block主要的工作函数：work()和general_work()的区别，这两个的函数是block的主要功能函数，处理输入管道进来的数据，并输出产生的新数据。
* consume_each()，告诉runtime系统这次调用work()函数里消耗了多少输入数据，runtime系统会根据消耗数据安排下次输入数据数量。
* return noutput_items，告诉runtime系统work()函数产生了多少新数据输出。
* 对于sync block来说，消耗的输入跟产生的输出相等，其使用work()。
* 对于自定义的输入输出比，使用general_work()并配合forecast()函数，forecast()用来告诉runtime系统输入与输出数据的比例关系。
* 说了这么多，其实编写好一个block很简单，就是利用前提所提的这些函数处理好输入输出数据的比例关系，然后在工作函数work()里实现该block的功能，把输入数据转化成输出数据。

除了官方文档外，还有一些社区编写的资料，文件夹内GNU Radio教程4.6节介绍了已有的常用block有哪些。在官方文档有不理解的地方也可以来这里面查找一下看有没有解释。

对于USRP B210，插上去就能用，运行GNU Radio的程序，就会自动调用UHD相关接口烧写程序到USRP运行，因此主要搞清楚USRP B210的构造和一些参数。

USRP B210有两块子板，A板和B板相同，各有两个发射天线，TX/RX表示即可作为发送端也可作为接收端，RX表明只可作为接收端。B210的带宽最高可到70Mhz左右，即采样率最高可到这么多，具体记不清楚了，可以去USRP_Documentation.pdf文档里查，这个文档里把General Questions及之前的章节看明白就差不多了。

### 数据分析

GNU Radio非图形化界面提供了File Sink block，可以把通信数据存储下来：

```python
self.connect(self.pk_detect, blocks.file_sink(gr.sizeof_char, "ofdm_sync_pn-peaks_b.dat"))
```

其中connect()用来连接第一个block的输出端口到第二个block的输入端口，因此这里把pk_detect的数据输出到file_sink，file_sink的作用是把数据存在当前文件夹里。

对于图形界面，file_sink的图形化模块即为File Sink这一模块，把输出连接到这一模块即可存储数据，用用就知道了。

数据保存后可以用Octave或者matlab分析数据，不知道怎么用的可以看：https://wiki.gnuradio.org/index.php/Octave

用好了数据分析可以很好的debug每个block的问题出现在哪里，通过分析数据就可以知道该block实现了什么功能，或者要实现的功能有没有达到效果。

### 实战训练

GNU Radio文件夹：gr-digital\examples\ofdm下提供了一个802.11发送和接收的示例代码：benchmark_tx.py以及benchmark_rx.py，第一个是802.11发送端启动脚本，第二个是接收端启动脚本。

这里主要是要修改接收端的定时同步里的峰值检测block，benchmark_rx.py调用了同文件夹下receive_path.py类，receive_path.py调用了gr-digital\python\digital文件夹里的ofdm.py里面的ofdm_demod类，ofdm_demod类调用了同文件夹下ofdm_receiver.py类，ofdm_receiver.py调用了同文件夹下ofdm_sync_pn.py类，ofdm_sync_pn.py的主要作用是对接收信号做定时同步。定时同步主要分为两步，第一步将接收信号的preamble跟已知preamble互相关，互相关后在定时点位会产生比较高的峰值；第二步就是找出这些峰值位置，就完成了定时同步。ofdm_sync_pn.py调用了gr-blocks\lib文件夹下的peak_detector_XX这个峰值检测block，对应文件为peak_detector_XX_impl.cc.t，XX第一个X表示输入数据类型，第二个X表示输出数据类型，这里为X表明这是个模板类，可以为任何类型。

这里的主要任务就是模仿峰值检测block新写个相同功能的block，即peak_detector_XX这个block，其实其它地方代码都可以直接复制，主要是修改里面的工作函数，就是work()函数，设计一个自己的峰值检测代码，写好后保存，然后重新编译gnuradio就可以用了，不要求同步准确率，只要能用就行了。

修改过程中出现问题可用上一节数据分析的方法分析数据，对于benchmark_rx.py代码调用结构还不清楚的可以看看这篇文章：http://gnuradio.cc/read.php?tid-622-fpage-2.html

### 常用网站

完成实战训练后基本上GNU Radio和USRP就比较熟悉了，能够编写一些基础的block实现自己想要的功能，如果平时碰到啥解决不了的问题，下面是一些常用的网站，一般可以帮上忙。

官方文档，最有用，最权威的解释：https://wiki.gnuradio.org/index.php/Main_Page

GNU Radio编程的一些接口文档，所有函数和block都能在这里面查到：https://www.gnuradio.org/doc/doxygen/index.html

中文GNU Radio论坛，不算活跃，但也能查到一些问题：http://gnuradio.cc/index.php?m-bbs.html

如果上面都解决不了，可以去mailing list上查找或者提问：https://wiki.gnuradio.org/index.php/MailingLists