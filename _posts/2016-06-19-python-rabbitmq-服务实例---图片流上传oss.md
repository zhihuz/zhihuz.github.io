---
layout: post
title: "python + rabbitmq 服务实例--图片流上传oss"
tags: [python , rabbitmq, oss]
---
## 需求

按生产和消费者模型，写一个图片流上传至阿里oss的服务。其中，生产者是python爬虫，抓取需要的图片链接url。消费者获取url对应的文件流，将其上传至oss。中间件采用rabbitmq。基于此需求，本人写了一个小demo，分享给大家。本文的中心将在消费者，至于生产者（python spider）不描述。由于本人对此不是很熟，技术略糙，其中难免会有很多需要改进的地方，欢迎交流。另外，这篇文章也参考了很多网友的代码，感谢他们。

## 环境准备

[阿里云oss Python-SDK 官方文档](https://help.aliyun.com/document_detail/32030.html?spm=5176.doc32027.6.299.gZOACS)

[rabbitmq 安装及入门教程](http://blog.csdn.net/column/details/rabbitmq.html)

## 代码

### sender.py
这部分的代码不做详细介绍，直接copy网上教程的代码。为简化处理，待发送的图片url直接写在body里面了，
如下：

{% highlight python linenos %}

#!/usr/bin/env python
#coding=utf8
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='upload2oss')

channel.basic_publish(exchange='', routing_key='upload2oss', body='https://raw.githubusercontent.com/zhihuz/BLOG-RES/master/2016-5-10%2034354.jpg')
print " [sender] message Sented!"
connection.close()

{% endhighlight %}

### receiver.py

消费者从队列中获取url，函数download获取文件流，函数upload将其上传至oss。由于并非所有的url都是有效的，分别为这两个函数设置了失败超时次数（downloadRetryTimes和uploadRetryTimes），并用日志模块记录这个过程。文中为简化处理，将已经封装好的logger替换为print。代码中参数较多，因此统一写在了oss.ini文件中。配置文件如下：

{% highlight python linenos %}

[oss]
AccessKeyId = ********
AccessKeySecret = ********
Endpoint = ********
Bucket = ********
downloadRetryTimes = 3
uploadRetryTimes = 3

[threading]
threadingNum = 2

{% endhighlight %}

receiver.py代码主要分三步：1.建立connection 2.建立channel 3.建立queue
读取配置文件及建立连接的代码如下：
{% highlight python linenos %}

import ConfigParser
import oss2
import requests
import hashlib
import time
import threading
import traceback

cp = ConfigParser.SafeConfigParser()
cp.read("/home/solosseason/oss/oss.ini")

AccessKeyId = cp.get('oss', 'AccessKeyId')
AccessKeySecret = cp.get('oss', 'AccessKeySecret')
Endpoint = cp.get('oss', 'Endpoint')
Bucket = cp.get('oss', 'Bucket')
downloadRetryTimes = int(cp.get('oss', 'downloadRetryTimes'))
uploadRetryTimes = int(cp.get('oss', 'uploadRetryTimes'))

auth = oss2.Auth(AccessKeyId, AccessKeySecret)
bucket = oss2.Bucket(auth, Endpoint, Bucket)

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='upload2oss')

{% endhighlight %}

下载文件流的函数download声明如下：
{% highlight python linenos %}

def download(url,retryTime):
    input = requests.get(url)
    if input.status_code == 200:
        return input
    if retryTime < downloadRetryTimes:
        download(url,retryTime+1)
    return

{% endhighlight %}

在上传文件流至oss之前，对url对应的文件流生成唯一的名称。
{% highlight python linenos %}

def calSha1(url):
    sha1obj = hashlib.sha1()
    sha1obj.update(url)
    hash = sha1obj.hexdigest()
    return hash+'.jpg'

{% endhighlight %}

upload函数将文件流上传至阿里oss
{% highlight python linenos %}

def upload(input,url,retryTime):
    name = calSha1(url)
    result = bucket.put_object(name, input)
    if result.status == 200:
        print result.status,result.etag
    if retryTime < uploadRetryTimes:
        upload(input,retryTime+1)

{% endhighlight %}

MQ的receiver.py还需要一个callback函数，负责接收从队列中取出的元素，并自定义操作，本文的操作是download()下载文件流后upload()将其上传至阿里oss。

{% highlight python linenos %}

def callback(ch, method, properties, body):
    try:
        input = download(body,0)
        if input:
            self.upload(input,body,0)
        ch.basic_ack(delivery_tag = method.delivery_tag)
    except Exception,e:
        traceback.print_exc()
        print "Warning##Upload2oss##callback {0} failed##{1}".format(body, e)

# 启动服务，写在代码的最后面。
print ' [*] Waiting for messages. To exit press CTRL+C'
channel.start_consuming()

{% endhighlight %}

### 多线程及封装

上文已经详细描述receiver.py的代码构成。不难想象，生产者只是负责publish 一个个url。相比而言，消费者的操作复杂的多，考虑到负载均衡，需要对消费者进行多线程拓展，同时为了提高代码整洁性，将代码封装成类（封装的过程有参考网友的代码）。此处直接贴出最终版。（ps:篇幅限制，此处不提供logger部分的代码）

{% highlight python linenos %}

#!/usr/bin/env python
#coding=utf8

import sys
sys.path.append("/home/solosseason")

import pika
import ConfigParser
import oss2
import requests
import hashlib
import threading
import traceback
from oss import logger

cp = ConfigParser.SafeConfigParser()
cp.read("/home/solosseason/oss/oss.ini")

AccessKeyId = cp.get('oss', 'AccessKeyId')
AccessKeySecret = cp.get('oss', 'AccessKeySecret')
Endpoint = cp.get('oss', 'Endpoint')
Bucket = cp.get('oss', 'Bucket')
auth = oss2.Auth(AccessKeyId, AccessKeySecret)
bucket = oss2.Bucket(auth, Endpoint, Bucket)

downloadRetryTimes = int(cp.get('oss', 'downloadRetryTimes'))
uploadRetryTimes = int(cp.get('oss', 'uploadRetryTimes'))

threadingNum = int(cp.get('threading', 'threadingNum'))

class Upload2oss(threading.Thread):

    def __init__(self):
        threading.Thread.__init__(self)
        self.connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue='')
        self.channel.basic_qos(prefetch_count=1)
        self.channel.basic_consume(self.callback, queue='upload2oss')

    def run(self):
        print 'start consuming'
        self.channel.start_consuming()

    def callback(self,ch, method, properties, body):
        try:
            input = self.download(body,0)
            if input:
                self.upload(input,body,0)
            ch.basic_ack(delivery_tag = method.delivery_tag)
        except Exception,e:
            traceback.print_exc()
            logger.warning("Warning##Upload2oss##callback {0} failed##{1}".format(body, e))

    def calSha1(self,url):
        sha1obj = hashlib.sha1()
        sha1obj.update(url)
        hash = sha1obj.hexdigest()
        return hash + '.jpg'

    def download(self,url,retryTime):
        input = requests.get(url)
        if input.status_code == 200:
            return input
        if retryTime < downloadRetryTimes:
            self.download(url,retryTime+1)
        return

    def upload(self,input,url,retryTime):
        name = self.calSha1(url)
        result = bucket.put_object(name, input)
        if result.status == 200:
            print result.status,result.etag
        if retryTime < uploadRetryTimes:
            self.upload(input,retryTime+1)

for _ in range(threadingNum):
    print 'launch thread'
    td = Upload2oss()
    td.start()

{% endhighlight %}

