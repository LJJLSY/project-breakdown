# 项目整体架构图  
<img width="1077" height="612" alt="image" src="https://github.com/user-attachments/assets/12b10aef-0f6c-4700-9f75-c1088591cf2e" />  



# 核心业务流程  
<img width="1200" height="710" alt="image" src="https://github.com/user-attachments/assets/d1a8ab9b-5c30-468f-8a52-528b02704486" />
  

# 后端流程  
<img width="1197" height="819" alt="image" src="https://github.com/user-attachments/assets/3b6867e8-7c3d-4e64-a4aa-1acdbde0f51b" />
  

# 主要模块职责  
**config模块**  
存放postgres、redis、rpc、Markets、MatchEngine撮合引擎、FundingRate、Liquidator、Chainlink、USDC 转收款地址、Coinbase等配置  
**数据存储**  
postgres存储链上的数据进行持久化，redis存储缓存数据  
**USDC Transfer监听**  
用轮询+事件日志解析的方式同步链上USDC Transfer(to是链上USDC转收款接收地址)事件的数据，写入数据库里deposits表并累加ledger_balances表，可以批量同步过去时间指定范围的区块  
**IndexPrice监听**  
连接Coinbase交易所获取指数价，写入数据库market_quotes表  
**MarketWebSocket**  
注册API定时从链上预言机刷新各市场价格（链上标记价）  
**路由模块**  
按account、funding、liquidation、market、order、position、trade等分成多个Group handler  
account查询账户余额、存款记录、取款记录  
funding获取资金费率历史  
liquidation获取清算记录  
market获取K线数据、所有市场及市场详情、深度、行情等数据  
order创建订单、查询订单列表、查询订单、取消订单  
position查询仓位和风险信息  
trade查询交易记录  
**撮合引擎模块**  
内存订单簿 + 周期性调用 chain.BuildMatchTradeData / Perpetual.trade  
用perpAddresses从 DB 恢复未成交（pending / partial）订单到订单簿，进行撮合，调用chain.BuildMatchTradeData编码撮合结果生成tradeData，构建交易信息并签名，调用Perpetual.trade提交上链，等待收据；链上成功更新 DB  
**清算机器人模块**  
从数据库查询有仓位的交易者，检查交易者仓位，查询链上安全状态，获取仓位信息，执行清算，等待收据，链上成功更新 DB  
**资金费率模块**  
资金费率结算，用标记价（MarketWebSocket获取的价格）与指数价（IndexPrice）差计算 delta，配置周期调用链上Dealer.updateFundingRate，再把更新记录入库  

# 从业务角度分析代码实现  
