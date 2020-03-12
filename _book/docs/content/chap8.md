
继续上一章，接下来将进行智能合约合约的升级
还是先在控制台输入 `docker exec -it cli bash` 进入cli的控制台，默认cli的环境变量节点为 peer0.org1.example.com.

## 1.查看需要升级的智能合约信息

目前已部署一个智能合约到 peer0.org1.example.com，cli控制台输入命令：

```bash
peer lifecycle chaincode queryinstalled
```

查看mycc合约的package_id如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200215215610268.png)



查看mycc在mychannel通道的合约定义，cli控制台输入命令：

```bash
peer lifecycle chaincode querycommitted -C mychannel
```



可以看到mycc具体信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020021521562028.png)
| 参数 |中文描述  | 值
|--|--|--|
| Name  |合约名称  |mycc
|Version |合约版本|1
|Sequence |合约序列号(升级1次，加1) |1
|Endorsement Plugin|背书插件|escc
|Validation Plugin|校验插件|vscc

执行一次query操作方便后面对比

```bash
 peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```

此时控制台输出a的值为90
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200215215910123.png)

## 2 修改合约代码

假如本地不存在该合约代码，2.0提供从节点上获取代码包的操作
控制台输入以下命令

```bash
 peer lifecycle chaincode getinstalledpackage --package-id  mycc_1:00ef9e95ea103b2c27eacd5a62efd9b34863c672d236a1ce99a7d539b2f9ef7a
```

packge-id：使用1步骤中查询获取的合约packge_id

当前目录出现mycc.tar.gz包为合约包，解压合约包里面的code.tar，编辑合约代码
添加一个方法默认加10
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020021521591983.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200215215925475.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

## 3. 重新打包合约

控制台输入以下命令打包合约：

```bash
 peer lifecycle chaincode package mycc.tar.gz --path github.com/hyperledger/fabric-samples/chaincode/abstore/go/ --lang golang --label mycc_1
```

将根据新的合约代码打包成新的智能合约包mycc.tar.gz


## 4. 重新安装合约

更新合约代码需要重新将合约安装到节点上 

```bash
peer lifecycle chaincode install mycc.tar.gz
```

控制台输出合约安装信息：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200215215941863.png)

此时我们再查询一下节点安装合约信息，控制台输入：

```bash
peer lifecycle chaincode queryinstalled
```

控制台输出：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200215215955218.png)

可以看到新增了一个package_id不一样的mycc,原来的还在。

## 5.修改合约定义

真正升级的操作其实是在合约定义这一步骤
控制台输入以下命令，``与原来相比将package_id修改为新安装的合约package_id，sequence要改成2，因为install了2次 ``

```bash
peer lifecycle chaincode approveformyorg --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc --version 1 --init-required --package-id  mycc_1:2f358faa3475e5c37a90be9a7c0db2f608ecb09b13f64b001f83799be9fccc77 --sequence 2 --waitForEvent
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200215220007895.png)


检查approve状态，控制台输入：

```bash
peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name mycc --version 1 --sequence 2 --output json --init-required 
```

``sequence要改成2，因为install了2次，否则报错Error: query failed with status: 500 - failed to invoke backing implementation of 'CheckCommitReadiness': requested sequence is 1, but new definition must be sequence 2``



## 6.切换节点重复 3，4，5操作
环境变量切换peer0.org2.example.com，重复3，4，5步，知道出现以下结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020021522010997.png)

## 7.重新提交合约定义

完成智能合约lifecycle策略后，重新提交合约定义

```bash
 peer lifecycle chaincode commit -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --version 1 --sequence 2 --init-required
```

``注意序列号sequence``

控制台输出以下结果：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200215220132660.png)

此时通过新的智能合约查询a的值，还是为90

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200215220139267.png)
## 8.检验新合约代码

刚刚添加了addTen这个方法，理论上a应该加10等于100
控制台输入以下命令调用addTen

```bash
 peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt  -c  '{"Args":["addTen","a"]}'  
```

查看结果a果然为100 升级成功

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200215220152174.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)