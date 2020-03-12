Fabric2.0动态添加组织是很多朋友比较关注，我们从上一章更新通道配置的思路来看，动态添加组织实践上也是通过修改区块配置来实现，
接下来的操作基于first-network已部署好的网络。

## 1. 新增org3证书配置
对于fabric网络来说，要新增一个组织，首先是从证书开始，因为证书就是fabric里面的身份。
编辑证书配置org3-crypto.yaml （first-network/org3-artifacts）

```bash
PeerOrgs:
  # ---------------------------------------------------------------------------
  # Org3
  # ---------------------------------------------------------------------------
  - Name: Org3
    Domain: org3.example.com
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
```

配置了Org3组织2个节点1个普通用户User

执行以下命令，生成证书

```bash
 cd org3-artifacts/
 ../../bin/cryptogen generate --config=org3-crypto.yaml --output="../crypto-config"
```

执行完成后查看../crypto-config/peerOrganizations目录可以看到新增的组织证书目录
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155532523.png)

## 2. 新增org3定义到区块链

之前我们启动的网络的时候，在启动前需要是创建创始区块与通道配置，因此在为了让区块链知道这个新来的组织，需要把组织的配置添加到区块配置中
配置文件~\first-network\org3-artifacts\configtx.yaml
注意证书目录必须对应正确的org3证书目录

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155542424.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
在first-network目录控制台输入以下命令生成org3定义

```bash
export FABRIC_CFG_PATH=$PWD
 bin/configtxgen  -printOrg Org3MSP -configPath org3-artifacts > channel-artifacts/org3.json
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155553392.png)



## 3.启动org3相关节点容器

容器配置 ~/first-network/docker-compose-org3.yaml

执行以下命令启动 

```bash
docker-compose -f docker-compose-org3.yaml up -d
```



``docker ps -a |grep org3`` 查看状态，启动失败了，查看日志如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155625467.png)

``Cannot run peer because error when setting up MSP of type bccsp from directory /etc/hyperledger/fabric/msp: could not load a valid signer certificate from directory /etc/hyperledger/fabric/msp/signcerts: stat /etc/hyperledger/fabric/msp/signcerts: no such file or directory``

从错误来看就是找不到证书，这时候看看docker-compose-org3.yaml，目录在org-artifacts下，我们之前是放在first-network/crypto-config下的，为了方便原本cli切换环境变量。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155644130.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)




知道目录错了修改为正确目录即可
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155651116.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

删除错误容器

```bash
 docker-compose -f docker-compose-org3.yaml rm
```

重新启动

```bash
 docker-compose -f docker-compose-org3.yaml up -d
```

再次查看容器状态成功启动
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155705211.png)



看到还多启动了一个Org3cli容器，是org3环境变量的cli，可以不起的，不过启动了就启动了

 ## 4. 更新通道配置
接下需要将org3添加到通道里面，步骤与上一章差不多。

Org3cli是默认org3配置的cli容器，不用白不用。

进入org3cli  ``docker exec -it Org3cli bash``

```bash
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CHANNEL_NAME=mychannel
echo $ORDERER_CA && echo $CHANNEL_NAME
```


### 4.1 获取配置
将配置切换成org1，因为当前org3不能获取当前通道最新配置


```bash
    export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
    export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
    export CORE_PEER_LOCALMSPID="Org1MSP"
   export  CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
```

输入以下命令获取最新块

```bash
peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
```

获取到第8块区块
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155754291.png)



### 4.2 修改配置

将pb文件转json

```bash
configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json
```

将之前org3的配置org3.json添加到config.json
先把之前生成的org3.json放进去Org3cli容器

 打开新的控制台，宿主机输入：

```bash
docker cp channel-artifacts/org3.json 6b9b3f2a22c4:/opt/gopath/src/github.com/hyperledger/fabric/peer
```

``6b9b3f2a22c4： Org3cli容器id``


Org3cli容器：

```bash
jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json org3.json > modified_config.json
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155831842.png)




将config.json 跟modified_config.json 转pb编码

```bash
configtxlator proto_encode --input config.json --type common.Config --output config.pb

configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
```

计算两个pb差异

```bash
configtxlator compute_update --channel_id mychannel --original config.pb --updated modified_config.pb --output org3_update.pb
```

将更新的pb解析为json

```bash
configtxlator proto_decode --input org3_update.pb --type common.ConfigUpdate | jq . > org3_update.json
```

现在我们有一个解码后的更新文件org3_update.json，我们需要将其包装在信封消息中。此步骤将使我们返回之前删除的header字段。我们将这个文件命名为org3_update_in_envelope.json：


```bash
echo '{"payload":{"header":{"channel_header":{"channel_id":"'$CHANNEL_NAME'", "type":2}},"data":{"config_update":'$(cat org3_update.json)'}}}' | jq . > org3_update_in_envelope.json
```


使用我们正确格式的JSON – org3_update_in_envelope.json我们将configtxlator最后一次使用该工具，并将其转换为Fabric所需的完整protobuf格式。我们将命名我们的最终更新对象org3_update_in_envelope.pb：



```bash
configtxlator proto_encode --input org3_update_in_envelope.json --type common.Envelope --output org3_update_in_envelope.pb
```

### 4.3 签名并提交更新配置

添加组织需要目前通道大部分的组织Admin签名，目前Org3cli的环境变量是org1

输入以下命令

```bash
peer channel signconfigtx -f org3_update_in_envelope.pb
```

切换环境为org2执行更新配置，因为update也会为当前组织签名，所以不需要再org2签名

```bash
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
   export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
   export CORE_PEER_LOCALMSPID="Org2MSP"
   export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2
```

更新命令：

```bash
peer channel update -f org3_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050 --tls --cafile $ORDERER_CA
```

更新成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155917248.png)


## 5. org3加入通道
切换成org3环境变量

```bash
export CORE_PEER_LOCALMSPID=Org3MSP
export CORE_PEER_ADDRESS=peer0.org3.example.com:11051
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
```

获取mychannel 0号块创始块

```bash
peer channel fetch 0 mychannel.block -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155940536.png)



该命令将创世块返回到名为的文件mychannel.block。现在，我们可以使用此块将org3的节点加入通道。

输入以下命令：

```bash
peer channel join -b mychannel.block
```

通过peer channel list 验证

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226155944923.png)

## 6. 总结
动态组织加入大概就是这样，想必删除组织也是相似的，洞悉fabric的配置原理就可以应对大部分fabric的应用需求。