<h1 align="center">去中心化数据交换应用框架</h1>
<p align="center" class="version">Version 0.8.0 </p>

## 概述

DDXF基于Ontology BlockChain和ONT DATA协议，通过一致性账本、智能合约、密码学技术完美实现数字资产去中心化交易。DDXF提供一系列智能合约模板、交易组件和密码学组件，上层应用可以非常方便地实现版权控制、契约式数据分享等场景需求。

DDXF提供的主要功能包括：

* 数据资产化DataToken
* 数据交易智能合约eXchange Smart Contract
* 数据交易客户端DataRot
* 一系列密码学组件

![](http://on-img.com/chart_image/5b9b529de4b0fe81b63605f9.png)


## 系统架构

![](http://on-img.com/chart_image/5b9fd665e4b0534c9bdfc0bb.png)

## 数据Token



## 数据交易组件DataRot

在链下数据交割过程中，为了保证数据的去中心化可信存储，原则上由数据提供方自行保存自己的数据，在进行链上数据DToken化时绑定数据URL。数据交易参与方可集成DataRot组件，由组件为提供方完成数据DToken化、DToken动作监听、权限管理及数据安全传输。


去中心化数据交易标准组件DataRot，可由数据提供方主动部署。ExDataClient作为数据提供方、数据需求方和区块链三者的交互桥梁，既要与链上智能合约交互又要与链下真实数据交互。需提供的标准功能包括：

- 数据链上DToken化
- DToken链上动作监听
- 请求方权限验证
- 数据加密安全传输
- 数据本地存储

由于组件需要调链上智能合约发交易，数据提供方需要把自己的OntId和account.json账户文件配置在组件配置文件和目录中。

### 数据链上DToken化
DataRot组件提供标准Restful API替数据提供方完成本地数据在链上智能合约中的DToken化。数据提供方需提供dataName，dataPath，auditorOntId等信息。组件会生成唯一数据URL，获取配置的数据提供方OntId绑定到DToken中。
```
{
	"auditorOntId":"did:ont:T2nGst12KJ769n0N1GC7901n2",
	"dataName":"did_contract.abi",
	"dataPath":"/var/mydata/contract/"
}
```


### DToken链上消息监听
当实例化好DToken后，组件通过websocket订阅机制，监听该DToken的链上动作。
- 买单动作
当有数据需求方进行买单完成资金锁仓时，ExDataClient组件需监听该买单动作，更新DToken状态为"已下单"并获取需求方OntId，用于后续权限验证。
- 审核动作
当审核方做完数据审核，完成链上DToken打分时，ExDataClient组件需监听该打分动作，更新DToken状态为"已审核"并获取分数。
- 资金交割
当数据需求方进行订单确认，完成链上资金交割时。ExDataClient组件需监听该资金交割动作，更新DToken状态为"已交割"。


### 请求方权限验证
当进行链下数据请求时，请求方必须带上OntId，数据提供方的DToken标识以及自己的签名
```
{
	"ontId":"did:ont:TA4zNs7vk6MZwSY1FXHCZi5F9jnsgZ8fRq",
	"dToken":"0x8018jgf7645a84298a1a52aa3745f84dba6615cf",
	"siganture":"",
}
```

DataRot组件首先需检查该DToken的状态是否是"已审核"，然后再对请求签名进行验签，确认数据请求附带的OntId属于请求方。最后检查该OntId是否已经是DToken的属主。完成以上步骤才算权限验证通过。



## 工作模式


![](http://on-img.com/chart_image/5a92878de4b059c41ac98e46.png)









