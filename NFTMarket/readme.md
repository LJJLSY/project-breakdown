# 项目整体架构图  
<img width="867" height="612" alt="image" src="https://github.com/user-attachments/assets/443fec9b-71df-4b8a-acf0-df49501f2537" />


# 核心业务流程  
<img width="1200" height="441" alt="image" src="https://github.com/user-attachments/assets/20aa70b2-d081-40d6-bd41-0c45160616aa" />  
1、初始化所有继承的合约：  

OrderStorage: 订单存储  
ProtocolManager: 协议费用管理  
OrderValidator: 订单验证（EIP712签名验证）  
以及安全管理合约：  
Context: 上下文管理  
Ownable: 所有权管理  
ReentrancyGuard: 重入保护  
Pausable: 暂停功能  

并设置Vault金库合约  

2、批量创建订单：  
对于Bid订单，会累计ETH总金额，将ETH转移到Vault金库，数量不能为0，如果传入的ETH多于实际需要的金额则退回多余ETH，如果ETH不足则回滚交易。  
对于List订单，要先授权NFT给Vault金库，创建订单时会将NFT转移到金库，限制数量为1。  
创建订单时要验证订单规则：  
*只有订单创建者可以创建订单（order.maker == msg.sender）  
*价格不能为0  
*salt不能为0  
*过期时间必须大于当前区块时间戳，或为0（永不过期）  
*订单不能已被取消或完全成交  
验证完后将订单写入Storage存储合约  

3、批量取消订单  
只有订单创建者可以取消自己的订单，并且订单必须未完全成交，否则跳过订单，取消订单时从订单存储Storage合约中移除订单。对于List订单，从金库提取NFT返回给创建者；对于Bid订单，从金库提取未成交部分的ETH返回给创建者  

4、批量编辑订单  
编辑订单实际是先取消旧订单再创建新订单的过程  
编辑限制检查：saleKind、side、maker、nft（collection和tokenId）必须与旧订单一致，只能修改价格price和数量amount，订单不能已完全成交  
新订单验证：新订单的maker必须是调用者，salt不能为0，过期时间必须有效（大于当前时间或为0），新订单不能已被取消或完全成交  
验证完先取消旧订单再创建新订单  
资产处理：对于List订单：直接更新金库中的NFT关联；对于Bid订单：如果新价格更高：需要补足差额ETH，如果新价格更低：金库会退回多余ETH  

5、撮合单个订单  
要验证sellOrder订单和buyOrder是否匹配，只有匹配才能撮合  
支持两种撮合场景：  
*卖家接受出价：sellOrder.maker调用，验证buyOrder，要求Bid订单必须存在于订单存储中，此时成交价格为Bid订单的价格，然后更新成交数量，进行资产转移，从金库提取ETH到OrderBook合约，扣除手续费后转给卖家，NFT则从卖家转移到买家  
*买家接受挂单：buyOrder.maker调用，验证sellOrder，要求List订单必须存在于订单存储中，此时成交价格为List订单的价格，然后更新成交数量，进行资产转移，从金库提取ETH到OrderBook合约，扣除手续费后转给卖家，NFT则从卖家转移到买家。如果买家出价高于成交价，退回多余ETH，如果买家传入的ETH高于实际花费的ETH金额，退回多余ETH  

6、批量撮合订单  
使用delegatecall执行步骤5进行批量撮合，如果撮合失败，记录事件但不回滚，如果撮合成功，判断是否买家发起的撮合，买家发起撮合就累积已花费的ETH，批量撮合完，如果传入的ETH多于实际需要的金额，退回多余的ETH  

7、聚合调用多个操作（单笔交易内串联多个操作）  
支持的操作：makeOrders、cancelOrders、editOrders、matchOrder、matchOrders。要验证调用的函数是否在其中  
由于delegatecall下每个子调用看到的msg.value相同，为避免资金语义歧义，一次聚合调用最多允许1个“可能消耗msg.value”的子调用  
根据revertOnFail参数决定策略：为true时任一失败将整笔回滚；为false时仅记录失败并继续  

# 后端流程  
<img width="1161" height="488" alt="image" src="https://github.com/user-attachments/assets/73390884-f2fe-4695-8a7a-ec9fb61a87f7" />  

# 主要模块职责  
**config模块**  
存放mysql、redis、rpc及其他配置  
**数据存储**  
mysql存储链上的数据进行持久化，redis存储缓存数据  
**路由模块**  
经Sync模块将链上数据同步到数据库后，由Api路由获取给到前端  
按Collection、Use、Order、Activity、Admin等分成多个Group handler  
Collection处理Collection的详细数据，指定Collection的item列表或指定item数据，以及Collection相关其他数据  
User处理User相关的Collection、item数据以及login  
Order处理User相关的bid、list数据，以及Collection的bid、list数据，Collection指定item的bid、list数据  
Activity处理Activities数据  
Admin处理Admin权限数据  
**Sync模块**  
用轮询+事件日志解析的方式同步链上数据到数据库，可以批量同步过去时间指定范围的区块  

# 从业务角度分析代码实现  
1、合约升级：  
使用Initializable、OwnableUpgradeable、UUPSUpgradeable实现合约的可升级结构，核心合约如OrderBook、Vault都支持UUPS升级方式  

2、批量创建订单：  
传入Order结构体数组，用for循环对每个Order都执行makeOrder函数操作，makeOrder函数返回OrderKey。makeOrder函数操作前判断Order的side，如果是Bid订单，计算该订单需要的ETH（即buyPrice）：单价X数量，将buyPrice一起传入makeOrder函数。如果创建成功，则OrderKey有效不是哨兵值，累计每个Bid订单的buyPrice，否则创建失败，ETH会被退回。  
执行makeOrder函数时：  
先验证订单规则，验证通过则将order进行hash为OrderKey作为订单的唯一标识；然后验证订单数量（List订单限制数量为1，Bid订单数量不能为0），然后再调用Vault合约的depositNFT或depositETH函数将OrderKey和资产（NFT或ETH）存入金库，然后调用Storage合约的addOrder函数将订单存入订单存储，存入订单存储的过程要用到红黑树进行价格档排序（价格优先，时间优先）。如果订单创建失败，则跳过发出跳过该订单事件  
最后如果传入的ETH多于实际所需金额，退回多余部分    

3、批量取消订单（即创建订单的反向操作）  
传入OrderKey数组，用OrderKey获取对应订单，验证订单是否满足取消规则，然后调用Storage合约的removeOrder函数移除订单，再调用Vault合约的withdrawNFT或withdrawETH提取资产（其中Bid订单要按未成交数量计算可提取的ETH），然后调用Storage合约的cancelOrder标记订单已取消，如果取消失败，发出跳过事件  

4、批量编辑订单  
传入旧订单标识oldOrderKey和newOrder，用oldOrderKey获取旧订单，进行编辑限制检查和新订单验证后，调用Storage合约的removeOrder和cancelOrder函数移除订单，

编辑订单实际是先取消旧订单再创建新订单的过程  
编辑限制检查：saleKind、side、maker、nft（collection和tokenId）必须与旧订单一致，只能修改价格price和数量amount，订单不能已完全成交  
新订单验证：新订单的maker必须是调用者，salt不能为0，过期时间必须有效（大于当前时间或为0），新订单不能已被取消或完全成交  
验证完先取消旧订单再创建新订单  
资产处理：对于List订单：直接更新金库中的NFT关联；对于Bid订单：如果新价格更高：需要补足差额ETH，如果新价格更低：金库会退回多余ETH  

5、撮合单个订单  
要验证sellOrder订单和buyOrder是否匹配，只有匹配才能撮合  
支持两种撮合场景：  
*卖家接受出价：sellOrder.maker调用，验证buyOrder，要求Bid订单必须存在于订单存储中，此时成交价格为Bid订单的价格，然后更新成交数量，进行资产转移，从金库提取ETH到OrderBook合约，扣除手续费后转给卖家，NFT则从卖家转移到买家  
*买家接受挂单：buyOrder.maker调用，验证sellOrder，要求List订单必须存在于订单存储中，此时成交价格为List订单的价格，然后更新成交数量，进行资产转移，从金库提取ETH到OrderBook合约，扣除手续费后转给卖家，NFT则从卖家转移到买家。如果买家出价高于成交价，退回多余ETH，如果买家传入的ETH高于实际花费的ETH金额，退回多余ETH  

6、批量撮合订单  
使用delegatecall执行步骤5进行批量撮合，如果撮合失败，记录事件但不回滚，如果撮合成功，判断是否买家发起的撮合，买家发起撮合就累积已花费的ETH，批量撮合完，如果传入的ETH多于实际需要的金额，退回多余的ETH  

7、聚合调用多个操作（单笔交易内串联多个操作）  
支持的操作：makeOrders、cancelOrders、editOrders、matchOrder、matchOrders。要验证调用的函数是否在其中  
由于delegatecall下每个子调用看到的msg.value相同，为避免资金语义歧义，一次聚合调用最多允许1个“可能消耗msg.value”的子调用  
根据revertOnFail参数决定策略：为true时任一失败将整笔回滚；为false时仅记录失败并继续
