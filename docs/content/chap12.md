

根据前面的步骤，我们基于cli客户端完成了一系列操作，但是正常情况下，我们的工程一般会使用SDK去调用
网络完成交易，因此从这一章开始实践何基于Java SDK调用Fabric2.0网络完成交易。

## 1.Gateway
在进行实践前，有一个比较新的概念需要了解就是Gateway。
为了应对由于Fabric网络中的变化频繁所造成的后果，Fabric2.0 原本的SDK之上盖了一层网关Gateway，用于减轻应用程序的负担。

具体拓扑结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200222214525147.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

MagnetoCorp和DigiBank应用程序（发行和购买）将各自的网络交互委托给其网关。每个网关都了解网络通道拓扑，其中包括两个组织MagnetoCorp和DigiBank的多个peers和order，使应用程序专注于业务逻辑。节点间可以使用gossip协议在组织内部和组织之间相互共识交互。

## 2.环境准备
|系统工具  |版本  |备注
|--|--|--|
| Window | 10 |
| Fabric | 2.0 |已部署好mychannel通道以及mycc合约到两个组织节点
| Java | 1.8 |
| Maven | 3.5.2 |

Fabric网络结构
|节点类型 |节点名 |所属组织|ip  |服务端口
|--|--|--|--|--|
| orderer| orderer.example.com|-|192.168.2.104|7050
| peer | peer0.org1.example.com |org1|192.168.2.104|7051
| peer | peer1.org1.example.com |org1|192.168.2.104|8051
| peer | peer0.org2.example.com |org2|192.168.2.104|9051
| peer | peer1org2.example.com|org2|192.168.2.104|10051
| ca | ca1.org1.example.com|org1|192.168.2.104|7054
| ca | ca2.org1.example.com|org2|192.168.2.104|8054

## 3.创建基础工程
新建一个Maven工程，添加以下依赖：

```java
<dependency>
  <groupId>org.hyperledger.fabric</groupId>
  <artifactId>fabric-gateway-java</artifactId>
  <version>2.0.0</version>
</dependency>
```

## 4.创建connectionProfile
connectionProfile用于创建一个连接网络对象，如果有跑过first-network的朋友，相比记得一个ccp.sh的命令，生成的connection-org1.json、connection-org1.yaml这种文件，其实这些就是我们所需要的connectionProfile，可以直接使用。当然也可以自行编写，自行编写请参考[官方模板](https://hyperledger-fabric.readthedocs.io/en/release-2.0/developapps/connectionprofile.html)

### 4.1 配置文件结构说明
整体connectionProfile结构如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200222221024680.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
包含对象说明：

|参数名 |描述|
|--|--|
|name|自定义网络名称
|version|自定义网络版本
|client|客户端相关信息
|channels|网络所包含的通道信息
|organizations|网络中组织信息
|orderers|排序节点信息
|peer|pee节点信息
|certificateAuthorities|ca节点信息

#### 4.1.1 client
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200222221711506.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

#### 4.1.2 channels
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200222221838973.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
其中节点对象有4中角色定义
|角色 |描述|
|--|--|
|endorsingPeer| 具有背书权限节点
|chaincodeQuery| 具有合约查询权限节点
|ledgerQuery|具有账本查询权限节点
|eventSource|  event hub节点

#### 4.1.3 organizations
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200222222702232.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

其中admin私钥与admin签名证书支持路径与文件内容，路径使用参数``path``，对应值填写路径，文件内容使用参数``pem``，对应值填写私钥或者证书内容。

#### 4.1.4 orderers
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200222223321769.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)


#### 4.1.5 peer
与排序节点类似，如果是tls必须配置tlsCACerts
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020022222343458.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

#### 4.1.6 certificateAuthorities
与排序节点类似，如果是tls必须配置tlsCACerts

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200222223623958.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
## 5. JAVA工程目录说明
新建的工程目录如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200222223714865.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

    src/main/java ： 存放demo主程序类
    src/main/resources/connection.json ： 上面新建好的connectionProfile
    src/main/resources/crypto-config: 存放fabric网络证书内容(选择用到的就行)

## 6. 实践
### 6.1 创建网关账户
网关账户就是相当于连接fabric网络的fabric用户对象。

```java
          //使用org1中的user1初始化一个网关wallet账户用于连接网络
            Wallet wallet = Wallets.newInMemoryWallet();
            Path certificatePath = credentialPath.resolve(Paths.get("signcerts", "User1@org1.example.com-cert.pem"));
            certificate = readX509Certificate(certificatePath);

            Path privateKeyPath = credentialPath.resolve(Paths.get("keystore", "priv_sk"));
            privateKey = getPrivateKey(privateKeyPath);
             //放进wallet
            wallet.put("user",Identities.newX509Identity("Org1MSP",certificate,privateKey));
```

账户对象都可以存放到wallet里面，方便存取。
证书对象使用的是org1的user私钥，证书。

### 6.2 创建网关
通过connectionProfile以及网关账户创建网关

```java
 			//根据connection-org1.json 获取Fabric网络连接对象
            GatewayImpl.Builder builder = (GatewayImpl.Builder) Gateway.createBuilder();

            builder.identity(wallet, "user").networkConfig(NETWORK_CONFIG_PATH);
```

``NETWORK_CONFIG_PATH ： connectionProfile文件路径``
### 6.3 连接网关

```java
           //连接网关
            gateway = builder.connect();
            //获取mychannel通道
            Network network = gateway.getNetwork("mychannel");
            //获取合约对象
            Contract contract = network.getContract("mycc");
```

网关连接后
``getNetwork``:可以根据通道名称获取Fabric具体通道网络
``network.getContract``：可以根据合约名称获取部署到对应通道的智能合约对象

### 6.4 交易
基于上一部分的智能合约，有一个addTen的交易，结果是对象加10

首先我们对合约对象``a``当前的值进行查询

```java
      //查询合约对象evaluateTransaction
            byte[] queryAResultBefore = contract.evaluateTransaction("query","a");
            System.out.println("交易前："+new String(queryAResultBefore, StandardCharsets.UTF_8));
```
然后调用``addTen`` 进行a+10

```java
 // 创建并且提交交易 
            byte[] invokeResult = contract.createTransaction("addTen")
                    .setEndorsingPeers(network.getChannel().getPeers(EnumSet.of(Peer.PeerRole.ENDORSING_PEER)))
                    .submit("a");
            System.out.println(new String(invokeResult, StandardCharsets.UTF_8));

            byte[] invokeResult = contract.createTransaction("invoke")
                    .setEndorsingPeers(network.getChannel().getPeers(EnumSet.of(Peer.PeerRole.ENDORSING_PEER)))
                    .submit("a","b",10);
            System.out.println(new String(invokeResult, StandardCharsets.UTF_8));
```

此处``setEndorsingPeers`` 设置背书节点，这里选择了通道中背书权限节点集合

交易完成后再次进行查询

```java
  //查询合约对象evaluateTransaction
            byte[] queryAResultAfter = contract.evaluateTransaction("query","a");
            System.out.println("交易后："+new String(queryAResultAfter, StandardCharsets.UTF_8));
```


最后控制台输出：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200222225455136.png)

交易成功。

## 7. 总结
在调用交易方面，通过网关这种确实简单了很多，后面应用程序进行设计的时候，可以将网关这个包单独封装，应用程序再进行调用，可以基于网关这层保障应用程序的业务稳定性。而再创建通道、加入通道、部署合约等操作应该还是SDK那套，后续章节也将继续实践。需要本章源码的可以评论留言。

源码：https://github.com/llzz9595/fabricdemo