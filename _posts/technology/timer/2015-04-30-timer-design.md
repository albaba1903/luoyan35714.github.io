---
layout: post
title:  定时器之（一） - 定时器模块初稿设计
date:   2015-04-30 17:15:00 +0800
categories: 技术文档
tag: 定时器设计
---

* content
{:toc}


定时器设计
-----------------------
前段时间设计架构需要设计一个定时器的公共模块。从需求初稿到最终确定版。中间经历了多重磨难，终于完工。记录下期间的心得。

需求
=======================
* 最开始的需求很简单，只是业务人员的一句话 - “做一个定时器功能”，然后问有没有其他需求，不清楚。只是要一个定时器就可以了。
* 分析之后决定用Java自己的Timer来做，不打算用Spring Schedule或者Quartz，那就开工先做个Demo出来吧。于是，总结了下需求，做了如下的功能
<br />
系统启动的时候Delay一段时间，定时器会被触发，每隔一定时间执行一次。当然所有的这些参数是放在配置文件的。（这个功能是Java的Timer自带的，有API，没啥技术含量）
{% highlight text %}
delay = 10
period = 100
{% endhighlight %}

* 然后拿给业务看，反馈是太简单了，需要加需求，在每天固定时间执行。
<br />
拿回来之后，继续改造，每天的固定时间执行，其实“每天”比较简单。相当于间隔时间是24小时就可以了，难点在这个固定时间执行。分析后决定做法如下：系统启动取得当前时间和这个固定时间，先对这两个时间做判断，如果当前时间在前，只需让定时器Delay(固定时间-当前时间)的秒数就可以了。如果当前时间在后，则让定时器Delay(24*60*60 - 当前时间 + 固定时间)。
{% highlight text %}
# 兼容第一步中的需求
delay = 10
# 兼容第一步中的需求
period = 100 

start = 03:00:00
{% endhighlight %}

* 再给业务看，反馈是功能差不多了，如果这个时间放在数据库，有个后台系统可以随时改动这个时间就更好了。
<br />
先分析一下第二步之后的定时器的限制：
	* 固定时间是在系统初始化的时候加载进来的，所以在系统运行中，现有基础上修改这个时间是不可能实现的。
	* 现有项目是J2ee项目，Mybatis的Bean是由Spring MVC扫描得到的，定时器的类是自己实例化的，获取不到Mybatis的持久层MapperBean
<br />
继续分析之后。
	* 第一个问题的解决的方式是：在系统初始化固定时间加载进来之后,把这个固定时间放在一个全局的static的变量中。然后再添加一个计时器，在每天的0点整先去更新这个时间变量，再执行第二步中的步骤。这样就搞定了。
	* 第二个问题的解决方案是：实现Spring的ApplicationContextAware接口，通过ApplicationContext的getBean方法来获取Spring的托管Bean。

{% highlight text %}
# 兼容第一步中的需求
delay = 10
# 兼容第一步中的需求
period = 100 

# 更新缓存中的时间
update = 00:00:00

# 转移到数据库中了
#start = 03:00:00
{% endhighlight %}

* 这时候拿给业务看，已经可以了，然后大家做了很多的开发，系统运行了一段时间没有什么问题。但是有一天突然又有一个新的需求 - 每周/每月/每年 的某天执行这个定时任务。
<br />
周还好说，Delay7天就可以了。月呢，每月天数是不一定的。年呢，每年也是不一样的。并且系统已经开发了很多功能，在不改变现有业务代码的前提下，加上这个需求，还是有一定难度的。
<br />
好吧。分析了一段时间之后。决定添加一个配置字段period_type来计算Delay时间。
	* DAY的情况是默认的，在上一个需求已经搞定。
	* WEEK的情况，用Calender来获取本周的Sunday和Saturday(JDK中认为每周的结束是周六，每周的开始是周日)，每周日的update时间更新缓存中的开始和Delay时间，通过周六，周日来计算中间的Delay值
	* MONTH的情况，用Calender来获取本月的开始和结束，每个月的第一天的update时间更新开始和Delay时间，通过本月的开始和结束来计算中间的Delay值
	* YEAR的情况，用Calender来获取本年的开始和结束，每年的第一天的update时间更新开始和Delay时间，通过本年的开始和结束来计算中间的Delay值

{% highlight text %}
# 兼容第一步中的需求
delay = 10
# 兼容第一步中的需求
period = 100 

# 更新缓存中的时间
update = 00:00:00

# 周期
period_type = DAY(Default)|WEEK|MONTH|YEAR

# 转移到数据库中了
#start = 03:00:00
{% endhighlight %}

结语
=============
最后发现，其实自己又做了一套简易的Quartz而已，如果当初直接选用Quartz就会比现在好多了。所以说，技术选型很重要，前期调研很重要，一定要搞清楚业务需求和隐含需求。做到对已知的轻车熟路，对未知的有所规划和把控。

<br />
<br />