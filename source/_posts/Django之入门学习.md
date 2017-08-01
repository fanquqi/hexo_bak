---
title: Django之入门学习
date: 2017-06-14 10:36:07
tags: Django
categories: Django
---
> 自己动手从0开始

python版本：3.5.1
Django版本：1.9.5

# 安装初始化
## python3

```
yum groupinstall 'Development Tools'
yum install zlib-devel bzip2-devel openssl-devel ncurese-devel
wget https://www.python.org/ftp/python/3.5.1/Python-3.5.1.tar.xz && mv Python-3.5.1.tar.xz /usr/local/ && cd /usr/local
tar -Jxvf Python-3.5.1.tar.xz &&  cd Python-3.5.1
./configure --prefix=/usr/local/python3
make && make install 
ln -s /usr/local/python3/bin/python3.5 /usr/bin/python3
ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3
# 
python3 -m venv myvenv

```

## django安装
```
source myvenv/bin/activate
pip install django==1.9.5
```

### 生成项目

```
django-admin.py startproject mysite
```

`mysite`目录结构如下

```
mysite
├───manage.py
└───mysite
        settings.py
        urls.py
        wsgi.py
        __init__.py
```

- manage.py是管理网站的脚本，可以使用它来启动一个简单的web服务器，这个对于开发调试非常有用。
- setting.py是工程的核心配置文件。
- urls.py是路径配置文件，可以配置URL到实际Controller的映射关系。



### 配置更改
更给`settings.py`语言，时区

```
LANGUAGE_CODE = 'zh-cn'
TIME_ZONE = 'Asia/Shanghai'
```

数据库不用改 先使用默认的sqlite3,
生成数据库
```
python manage.py migrate
```
runserver 看下，可以再浏览器访问下 ip:8000
python manage.py runserver 0.0.0.0:8000

### 创建应用

django 中 工程(project)与应用(application)概念要分清，我们使用 `django-admin.py manage.py startproject mysite`创建`mysite`的是工程.
下边使用`python manage.py startapp blog` 创建的`blog`是应用，一个工程可以包括很多个应用。

创建应用
`python manage.py startapp blog`
现在整个工程目录结构如下
```
.
├── blog
│   ├── admin.py
│   ├── apps.py
│   ├── __init__.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── db.sqlite3
├── manage.py
└── mysite
    ├── __init__.py
    ├── __pycache__
    │   ├── __init__.cpython-35.pyc
    │   ├── settings.cpython-35.pyc
    │   ├── urls.cpython-35.pyc
    │   └── wsgi.cpython-35.pyc
    ├── settings.py
    ├── urls.py
    └── wsgi.py
```

然后，我们需要把这个应用在`settings.py`注册一下。
```
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog',
]
```
