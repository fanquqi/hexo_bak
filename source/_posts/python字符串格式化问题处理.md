---
title: 'python字符串格式化问题'
date: 2017-08-11 15:59:05
tags: python
categories: python
--- 


## 场景描述


![](http://or2jd66dq.bkt.clouddn.com/unicode_print_error.png)

我格式化输出一个值竟然报出这种神奇的错误，很不解。

仔细看了下 `douban.critic[i]`这个变量type是个Unicode。

而前面`影评`两个字是str不是Unicode 两个值在一起输出就会报错。

## 原因

Python在格式化字符串的时候有一些隐藏着的小动作：如果%s对应的参数里有unicode，那么最终的结果也是unicode。在这种情况下模版字符串以及所有的%s参数中的str都会被decode成unicode，然而这个decode是隐式的，用户无法指定其使用的charset，Python只能用默认的ASCII。如果正好里面有非ASCII编码的字符串，就完蛋了……

## 解决
统一格式

参考：
print u"影评：%s" % douban.critic[i]
