## 1.创建通道准备
### 1.1 创建通道配置文件

由于first-network目录已存在configtx.yaml，如果需要修改通道配置的，可备份原本configtx.yaml，修改相关通道配置。


### 1.2 环境准备
打开控制台，执行以下命令

 - 设置二进制文件configtxgen目录到环境变量，方便调用

```bash
 export PATH=${PWD}/../bin:${PWD}:$PATH
```

 - 设置环境变量 FABRIC_CFG_PATH为configtx.yaml所在目录

```bash
 export FABRIC_CFG_PATH=${PWD}
```

### 1.3 创建通道tx文件
控制台执行以下命令

```bash
  configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel2.tx -channelID channel2
```
-outputCreateChannelTx ：输出tx文件路径
-channelID： 通道ID

执行结果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214162722848.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

## 2.创建通道
原本创建通道是通过cli客户端创建的，2.0这次可以直接通过二进制文件创建，二进制与cli的区别，除了表面形式的区别外，其实都是一样，不同在于cli的环境变量一开始就设置好一个默认的，例如peer的证书路径，使用二进制的话，就直接在控制台设置环境编码，详情查看fabric-samples/test-network的脚本，这里不做详细介绍，接下来我们还是使用cli比较快捷创建一个测试通道。

进入cli容器

```bash
docker exec -it cli bash
```
进入后:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214162835231.png)
由于cli的channel-artifacts已经与宿主机的~/first-network/channel-artifacts建立映射，因此上面新建的channel文件也存在cli的目录下

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214162916892.png)

在当前目录输入命令：

```bash
 peer channel create -o orderer.example.com:7050 -c channel2 -f ./channel-artifacts/channel2.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

` 自定义的话只需要修改
-o 参数排序节点服务域名端口 
-c 通道ID
 -f 通道文件所在路径
-tls 是否启用tls 
-cafile ca路径`

控制台数据结果如下，表示通道创建成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020021416300325.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
当前目录出现通道区块文件如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214163038555.png)
## 3.节点加入通道
设置cli连接节点对象只需要设置相应的环境变量，目前cli设置的节点为peer0.org1.example.com

输入命令，查看环境变量

```bash
env|grep CORE
```
输出结果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214163110244.png)
如需要修改节点只需要修改上面的环境变量为对应节点的配置，现在将peer0.org1.example.com添加到通道

控制台输入以下命令：

```bash
 peer channel join -b channel2.block
```
``-b 区块文件路径``

控制台输出如下结果，表示节点加入成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020021416325128.png)

查看排序节点日志如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214163324354.png)
排序节点写入了新的区块，同时为该通道创建了一个raft集群。

## 4.验证节点加入通道
控制台输入

```bash
 peer channel list
```
控制台输出结果如下：
可以看到当前节点已经加入到channel2

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200214163650827.png)

## 5.总结
在通道方面，客户端的操作上相较于1.x没什么变化，相关通道策略配置跟1.4基本一致，变化在于底层共识，原本的创建通道只是创建一个区块，在kafka开个订阅Topic，现在2.0的是一个通道就创建一个raft服务集群实现通道交易一致性。以上的脚本主要参照first-network下的script/script.sh。
