---
layout:     post
title:      XuperChain学习之超级链网络搭建
subtitle:   百度开源的区块链XuperChain学习记录
date:       2020-05-09
author:     BaiYe
header-img: img/post-bg-01.png
catalog: true
tags:
    - 区块链
    - XuperChain
    - 学习笔记
---

# XuperChain学习（二）：超级链网络搭建

## 简介

在Linux下基于XuperChain搭建多机器多节点区块链网络，用于验证&学习XuperChain。主要是搭建测试验证环境，非高可用专业环境。

系统版本：Linux CentOS release 6.8
实例配置：1核/2GB/40GB

## 环境准备

### 基础环境准备

参考文档：https://chunhui01.github.io/2020/04/16/Linux%E4%B8%AA%E4%BA%BA%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/

### XuperChain下载编译

```
mkdir -p /home/rd/develop/go-path/src/github.com/xuperchain

cd /home/rd/develop/go-path/src/github.com/xuperchain

git clone git@github.com:xuperchain/xuperchain.git

# 编译
cd /home/rd/develop/go-path/src/github.com/xuperchain/xuperchain
make all

编译成功后产出在 output 目录下。

```

## 多节点部署
注：由于节点很少，就手动操作了，节点多可以编写脚本发布。

### 分发部署
```
cd /home/rd/develop/go-path/src/github.com/xuperchain/xuperchain/
mkdir -p /home/rd/apps/
cp -r ./output /home/rd/apps/xuper_node01
cp -r ./output /home/rd/apps/xuper_node02
cp -r ./output /home/rd/apps/xuper_node03

其他机器scp过去到同样的目录即可。
```

### 创建账户、网络key、背书人账户
cd进节点目录，每个节点依次执行（多节点写启动脚本解决）：
```
# 删除默认值
rm -rf ./data/endorser ./data/keys ./data/netkeys
# 创建普通账户
./xchain-cli account newkeys
# 创建背书人账户
./xchain-cli account newkeys --output ./data/endorser/
# 生成节点p2p网络密钥
./xchain-cli netURL gen
```

### 更新配置

#### 1.先更新创世块配置（./data/config/xuper.json)
```
cd /home/rd/apps/xuper_node01
vim ./data/config/xuper.json
```

```
	# 为指定账户预分配金额（address通过各节点 cat ./data/keys/address 获取）
    "predistribution": [
        {
            "address": "mTqoSrhxxxxx...xxxxxxx7L5fnS4t",
            "quota": "50000000000000000000"
        },
        {
            "address": "f2iKfxhxxxxx...xxxx4xStaNJbQFH",
            "quota": "50000000000000000000"
        }
    ],

	# 搭建tdpos共识网络
	"genesis_consensus": {
        "name": "tdpos",
        "config": {
	        # tdpos共识初始时间，声明tdpos共识的起始时间戳，建议设置为一个刚过去不旧的时间戳
            "timestamp": "1588998296000000000",
            # 每一轮选举出的矿工数，如果某一轮的投票不足以选出足够的矿工数则默认复用前一轮的矿工
            "proposer_num": "4",
            # 每个矿工连续出块的出块间隔
            "period": "3000",
            # 每一轮内切换矿工时的时间间隔，需要为period的整数倍
            "alternate_interval": "6000",
            # 切换轮时的出块间隔，即下一轮第一个矿工出第一个块距离上一轮矿工出最后一个块的时间间隔，需要为period的整数配
            "term_interval": "9000",
            # 每一轮内每个矿工轮值任期内连续出块的个数
            "block_num": "200",
            # 为被提名的候选人投票时，每一票单价，即一票等于多少Xuper
            "vote_unit_price": "1",
            # 指定第一轮初始矿工，矿工个数需要符合proposer_num指定的个数，所指定的初始矿工需要在网络中存在，不然系统轮到该节点出块时会没有节点出块
            "init_proposer": {
                "1": [
                    "mTqoSrhqbxxxx......xxxV7L5fnS4t",
                    "WQPYSYbbBxxxxx.....xxxdqumQUsB3",
                    "mRqQhybzxxxx......xxxxrcPKTqwaE",
                    "f2iKfxhrxxx....xxxHK4xStaNJbQFH"
                ]
            },
            "init_proposer_neturl": {
                "1": [
                    "/ip4/xxx.xxx.xxx.xxx/tcp/8300/p2p/QmPweQBxxxYY5Exxxx9ceJyW4x",
                    "/ip4/xxx.xxx.xxx.xxx/tcp/8310/p2p/QmbLUuKexxxCixxzJxxxAF46u5",
                    "/ip4/xxx.xxx.xxx.xxx/tcp/8320/p2p/QmbVZyexxo4pFv1xxxxxa7DQGy",
                    "/ip4/xxx.xxx.xxx.xxx/tcp/8300/p2p/QmVq18nkGxxMKRKkKxxxxxwfpy"
                ]
            }
        }
    }

```

#### 2.更新xchain启动核心配置（./conf/xchain.yaml）
需要更新端口、账户地址、和p2p bootNodes。

分别是：
```
# RPC 服务暴露的端口
tcpServer:
  port: :8200
  # prometheus监控指标端口, 为空的话就不启动
  metricPort: :8201

# 区块链节点配置
p2p:
  # module is the name of p2p module plugin, value is [p2pv2/p2pv1], default is p2pv2
  port: 8300

  bootNodes:
    - "/ip4/xx.xx.xx.xx/tcp/8300/p2p/QmPweQxxxxxxxxMk9ceJyW4x"
    - "/ip4/xx.xx.xx.xx/tcp/8300/p2p/QmVq1xxxxxxxxxBNDRGGwfpy"

# 管理native合约的配置
native:
  # 与部署相关的配置
  deploy:
    # 部署白名单，列表为钱包地址
    whiteList:
      addresses:
        - mTqoSrhqbxxxxxxxxx7L5fnS4t
        - f2iKfxhrhxxxxxx4xStaNJbQFH
```

bootNodes网络地址通过在相应节点目录下执行：./xchain-cli netURL preview 获取，主要爱修改host和端口。
账户地址通过在相应节点目录下执行：cat ./data/keys/address 获取。

#### 分发配置

1../data/config/xuper.json ，直接scp到所有节点上即可，所有节点保存一致。
2../conf/xchain.yaml ，scp到所有节点上后，如果是同机多节点需要修改那三个端口，不要重复。

### 启动

#### 先启动bootNodes
1.创建链（创世区块）： ./xchain-cli createChain
2.启动xchain服务节点： nohup ./xchain >/dev/null 2>&1&

#### 启动其他nodes
1.创建链（创世区块）： ./xchain-cli createChain
2.启动xchain服务节点： nohup ./xchain >/dev/null 2>&1&

注：第一个bootNodes启动很快就会因为p2p链接失败而panic退出，有几秒的等待时间，需要快速启动其他节点。

## 验证

1.链接任意节点查看xuper链状态：
./xchain-cli status -H xx.xx.xx.xx:8210
各个节点高度一致增加，就ok。

2.转账：
./xchain-cli transfer --to czojZcZ6cxxxxxxxxxMB1PjKnfUiuFQ --amount 10000 --keys data/keys/ -H xxx.xxx.xxx.xxx.:8300

3.查询余额：
./xchain-cli account balance --keys data/keys -H xxx.xxx.xxx.xxx.:8300

## 参考资料
https://xuperchain.readthedocs.io/zh/latest/quickstart.html
https://xuperchain.readthedocs.io/zh/latest/advanced_usage/multi-nodes.html

