**Scrapy**是一个为了爬取网站数据，提取结构性数据而编写的应用框架。 其可以应用在数据挖掘，信息处理或存储历史数据等一系列的程序中。其最初是为了页面抓取 (更确切来说, 网络抓取 )所设计的， 也可以应用在获取API所返回的数据(例如 Amazon Associates Web Services ) 或者通用的网络爬虫。Scrapy用途广泛，可以用于数据挖掘、监测和自动化测试。

Scrapy 使用了 Twisted异步网络库来处理网络通讯。整体架构大致如下

![img](https://upload-images.jianshu.io/upload_images/7058492-b65931bd28f58a10.png?imageMogr2/auto-orient/strip|imageView2/2/w/1080)

Scrapy主要包括了以下组件：

- - **引擎(Scrapy)**
    用来处理整个系统的数据流, 触发事务(框架核心)
  - **调度器(Scheduler)**
    用来接受引擎发过来的请求, 压入队列中, 并在引擎再次请求的时候返回. 可以想像成一个URL（抓取网页的网址或者说是链接）的优先队列, 由它来决定下一个要抓取的网址是什么, 同时去除重复的网址
  - **下载器(Downloader)**
    用于下载网页内容, 并将网页内容返回给蜘蛛(Scrapy下载器是建立在twisted这个高效的异步模型上的)
  - **爬虫(Spiders)**
    爬虫是主要干活的, 用于从特定的网页中提取自己需要的信息, 即所谓的实体(Item)。用户也可以从中提取出链接,让Scrapy继续抓取下一个页面
  - **项目管道(Pipeline)**
    负责处理爬虫从网页中抽取的实体，主要的功能是持久化实体、验证实体的有效性、清除不需要的信息。当页面被爬虫解析后，将被发送到项目管道，并经过几个特定的次序处理数据。
  - **下载器中间件(Downloader Middlewares)**
    位于Scrapy引擎和下载器之间的框架，主要是处理Scrapy引擎与下载器之间的请求及响应。
  - **爬虫中间件(Spider Middlewares)**
    介于Scrapy引擎和爬虫之间的框架，主要工作是处理蜘蛛的响应输入和请求输出。
  - **调度中间件(Scheduler Middewares)**
    介于Scrapy引擎和调度之间的中间件，从Scrapy引擎发送到调度的请求和响应。

**4、文件说明：**

- scrapy.cfg  项目的配置信息，主要为Scrapy命令行工具提供一个基础的配置信息。（真正爬虫相关的配置信息在settings.py文件中）
- items.py    设置数据存储模板，用于结构化数据，如：Django的Model
- pipelines    数据处理行为，如：一般结构化的数据持久化
- settings.py 配置文件，如：递归的层数、并发数，延迟下载等
- spiders      爬虫目录，如：创建文件，编写爬虫规则

注意：一般创建爬虫文件时，以网站域名命名

重点总结：

#### scrapy安装（最简单的方法）

`conda install scrapy requests simplejson`

需要的包都一键安装上



### 01 爬取豆瓣读书的标题title 和 评分rate

#### 1.  创建工程

```
(F:\Pyenv\conda3.8) F:\Projects\spider>scrapy startproject mspider .
```
#### 2.  创建爬虫程序
```
# 创建模板并指定域
(blog) F:\Projects\scrapy\mspider>scrapy genspider -t crawl book douban.com
Created spider 'book' using template 'crawl' in module:
  mspider.spiders.book
```



![1599577884590](C:\Users\dell\AppData\Roaming\Typora\typora-user-images\1599577884590.png)


#### 3. 文件说明：

scrapy.cfg  项目的配置信息，主要为Scrapy命令行工具提供一个基础的配置信息。（真正爬虫相关的配置信息在settings.py文件中）

>items.py    设置数据存储模板，用于结构化数据，如：Django的Model
>pipelines    数据处理行为，如：一般结构化的数据持久化
>settings.py 配置文件，如：递归的层数、并发数，延迟下载等
>spiders      爬虫目录，如：创建文件，编写爬虫规则

注意：一般创建爬虫文件时，以网站域名命名




#### 4. 设置数据存储模板
```
# items.py
import scrapy

class BookItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    title = scrapy.Field()
    rate = scrapy.Field()
```

#### 5. 编写爬虫

```
# book.py

import scrapy
from scrapy.linkextractors import LinkExtractor
from scrapy.spiders import CrawlSpider, Rule
from scrapy.http.response.html import HtmlResponse
from ..items import BookItem

class BookSpider(CrawlSpider):
    name = 'book1'
    allowed_domains = ['douban.com']
    start_urls = ['https://book.douban.com/tag/%E7%BC%96%E7%A8%8B?start=0&type=T']

    rules = (
        Rule(LinkExtractor(allow=r'start=\d+'), callback='parse_item', follow=False),
    )

    custom_settings = {
        'filename':'./books2.json'
    }

    def parse_item(self, response:HtmlResponse):
        print(response.url)

        subjects = response.xpath('//li[@class="subject-item"]')

        for subject in subjects:
            title = "".join((x.strip() for x in subject.xpath('.//h2/a//text()').extract()))
            rate = subject.css('span.rating_nums').xpath('./text()').extract()  # extract_first()/extract()[0]

            item = BookItem()
            item['title'] = title
            item['rate'] = rate[0] if rate else '0'
            print(item)

            yield item

```

#### 6. 编写 代理（反爬）

```

# middlewares.py

# https://docs.scrapy.org/en/latest/topics/spider-middleware.html

from scrapy import signals
from scrapy.http.response.html import HtmlResponse
from scrapy.http.request import Request

import random

class MspiderSpiderMiddleware:

    @classmethod
    def from_crawler(cls, crawler):
        # This method is used by Scrapy to create your spiders.
        s = cls()
        crawler.signals.connect(s.spider_opened, signal=signals.spider_opened)
        return s

    def process_spider_input(self, response, spider):
        return None

    def process_spider_output(self, response, result, spider):
        for i in result:
            yield i

    def process_spider_exception(self, response, exception, spider):
        pass

    def process_start_requests(self, start_requests, spider):
        for r in start_requests:
            yield r

    def spider_opened(self, spider):
        spider.logger.info('Spider opened: %s' % spider.name)

class MspiderDownloaderMiddleware:
    @classmethod
    def from_crawler(cls, crawler):
        # This method is used by Scrapy to create your spiders.
        s = cls()
        crawler.signals.connect(s.spider_opened, signal=signals.spider_opened)
        return s

    def process_request(self, request, spider):
        return None

    def process_response(self, request, response, spider):
        return response

    def process_exception(self, request, exception, spider):
        pass

    def spider_opened(self, spider):
        spider.logger.info('Spider opened: %s' % spider.name)

class ProxyDownloaderMiddleware:

    proxy_ip = '223.215.13.40'
    proxy_port = 894
    proxies = [
        'http://{}:{}'.format(proxy_ip, proxy_port)
    ]

    def process_request(self, request:Request, spider):

        proxy = random.choice(self.proxies)
        request.meta['proxy'] = proxy   # 换代理
        print(request.url,request.meta['proxy'])
        # return None

# # 看造出的response怎么处理；
# class After(object):
#     def process_request(self, request, spider):
#         print('After ~~~~~{}')
#
#         return None

```




#### 7. 配置文件设置IP代理下载中间件
settings.py 增加代理下载中间件；
```
# settings

BOT_NAME = 'mspider'

SPIDER_MODULES = ['mspider.spiders']
NEWSPIDER_MODULE = 'mspider.spiders'

USER_AGENT = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.106 Safari/537.36"

ROBOTSTXT_OBEY = False
CONCURRENT_REQUESTS = 4
DOWNLOAD_DELAY = 1

COOKIES_ENABLED = False

SPIDER_MIDDLEWARES = {
   'mspider.middlewares.MspiderSpiderMiddleware': 500,
}

DOWNLOADER_MIDDLEWARES = {
   # 'mspider.middlewares.MspiderDownloaderMiddleware': 543,
   'mspider.middlewares.ProxyDownloaderMiddleware':150,
}

ITEM_PIPELINES = {
   'mspider.pipelines.MspiderPipeline': 300,
}
```

![IP代理成功](https://upload-images.jianshu.io/upload_images/7058492-914b9833aa8c4696.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 8. 编写数据处理 持久化脚本

```
# pipelines.py
import simplejson
from scrapy import Spider

class MspiderPipeline(object):
    def __init__(self):   # 实例化过程， 可以不用
        print('~~~~~init~~~~~')

    def open_spider(self, spider:Spider):  # 不同的spider做不同的任务
        self.count = 0
        print(spider.name)
        filename = spider.settings['filename']
        self.file = open(filename, 'w', encoding='utf-8')
        self.file.write('[\n')
        self.file.flush()

    def process_item(self, item, spider):
        print(item, '~----~~~~~~~~~')   # 打印
        self.count += 1
        self.file.write(simplejson.dumps(dict(item)) + ',\n')   # dict

        return item

    def close_spider(self, spider):
        print('=========={}'.format(self.count))
        self.file.write(']')
        self.file.close()

```

#### 9. 执行爬虫
豆瓣读书只允许爬取50页的内容，每一页 20 项，共计1000条数据爬取成功

```
(F:\Pyenv\conda3.8) F:\Projects\spider>scrapy crawl book1 --nolog

{'rate': '8.8', 'title': '黑客与画家: 来自计算机时代的高见'} ~----~~~~~~~~~
https://book.douban.com/tag/%E7%BC%96%E7%A8%8B?start=1060&type=T http://223.215.13.40:894
https://book.douban.com/tag/%E7%BC%96%E7%A8%8B?start=1040&type=T
https://book.douban.com/tag/%E7%BC%96%E7%A8%8B?start=1060&type=T
==========1000   # 
```


