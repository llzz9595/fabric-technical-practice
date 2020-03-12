上一章可知fabric网络启动主要依赖脚本`./byfn.sh up`
接下针对这个脚本进行剖析，研究fabric2.0 first-network的启动过程。

## 1.byfn.sh
查看byfn.sh找到up模式主要做了什么，如下图可见，执行networkUp这个function来实现fabric网络启动。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213163814266.png)

接下来进入networkUp详细阅读：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213163946763.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
networkUp这个function里面核心脚本主要为以上红色框住的部分，分别为：
	1. 检查二进制文件是否可用以及对应版本docker镜像是否存在。
	2. 假如当前sh所在父目录不存在crypto-config目录就执行生成区块、通道以及证书脚本，详情请查看[Fabric2.0 first-network 生成配置说明](https://blog.csdn.net/qq_28540443/article/details/104282768)
	3. 使用docker-compose命令启动fabric网络。
	4. 加载go合约依赖包
	5. 使用cli客户端执行脚本操作
	
	     其中在默认条件下启动yaml文件包括：
		 docker-compose-cli.yaml
	     docker-compose-etcdraft2.yaml

## 2.docker-compose-cli.yaml     

为了看清楚docker-compose-cli.yaml具体启动了什么，我们将文件拆分执行
首先打开控制台，输入以下命令
	

```bash
cd first-network
docker-compose -f docker-compose-cli.yaml up -d 2>&1
```

执行结果如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213164756714.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
	总共启动了6个容器分别是：
 
   5个排序节点中的其中一个节点

     orderer.example.com

  两个组织org1、org2的节点 

	 peer0.org1.example.com**
     peer1.org1.example.com**
     peer0.org2.example.com**
     peer1.org2.example.com**

fabric客户端
   
     cli	

### 2.1 排序节点启动
orderer.example.com
   	从排序节点入手：
	一般来说，我会从启动日志入手，先看结果再看配置，打开控制台，输入以下命令
```bash
docker logs -f orderer.example.com --tail=300
```
	
控制台输出了orderer.example.com的启动日志如下

由于太长我分开几个部分来说

（1）排序节点配置项输出：
排序节点启动的时候，会根据环境变量输出相应的配置，包括证书配置、监听端口配置等，
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213165359148.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
而这些配置对应就是docker-compose-cli.yaml的以下配置：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213165424524.png)
	我们继续追踪base/docker-compose-base.yaml，找到orderer.example.com服务

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213165456781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
		从上图来看，orderer的配置还是引入peer-base.yaml中的orderer-base服务


![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213165510237.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
	
从上图来看与日志输出配置匹配，就是yaml里面加了ORDERER_前缀，其余配置的修改也可通过这种方式。详细排序节点配置说明，晚点补充。

（2）加载创始区块，启动排序节点。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213165546881.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
	好的系统的一个好的日志输出，fabric的日志输出还是很优秀的，从日志输出可以跟踪排序节点启动究竟干了什么，通过上图日志输出，可以读到：

	1.根据1的配置初始化排序节点
	2. 创建本地账本目录
	3. 设置排序节点服务监听端口
	4. 根据创始区块文件初始化本地账本
	5. 启动fabric系统通道
	6. 启动raft服务

（3）raft服务启动
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213165916906.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
	上图已经标识了核心日志并且进行说明，有一点还是很清晰的就是，(对于没仔细研究fabric2.0 raft的人来说)一个通道就是一个raft服务集群。红色的日志是错误，在分配完raft集群节点后，其余节点没启动，所以都是失败的，后面一直在做重连操作。
	
以上就是orderer.example.com启动剖析过程。

### 2.2 节点启动
然后我们来看以下peer0.org1.example.com
打开控制台，输入以下命令

```bash
docker logs -f peer0.org1.example.com --tail=300
```
控制台输出结果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213170230304.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
	peer节点启动主要是
	 1. 启动peer grpc服务
	2. 初始化gossip服务
	3. 初始化本地账本
	4. 安装系统链码,这里比起1.x，2.0多了一个系统链码就是_lifecycle，是跟智能合约相关的链码，我们下一篇进行详解。
	5. 基于gossip对同组织节点进行交互
## 3.docker-compose-etcdraft2.yaml
同样的套路，首先打开控制台，输入以下命令：

```bash
cd first-network
docker-compose -f docker-compose-etcdraft2.yaml up -d 2>&1
```
控制台输出结果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020021317041874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
由上图可知，主要启动了其余4个排序节点

先不看新启动的排序节点，查看原本开始启动的orderer.example.com
控制台输入以下命令

```bash
 docker logs --tail=300  -f orderer.example.com
```
控制台输出结果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213170445619.png)
由日志可以观察到，orderer.example.com以及成功连接其他raft节点，**并且在raft 的term2 leader 从orderer.example.com (0)变成orderer5.example.com（5）**

接下来我们看一下orderer5.example.com节点的情况
控制台输入以下命令： 
```bash
docker logs --tail=300  -f orderer5.example.com
```
控制台输出：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213170538727.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
由日志可见，在term2 经过一轮投票后，orderer5获得5票回应成为了term2的raft leader。
整体网络处于完全启动状态。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200213170655345.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)


## 4.总结
上面是对网络启动的脚本以及配置剖析，虽然比较简单，但是主要是展示整个学习的过程，一些比较详细的配置说明还是从官方文档学习比较好，通过上面的学习，也可以看到fabric2.0的一些新特性，主要集中在智能合约以及raft共识，后面将继续往这个方向进行研究。