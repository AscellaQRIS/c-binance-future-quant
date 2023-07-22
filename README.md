# 简介

该项目为币安合约的架构，有超过100亿美金交易量和超过一年时间实盘验证，包含数据录入，风控，交易，数据分析，但不包含具体策略。

个人认为其运行速度达到了除获取币安内网地址读取行情，下单外的最优解

前端演示地址：[8.217.121.203](http://8.217.121.203/)，目前放1000美金运行着一个年化100%~200%的高频左侧回归策略

你可以利用它简单，低成本的实现你的交易逻辑，其大量运用阿里云服务器进行分布式架构，多进程处理，以及飞书进行异常报错和交易信息披露

如果你愿意详细阅读该readme的所有信息，尤其是 [模块详细解析](#模块详细解析) ，那么他同时也会是一部关于币安合约的交易风控，设计架构的经验理解历史，总结了几乎本人所有成功和失败的经验，希望能让后来者少踩些坑


# 优势

低成本，高效率，简单实现是这套系统的三个优势

不到1000人民币一个月的成本，实现每分钟扫描约1500万次交易对是否满足交易条件

除了撮合服务器（C++）外都采用python编写，简单易懂

大量的分布式架构实现更快的速度，并且可以根据个人需求，自由伸缩的调控服务器数量来实现成本和性能的平衡

通过多重接口读取行情/账户信息，并根据更新时间戳进行整合，最大程度的降低数据风险

企业级别的风控安全解决方案

# 架构

该系统通过一个C++服务器作为主撮合服务器，大量的可伸缩调整的分布式python服务器作为数据采集服务器。

将采集的数据，包括Kline ，trades，tick等，喂给C++服务器。

交易服务器再从C++服务器统一读取数据，并且在本地端维护一个K线账本，来避开交易所的频率限制，实现高效率，低成本的数据读取。

又比如，账户的余额，持仓数据，在币安上有三种方式获取，a是position risk，b是account，c是ws，那么会有三台服务器分别采用这三种方式读取，然后汇入C++服务器进行校对，通过对比更新时间截取最新的数据，然后服务给交易服务器

前端数据板块，通过阿里云oss为中介，网页读取oss数据的方式进行展示，隔离数据风险

# 作者自述

2021年，我从一家top量化公司辞职后做起量化交易，主要战场在币安，这两年间，我从做市商->趋势->套利等类型均有涉及，最高峰的时候，在币安一个月有接近20亿美金的交易额。

至2023年7月，因为各种原因，大方向上失败了，只遗留下一个朋友的资金在继续运行一个比较稳定盈利的左侧交易策略。

这是在这两年时间里面探索出来的一套高效率，低成本的数据读取，录入框架，同时包含了一套风控系统，他更像一个架构，而不是一个实现，你同样可以通过简单的修改替换运用到okex，bybit等等上

资金合作或者工作机会（不谈任何涉及策略原理和源码，请开门见山节省双方时间），请联系微信号 melelery 或邮件至c.binance.quant@gmail.com

# 环境与启动

我方实盘的项目运行环境为Ubuntu 22.04 64位，Python 3.10.6

python文件采用以下方式运行，以webServer为例

```
ps -efwwww |grep webServer.py | awk '{print $2}' | xargs kill -9
dos2unix webServer.py
chmod +x  webServer.py
nohup ./webServer.py >/dev/null &
```

但是更推荐使用 [dataPy/uploadDataPy](#dataPy/uploadDataPy) 的方式

react前端项目

```
npm install
```

```
npm start
```

C++文件

```
g++ wsServer.cpp -o wsServer.out -lboost_system
```

```
dos2unix wsServer.out
chmod +x  wsServer.out
nohup ./wsServer.out >/dev/null &
```

由于项目使用的全部库和包都是官方或者热门项目，在google可以查找到安装方式，此处不再累述如何安装环境

如果需要最简启动方案，可邮件 c.binance.quant@gmail.com 联系我方，直接共享系统镜像到你方阿里云账号，收取100 USDT的技术费用


# 模块详细解析

我方设计的模块包括 [通用部分](#通用部分) ，[数据处理部分](#数据处理部分) ，[关键操作部分](#关键操作部分) ，[安全风控部分](#安全风控部分) ，以及没有开源的具体交易逻辑部分

该项目的最小化开启需求为 一台web服务器，一台ws服务器，一台tick数据读取服务器，一台one min kline数据读取服务器，以及一台交易服务器

你可以根据自己的需求，扩展相关的服务器，例如需要 15 m 的kline，需要trades等等，只需要在这个基础上增加即可，这里没有盘口数据的整合，除了买一卖一。

对于更高维度的盘口数据，我方选择在交易服务器里面进行读取

因为除了kline数据，其他数据进行这种整合并没有太大的实际意义

200个交易对的kline数据，通过这种方式可以减少调用接口的频率，以及在某些接口有延迟的情况下依然能通过校对获得最新数据

但是盘口数据的api就一个，所有没有整合需求。

我方对于盘口数据延迟的解决方案是通过交易服务器的分布式运行。

如架设了五台交易服务器，运行着一样的开仓关仓逻辑，在满足前置开仓条件后，五台服务器会同时进入盘口数据读取的流程，同时间多进程的读取同一个数据，可以优化延迟等意外状况。

这里顺便引申出一个交易服务器发出订单的方式。

有两种，一种是直接在交易服务器上发出，一种是做一个发出交易指令的web服务器，交易服务器通过http请求到内网web服务器后统一发出。

比方，五台交易服务器，假设总交易量需求是100u，那么就每台服务器负责20u的交易量，如果延迟了或是其他问题，某一台服务器丢失了信号那就是损失20u的交易量

而如果采用web服务器统一发出，那么会在第一次接收到http请求的时候，即开出100%的交易量

两种方式各有适用的地方和优势，需要自行抉择，实际上在webserver文件里面已经整合了第二种方式。


我方目前维护的实盘项目，具体服务器配置为，（服务器名即为开源的文件名，在下方有更详细的单个文件介绍）

一台 wsPosition 服务器，用于通过ws的方式读取币安的仓位和余额数据，汇入ws数据撮合服务器

一台positionRisk服务器，用于读取/fapi/v2/positionRisk接口，通过该接口获取仓位信息并实时判断损失，以便于及时止损，该服务器和wsPosition，makerStopLoss，getBinancePosition具备类似功能，之所以设计多种交叉相同功能的服务器，是为了最大限度的防止风险，在币安的某一接口出现延迟的情况下，系统依然可以健壮的运行，以下不再重述

一台makerStopLoss服务器，用于从getBinancePosition服务器读取仓位信息后，读取单独的挂单信息，然后预设止损单，之所以与getBinancePosition服务器进行拆分，是因为读取挂单需要的权重较高，拆分成两个ip可以更高频率的进行操作

一台getBinancePosition服务器，用于读取/fapi/v2/account接口，获得仓位信息和余额后汇入ws数据撮合服务器

一台commission服务器，用于记录流水信息

一台checkTimeoutOrders服务器，用于读取/fapi/v1/openOrders接口，查询全部挂单，然后取消超过一定时间的挂单，或者进行一些额外的操作，例如挂单三秒没被吃，则转换成take订单等等

一台cancelServer服务器，本质上也是web server服务器，运行webServer文件，用于取消订单

一台webServer服务器，本质上也是web server服务器，运行webServer文件，用于大部分程序一开始读取交易对等等

两台oneMinKlineToWs服务器，用于低频率读取一分钟线的kline

两台volAndRate服务器，用于读取交易量数据做分析并提供给其他服务器

十台specialOneMinKlineToWs 服务器，用于高频率读取一分钟线的kline

十台tickToWs服务器，用于读取 tick 群信息

一台ws服务器，用于C++的数据戳合服务器

以及五台交易服务器，除了运行交易程序同时运行webServer，提供接口给checkTimeout获取全部挂单数据

合计38台服务器，以及一台最低配置的mysql数据库

其中对于开仓服务器和ws服务器，选择了高主频类型的服务器，约有一倍速度的提升，其他的行情和数据录取服务器并无必要进行这个优化，只需要选用最低配置的抢占式服务器

综合成本一个月不足3000元人民币，大头在流量费用，个人认为上述数目对半砍依然可以满足大部分量化的风控和延迟需求


在该项目里，一个单独的模块，需要一台服务器一个IP单独运行，目前基本已经将单模块的https读取频率调教到币安允许的最大值。

此处需要说明的是，这里都是以阿里云东京为例子，币安的服务器在亚马逊云东京。

而本文所叙述的延迟，其实包含两种延迟，一是读取频率的延迟，二是网络的延迟，这两种延迟综合计算后，才是真实环境下的最终延迟。

之所以使用阿里云是因为阿里云抢占式服务器具备成本优势，从而具备读取频率延迟的优势。

阿里云网络延迟约为10ms，而亚马逊在不申请内网权限的情况下预计为1~3ms，申请内网则面临锁定ip的问题，即无法通过铺开更多ip的手段去降低延迟。

虽然亚马逊云具备更低的延迟，但是由于

1. ws类型的数据读取通常被币安锁定100ms以上的延迟，且ws类型的读取数据具备某些不可确认的风险因素，所以该方式被排除

2. 如果采取https读取的方式，某些数据的读取权重高达20，甚至是30，由此推演出需要多IP，进行分布式读取才能具备更高频率，而这个时候，单个IP的成本价格即成了需要考虑的因素，阿里云抢占式服务器的成本一个月不足20人民币，在综合的性价比考虑后，我方选择了日本阿里云。

3. 该方案并不是一套服务于高频（纳秒级别）交易的解决方案，否则全部都会用C++编写，实际上他是一套追求成本，延迟，开发速度，三者均衡最优解的方案，并为毫秒级别的策略服务

## 通用部分

### react_front文件夹

前端文件，该网页为对外网页，所以强制锁定了一分钟的时间间隔，展示的数据也比较简单，毕竟是对外的

我亦有设计内网详细的数据分析网站，但是这一部分实际上需要更具自己的量化策略去进行定制，且一旦公开存在策略泄露的风险，所以不在此处披露。

另外一提，交易后的数据处理，以及前端代码，都是以实现功能为主，所以并不注重性能等其他指标，可参考最下方faq

### afterTrade文件夹

服务器如何更新到前端数据的文件，主要原理是利用阿里云oss作为中间件，隔离数据风险

其中的tradesUpdate.py为更细trade记录，需要你在下单的时候先调用webServer的begin_trade_record接口插入交易数据

请注意这个接口的设计是基于我自身的需求，因为每个人的量化模型，参数等不同，需要研究的也不一样，所以建议这一块自己重写

positionRecord.py主要记录每分钟的账户余额和持仓价值 ，服务于下面的文件

webOssUpdate.py会整理数据，上传到oss，web前端则从oss中读取数据，并且会整理交易流水记录，形成一个以天为单位的统计表格，该处只是简单的统计了盈利和手续费，你可以自行扩展。

### binance_f文件夹

币安涉及到api key的接口的处理包，是从币安官网推荐的github下载后，进行了二次改造的版本

### config.py

通用配置，需要自行配置mysql数据库，以及申请飞书api key等

### commonFunction.py

通用方法

### updateSymbol/trade_symbol.sql

在数据库中生成trade_symbol表格，该表格将控制系统可执行交易的交易对信息

### updateSymbol/updateTradeSymbol.sql

向trade_symbol表格录入交易对信息

此处进行了一些特殊处理，主要是适应我方的情况，包括只录入usdt的交易对，不录入指数类型的交易对（如btcdom，football这类型）

此处大部分字段为配合另一个项目，专业手操工具而设计，用于量化的数据字段实际上只有symbol，status

### simpleTrade

一个最基础的交易演示程序，当某个交易对，持仓价值为0且一分钟涨幅>1%的时候开多，当他持仓价值>0且一分钟跌幅小于-0.5%平多

如果你是新手，建议关注updateSymbolInfo()这个函数，价格精度，数量精度，最大平仓数量应该是新手会遇到最多的问题。

## 数据处理部分

### wsServer.cpp

撮合服务器

所有的数据都会汇入这里，部分多来源数据会根据数据自带的更新时间戳判断是否更新该条数据，使用以下命令行可编译成可执行文件

g++ wsServer.cpp -o wsServer.out -lboost_system

该源码使用了两个库 一个是websocketpp，一个是boost

### dataPy/uploadDataPy

将程序从一个阿里云的主控服务器，上传到各个对应的阿里云服务器，运行，然后销毁。

这里只展示tick，oneMinKlineToWs，specialOneMinKlineToWs三个数据录入程序的使用，其他程序亦同理

该程序可以简单快速的发布分布式运行的程序，到所有符合命名规则的服务器上云运行和销毁。

使用前需要将阿里云服务器进行统一命名，如tickToWs_1,tickToWs_2...

使用前先从本地上传文件到某一个主控服务器，然后在主控服务器运行该程序，正常运行后，包括主控服务器和实际运行的服务器上的所有硬盘应都被覆盖掉源文件的信息。

程序会调用get_aliyun_private_ip_arr_by_name函数，搜索对应字符段的阿里云服务器的私网地址，然后上传，执行，并且在三秒后判断有没有在正常运行

如正常运行，则其后会对硬盘数据进行覆盖销毁，防止机密数据泄露，只保留程序在内存运行

由于是私网地址的操作，需要在同地域的阿里云服务器上执行该程序

### dataPy/oneMinKlineToWs.py

该程序属于分布式运行架构，只需要标准化命名即可无限扩展服务器降低延迟
![image](https://github.com/Melelery/c-binance-future-quant/assets/139823868/801409a3-25b7-41c8-b795-d7aa0efd0fe6)

缓更新的1分钟的K线数据读取程序

每次程序运行的时候，会像ws服务器发送目前交易对的总数。

每次读取kline数据前，会从ws服务器拿到一个交易对编号，拿取的同时，ws服务器会对编号执行 +1的操作，确保分布式架构的时候，每台扩展oneMinKlineToWs都可以按照最佳顺序读取交易对的kline数据。

由于币安的k线数据更新延迟略高于tick数据，k线数据来源于数据库，而tick数据来源于缓存，同时，单symbol的延迟会低于全部交易对信息，所以每次更新前还会再读取一次交易对单独的tick数据去校正最后一条k线数据

kline数据会向ws服务器发送两次数据，一次是所有读取到的数据，比如说读取kline的时候，设置limit=45，即读取了最近45条kline的数据，但很显然，前面43条的数据是一直不变的，所以交易服务器只需要长时间（30秒/1分钟...）去校正一次即可，只有最新的两条数据的变化概率是大的，需要实时的去读取。

所以我拆分成了两条信息，一个是前两段kline，一个是全部kline，前两段kline用于交易服务器实时读取，实时更新，后面两端则用于一定时间间隔后的校对。

缩减消息的长度对于交易服务器解析消息所用的时间有极大的帮助。

其他时间间隔的k线，如5分钟 ，15分钟，1小时等等读取和打入ws服务器的过程亦同理，只需要简单的替换文件中的参数即可实现，所以此处不再列出

### dataPy/specialOneMinKlineToWs.py

该程序属于分布式运行架构，只需要标准化命名即可无限扩展服务器降低延迟

急速更新的1分钟的K线数据读取程序

与上面不同的是，此处的读取数据有一个前置的交易量条件，你可以理解为我的量化系统只有满足某个交易量条件的要求时候才会开仓，所以针对这一部分可能开仓的交易对，铺设了专门读取数据的服务器。

假设本来有200个交易对轮流读取，限制条件后，变成了20个交易对，那么等于你单个机器的数据读取数据提高了10倍

这个只是一个展示程序，实际上你应该根据你的交易条件去自己编写相应的条件，限制数据读取的交易对。

其他时间间隔的k线，如5分钟 ，15分钟，1小时等等读取和打入ws服务器的过程亦同理，只需要简单的替换文件中的参数即可实现，所以此处不再列出

### dataPy/tickToWs.py

该程序属于分布式运行架构，只需要标准化命名即可无限扩展服务器降低延迟

tick数据读取程序会读取阿里云上，所有的tick服务器数量，然后自动锁定该服务器在一秒内的某个时间段内去进行数据读取

打个比方，现在我们开通了五台tick服务器，那么tick 1服务器会在每一秒的>=0 <200毫秒的时间段内去读取数据，tick 2会在每一秒的>=200 <400毫秒的时间段内去读取数据...以此类推

tick数据在汇入ws服务器后，交易程序读取这部分主要用于修正最新一条kline的最高价格，最低价格和最新价格。

### dataPy/useData.py

展示了如何从ws服务器拿取one min kline数据和tick数据后，如何在本地端自行拼合，维护一个k线数据，此处还可以扩展到加入trade vol等数据，原理相同所以不再展示

## 关键操作部分

### binanceOrdersRecord.py

记录orders信息，方便后续分析，例如可以通过orders的记录分析出发出去的订单的总成交比例等等

### binanceTradesRecord.py

记录trades信息，方便后续分析，例如可以通过trades计算出总交易量等等

### checkTimeoutOrders.py

检查是否有超时订单并取消，同时可以附加一些交易操作，如maker挂单超过多少秒没有成交则按比例转化成take订单

由于需要调用币安获取全部挂单的api，而该api权重极高，为了满足更加灵敏的扫描，我的五个交易服务器同时运行有webserver程序，checkTimeoutOrders会轮流从五个交易服务器读取全部挂单信息

### commission.py

记录所有资金流水，这个是最重要的数据，通过他可以计算出手续费，盈利，资金费用等等数据

由于commission长时间积累的数据量大，所以这里有两个表，一个是持续记录的表，一个是24小时临时记录的表。

24小时记录的临时表主要用于分析最近一天的亏损情况，从而给交易系统进行风险控制

比如以下这段代码
```
for key in fourHoursProfitObj:
    if fourHoursProfitObj[key]<=-150 or oneDayProfitObj[key]<=-1800:
        banSymbolArr.append(key)

if allOneDayProfit<=-3000:
    banSymbolArr = ["ALL"]
```
当读取到某个交易对四小时盈利小于150u或者24小时利润小于1800u时，会向ws服务器发送一个禁止交易的交易对列表，当所有交易对总利润小于-3000u时，则直接全部暂停交易

### getBinancePosition.py

通过/fapi/v2/account接口获取币安仓位和余额信息，并且上传到本服务器的80端口，以及ws服务

上传到ws服务器的信息，会与positionRisk和wsPosition拿到信息的更新时间戳进行对比，选择最新的信息发送给交易服务器

其他服务器通过80端口读取json文件获取数据，旧版本使用的方案，后面采用了ws但是这里保留下来了

### positionRisk.py

通过/fapi/v2/positionRisk接口获取币安仓位和余额信息，其他同上

### wsPosition.py

通过websocket接口获取币安仓位和余额信息，其他同上

### makerStopLoss

读取ws服务器读取仓位信息后，读取该币种的币安接口挂单信息，之所以不采用所有symbol的挂单信息，是因为权重太高，会导致止损过于迟钝。

在仓位最大值发生超过5%变化的同时，挂出最大止损单，演示文件的写法，是以成本价5%为初始止损价格，并且拆分成五单，每单往后增加0.5%的价格进行止损，防止深度的影响

例：现在仓位是1000u，已有止损单，那么当仓位增加到1001u的时候，该系统不会重设止损，因为不满足数量变化>5%的情况，太过灵敏的重设会到权重消耗等等问题

如增加到1060u，那么会重新设定五个止损单，初始止损价格为成本价的5%，后续每个止损单依次为5.5%,6%,6.5%,7%,

新止损设置完后，系统会读取挂单检查是否成功，确认成功后，才会撤销旧的止损单。


## 安全风控部分

由于程序源码会带有敏感信息，建议采用dataPy/uploadDataPy的上传方式，统一进行文件的上传，运行，销毁，实现最终只在内存中运行，而覆盖硬盘所有储存的程序信息的目的，仅在本地段保有源码。

建议关闭阿里云所有对外的端口，在购买服务器的时候选中将所有服务器置于统一私网前缀IP下，这样即可在关闭外网端口的同时实现功能正常运转和互通，如没有在统一私网前缀IP下，则需要添加对应的私网IP到部分服务器的安全组内

在需要操作服务器的时候，再将本地的IP添加到安全组内，并且使用完后即时删除

建议币安的api绑定服务器的IP

建议保持操作系统的更新

# FAQ

## 1.为什么不用历史数据先回测

事实上我个人探究量化实打实算起来已经有三年。

以历史数据回测后拟合一条折线出来的方式并不是没有尝试过。

但是回测的环境要做到与实盘一致的难度可能超过了目前大部分人的估计，我说的一致是绝对的一致，任何一点小差异其实都会在整个过程被放大到最后让你无法接受的程度。

并且存在一个无法验证到底是误差还是策略造成的损益差异的问题。

因为很可能到你结束项目，还存在数十个你没有发现的误差的地方，而且不具备量化判断这些误差造成损益的可能性。

当然这是以我的能力和视角得出的结论。

所以自最近半年起，我的思路都是直接上实盘，哪怕是小资金验证后，以实盘的数据为基础去调参

## 2.部分存在性能改进的可能性

是的，因为这个是一个个人项目，我要负责的事情非常多。

所以对于一些不需要追求性能的地方，我都会用最简单的写法带过。

比如说orders，trades这些录入数据库是拿最近1000条数据出来比较，没有重复的就插入，这里当然存在性能更优解的写法，但是在我看来意义不大所以我没有花时间去改进。

为什么意义不大，因为我用最低配的mysql整套系统运行的过程中cpu和内存的使用比例也没有超过50%。

我大部分精力关注在数据录入，交易的性能优化，而忽略交易后数据拉取分析这一块的优化，这一块只要最后的结果是对的即可。

交易后的过程，消耗了1个性能还是100个性能，只要没有达到我硬件的峰值，我都不会去寻求改变

