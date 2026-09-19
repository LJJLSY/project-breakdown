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
对于Bid订单，会累计ETH总金额，将ETH转移到金库，数量不能为0，如果传入的ETH多于实际需要的金额则退回多余ETH，如果ETH不足则回滚交易。  
对于List订单，要先授权NFT给金库合约，创建订单时会将NFT转移到金库，限制数量为1。  
创建订单时要验证订单规则：  
*只有订单创建者可以创建订单（order.maker == msg.sender）
*价格不能为0
*salt不能为0
*过期时间必须大于当前区块时间戳，或为0（永不过期）
*订单不能已被取消或完全成交  

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
