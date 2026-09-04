# 项目整体架构图  
<img width="1241" height="579" alt="image" src="https://github.com/user-attachments/assets/e25f237d-75ef-48dd-ac9c-62c17008017e" />

# 核心业务流程  
<img width="1275" height="831" alt="image" src="https://github.com/user-attachments/assets/f49e1290-ef19-466d-b895-dbd53936d506" />  

1、创建质押池，设置质押池基本信息和统计信息参数，包括结算时间settleTime和计息结束时间endTime，池子默认状态是MATCH（池子一共5个管理状态：MATCH、EXCUTION、FINISH、LIQUIDATION、UNDONE）  
2、如果还没到结算时间，借贷双方可以往池子里面存入资产，直到结算时间为止不能再存入资产，此时池子进行结算  
3、结算过程判断双方是否都存入资产，如果有任一方没存入，则池子状态变成UNDONE，存入的一方要紧急取回资产；如果双方都有存入，则调用预言机获取抵押物价格，计算贷款人的实际借出数量和借款人的实际抵押数量，然后池子状态变成EXCUTION  
4、执行期间如果从预言机获取的抵押物价格跌到低于清算阈值，则触发清算，清算过程会按照endTime-settleTime整个期限计算利息，再加上本金（实际借出数量）以及平台手续费。再通过UniswapV2卖出一定数量的抵押物，换成贷款人需要的代币，然后计算清算后贷款人可取的本息数量和借款人可取的剩余抵押物数量。池子状态变成LIQUIDATION  
5、执行期间如果抵押物价格没有低于阈值，等到结束时间后，按照endTime-settleTime整个期限计算利息，再加上本金（实际借出数量）以及平台手续费，通过UniswapV2卖出一定数量的抵押物，换成贷款人需要的代币，然后计算清算后贷款人可取的本息数量和借款人可取的剩余抵押物数量，池子状态变成FINISH。和清算不同的是，FINISH过程在用UniswapV2卖出抵押物之后，会先检查换来的代币数量能否覆盖贷款人的本息，如果不能覆盖则直接报错提示滑点过高  
6、池子状态在FINISH或者LIQUIDATION时，借贷双方可以按份额提取资产，贷款人可以按份额提取本息，借款人可以按份额提取剩余抵押物  
7、在结算之后，只要池子状态不是MATCH和UNDONE，双方可以退回没有被结算的资产，贷款人退回没有被借走的资产，借款人退回没有用到的抵押物数量。贷款人可以申领本息，系统会根据份额给贷款人铸造相应数量的凭证代币，以此等到FINISH或LIQUIDATION状态时按照凭证代币数量提取本息。借款人可以提取借走的资产，并申领剩余抵押物，系统也会按照份额铸造凭证代币，以此等到FINISH或LIQUIDATION状态时按照凭证代币数量计算可取抵押物份额，提取剩余抵押物

# 后端流程  
<img width="1326" height="810" alt="image" src="https://github.com/user-attachments/assets/9abbb730-e6da-4201-99f7-6a8dad4c1cb0" />

# 主要模块职责  
**config**  
存放配置文件和配置管理代码  
**数据存储**  
mysql提供链上信息的数据存储，近期或历史数据管理和查看  
redis提供数据缓存和数据变更校验  
gorm框架提供mysql、redis等db连接  
**路由模块**  
用gin框架提供api接口与前端交互，通过数据库查看链上信息
