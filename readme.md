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

主要需求分列如下:

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

------------------------------------------------------------

## 概要设计

### 展示 address token balance 的实现逻辑

* 通过 [token_data_table](./doc/实体关系表.md#代币信息储存表--tokendatatable) 维护经过删选的链上代币信息列表,
  并只展示该列表范围内的资产信息
* 为了快速获取 资产数据, 优先通过由服务端维护的
  [account_amount_cache_table](./doc/实体关系表.md#用户资产缓存表--accountamountcachetable) 获取 address 资产 amount.
* 为了数据准确性, 允许参数触发实时获取链上 address 对应 amount 数据.
* 通过 join [token_data_table](./doc/实体关系表.md#代币信息储存表--tokendatatable) 获取 address 资产 balance
* 通过 join [token_price_table](./doc/实体关系表.md#代币币价表--tokenpricetable) 获取 address资产 usd balance
* 对所有代币 usd balance 加总展示 address 资产总额度



------------------------------------------------------------

## 参考文档

* [OneKey Final 测试题目](https://chip-asphalt-ca1.notion.site/OneKey-Final-c60da4846a45428e9340749a124e4f11)

## 问题:

- [ ] 主币和代币发送, 接收, 闪兑是否在本系统功能内?
- [ ] 页面中价值后的 +0.90% 含义?
