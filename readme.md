# OneKey Software Wallet

## 概念定义

 concept         | definition                                
-----------------|-------------------------------------------
 account_address | 链上账户地址                                    
 token_address   | 链上代币地址                                    
 amount          | 最小单位的资产数量, 如 eth 单位为(wei), ERC1155 单位为(个) 
 decimals        | 以10为底指数精度                                 
 balance         | 金额, 通常为 amount/(10**decimals)             
 usd_price       | 单位金额主代币的美元值                               
 chain           | 链标识, 主网和测试网属于不同的 chain                    
 timestamp       | 以秒为单位的Unix时间戳                             
 block_number    | 区块编号整数                                    
 transaction     | 交易(事务)                                    
 transfer        | 代币转移

------------------------------------------------------------

## 需求分析

### 主要需求分列如下:

* 主要支持 BTC, EVM, SOL, TRON 链
* 主币支持
    * 获取 account_address 当前主币 balance
    * 获取主币当前法币价值
* 同质化代币资产支持
    * 获取 account_address 当前同质化代币列表
    * 获取 account_address 当前同质化代币 balance
    * 获取同质化代币当前法币价值
* 非同质化代币资产支持
    * 获取 address 当前非同质化代币列表
    * 获取 address 当前非同质化代币 balance
    * 获取非同质化代币当前法币价值
* 交易历史支持
    * 获取 address 主币交易历史
    * 获取 address 同质化代币交易历史
    * 获取 address 非同质化代币交易历史

### 未来可能要求支持的需求:

*以下需求在本版本系统设计需求外, 但参考竞品设计后可预计, 很有可能在后续产品中要求支持, 在系统设计中应当给予考虑*

* 产品有同一链支持很多地址的需求可能: 如 BTC 链有 4 种类, 不同格式的地址, 一个 EOA 账户可能控制多个 AA 地址账户等
* 产品有同一页面展示所有链资产总和的需求可能:
* 产品有展示其他法币的需求可能:
* 产品有列表隐藏小额资产的需求可能: 这里的小额指法币价值低于一定数量
* 地址详细信息维护: 如 ENS, 风险地址标识, 通用地址标识(如 UniSwap 池子地址)等.

------------------------------------------------------------

## 概要设计

### 展示 address token balance 实现流程

* 通过 [token_data_table](./doc/实体关系表.md#代币信息储存表--tokendatatable) 维护经过删选的链上代币信息列表,
  并只展示该列表范围内的资产信息
* 为了快速获取 资产数据, 优先通过由服务端维护的
  [account_amount_cache_table](./doc/实体关系表.md#用户资产缓存表--accountamountcachetable) 获取 address 资产 amount.
* 为了数据准确性, 允许参数触发实时获取链上 address 对应 amount 数据.
* 通过 join [token_data_table](./doc/实体关系表.md#代币信息储存表--tokendatatable) 获取 address 资产 balance
* 通过 join [token_price_table](./doc/实体关系表.md#代币币价表--tokenpricetable) 获取 address资产 usd balance
* 对所有代币 usd balance 加总展示 address 资产总额度

### 展示 address transfer 实现流程

* 通过 [token_data_table](./doc/实体关系表.md#代币信息储存表--tokendatatable) 维护经过删选的链上代币信息列表,
  并只展示该列表范围内的资产信息
* 对于大部分已经添加扫快信息的链, 通过 [token_transfer_table](./doc/实体关系表.md#资产转移表--tokentransfertable) 获取
address 历史交易数据, (注意可选择 0 额转账过滤)
* 对于没有扫快信息的链, 允许支持通过第三方数据源获取历史交易数据.
* 转账历史展示中, 由于可能动态过滤某些项目(如 0 金额), 因此使得计算总数和随机翻页比较困难,
  不要设计转账总数和页码翻页, 要支设计支持通过游标翻页, 可能要支持按时间筛选范围.
* 因为同一 transaction 可能产生多笔 transfer, 在展示历史条目时, 需要对 transfer 进行聚合.
  几个聚合方案:
  * 找出同一 transaction 下的所有与 address 相关的 transfer.
  * 消除 0 额代币转移, 如同一地址下两笔 1ETH -> 1WETH, 1WETH -> 2000USDT, 应当 WETH 转移金额为 0
  * 扣除回款代币转移, 如BTC下 output:[3,2], input[2], 则实际代币花费为 (3+2)-2 = 3.
  * 对于多输入单多输出的交易, 可以简单聚合会地址 address 在这笔交易中共计花出或收入数量, 而不计对手地址.


### 维护 token_data_table 实现流程

* [token_data_table](./doc/实体关系表.md#代币信息储存表--tokendatatable) 用于管理经过钱包验证的代币信息
* 代币信息可以设计管理员接口进行维护
* 代币信息的 name, symbol, decimals 等项, 可以支持智能查询自动填充, 帮助管理员管理表项

### 维护 token_price_table 实现流程

* [token_price_table](./doc/实体关系表.md#代币币价表--tokenpricetable) 用于存储一段时期内代币的法币价格信息
* 币价为最终值, 其数据源可以来源于交易所或预言机等聚合
* 对于数据源单一, 比较波动大的代币, 可以采用平滑算法减小波动
* 一段时间内币价为空时, 可以采用最后更新币价更新到最新币价, 并人工积极寻找可用的数据源
* 对于同一代币的不同表现, 可以采用同一数据源, 但需要记录在不同表: 如 ETH 和 WETH 需要分别生成两条记录, 
因为对于同一代币的不同表现, 可能会有不同的数据, 如某链上 xBTC 产品价格可能和 BTC 价格脱锚.

### 维护 token_transfer_table 实现流程

* [token_transfer_table](./doc/实体关系表.md#资产转移表--tokentransfertable) 用于储存全量账户转移记录
* 对于以太坊及大部分 EVM 链: 
  * 通过搜集 transaction, trace 信息可以获取主币转移信息
  * 通过收集 Transfer 事件信息, 可以获取代币转移信息
  * 对于特殊的代币合约(如加密猫), 需要监听相应的 Event 事件
* 对于 BTC 链:
  * 解析交易中 input, output 信息可以获取主币转移信息
  * BTC 代币主要为铭文系列 (TODO)
* 对于 SOL 链: **有待调研**
* 对于 TRON 链: **有待调研**
* 交易记录的获取需要逐块获取, 初始化时, 可以从中间任一区块(安全)高度启动, 向两端同步区块数据,
使得新数据能尽快展示, 并且能逐步恢复旧数据
* 向新数据端同步时, 注意要处理区块校验问题, 对于区块 B(n), 要检查其 parent_hash 是否与库中区块 B(n-1) 的 hash 相同, 
否则库中区块 B(n-1) 数据为失效数据, 应当移除其相关数据.
* 向旧数据端同步时, 不需处理区块校验问题, 单有可能因为数据容量问题, 需要具备移除某个时间段之前数据的能力.

### 与区块链交互 rpc 提升稳定性方案

* 对于重点和高频访问区块链, 可以采用自建节点的方案, 如自建多节点, 可以利用负载均衡方式将其并联起来.
如有必要, 可以采购商用节点, 处于成本考虑, 可以将商用节点作为自建节点的备份.
* 对于次要链或重要但自建成本较大的链(如BTC), 可以采购商用节点和数据源提供商, 如有必要可采购多家服务, 并将其进行串或并连.
* 节点的管理可依赖区块高度进行筛选, 以官方公开节点或大多数节点提供的区块高度为准, 将区块高度相差过大的节点剔除在服务外并报告异常情况

------------------------------------------------------------

## 参考文档

* [实体关系表](./doc/实体关系表.md)
* [OneKey Final 测试题目](https://chip-asphalt-ca1.notion.site/OneKey-Final-c60da4846a45428e9340749a124e4f11)
