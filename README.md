# 1. 不同券商行情处理

我们有个程序，需要部署到很多券商处。该程序其中一个模块在收到券商推送的股票行情时，需要利用行情计算该股票此时的中间价并打印出来，假设该函数名为`printMidPrice`。中间价 = (申卖价一+申买价一)/2。

每个券商行情数据结构不一样，但表达的意义是一样的，都包含某个股票某个时刻的买卖五档上的价格和数量。

譬如券商A给的行情数据结构为`MarketDataA`，定义为：
```c++
struct MarketDataA {
    char instrumentId[20];  //股票代码
    double askPrice1;       //申卖价一
    uint64_t askVolume1;    //申卖量一
    double bidPrice1;       //申买价一
    uint64_t bidVolume1;    //申买量一
    double askPrice2;       //申卖价二
    uint64_t askVolume2;    //申卖量二
    double bidPrice2;       //申买价二
    uint64_t bidVolume2;    //申买量二
    double askPrice3;       //申卖价三
    uint64_t askVolume3;    //申卖量三
    double bidPrice3;       //申买价三
    uint64_t bidVolume3;    //申买量三
    double askPrice4;       //申卖价四
    uint64_t askVolume4;    //申卖量四
    double bidPrice4;       //申买价四
    uint64_t bidVolume4;    //申买量四
    double askPrice5;       //申卖价五
    uint64_t askVolume5;    //申卖量五
    double bidPrice5;       //申买价五
    uint64_t bidVolume5;    //申买量五
    // 其余字段
    // .......
};
```
假设券商A通过`onQuoteA`回调函数通知我们收到新的行情，我们在里面调`printMidPrice`打印该股票的中间价：
```c++
void onQuoteA(const MarketDataA& quote)
{
	printMidPrice(XXX);
}
```

而券商B给的行情数据结构为`MarketDataB`，定义为：
```c++
struct MarketDataB {
    std::string code;       //股票代码
    double askPrice[5];     //申卖价
    uint64_t askVolume[5];  //申卖量
    double bidPrice[5];     //申买价
    uint64_t bidVolume[5];  //申买量
    // 其余字段
    // .......
};
```
假设券商B通过`onQuoteB`回调函数通知我们收到新的行情，我们在里面调`printMidPrice`打印该股票的中间价：
```c++
void onQuoteB(const MarketDataB& quote)
{
	printMidPrice(XXX);
}
```

`printMidPrice`的实现会放在一个库里，不管接入多少券商，其实现不能修改。

请实现`printMidPrice`函数以及从A和B券商收到行情后的处理流程。

不能使用**额外的内存**来转换和存放行情。比如定义一个中间的数据结购，`MyMarketData`，在所有券商行情回调里将行情转换一下再调用`printMidPrice`：

```c++
struct MyMarketData {
	std::string code;
    double askPrice;       //申卖价一
    double bidPrice;       //申买价一
};

void printMidPrice(const MyMarketData& quote)
{
    double midPrice;
    // 计算mid price
	midPrice = (quote.askPrice + quote.bidPrice) / 2;
    cout << "symbol:" << quote.code
        << ",mid price:" << midPrice << endl;
}

void onQuoteA(const MarketDataA& quote)
{
    MyMarketData data;
    data.code = quote.instrumentId;
    data.askPrice = quote.askPrice1;
    data.bidPrice = quote.bidPrice1;
	printMidPrice(data);
}

void onQuoteB(const MarketDataB& quote)
{
    MyMarketData data;
	data.code = quote.code;
    data.askPrice = quote.askPrice[0];
    data.bidPrice = quote.bidPrice[0];
	printMidPrice(data);
}
```

# 2. 风控设计

有一套订单执行系统，它接收策略的报单信号，进行一系列处理，根据处理结果确定是拒绝报单还是把订单送到市场。

其中一个流程是风控（风险控制），如果报单满足风控要求，返回true，否则返回false。

风控模块需要满足不同产品的要求。假设有3个产品（A，B，C），产品A的要求是创业板股票市值不超过30%，每个股票市值占产品总市值比重不超过5%。产品B风控要求是不能持有创业板股票，撤单率不能大于50%。产品C的风控，每个股票市值占产品总市值比重不超过10%，不能交易商品期货。

假设每个产品对应一个账户，策略的报单信息在`Order`结构里，其中包含了账户信息：
```c++
struct Order
{
    uint64_t accountId; //报单账户（对应某个产品）
    double price;       //报单价格
    int32_t qty;        //报单数量
    // 其他字段...
};
```


请设计一个风控系统满足上述要求。假设风控系统入口是`bool Risk::check(const Order& order)`。

该系统需要哪些类构成？各需要什么样的对外接口？
请实现入口函数`Risk::check`的逻辑，以及Risk类的必要初始化操作。其他具体的风控的逻辑可以不实现，只需要写好接口以及注释。




# 3. 回报分发

订单执行系统除了要处理报单，也要处理回报。

系统里一般有多个模块需要回报。譬如有的模块需要收回报，做一些统计用于风控。有的模块需要收回报，然后通过tcp连接发给策略。假设模块`class Api`负责和市场通信，`Api::onRtnOrder`回调函数从市场收回报并负责分发给内部各个模块。
```c++
// 从市场收到订单的回报
void Api::onRtnOrder(const OrderRtn& rtn)
{
    // 把回报发送给其他模块
}

```
假设有3个模块`class ModuleA`，`class ModuleB`和`class ModuleC`需要回报。请设计一个回报分发系统，在`class Api`收到市场里的订单回报后分发给其他模块。需要考虑扩展性，譬如又有一个新的`class ModuleD`需要回报的情况下，不能修改`Api`模块的代码。

# 4. 回报触发

有一个低频策略，其根据收到行情做一些计算，确定是否要报单。报单可能有两种行为：a) 报到内部系统，b) 报到外部（报给订单执行系统）。
报到内部系统还是外部，是初始化时确定的（通过参数），初始化后就不再改动。不管订单是报给内部还是外部，策略的行为要完全一样（包括内部状态维护和log打印）。

策略大致框架如下：

```c++
// 收到行情
void Strategy::onMarketData(const MarketData& md)
{
	std::unique_lock<std::mutex> lock(m_Lock);
    Order order;
    // 获取策略内部数据，并结合行情做一些计算
    // .......
    if (m_orderInternal)
    {
        sendOrderInternal(order);
    }
    else
    {
        sendOrderExternal(order);
    }
}

// 内部报单
void Strategy::sendOrderInternal(const Order& order)
{
    // 报到内部程序
    // ....

    /// 认为订单已成交，请完成回报触发逻辑。不能修改其他函数。
}

// 外部报单
void Strategy::sendOrderExternal(const Order& order)
{
    // 报到外部程序
    // ....
}


// 回报，反馈订单的状态，如成交或撤单
void Strategy::onRtnOrder(const OrderRtn& rtn)
{
	std::unique_lock<std::mutex> lock(m_Lock);
    // 仓位更新，log打印，策略状态维护等
    // ....
}
```

如果是外部报单，有一个线程从订单执行系统收到回报后会调`Strategy::onRtnOrder`回调将订单的状态更新通知给策略。如果是内部报单，则没有这个逻辑。假如内部报单的话策略报出去后即可认为已经成交，为了维护策略内部状态正确，内部报单报完后需要自己触发回报。

请完成`Strategy::sendOrderInternal`的回报触发逻辑。可以借助标准库或任何第三方库。
