---
layout: post
title: "python ajax爬虫 --36Kr为例"
tags: [python , 爬虫, ajax]
---

最近在倒腾ajax爬虫，顺便记录一下过程。以36Kr “早期项目” 一栏为例，大致分为如下两步。

## 解析索引页

难点在于如何获取索引页的url。浏览器打开[36Kr](http://36kr.com/),按F12打开开发者工具，切换到Netwotk 面板。AJAX 一般是通过 XMLHttpRequest 对象接口发送请求的，XMLHttpRequest 一般被缩写为 XHR。所以点击XHR，清空监听到的内容。如下图:

![](https://raw.githubusercontent.com/zhihuz/BLOG-RES/master/2016-5-10%2034353.jpg)

点击页面导航栏“早期项目”，method 为 GET 的响应就是我们需要的，选中项目鼠标右键 “Copy link address”，内容如下：http://36kr.com/asynces/posts/feed_column_more.json?column=starding,这就是起始url，将url输入到浏览器地址栏，得到索引页面的josn字数据，内容如下：

![](https://raw.githubusercontent.com/zhihuz/BLOG-RES/master/2016-5-10%2034354.jpg)

推荐安装 [JSONView 插件](https://chrome.google.com/webstore/detail/jsonview/chklaanhfefbnpoihckbnefhakgolnmc?hl=zh-CN)，这样可以看到更好看的 JSON 格式，展开折叠列等功能。然后，我们根据 JSON 数据的结构提取资讯标题，时间，和url_code（后文介绍这个字段）。

{% highlight python linenos %}

def parseIndex(url):
	jsContent = requests.get(url)
	jsDict = json.loads(jsContent.content)

	for data in jsDict['data']['feed_posts']:
		# 存放一条资讯
		item = {}
		item['title'] = data['title']
		item['publish_time'] = data['published_at']
		item['index'] = data['url_code']
		contentUrl = "http://36kr.com/p/"+str(item['index'])+".html"
		item['url'] = contentUrl
		item['summary'] = parseContent(contentUrl)
		db.append(item)

{% endhighlight %}

获取了起始url，那么如何模拟点击“获取更多”？可以认为这是一个翻页按钮，只要获取下一页对应的url即可。清空开发者工具，在“早期项目”栏下点击“获取更多”，同样的方法获取新的链接http://36kr.com/asynces/posts/feed_column_more.json?b_url_code=5046782&column=starding，这就是下一页的url，可将其调整为http://36kr.com/asynces/posts/feed_column_more.json?column=starding&b_url_code=5046782，并将起始url调整为http://36kr.com/asynces/posts/feed_column_more.json?column=starding&b_url_code=，注意到这个b_url_code=5046782是什么鬼，尝试在刚刚获取的json数据里搜索5046782，发现这正是最后一条资讯的url_code，想必大家应该明白了如何生成下一个索引页的链接了吧。

{% highlight python linenos %}

while (totalPage >= 1):
		parseIndex(url)
		# 获取最后一条资讯的url_code，拼接下一个索引页的url
		index = db[-1]['index']
		url = "http://36kr.com/asynces/posts/feed_column_more.json?column=starding&b_url_code="
		url += str(index)
		totalPage -=1

{% endhighlight %}

至此，解析索引页的工作结束。

## 解析资讯页

点击“早期项目”的第一条资讯，并从浏览器地址栏获得url为http://36kr.com/p/5046804.html,根据前文的经验，很快发现这个5046804正是其url_code,“http://36kr.com/p/”+url_code+“.html”便是资讯链接，因此这个字段成功获取。如何获取资讯正文呢？

查看网页源码，不难发现正文对应的xpath是“//div/@data-props”，解析后发现response同样为json数据，内容如下，根据json结构便可轻松获取内容。

{% highlight json linenos %}

{"status":{"code":"200","message":"返回成功"},"data":{"router":"/p/5046804.html","post":{"id":45805,"title":"大咖拍卖：做线上艺术品拍卖，“经纪人”或许是互联网更容易撬动的一环","summary":"“传统机构是典型的一级市场，而拍卖是二级市场行为”","cover":"http://a.36krcnd.com/nil_class/e0edfc83-02c3-4597-9f6f-80fd734bf8f3/1.pic.jpg","url_code":5046804,"published_at":"2016-05-10T14:24:28.168+08:00","key":"2c515a1d-364c-4041-b524-b192319310c1","extra":{"source_urls":null,"jid":"","close_comment":false,"mobile_views_count":0,"related_company_type":"domestic"},"source_type":"original","related_company_id":141335,"display_tag_list":["大咖拍卖","艺术收藏","泛娱乐"],"plain_summary":"“传统机构是典型的一级市场，而拍卖是二级市场行为”","display_content":"...内容略去...","author":{"id":458247,"domain_path":"/posts/zibing","display_name":"二水水","avatar":"https://krid-assets.b0.upaiyun.com/uploads/user/avatar/327099/09d80114-dabb-4f1f-8cdc-d12f1fe7183c.jpeg!480"},"crowd_funding":"https://mobilecodec.alipay.com/show.htm?code=rex096303svi4ij2pmmckd3","internal_links":[],"source_message":"原创文章，作者：二水水"}}}

{% endhighlight %}

{% highlight python linenos %}

def parseContent(contenturl):
	res = html.parse(contenturl).xpath("//div/@data-props")[0]
	data = json.loads(res)
	# 获取摘要，资讯正文对应字段为display_content，此处并未获取
	return data['data']['post']['summary']

{% endhighlight %}

至此整个工作接近尾声，下面组织一下代码。

## 完整代码如下

{% highlight python linenos %}

#-*-coding:utf8-*-

import requests
import json
from lxml import html

# 列表存储爬取的资讯
db = []

def parseContent(url):
	res = html.parse(url).xpath("//div/@data-props")[0]
	data = json.loads(res)
	return data['data']['post']['summary']

def parseIndex(url):
	jsContent = requests.get(url)
	jsDict = json.loads(jsContent.content)

	for data in jsDict['data']['feed_posts']:
		# 每条资讯对应一个字典
		item = {}
		item['title'] = data['title']
		item['publish_time'] = data['published_at']
		item['index'] = data['url_code']
		contentUrl = "http://36kr.com/p/"+str(item['index'])+".html"
		item['url'] = contentUrl
		# 解析正文页
		item['summary'] = parseContent(contentUrl)
		db.append(item)

def main(totalPage):
	'''
	totalPage: 抓取的索引页页数
	'''
	# 起始url
	url = "http://36kr.com/asynces/posts/feed_column_more.json?column=starding&b_url_code="
	while (totalPage >= 1):
		# 解析索引页
		parseIndex(url)
		# 获取最后一条资讯的url_code
		index = db[-1]['index']
		# 拼接下一个索引页的url
		url = "http://36kr.com/asynces/posts/feed_column_more.json?column=starding&b_url_code="
		url += str(index)
		totalPage -=1

	# 打印数据
	print len(db)
	for item in db:
		print item

if __name__ == '__main__':
	main(3) # 设置为抓取3页资讯

{% endhighlight %}
