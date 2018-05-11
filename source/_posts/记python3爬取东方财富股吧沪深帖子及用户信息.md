---
title: 记python3爬取东方财富股吧沪深帖子及用户信息
date: 2018-05-11 20:45:30
tags: 爬虫
---

### 一. 简介

爬虫并没有使用流行的框架，原因是以前并没有接触过爬虫 想通过自己慢慢实现来找找感觉。

程序使用了多进程+多线程，实现了首次全部爬取和增量爬取。

按我的理解，爬虫主要要干的就是：

- 下载网页
- 解析网页
- 存储内容

下载请求使用的是**requests**，解析和获取内容使用的是**lxml、xpath和re**。

<!--more-->

存储方面我则是用了**mongodb**，这也是我第一次使用NoSQL。其实使用mongodb部分原因是想用用非关系数据库，借机了解学习= =。还有使用了**redis**作为消息队列。

### 二. 数据库存储

使用非关系数据库的话，就要学会摒弃关系型数据库的思维，反范式化。

不得不说，开头的时候我还是傻傻地把表按关系型的思维去设计了。

**东方财富股吧数据表字段**：

**User集合**

| 字段名          | 类型             | 描述         |
| --------------- | ---------------- | ------------ |
| _id             | ObjectId         | ObjectID     |
| id              | string           | 网站用户id   |
| url             | string           | 用户主页链接 |
| nickname        | string           | 用户昵称     |
| avator          | string           | 用户头像链接 |
| reg_date        | ISODate          | 注册日期     |
| following_count | int              | 关注数       |
| fans_count      | int              | 粉丝数       |
| influence       | int              | 影响力       |
| introduce       | string           | 用户简介     |
| visit_count     | int              | 总访问数     |
| post_count      | int              | 发帖数       |
| comment_count   | int              | 评论数       |
| optional_count  | int              | 自选数       |
| capacity_circle | List（存放code） | 能力圈       |
| source          | string           | 所属论坛     |

**Post 集合**

| 字段名         | 类型     | 描述                                  |
| -------------- | -------- | ------------------------------------- |
| _id            | ObjectId | Objectid                              |
| id             | string   | 帖子id                                |
| url            | string   | 帖子链接                              |
| user_nickname  | string   | 作者昵称                              |
| title          | string   | 帖子标题                              |
| created_at     | ISODate  | 发表时间                              |
| content        | string   | 帖子内容（见下）                      |
| type           | string   | 帖子类型                              |
| code           | string   | 所属板块代码                          |
| comment_count  | int      | 评论数                                |
| comments       | List     | 评论列表，里面存放comment文档（见下） |
| last_update_at | ISODate  | 最后更新时间                          |
| like_count     | int      | 点赞数                                |
| page_count     | int      | 帖子页数                              |
| source         | string   | 来源（eastmoney）                     |
| type           | string   | 帖子类型（有hinfo、normal还有qa）     |
| user_id        | string   | 作者id                                |
| user_influence | string   | 作者影响力                            |
| user_age       | string   | 作者吧龄                              |
| view_count     | int      | 浏览量                                |

文章与评论采用**内嵌形式**

采用内嵌而不是引用的理由：

- 场景是使用爬虫爬取文本内容做分析而不是作为论坛数据库使用，多数帖子评论较少，最多的大概是几千条，基本不会超过16MB的限制。
- 且未来文本分析主要操作是读取，读取内容为主帖+每条评论，使用内嵌形式的话进行一次查询即可。
- 缺点是占用更多空间。

------

**Comment 评论文档**

| 字段名         | 类型     | 描述                       |
| -------------- | -------- | -------------------------- |
| id             | string   | 评论id（唯一）             |
| user_nickname  | string   | 评论用户昵称               |
| user_id        | String   | 评论用户id                 |
| created_at     | ISODate  | 发表时间                   |
| content        | string   | 评论内容                   |
| like_count     | int      | 点赞数                     |
| user_influence | int      | 评论用户影响力             |
| user_age       | String   | 评论用户吧龄               |
| reply_to       | document | 回复的评论的内容（见下表） |

**reply_to 文档**

| 字段名                 | 类型   | 描述                 |
| ---------------------- | ------ | -------------------- |
| reply_to_user_nickname | string | 回复的评论的用户昵称 |
| reply_to_comment       | string | 回复的评论的内容     |
| reply_to_comment_id    | string | 回复的评论的id       |

------

当**帖子类型为qa**（问董秘）时，帖子的内容形式有变化。此时帖子的content则存放question和answer，其中question为string，而answer为文档，如下。

**answer 文档**

| 字段名  | 类型    | 描述     |
| ------- | ------- | -------- |
| content | string  | 答复内容 |
| from    | string  | 答复来自 |
| time    | ISODate | 答复时间 |

实例如下：

```
{
    "_id" : ObjectId("5af30cc7e99de146ba385258"),
    "url" : "http://guba.eastmoney.com/news,600000,758450707.html",
    "code" : "600000",
    "comment_count" : "1",
    "comments" : [ 
        {
            "id" : "8692649074",
            "user_nickname" : "很S很天真",
            "user_id" : "6303084653682694",
            "created_at" : ISODate("2018-05-07T18:51:56.000Z"),
            "content" : "浦发的每一次反弹都是撤离的好机会！",
            "reply_to" : "",
            "like_count" : 0,
            "user_influence" : 5,
            "user_age" : "1.9年"
        }
    ],
    "content" : {
        "question" : "为何2017年的现金分红对比2016年大幅缩减？导致5月2号股票价格大跌。",
        "answer" : {
            "from" : "上证e互动",
            "time" : "2018-05-07 17:32:47",
            "content" : "公司2017年度利润分配预案主要基于：一是在国家持续推进供给侧改革和去杠杆过程中，银行业风险管控压力持续上升，对银行抗风险能力提出更高的要求；二是2018年为资本达标过渡期最后一年，对商业银行资本充足水平提出了更高要求；三是随着公司集团化、国际化战略的不断推进，集团及各子公司健康快速发展，对于资本补充的需求也显著上升；四是公司加快结构转型，注重科技引领，加快建设数字生态银行，科研投入占比有所增加，需要充足的资本支撑。公司综合考虑监管机构的相关要求、自身盈利水平和资本充足状况、以及转型发展的需要，适当提高了利润留存比例以补充资本，提升公司防范金融风险、服务实体经济、深化金融改革的能力。感谢您的关注！"
        }
    },
    "created_at" : ISODate("2018-05-03T11:15:25.000Z"),
    "id" : "758450707",
    "last_update_at" : ISODate("2018-05-07T18:51:56.000Z"),
    "like_count" : 0,
    "page_count" : 1,
    "source" : "eastmoney",
    "title" : "为何2017年的现金分红对比2016年大",
    "type" : "qa",
    "uesr_nickname" : "顺民izeoyl",
    "user_age" : "2.8年",
    "user_id" : "4661094379663388",
    "user_influence" : 0,
    "view_count" : "2642"
}
```

### 三.爬取过程

爬取的思路大概是这样的：

**1.首先要获取到所有的股票代码**

在[东方财富个股吧](http://guba.eastmoney.com/remenba.aspx?type=1)里有所有的股票信息，我只选取了沪A和深A的股票作为爬取对象。

所以的话，第一步就是就**将这些股票代码爬取下来，保存到redis中。**

**2.能够获取一个股票代码版块的所有帖子链接**

可以选择页数较少的股票先作为目标，比如 浙江美大002677 这只股票，其版块链接为http://guba.eastmoney.com/list,002677.html

大概有167页，每页最多有80个帖子。要能够把这么多个帖子链接记录下来，然后后面再挨个爬取里面的详细内容。

##### **3.能够获取一个帖子里面的所有内容**

每获取完一个帖子里的所有内容，就把它保存到数据库中。

在这个过程中，把所有发帖留言的用户id都记录下来。

##### **4.能够获取用户的信息**

爬取前先判断用户id是否已经保存在redis的set中。若不存在，则获取用户信息，然后保存到数据库，并且把用户id插入到redis的set中（set能够自动去重）。

------

这四步都完成后，爬虫就完成啦。

第二步就可以使用多进程去操作，每个进程从redis里取一个任务，然后执行第三步。

而第三、四步则可以用多线程，因为这两步属于**IO密集型**的任务。



### **四. 细节**

我在爬取东方财富股吧的过程中发现它好像没有设置反爬机制？对新手很友好哇。但是我们还是秉着原则`time.sleep(x)`一下。

------

##### **关于第一步** 获取所有股票代码

爬虫的第一步没有什么坑，按照静态页面思路爬取即可，主要是熟悉了xpath的使用。

------

##### **关于第二步** 获取某个版块的所有帖子

要获取所有帖子，第一反应就是想到遍历每一页，那得先知道版块页数对吧？然后发现版块下面就有了，如下图。

![屏幕快照 2018-05-11 下午5.45.31.png](https://upload-images.jianshu.io/upload_images/9009359-930e9065b94409a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但是发现这个是属于动态内容，所以我是用到了selenium无头模式获取这个版块的页数，比如浙江美大版块 共 **167** 页。其中，在服务器中如果要用selenium得要用库pyvirtualdisplay。然后在使用selenium的地方加上

```
display = Display(visible=0, size=(800, 800))   

display.start() 
```

获取到页面数之后，就开始用for循环获取每一页内容，把页面上基本的信息保存下来。

**尤其是帖子的链接，**在第三步获取帖子里面的内容，就得请求帖子链接。

```
页数范围 = 用selenium获取的页数
for i in 页数范围：
	获取（http://guba.eastmoney.com/list,002677_i.html）的帖子信息
```

其实后面我想到没必要用selenium获取这个页数，直接在while循环内让页数递增，当获取不到页面元素时直接捕捉异常退出while循环就可以了吧= =。

这步遇到的坑就是，**当爬取内容前，得先了解要爬取的目标！！**

比如帖子原来有**公告**，**比赛**，**问董秘**，**研报**，**新闻**及**普通帖子**多种形式。

所以要分析这些帖子里面的内容形式是不是相同的！！否则解析部分就会不对= =。比如一开始我看了几页数据，都没注意到有**问董秘**这种东西。。。

------

##### 关于第三步 获取帖子详细内容

帖子的标题，内容还有评论内容都是静态内容，可以很容易获取到的。

但是作者、评论用户的吧龄及影响力，还有帖子及评论的点赞数则都是要用ajax获取到的。

用chrome打开控制台Network得到这些链接。如下图所示，以下得到的是每个用户的id、吧龄和影响力。

![屏幕快照 2018-05-11 下午6.08.01.png](https://upload-images.jianshu.io/upload_images/9009359-8f8881cca7f79764.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

请求链接Requests URL如下图所示，可以看到**关键**就是要**构造action=xx后面的id和replyids**，![屏幕快照 2018-05-11 下午6.10.14.png](https://upload-images.jianshu.io/upload_images/9009359-a5bbf01a8bb16b66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以的话，每次想要获取帖子里所有用户的吧龄和影响力，就是先获取到他们的用户id，以及评论id，构造出类似上面的链接获取信息。

构造形式大概就是`/../guba.aspx?action=getreplylikegd&id=xxx&replyids=xxx%7Cxxx%2Cxxx%7Cxxx….`

7C后面接的是评论的id，2C后面接的是评论用户的id。即对于评论，是要有**这条评论本身的id**以及**评论用户的id**一起才行的。

![屏幕快照 2018-05-11 下午6.14.11.png](https://upload-images.jianshu.io/upload_images/9009359-9b3def33dae77029.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

返回结果为如上，把最外层的括号去掉，然后用json库解析数据获取里面的内容即可。

**再后面就遇到坑了！！！：**

有的帖子有很多的评论用户，构造出来的链接就会包括很多用户和评论id。此时就可能超过了一定的上限，比如几百条回复，你构造了超级长的链接请求，结果可能只返回了前30条。。。（具体多少我没去留意）

所以，我就分开多次获取，每次获取30条再保存起来。

其它的部分也类似上述过程啦。

------

#### 最最后

程序可以跑啦！

结果，却发现还是会有报错。。。（痛苦）

后面尝试输出错误页面链接，发现了东财很多奇怪的地方。。。

**比如坑1：**

![屏幕快照 2018-04-28 下午6.18.13.png](https://upload-images.jianshu.io/upload_images/9009359-1b3becc4263fc3d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![屏幕快照 2018-04-28 下午6.18.22.png](https://upload-images.jianshu.io/upload_images/9009359-abeebea3e6fa58af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

中科信息吧的帖子打开竟然是中科曙光的！？= =

**坑2：**
![屏幕快照 2018-05-09 下午9.12.29.png](https://upload-images.jianshu.io/upload_images/9009359-cda0579d3defe65b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
有的帖子出现在了版块中，打开却会是不存在。

**坑3：**
![屏幕快照 2018-05-10 下午9.34.14.png](https://upload-images.jianshu.io/upload_images/9009359-b78b7be1a81a20df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这虽然是问董秘类型的帖子，但其实应该是公告类型的。。。（仔细看，会发现和上面两条的公告是一样的）

进去里面并没有问答的内容，所以解析页面的时候就会报错！

**坑4：**

![屏幕快照 2018-05-09 下午9.28.48.png](https://upload-images.jianshu.io/upload_images/9009359-d1b66486d67e2a02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

好吧，原来还有“上海手机网友”这种玩意。。。
你没法获取到它的用户id（压根没有）和昵称（内容不再是在a标签里，而是在span标签里）。评论里也一样会出现“上海手机网友”。。。

针对上述情况，修改下代码即可。

ok，单进程单线程能正常跑了，加上多线程和多进程提高爬虫速度。

### 五.增量爬取

首次爬取完全部的内容了，假设后面想获取更新内容呢？立马想到的是**根据时间来判断**。

取**版块中帖子的最后更新时间**与**数据库中该板块的最新时间**作比较，如果发现页面中的时间比数据库里的新，就把帖子爬下来。

结果还是遇到坑了，首先，东财版块页面每条帖子后面显示的最后更新时间是不包含年份的，这样的话可能遇到某些特殊情况会出问题？于是，我就尝试获取帖子的最新评论的发表时间，这个时间是包含年月日的。

请求链接为：[http://guba.eastmoney.com/news,xxxxxx,xxxxxxxx,d.html#storeply](http://guba.eastmoney.com/news,xxxxxx,xxxxxxxx,d.html#storeply)

请求带d的帖子链接，第一条评论就是最新评论。

------

本以为这就可以了，实际运行发现爬没几条就停止了。发现这些帖子都有一个问题，就是它们**在版块中显示的最后更新时间**和**最新一条评论的发表时间**并不相同！！（好吧，此时此刻我以为我又遇到bug了）

后面细心点才发现**问题所在**：

版块页面每条帖子后面显示的最后更新时间并不一定代表帖子有新评论，也可能是帖子中有点赞行为（**猜测是吧，实际中我尝试回帖、点赞，帖子都不是立即被置顶的**）。

而当时**保存在数据库中的最后更新时间** 其实是 **主帖中最后一条评论的发表时间**，这个不一定和版块页面中显示的最后更新时间是相同的。

所以就出现了这样的情况：有的帖子没有新评论，但是有点赞行为，帖子被顶上去了，然而程序如果看到这样的帖子，就以为爬完了（因为版块页面的时间是最新的，但最后一条评论的发表时间和数据库中的最后更新时间相同）。

后面想了一个解决方法是：设定一个上限N，只有连续获取的N条帖子情况都这样（连续N条帖子都是由纯点赞行为导致的可能性是很小的吧）才认为把最新的内容都爬取完。

这么弄了后，观察下来增量爬取好像没什么问题了。

### 总结

我们很难想象是不是已经考虑到了所有的情况，所以可以忽略的部分就尽可能善用异常机制，不能忽略的就观察错误，改进代码。庆幸东财没反爬虫。。。通过这次的爬取，还是对爬虫有了一定的感觉的。