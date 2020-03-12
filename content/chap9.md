

## 1.背书策略定义
智能合约背书策略用来定义交易是否合法的判断条件，策略以主体的形式表示。主体格式为'MSP.ROLE'， MSP代表所要求的MSPID， ROLE表示角色，一共有四种合法角色：member, admin, client, peer。

## 2. 背书策略语法
背书策略的语法如下：
EXPR(E[, E...])
EXPR可以是AND、OR、OutOf，E可以是一个上面示例的主体或者是另一个嵌套的EXPR策略。示例如下：
AND('Org1.member', 'Org2.member', 'Org3.member') ：要求三个主体中每一个主体都要签名。
OR('Org1.member', 'Org2.member') ：要求三个主体中至少有一个主体签名。
OR('Org1.member', AND('Org2.member', 'Org3.member'))：要求同时有主体Org1.member的签名，以及主体Org2.member与Org3.member中至少一个主体的签名。
OutOf(2, 'Org1.member', 'Org2.member', 'Org3.member') ：要求三个主体中，至少有两个主体签名。

## 3. 设置背书策略
fabric2.0智能合约设置背书策略得方式主要有两种，一种是通过提交合约的时候设置，我们称之为合约级别的背书策略，一种是直接通过合约动态设置，我们称之为键级别背书策略。

### 3.1 合约级别背书策略设置

所谓合约级别背书策略，就是在这个合约的交易都必须遵循这个策略，在默认情况下，即不设置背书策略，合约的背书策略为majority of channel members,过半数通道成员。

输入以下命令设置合约级别背书策略
背书策略"OR('Org1.member', 'Org2.member')" 组织1成员或者组织2成员背书即满足


```bash
 peer lifecycle chaincode approveformyorg  --signature-policy "OR('Org1MSP.member','Org2MSP.member')" --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc --version 1 --init-required --package-id mycc_1:4ad799ccef18d596f8c175fe1849cadc63f92a5efb1e7332712fbb2827a2ec6f --sequence 2 --waitForEvent
```

 --signature-policy：设置背书策略

出现以下错误：
``Error: proposal failed with status: 500 - failed to invoke backing implementation of 'ApproveChaincodeDefinitionForMyOrg': currently defined sequence 3 is larger than requested sequence 2``

序列号不是当前合约序列号，只需要改成最新的序号即可，假如是这里，sequence值改成3即可

``Error: proposal failed with status: 500 - failed to invoke backing implementation of 'ApproveChaincodeDefinitionForMyOrg': attempted to redefine uncommitted sequence (4) for namespace mycc with unchanged content ``

存在当前最新的合约没有commit，无法进行新的approve，只需要将最新的commit后再进行这次新的approve即可。


切换节点重复命令，知道满足lifecycle策略
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218173818197.png)

假如在操作上都approve成功了，还是出现以下情况：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218173828886.png)
``我的实践是直接先commit，commit成功就可以继续走``

#### 3.1.2 提交合约
每次调用完approve之后，必须commit才能起效。

控制台输入以下命令

```bash
 peer lifecycle chaincode commit -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --version 1 --sequence 2 --init-required    --signature-policy "OR('Org1MSP.member','Org2MSP.member')"
```
控制台输出以下结果表示成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218173853438.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)





此时查询a的值为90

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218173920415.png)

#### 3.1.3 验证背书策略
3.2设置的背书策略为组织1成员或者组织2成员背书即满足，
此时指定背书节点为peer0.org1.example.com

```bash
 peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt  -c '{"Args":["invoke","a","b","10"]}'
```
控制台输出如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218173938790.png)
重新查询a的值为80，更新成功

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218173957223.png)









将背书策略修改为 "AND('Org1MSP.member','Org2MSP.member')"

```bash
 peer lifecycle chaincode approveformyorg  --signature-policy "AND('Org1MSP.member','Org2MSP.member')" --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc --version 1 --init-required --package-id mycc_1:4ad799ccef18d596f8c175fe1849cadc63f92a5efb1e7332712fbb2827a2ec6f --sequence 3 --waitForEvent
```

```bash
 peer lifecycle chaincode commit -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --version 1 --sequence 3 --init-required    --signature-policy "AND('Org1MSP.member','Org2MSP.member')"
```

同样只设置peer0.org1.example.com节点，重新invoke，查看节点日志结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174036456.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

`` ERRO 0ed VSCC error: stateBasedValidator.Validate failed, err validation of endorsement policy for chaincode mycc in tx 15:0 failed: signature set did not satisfy policy``

不满足背书策略，因为我们已经重新设置为"AND('Org1MSP.member','Org2MSP.member')"

接下来，我们通过设置键级别背书策略的方法，将上面操作完成

### 3.2 键级别背书策略设置

键级别的背书策略是通过智能合约内部调用SDK完成

shim包提供了以下的函数设置或者恢复键对应的背书策略。下面的ep代表是“endorsement policy”的缩写。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174101358.png)

对于私有数据，以下功能适用：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174107564.png)

Go shim提供了扩展功能，允许链码开发人员根据组织的MSP标识符来处理认可策略
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174112815.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)


根据官方给的说明
假如需要两个特定组织来批准key更改，设置key的背书策略，请将两个org都传递MSPIDs给AddOrgs()，然后调用Policy()构造可以传递给的认可策略字节数组SetStateValidationParameter()
接下来我们进行实践

#### 3.2.1 编辑合约提供修改背书策略方法
首先导入相关依赖包
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174118973.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

新增function 设置背书策略

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174124300.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)


endorsement具体实现如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174130426.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
由于新增了引入包，先下载依赖
进入合约目录输入以下命令

```bash
 go mod vendor
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174140682.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)


#### 3.2.2 升级合约

参考
[Fabric2.0 智能合约实践- 升级智能合约](https://blog.csdn.net/qq_28540443/article/details/104335498)


#### 3.2.3 验证背书策略
设置a的背书策略为 AND("Org1MSP.member","Org2MSP.member") 

 

```bash
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt  -c  '{"Args":["endorsement","a","Org1MSP","add"]}'
```

```bash
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt  -c  '{"Args":["endorsement","a","Org2MSP","add"]}'
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174232892.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

查看peer日志，交易成功

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020021817424054.png)


只设置peer0.org1节点作为背书节点

```bash
 peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt  -c '{"Args":["addTen","a"]}'
```
查看节点日志如下，提示不满足策略，因为前面设置的是AND("Org1MSP.member","Org2MSP.member") 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174251577.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)



修改背书策略，删除Org2MSP

```bash
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt  -c  '{"Args":["endorsement","a","Org2MSP","del"]}'
```

修改成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174405767.png)


重新执行以下命令：

```bash
 peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt  -c '{"Args":["addTen","a"]}'
```

修改a值从110变成120，执行成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200218174412222.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

## 4.总结
经过实践，合约的approve跟commit是明显有问题的，会出现approve完成后，checkcommit还是false,但是最终还是可以commit成功的情况，这时候只要commit成功就行了。以上这一点是比较需要注意的。Fabric2.0灵活的背书策略配还是可以的，需要chaincode源码留下评论。