## 1. fabric2.0合约新特性

从Fabric 2.0开始，对链码的管理是完全分散的：多个组织可以使用Fabric链码生命周期来就链码的参数（例如链码认可策略）达成协议，然后再使用链码与分类帐进行交互。

新的智能合约在生命周期中提供了一些改进：

 - 多个组织必须同意链码的参数：

在Fabric的1.x版本中，一个组织可以为所有其他渠道成员设置链码的参数（例如，认可策略）。新的Fabric链码生命周期更加灵活，因为它既支持集中式信任模型（例如先前生命周期模型的模型），也支持分散模型，这些模型需要足够数量的组织才能在背书策略上生效。

 - 更安全的链码升级过程：

在先前的链码生命周期中，升级事务可能由单个组织发出，这给尚未安装新链码的渠道成员带来了风险。新模型仅在足够数量的组织批准升级后才允许升级链码。

 - 更容易的认可策略更新：

Fabric生命周期允许您更改认可策略，而无需重新打包或重新安装链码。用户还可以利用新的默认策略，该策略需要频道中大多数成员的认可。在从渠道中添加或删除组织时，该策略会自动更新。

 - 可检查的链代码包：

 
Fabric生命周期将链码打包在易于阅读的tar文件中。这使得检查链码包和协调跨多个组织的安装变得更加容易。

 - 使用一个程序包在通道上启动多个链码：

上一个生命周期使用安装链码包时指定的名称和版本定义了通道上的每个链码。现在，您可以使用单个chaincode程序包，并在相同或不同的通道上以不同的名称多次部署它。

## 2 智能合约实践
智能合约的lifecycle实践主要分成以下4个部分

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214195216637.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

### 2.1 安装以及定义智能合约

#### 2.1.1  打包合约
2.0的智能合约需要打包成一个.tar.gz压缩包。
打开控制台，执行 `docker exec -it cli` 进入cli容器
由于没接触过新的智能合约命令，我们先help下
控制台输入

```bash
peer lifecycle chaincode --help
```

	
 peer lifecycle chaincode命令定义说明：
	
	#智能合约操作合集
	Perform chaincode operations: package|install|queryinstalled|getinstalledpackage|approveformyorg|checkcommitreadiness|commit|querycommitted
	
	Usage:
	  peer lifecycle chaincode [command]
	
	Available Commands:
	  approveformyorg      同意智能合约定义
	  checkcommitreadiness  检查合约是否在通道上已经定义
	  commit               提交合约定义到指定通道
	  getinstalledpackage  从节点中获取已经安装的链码包
	  install              部署链码
	  package           打包链码
	  querycommitted       查询节点上已提交的链码定义
	  queryinstalled       查询节点已经安装的链码
	
	
由于要打包合约此处我们使用  peer lifecycle chaincode package
	在容器内部执行命令
	
```bash
 peer lifecycle chaincode package mycc2.tar.gz --path github.com/hyperledger/fabric-samples/chaincode/abstore/go/ --lang golang --label mycc_2
```
	
	mycc2.tar.gz ：打包合约包文件名 
	--path 智能合约路径
	--lang 智能合约语言 支持golang、node、java
	--label  智能合约标签，描述作用
	
执行完成后，查看当前目录，出现mycc2.tar.gz
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214195430558.png)
	
查看打包后的文件内容，主要包含：
	
	（1）.chaincode-Package-Metadata.json文件，主要是定义合约的代码路径、合约语言以及合约标签
	
	{"path":"github.com/hyperledger/fabric-samples/chaincode/abstore/go/","type":"golang","label":"mycc_2"}
	
	（2）压缩的合约代码文件code.tar.gz,包含合约代码以及依赖包

	
#### 2.1.2 部署合约到节点
	
继续在cli执行以下命令
	
```bash
peer lifecycle chaincode install mycc2.tar.gz
```
	mycc2.tar.gz： 智能合约压缩包路径
	
控制台输出：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214195532461.png)
		
验证合约安装是否安装到节点
	
```bash
peer lifecycle chaincode queryinstalled
```

	
控制台输出：
	
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214195813802.png)
	
	
在宿主机执行 docker ps 发现目前合约容器还未启动。
	
#### 2.1.3 当前组织同意合约定义

2.0智能合约由合约定义控制。当通道成员批准合约定义时，该批准作为组织对其接受的合约参数的投票。这些经过批准的组织定义允许通道成员在链码在通道上使用之前对链码达成一致。链码定义包括以下参数，这些参数需要在组织之间保持一致:

**名称:** 智能合约名称

**版本:** 与给定的合约包相关联的版本号或值。如果您升级了合约二进制文件，那么您也需要更改合约版本。

**序列:** 定义链码的次数。此值为整数，用于跟踪合约升级。例如，当您第一次安装并批准一个chaincode定义时，序列号将是1。下次升级合约时，序列号将增加到2。

**背书策略:** 哪些组织需要执行和验证事务输出。背书策略可以表示为传递给CLI或SDK的字符串，也可以在通道配置中引用策略。默认情况下，背书策略被设置为通道/应用程序/背书，这默认要求通道中的大多数组织对交易进行背书。

**集合配置:** 指向与您的chaincode关联的私有数据集合定义文件的路径。有关私有数据收集的更多信息，请参见私有数据体系结构参考。

**初始化:** 所有的链码需要包含一个初始化函数来初始化链码。默认情况下，这个函数永远不会执行。但是，您可以使用chaincode定义来请求Init函数是可调用的。如果请求执行Init，则fabric将确保在调用任何其他函数之前调用Init，并且只调用一次。

**ESCC/VSCC插件**: 自定义认证或验证插件的名称。

cli容器输入以下命令:

```bash
peer lifecycle chaincode approveformyorg --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID channel2 --name mycc2 --version 1 --init-required --package-id mycc_2:33c137ade1dcc3dd383174a451549a1eda509336cf957dfb472854686d565b9e --sequence 1 --waitForEvent
```

    --tls 是否启动tls
    --ca ca证书路径
    --channelID 智能合约安装通道
    --name 合约名
    --version 合约版本
    --package-id queryinstalled查询的合约ID
    --sequence 序列号
    --waitForEvent 等待peer提交交易返回
    --init-required 合约是否必须执行init

控制台日志：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214200118941.png)
能看到通过orderer.example.com产生了一个交易

#### 2.1.4 检查合约定义是否满足策略
2.0的智能合约的部署必须满足一定合约定义策略，策略的定义在通道配置中具体定义
控制台输入以下命令：

 peer lifecycle chaincode checkcommitreadiness --channelID channel2 --name mycc2--version 1 --sequence 1 --output json --init-required

输出以下结果，可以得知当前合约获取了Org1MSP的同意，参考2.1.3，而还没得到Org2MSP的定义
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214200136817.png)
	
查看原本的lifecycle策略定义是满足过半数即可
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214200148442.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)


现在明显不满足，我们还是执行commit合约看一下会怎么样
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214200201434.png)
Error: transaction invalidated with status (ENDORSEMENT_POLICY_FAILURE) ：不满足背书策略

这时候我们需要把Org2MSP也同意这个合约定义，切换环境变量为peer0.org2.example.com的配置

``重复执行2.1.2 跟2.1.3操作后，再执行策略查询，可以看到两个都true了，满足策略。``
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214200218368.png)


#### 2.1.5 提交合约
在满足合约定义的策略后，可以提交合约
控制台执行以下命令提交合约

```bash
 peer lifecycle chaincode commit -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --version 1 --sequence 1 --init-required
```
    --tls 是否启动tls
    --ca ca证书路径
    --channelID 智能合约安装通道
    --name 合约名
    --version 合约版本
    --package-id queryinstalled查询的合约ID
    --sequence 序列号
    --waitForEvent 等待peer提交交易返回
    --init-required 合约是否必须执行init
    --peerAddresses 节点路径 
    --tlsRootCertFiles 节点ca根证书路径(--peerAddresses --tlsRootCertFiles  连用，可多个节点，多个节点即将合约部署到对应节点集合上)

控制台输出日志：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214200343570.png)
查看智能合约容器，此时合约容器已经启动
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214200349485.png)
####  2.1.6 查看节点已提交合约

cli容器内输入以下命令

```bash
 peer lifecycle chaincode querycommitted --channelID mychannel --name mycc
```

控制台输出合约具体信息如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214200442834.png)


2.1.7 操作合约
fabric智能合约操作主要有invoke跟query,此处与1.x完全一致

初始化合约，执行init方法，设置a:100 b:100

```bash
 peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --isInit -c '{"Args":["Init","a","100","b","100"]}'
```

控制台输出：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214200455249.png)

查询a的余额

```bash
 peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```

输出值为100

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214200512819.png)


## 3.由于篇幅过长，其余下一篇补充。先小小总结
2.0的智能合约合约安装部署过程做了很多的修改，首先是合约打包，合约打包这个就是方便了，打包一次，部署多次，从安全性上，合约定义的同意策略控制极大提高了合约的合规性。虽然步骤麻烦，但是总体来说2.0智能合约的改动还是比较认可。
