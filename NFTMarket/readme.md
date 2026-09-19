# 项目整体架构图  


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
支持两种匹配场景：  
*卖家接受出价：sellOrder.maker调用，验证buyOrder，要求Bid订单必须存在于订单存储中，此时成交价格为Bid订单的价格，然后更新成交数量，进行资产转移，从金库提取ETH到OrderBook合约，扣除手续费后转给卖家，NFT则从卖家转移到买家  
*买家接受挂单：buyOrder.maker调用，验证sellOrder，要求List订单必须存在于订单存储中，此时成交价格为List订单的价格，然后更新成交数量，进行资产转移，从金库提取ETH到OrderBook合约，扣除手续费后转给卖家，NFT则从卖家转移到买家。如果买家出价高于成交价，退回多余ETH，如果买家传入的ETH高于实际花费的ETH金额，退回多余ETH  

6、批量撮合订单  


# 后端流程  


# 主要模块职责  
**config模块**  
  
**数据存储**  
  
**路由模块**  
  
**kucoin模块**  
    
**websocket模块**  
  
**定时任务**  
  
**其它模块**  
  

# 从业务角度分析代码实现  
