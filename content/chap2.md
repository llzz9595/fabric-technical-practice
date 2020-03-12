
为了加深对2.0的认识，从first-network的部署配置开始进行学习。
上篇有提到在运行Fabric网络前，先执行了`./byfn.sh generate` 实现创始区块、通道以及证书文件的生成，接下来让我们简单的剖析一下 `./byfn.sh generate` 的实现。


## 1.byfn.sh generate
首先从byfn.sh的脚本可以观察到，脚本的第一个参数为执行模式，其中模式包含up、down、generate、restart以及upgrade，代表启动、清除、生成以及升级网络。这里我们关注的是generate,在generate主要执行两个function，分别是  
generateCerts :生成证书、  
generateChannelArtifacts ：生成创始区块与通道文件。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200212180658344.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
## 2. generateCerts
generateCerts ：生成证书
详细来看脚本

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200212181305122.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
主要执行了两个核心脚本分别是：
1.`cryptogen generate --config=./crypto-config.yaml`  

    实现根据crypto-config.yaml 生成证书文件。
    
2.`./ccp-generate.sh`

    生成调用nodejs SDK的相关区块链配置文件。
   
### 2.1 生成证书文件
打开first-network目录下面的crypto-config.yaml文件。
crypto-config.yaml主要包含的fabric排序节点证书配置以及fabric组织证书配置。

 - **排序节点证书配置**
 配置了5个排序节点，域名为：Hostname + Domain eg:orderer.example.com

```powershellOrdererOrgs:
  - Name: Orderer
    Domain: example.com
    Specs:
      - Hostname: orderer
      - Hostname: orderer2
      - Hostname: orderer3
      - Hostname: orderer4
      - Hostname: orderer5

```
  **支持其他配置模式如下：**

（1）重写全限定域名

 CommonName

     默认值为Hostname.Domain

```
OrdererOrgs:
  - Name: Orderer
    Domain: example.com
    Specs:
      - Hostname: orderer
        CommonName: myorderer.example.com
```

（2）替换Specs为Template配置式

#Template 使用模板定义节点

#Count 节点总数

#Start 节点下标起始值

#Hostname 全限定域名 命名格式 

#Prefix 默认 peer

#Index 取Start值 无配置从1开始自增

```
OrdererOrgs:
  - Name: Orderer
    Domain: example.com
    Template:
      Count: 5
      Start: 1
      Prefix：order    
 
```

 - **组织证书配置**
配置了两个组织，每个组织2套公私钥和证书，包含普通User（Admin不包含在计数中）数量为1

```
PeerOrgs:
  - Name: Org1
    Domain: org1.example.com
    EnableNodeOUs: true
    Template:
      Count: 2
     Users:
      Count: 1

  - Name: Org2
    Domain: org2.example.com
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
```

#Domain 域名

#Template 参考OrdererOrgs 可替换为Specs配置式

#Users -> Count 添加到管理员的用户帐户数

#EnableNodeOUs 允许节点 OUS -> out of service，用于区分clients, admins, peers。假如true会在msp目录生成config.yaml如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200212183512768.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

此处证书结构不做详细说明，详情查看[Fabric2.0官方文档](https://hyperledger-fabric.readthedocs.io/en/release-2.0/msp.html#)

### 2.2 ./ccp-generate.sh

  生成调用nodejs SDK的相关区块链配置文件。此处不作详细配置说明。
  
## 3. generateChannelArtifacts
    generateChannelArtifacts ：生成创始区块与通道文件
    
### 3.1 生成创始区块
脚本：

```
 configtxgen -profile SampleMultiNodeEtcdRaft -channelID byfn-sys-channel -outputBlock ./channel-artifacts/genesis.block
```

由于我接触Fabric是从2018年开始就是1.x时代，2.x的话完全是没有接触，看到上面的shell，明显就有了点2.0的味道，按照常规套路先看2.0官方文档解释：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200212184447673.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
configtxgen 命令参数的使用在2.0发生了变化，为了方便比较我放一下1.4官方文档的：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200212184522666.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
差别在于 创世区块的网络共识模式定义变了，**删除了TwoOrgsOrdererGenesis(原本的排序节点创始区块配置)、SampleDevModeKafka (kafka共识配置)**。

剩余的还是解释一下：

**SampleMultiNodeEtcdRaft：用于生成创始区块的，配合-o一起用，支持etcdraft模式的共识。**

**TwoOrgsChannel：用于生成通道的。**

剩下生成通道的脚本与原先的没有太大差距此处不作详细说明，详情请看[官方文档说明](https://hyperledger-fabric.readthedocs.io/en/release-2.0/msp.html#)，

### 3.2 通道文件生成

**通道配置：**
  

```
  TwoOrgsChannel:
  # 通道联盟名称，现在默认是SampleConsortium
        Consortium: SampleConsortium  
     # 引用ChannelDefaults通道详细配置
        <<: *ChannelDefaults   
        Application:
        # // 引用ApplicationDefaults
            <<: *ApplicationDefaults 
            #  //通道组织定义            Organizations:
                - *Org1
                - *Org2
            Capabilities:
                <<: *ApplicationCapabilities
```
**通道详细配置：**

```
Channel: &ChannelDefaults
    #   通道权限策略  <ALL|ANY|MAJORITY> <sub_policy>
    Policies:
        # Who may invoke the 'Deliver' API
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        # Who may invoke the 'Broadcast' API
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        # By default, who may modify elements at this config level
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
    # Capabilities describes the channel level capabilities, see the
    # dedicated Capabilities section elsewhere in this file for a full
    # description
    Capabilities:
        <<: *ChannelCapabilities
```
上面主要是对通道权限策略配置，其中策略这块主要有两种类型：

1.Signature策略

 SIGNATURE策略指定通过签名来对数据进行认证，例如数据必须满足一定的签名身份组合 这种策略比较灵活，主要定义MSP主体组合规范。在验证签名策略的基础上，支持AND、OR、NOutOf，可以构建如：‘An admin of org A and 2 other admins, or 11 of 20 org admins’等规范。
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200212185039505.png)
 2.ImplicitMeta策略 
 
这种策略不如SignaturePolicy灵活，并且只在配置上下文中有效。它不直接进行签名检查，而是通过引用其子元素的策略（最终还是通过Signature策略）来进行检查 检查结果又Rule限制，它支持默认规则，如：‘A majority of the organization admin policies’。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200212185116229.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

**共识配置**

```

  SampleMultiNodeEtcdRaft:
        <<: *ChannelDefaults
        Capabilities:
	# 通道Capabilities 指定2.0
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
		  # 排序节点类型etcdraft
            OrdererType: etcdraft 
                  # 系统通道中raft节点配置 raft服务是在order节点内
            EtcdRaft:
                Consenters:
                - Host: orderer.example.com
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
              
            Addresses:
                - orderer.example.com:7050
               
            Organizations:
            - *OrdererOrg


            Capabilities:
		# 排序节点Capabilities 指定2.0

                <<: *OrdererCapabilities
        Application:
            <<: *ApplicationDefaults
            Organizations:
            - <<: *OrdererOrg
        Consortiums:
            SampleConsortium:
                Organizations:
                - *Org1
                - *Org2
```

## 4.总结
综合来看，比起1.x版本的Fabric,在配置最大的变化是删除了solo以及kafka模式，仅支持etcdraft共识模式其余并没有太大变化，因此对于有1.x部署经验并不会造成太大的困难,初学者可以慢慢根据教程入门。
