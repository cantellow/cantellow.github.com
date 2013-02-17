---
layout: post
title: "循环引用导致的内存泄漏"
date: 2013-01-31 17:46
comments: true
categories: ios
---
有时候didReceiveMemoryWarning不像女人的大姨妈来的那么确定，让人摸不着头脑，不过好在有Instruments这种神器，帮我们解决了不少问题。<br>
<!-- more -->
用Instruments分析了一下刚做好的app，发现如果反复触发同一个页面（没有做缓存），内存居高不下，仔细搜索controller的类名，发现根本就没有释放掉，触发代码如下：
{% codeblock code lang:objc %}
XXXViewController *viewController = [[XXXViewController alloc] init];  
    [self.navigationController pushViewController:viewController animated:YES]; 
{% endcodeblock %}
释放代码如下：
{% codeblock code lang:objc %}
[self.navigationController popViewControllerAnimated:YES];
{% endcodeblock %}
这几行代码没有什么复杂的，为什么pop之后XXXViewController没有释放呢，刚开始以为是NavigationController本身的机制就是这样的，后来新建了工程，写了例子代码之后发现pop之后controller是确定能够释放的。<br>
仔细用Instruments分析了一遍XXXViewController的retain和release历史，但由于经过ARC之后有大约70多条，一个一个的分析不知道能分析到什么时候，最后检查了一遍代码，发现是由于循环引用导致。<br><br>
简单来说就是XXXViewController引用了SubViewController，而在初始化SubViewController的时候把self传给了SubViewController，而且是strong引用，结果就导致XXXViewController pop之后retainCount仍然为1，从而内存泄漏了。<br>
相比较而言，java在循环引用上处理就比较给力，java的GC使用图算法从根对象出发，找到无法达到的对象：<br>
<img src="/images/posts/2.png" ><br>
而ARC只是编译时期的特性，在合适的地方插入retain和release代码，使用简单的对象引用计算来判断对象是否该释放。<br>
怎么解决循环引用呢？大致有两种方法：<br>
1、手动切断循环引用中的一端，比如在pop view的时候顺便把SubViewController对self的引用切断，置为nil<br>
2、使用weak，弱引用，本例中，在SubViewController中对self的持有置为weak引用，表示不对retainCount加1，ARC遵循这样的原则：“只要某个对象被任一strong指针指向，那么它将不会被销毁。如果对象没有被任何strong指针指向，那么就将被销毁”。<br><br>
大部分情况下，我们用不到weak引用，但它却时刻伴随着我们，一个常见的例子就是oc中常见的delegate设计模式，viewController中有一个strong指针指向它所负责管理的UITableView，而UITableView中的dataSource和delegate指针都是指向viewController的weak指针。<br>
<img src="/images/posts/3.png" ><br>
本例中引起循环引用的业务逻辑就是子controller要用到父controller的方法，这其实不用我们手工维持这么一个引用，可以使用self.parentViewController来获取，但在这之前你必须在addSubview的时候[self addChildViewController:subViewController]，这是一个好习惯，参见：<a target='_blank' href="http://blog.devtang.com/blog/2012/02/06/new-methods-in-uiviewcontroller-of-ios5/">iOS5中UIViewController的新方法</a>
<br>未优化时每触发一次都能泄漏0.5M，十次至少泄漏5M，优化后触发十次的内存：<br>
<img src="/images/posts/4.jpg" ><br>
基本上跟初始内存一模一样，可以断定没有任何泄漏了。
