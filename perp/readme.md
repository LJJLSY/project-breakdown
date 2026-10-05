# 项目整体架构图  
<img width="1077" height="612" alt="image" src="https://github.com/user-attachments/assets/12b10aef-0f6c-4700-9f75-c1088591cf2e" />  



# 核心业务流程  
<img width="1200" height="710" alt="image" src="https://github.com/user-attachments/assets/d1a8ab9b-5c30-468f-8a52-528b02704486" />  
1、资金流程  

用户通过USDC合约将保证金存入dealer合约，dealer合约将用户的primaryCredit或secondaryCredit主次资产增加记账。  
取款时用户发起请求取款，等待时间锁到达才能执行取款；如果是白名单用户，可以用快速取款直接取不用等待时间锁。  

2、交易流程  
多空双方用户存入保证金后，在链下构建订单并签名，由撮合引擎匹配订单生成撮合结果，然后构建交易信息提交到链上进行结算，结算后返回交易结果，更新双方余额（paper和credit）。  
有仓位后每8小时结算一次资金费率，如果资金费率为正，多头要支付资金费给空头；如果资金费率为负，空头要支付资金费给多头。  

3、清算/平仓流程  
系统检测可清算仓位，并验证是否安全，如果不安全则触发清算，合约计算清算价格，然后执行清算，变更被清算者和清算人的余额  
如果一直没有触发清算，用户可以主动平仓

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
1、资金流程  
用户先调用USDC合约的approve函数授权dealer合约足够的额度，然后调用dealer合约的deposit函数将保证金存入dealer合约，deposit函数里面会调用USDC合约的transferFrom函数将用户的存入保证金数量转到dealer合约，并在primaryCredit或secondaryCredit主次资产余额中增加记账，完成存款保证金流程  
取款时用户调用dealer合约的requestWithdraw函数发起请求取款，如果是普通用户，需要等待时间锁到达才能执行取款，执行取款时调用dealer合约的executeWithdraw函数，要先验证msg.sender是不是本人或者有授权的人取款，然后可以根据需要确定是否实际执行转账，可以传入一个isInternal，如果为true则调用safetransfer实际转账，如果为false则不实际转账，只在dealer合约内部变更primaryCredit或secondaryCredit主次资产余额的记账用户，以此减少gas消耗；如果是白名单用户，可以调用dealer合约的fastWithdraw函数快速取款，直接执行取款不用等待时间锁，但是也要先验证msg.sender是不是本人或者有授权的人取款。  

2、交易流程  
多空双方用户存入保证金后，在链下构建订单并签名，由撮合引擎匹配订单生成matchResult，然后构建tradeData交易信息，调用perpetual合约的trade函数提交到链上进行结算，trade函数会调用dealer合约的approveTrade函数，在里面验证：orderSender权限、订单签名 (EIP-712)、检查订单未过期，验证通过后计算并扣除手续费，返回交易结果给perpetual合约，perpetual合约再根据交易结果更新双方余额（paper和credit）。  
有仓位后链下Keeper定期（每8小时一次）获取标记价（MarketWebSocket获取的价格）与指数价（IndexPrice）差计算 delta，然后加上通过perpetual合约获取的旧资金费率，算出新资金费率，调用 dealer合约的updateFundingRate，updateFundingRate会调用perpetual合约的updateFundingRate更新该合约市场的资金费率  

3、清算/平仓流程
