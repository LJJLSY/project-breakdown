# 项目整体架构图  



# 核心业务流程  
  

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
