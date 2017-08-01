---
title: shell命令之xargs与exec
date: 2017-06-23 11:18:55
tags: shell
categories: 基础运维
---

## exec 
看起来就是执行某动作的命令
格式为 -exec echo {} \; 
其中 `echo` 为动作, `{}` 为参数(即为之前找到的文件) `\` 转义 `;` 是结束符

示例
```
find ./ -name \*.yml -exec echo {} \;
./logrotate_conf.yml
./filebeat.yml
./group_vars/all.yml
./group_vars/ehr.yml
./group_vars/http_proxy.yml
./group_vars/medweb.yml
```


## xargs 
xargs依赖管道,下边直接写了几个个实例

`基本使用`

把多行文件合并成一行 或者每行指定元素个数(-n 参数) 格式输出

```
[root@md6 ~]# cat test.txt
a b c d e f
 g h i j k l
 m n o p q r s
t u v w x y z

[root@md6 ~]# cat test.txt | xargs echo
a b c d e f g h i j k l m n o p q r s t u v w x y z
[root@md6 ~]# cat test.txt | xargs echo > test1.txt
[root@md6 ~]# cat test1.txt
a b c d e f g h i j k l m n o p q r s t u v w x y z

[root@md6 ~]# cat test.txt | xargs -n 4
a b c d
e f g h
i j k l
m n o p
q r s t
u v w x
y z
[root@md6 ~]# cat test.txt | xargs -n 3
a b c
d e f
g h i
j k l
m n o
p q r
s t u
v w x
y z
```

- 实例1 直接打印
```
find ./ -name \*.yml | xargs echo
./logrotate_conf.yml ./filebeat.yml ./group_vars/all.yml ./group_vars/ehr.yml ./group_vars/http_proxy.yml ./group_vars/medweb.yml
```

- 实例2 查找文件内容中带hostname的
```
find . -type f -print | xargs grep -n "hostname" (-n输出行号)
```

- 实例3 mv 或者 cp 使用-i参数 -p 打印过程(交互确认)
```
ls  *.txt | xargs -n1 -i -p cp {} /var/tmp/
```

- 实例4 查找当下文件按大小排序
```
find . -maxdepth 1 ! -name "." -print0 | xargs -0 du -b | sort -nr | head -10 | nl
```

- 实例5 结合sed替换
```
find . -name "*.txt" -print0 | xargs -0 sed -i 's/aaa/bbb/g'
```


## 总结

首先使用方式不一样，exec操作麻烦些 xargs更简单直接方法多样
另外从输出结果看得出  exec是遇到一个找到的文件就执行一次命令，xargs是把结果放到一起，执行一次。(这样可以使用在 把多行文件合并成一行的特定场景中)
需要强调xargs遇到文件名中有空格的行为是处理不了的。
这种情况可以这样

```
find . -name \*.txt -print0 | xargs -0 rm 
```
其中find找到每个文件定义以null字符结尾  xargs找文件定义null分割文件 就可以了 




