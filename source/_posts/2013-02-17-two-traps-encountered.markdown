---
layout: post
title: "APP升级遇到的两个问题"
date: 2013-02-17 11:35
comments: true
categories: ios
---
<h3>NO1,sqlite数据结构不兼容</h3>
现象：新的数据插不进去，功能性错误<!-- more --><br>
解释：之前一直很注意这个问题，但还是忘记了。前一个版本将sqlite建表逻辑更新出去了，当前版本开发中新增了几个列，导致插入数据出错，fmdb lasterror显示没有那几个column。<br>
临时解决方案：如果这个表里没有任何数据，就drop掉，重新create。<br>
最终解决方案：最初建表时多用一个extra column，里面保存json数据，可以兼容多个column扩展，也可以新建扩展表。后续版本最好不要更改表结构，如真的需要，考虑好数据兼容和迁移问题。
<h3>NO2,文件的绝对路径变动</h3>
现象：图片显示不出来<br>
解释：保存图片使用的是绝对路径，这些路径保存在sqlite中，更新之后图片的实际路径变动，通过sqlite中的路径找不到，所以就显示不出来。<br>
临时解决方案：存储路径时使用相对路径，显示图片时兼容上个版本的绝对路径，将其匹配为相对路径，同时更新数据库。<br>
最终解决方案：所有对文件的存储都应该使用相对路径，这是因为app升级的时候，它的class id会变：When you do an update, the new version is installed to a completely different location. The contents of a couple of key folders such as /Documents are copied from the old version to the new version. Then the old version is deleted.
<h3>怎么测试app update</h3>
测试是软件质量的最后一道防护线，所以测试的流程和质量非常重要，上面两个升级的问题就是因为对app update未测试到导致没有被发现，最后我们决定使用OTA来部署app，参考资料：<br>
<a target="_blank" href="https://help.apple.com/iosdeployment-apps/mac/1.1/?lang=en-us#app43ad871e">Installing apps wirelessly</a><br>
<a target="_blank" href="http://www.cnblogs.com/yingkong1987/archive/2012/10/28/2743774.html">IOS通过OTA部署App</a>

最后特别注意，每次提交app store时都应该保存一份源代码之后再开发下一版的功能，因为提交可能审核不通过，或者发布出去有严重bug需要紧急更新等等。
