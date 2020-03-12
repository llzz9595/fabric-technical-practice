
Fabric2.0的重大变化之一就是支持外部构建智能合约，在1.x版本的Fabric中，合约是由Peer以容器的方式进行启动和维护，依赖于Docker。这在一定程度上违反了安全准则，并且在管理运维中带来了麻烦。Fabric 2.0支持用户自行启动合约容器。


# 1. 合约外部构建器启动器介绍
要实现不通过Peer直接构建合约，需要适用的工具就是外部构建器和启动器，外部构建器和启动器按照官方的说法是基于buildpacks实现将源代码转化成可执行程序的工具。构建器负责准备要执行的应用程序，启动器负责启动应用程序，并定期调用运行状况检查以监视启动的应用程序的活跃性。

## 1.1 构建器与启动器组成

外部构建器和启动器由四个程序或脚本组成：

    • bin/detect：确定是否应使用此buildpack来构建chaincode程序包并启动它。
    • bin/build：将chaincode包转换为可执行的chaincode。
    • bin/release （可选）：向peer提供有关链码的元数据。
    • bin/run （可选）：运行链码。

到这里大家了解这几个脚本是什么用处的就好，我们在后面实践过程中会逐步解释。

# 2 外部构建启动合约
## 2.1 环境准备
|系统工具  |版本  |备注
|--|--|--|
| CentOS | 7 |
| Docker |   18.09.4|
| Docker-compose |   1.23.2| 参考：[CentOS7安装Docker-compose推荐方案](https://blog.csdn.net/qq_28540443/article/details/104262822)|
| GO |   1.13.4| 参考：[CentOS7安装Go](https://blog.csdn.net/qq_28540443/article/details/104265094)


## 2.2 构建器启动器脚本准备
根据上面对构建器启动器的介绍，我们可以只需要创建``bin/detect、bin/build、bin/release``三个脚本。

首先我们在 ~/first-network 下新建一个external/bin目录用于存放上面三个脚本

```bash
cd ~/first-network
mkdir external && mkdir external/bin
```

### 2.2.1 检测脚本
检测脚本为``bin/detect``,它用于检测当前install的合约是否适用外部构建器构建，假如``exit 0``就会继续执行构建发布外部合约的操作，``exit 1 ``就会按照原本适用节点安装部署合约的方式。

该脚本是自定义的，我这边提供一个模板

```bash
#!/bin/sh

# The bin/detect script is responsible for determining whether or not a buildpack
# should be used to build a chaincode package and launch it.
#
# The peer invokes detect with two arguments:
# bin/detect CHAINCODE_SOURCE_DIR CHAINCODE_METADATA_DIR
#
# When detect is invoked, CHAINCODE_SOURCE_DIR contains the chaincode source and
# CHAINCODE_METADATA_DIR contains the metadata.json file from the chaincode package installed to the peer.
# The CHAINCODE_SOURCE_DIR and CHAINCODE_METADATA_DIR should be treated as read only inputs.
# If the buildpack should be applied to the chaincode source package, detect must return an exit code of 0;
# any other exit code will indicate that the buildpack should not be applied.

CHAINCODE_METADATA_DIR="$2"

set -euo pipefail

# use jq to extract the chaincode type from metadata.json and exit with
# success if the chaincode type is golang
# 判断目录下有没有metadata.json文件 有就exit 0 
if [ "$(cat "$CHAINCODE_METADATA_DIR/metadata.json" | sed -e 's/[{}]/''/g' | awk -F"[,:}]" '{for(i=1;i<=NF;i++){if($i~/'type'\042/){print $(i+1)}}}' | tr -d '"')" = "external" ]; then
    exit 0
fi

exit 1

```

这里跟官方模板不一样在于，适用 ``sh`` 执行脚本，适用``sed`` 替换jq，尽量少安装东西到节点容器。



### 2.2.2 构建脚本

构建脚本``bin/build``，是将我们提交的合约配置文件按照构建规则放到它指定的目录。

```bash
#!/bin/sh

# The bin/build script is responsible for building, compiling, or transforming the contents
# of a chaincode package into artifacts that can be used by release and run.
#
# The peer invokes build with three arguments:
# bin/build CHAINCODE_SOURCE_DIR CHAINCODE_METADATA_DIR BUILD_OUTPUT_DIR
#
# When build is invoked, CHAINCODE_SOURCE_DIR contains the chaincode source and
# CHAINCODE_METADATA_DIR contains the metadata.json file from the chaincode package installed to the peer.
# BUILD_OUTPUT_DIR is the directory where build must place artifacts needed by release and run.
# The build script should treat the input directories CHAINCODE_SOURCE_DIR and
# CHAINCODE_METADATA_DIR as read only, but the BUILD_OUTPUT_DIR is writeable.

CHAINCODE_SOURCE_DIR="$1"
CHAINCODE_METADATA_DIR="$2"
BUILD_OUTPUT_DIR="$3"

set -euo pipefail

#external chaincodes expect connection.json file in the chaincode package
if [ ! -f "$CHAINCODE_SOURCE_DIR/connection.json" ]; then
    >&2 echo "$CHAINCODE_SOURCE_DIR/connection.json not found"
    exit 1
fi

#simply copy the endpoint information to specified output location
cp $CHAINCODE_SOURCE_DIR/connection.json $BUILD_OUTPUT_DIR/connection.json

if [ -d "$CHAINCODE_SOURCE_DIR/metadata" ]; then
    cp -a $CHAINCODE_SOURCE_DIR/metadata $BUILD_OUTPUT_DIR/metadata
fi

exit 0

```

### 2.2.3 发布脚本
在完成合约基础构建之后，剩下只需要发布就行。
发布脚本是``bin/release``


## 2.3 修改节点配置
### 2.3.1 core.yaml
``core.yaml``文件有的朋友可能不太熟悉，还是简单科普一下，core.yaml是节点的默认配置文件，详细的配置项可以在源码``~/fabric/sample-config/core.yaml``里面找到，一般来说，假如我们没有定义core.yaml，节点内部/etc/hyperledger目录下也会有这个文件，他的参数跟环境变量是基本一致的，但是环境变量的优先级别高于``core.yam``,这里我们设置的参数由于是一个数组对象，我们将创建core.yaml并且将``core.yaml``映射到节点的``/etc/hyperledger/fabric``目录下面。

具体操作如下：

1. 复制sample-config目录下的core.yaml到~first-network

```bash
cd ~/first-network
cp ../../sample-config/core.yaml .
```

2.修改core.yaml合约相关配置

定位``:524``行

```bash
    externalBuilders:
         - path: /opt/gopath/src/github.com/hyperledger/fabric/peer/external
           name: builder
        #   environmentWhitelist:
        #      - ENVVAR_NAME_TO_PROPAGATE_FROM_PEER
        #      - GOPROXY

```

``path``: externalBuilders路径 ,填写节点容器内存放externalBuilder脚本(1.2.2写的3个脚本)的目录，目前定义在节点内部的/opt/gopath/src/github.com/hyperledger/fabric/peer/external，各位可自定义路径，后面映射。
 
 ``name:`` 外部构建器名称，自定义。

``environmentWhitelist: ``环境变量白名单，节点在调用构建器脚本的时候可以采用的环境变量。



然后保存文件就行了。


### 2.3.2 节点容器配置
节点容器配置我们只需要完成2个事情，把external的3个脚本映射到externalBuilders.path以及将core.yaml挂载到容器的/etc/hyperledger/fabric目录

以first-network的配置为例

```bash
cd ~/first-network
vi base/peer-base.yaml
```

编辑peer-base服务的挂载参数

```bash
services:
  peer-base:
    image: hyperledger/fabric-peer:$IMAGE_TAG
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # the following setting starts chaincode containers on the same
      # bridge network as the peers
      # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=${COMPOSE_PROJECT_NAME}_byfn
      - FABRIC_LOGGING_SPEC=INFO
      #- FABRIC_LOGGING_SPEC=DEBUG
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_PROFILE_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
      # Allow more time for chaincode container to build on install.
      - CORE_CHAINCODE_EXECUTETIMEOUT=300s
      #挂载外部启动智能合约相关脚本文件以及core.yaml
    volumes:
      # 挂载目录
      - ../external:/opt/gopath/src/github.com/hyperledger/fabric/peer/external
      - ../core.yaml:/etc/hyperledger/fabric/core.yaml
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start

```

然后就可以快乐地适用``./byfn.sh up``启动网络了~,具体可参考[(一)HyperLedger Fabric 2.0-release编译镜像二进制文件+测试网络部署](https://blog.csdn.net/qq_28540443/article/details/104265844)。


## 2.4 部署合约到网络
要实现外部部署合约，对于要install到fabric的合约包跟之前[(五)Fabric2.0 智能合约实践- 安装以及定义智能合约](https://blog.csdn.net/qq_28540443/article/details/104318163)的包是不一样的，这个包里面不需要合约的源代码。

按照定义来说，这个合约包含有：

>  metadata.json ： 合约的属性，与原本的合约属性不一致的是他的path可以为空，他的type可以自定义。
> 
> connection.json :这个文件是用于节点去连接合约的相关配置信息，双方交互基于grpc服务。
> 
> metadata文件夹：应该是跟couchdb存放私有数据相关的，本实践不做详细说明。

根据我这边的情况，目前只需要有metadata.json跟connection.json就行了。

我们在 ~/fabric-samples/chaincode/abstore/go 下面创建一个packing目录用于存放打包文件

```bash
cd ~/fabric-samples/chaincode/abstore/go && mkdir packing && cd packing
```

### 2.4.1 编写metadata.json
在packing目录下，编辑metadata.json文件如下：

```bash
{"path":"","type":"external","label":"mycc2"}
```

path: 不需要填写内容。
type: 自定义，用于``bin/detect``检测。
label: 自定义。


### 2.4.2 编写connection.json


例子：

```bash
{
    "address": "192.168.2.103:7052",
    "dial_timeout": "10s",
    "tls_required": false,
    "client_auth_required": false,
    "client_key": "-----BEGIN EC PRIVATE KEY----- ... -----END EC PRIVATE KEY-----",
    "client_cert": "-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----",
    "root_cert": "-----BEGIN CERTIFICATE---- ... -----END CERTIFICATE-----"
}
```

``address``:  我们将要在外面运行/部署的合约开出来的grpc服务ip跟端口，最好先定下来(没定好随便写个也行，反正后面一个个节点改就是了)。

``dial_timeout``:grpc连接超时时间。指定为具有时间单位（例如“ 10s”，“ 500ms”，“ 1m”）的字符串。如果未指定，默认值为“ 3s”。

``tls_required``: 是否需要tls，视乎您等会在合约上面编辑的grpc服务端是否需要证书，不需要就false，需要true。假如false的话，下面3项可以就这样按模板填写，写空也行。

``client_key``: grpc 客户端访问合约服务端需要的私钥。

``client_cert``: grpc 客户端访问合约服务端需要的证书。

``root_cert``: grpc 客户端访问合约服务端需要的根证书。



### 2.4.2 打包合约

还是在packing目录
执行一下命令：

```bash
 tar cfz code.tar.gz connection.json  
 tar cfz mycc2.tgz metadata.json code.tar.gz
```

查看当前packing目录
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200311145537953.png)

### 2.4.3 安装部署合约
整个2.4.3操作通过cli操作

```bash
docker exec -it cli bash 
cd ../../fabric-samples/chaincode/abstore/packing
```

后面从install开始到commit完成合约安装部署，跟平常没什么不一样

下面命令自行替换参数，具体参考[(五)Fabric2.0 智能合约实践- 安装以及定义智能合约](https://blog.csdn.net/qq_28540443/article/details/104318163)

install合约，每个背书节点都要install

```bash
peer lifecycle chaincode install mycc2.tgz
```

批准合约定义，必须符合lifecycle策略

```bash
peer lifecycle chaincode approveformyorg --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc2 --version 1 --init-required --package-id mycc_1:84b3aaf583a6632e806a0ff46ad804539797d84ff7826c88a6d6009a9930cfee --sequence 1 --waitForEvent
```

提交合约定义到网络

```bash
 peer lifecycle chaincode commit -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --version 1 --sequence 1  --init-required
```
完全部署成功之后，我们可以看到跟往常部署不一样的是，合约容器并没有启动，在节点容器
``/var/hyperledger/production/externalbuilder/builds/``出现构建发布后的合约文件目录:

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020031115154175.png)
####  2.4.3.1 可能出现的错误与解决方案

``could not build chaincode :docker build failed: platform builder failed: Failed to generatea Dockerfile:Unknow Chaincode Type:EXTERNAL``
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200311150123140.png)
首先错误的意思就您install的合约类型不对，合约类型定义是在我们前面定义的metadata.json里面的type，我们为了区份原本的部署合约方式填写了external，一个不属于fabric支持的合约类型的type。

那是不是说明我们错了？

是的。

但是错的地方在外部构建器启动器的脚本，并不是合约定义metadata.json。必须让你的bin/detect,bin/build,bin/release脚本是能正常执行，比如说节点容器没有jq包，但是官方案例里面是使用jq解析配置，这是detect执行就会报错，在报错的情况下，会直接选择原本的合约部署方式。当然按照我上面给的脚本是不会报错，大家大胆install。

### 2.4.4 编写合约
这一次顺序跟之前不一样，但是其实也是可以一开始编写的，问题不大。

以abstore合约例子为例，只需要修改原有的main方法，继承shim.ChaincodeServer

```bash
func main() {

        server := &shim.ChaincodeServer{
                CCID:    os.Getenv("CHAINCODE_CCID"),
                Address: os.Getenv("CHAINCODE_ADDRESS"),
                CC:      new(ABstore),
                TLSProps: shim.TLSProperties{
                                Disabled: true,
                },
        }

        // Start the chaincode external server
        err := server.Start()

        if err != nil {
                fmt.Printf("Error starting Marbles02 chaincode: %s", err)
        }
}

```

``CCID`` ： 部署完成后生产的合约ID，这里使用环境变量读取比较方便，如 
mycc2:6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454

``Address``:定义grpc服务IP端口，如 0.0.0.0:9999，这里使用环境变量读取比较方便。

``CC``: 合约对象。

``TLSProps``: tls配置，与connection.json相呼应。

### 2.4.5 启动合约
``整个2.4.5操作都在宿主机``

其实到这里，启动合约可以选很多种方式，你可以编辑器启动，也可以go build一下二进制启动，这里的话，对于测试/生产环境，我希望这个chaincode更加方便迁移环境(所以上面读取环境变量也是为此)，我会选择打包合约镜像，运行合约容器，当然一般开发环境调试可以IDE来弄，但是测试生产环境目前还是偏向docker。

因此我们接下来按照一般的容器构建，来构建我们的合约镜像以及启动合约容器


#### 2.4.5.1  构建合约镜像


在abstore/go 目录，编辑Dockerfile如下：


```bash
#This image is a microservice in golang for the Degree chaincode
FROM golang:1.13.8-alpine AS build

COPY ./ /go/src/github.com/abstore
WORKDIR /go/src/github.com/abstore

# Build application
RUN  GOPROXY="https://goproxy.cn" GO111MODULE=on   go build -mod vendor -o chaincode -v .

# Production ready image
# Pass the binary to the prod image
FROM alpine:3.11 as prod

COPY --from=build /go/src/github.com/abstore/chaincode /app/chaincode

USER 1000

WORKDIR /app
CMD ./chaincode

```

这个脚本就是构建合约可执行文件，启动容器运行合约可执行文件。


开始构建镜像

```bash
cd ~fabric-samples/chaincode/abstore/go

docker build -t chaincode/mycc2:1.0 .

```

然后完成后，通过``docker images |grep mycc2 ``确认
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200311153240591.png)


#### 2.4.5.2  启动合约容器
使用docker-compose启动，编辑docker-compose-mycc.yaml如下：

```bash
version: '2'
networks:
 default:
  external:
   name: net_byfn
services:
  #合约容器
  mycc2:
    #定义主机名
    container_name: mycc2.chiancode.com
    #使用的镜像
    image: chaincode/mycc2:1.0
    #容器的映射端口
    ports:
      - 9999:9999
    networks:
      - default
    #环境变量
    privileged: true
    environment:
      - CHAINCODE_CCID=mycc2:6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454
      - CHAINCODE_ADDRESS=0.0.0.0:9999

```

注意CHAINCODE_CCID一定要对应我们之前安装好的合约id

启动容器：

```bash
docker-compose -f docker-compose-mycc.yaml up -d
```

使用``docker ps``确认容器是否启动成功

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200311153556181.png)

# 3 测试外部合约
对部署好的合约进行测试，``docker exec -it cli bash ``进入cli容器。

init合约

```bash
 peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc201 --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --isInit -c '{"Args":["Init","a","100","b","100"]}'
```
查询交易结果：

```bash 
peer chaincode query -C mychannel -n mycc201 -c '{"Args":["query","a"]}'
```

控制台输出：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200311153858384.png)
结论：十分成功。

当然成功乃失败之母，我可是跟成功隔了好几代的小伙子。下面开始一波错误总结

## 3.1 有可能的报错及解决方案
1. ``Error: endorsement failure during invoke. response: status:500 message:"error in simulation: failed to execute transaction 32e56961b2d9720f5e266ad0e61ce8b5ecaa5857e5b68b19a229ee3bd57da86f: could not launch chaincode mycc2:6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454: error building chaincode: malformed chaincode info at '/var/hyperledger/production/externalbuilder/builds/mycc2-6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454/release/chaincode/server/connection.json': invalid character '\\n' in string literal"``
    
 

>    简单来说就是之前在合约定义的connection.json写得不规范，直接进去所有安装合约的节点里面的
>     /var/hyperledger/production/externalbuilder/builds/mycc2-6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454/release/chaincode/server/connection.json这个目录就是错误提示文件路径，改你conntion.json改到符合json解析规范为止。

    
2. ``Error: endorsement failure during invoke. response: status:500 message:"error in simulation: failed to execute transaction 3b30d51113c87ae5279a5a4efc1b20b926afda53b356da5d95026b8ab08a83ff: could not launch chaincode mycc2:6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454: error building chaincode: malformed chaincode info at '/var/hyperledger/production/externalbuilder/builds/mycc2-6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454/release/chaincode/server/connection.json': invalid character '~' after top-level value"``
   

> 同1

4. ``Error: endorsement failure during invoke. response: status:500 message:"error in simulation: failed to execute transaction 6d756de4e462811da9a44aa0acf68c27e2fe66445de04c1083bb257da116ede1: could not launch chaincode mycc2:6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454: error building chaincode: malformed chaincode info at '/var/hyperledger/production/externalbuilder/builds/mycc2-6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454/release/chaincode/server/connection.json': json: cannot unmarshal string into Go struct field ChaincodeServerUserData.tls_required of type bool"``

> 还是connection.json的错误，但是不是格式，tls_required参数类型必须是boolean，不需要双引号。

5. ``Error: endorsement failure during invoke. response: status:500 message:"error in simulation: failed to execute transaction 556ddd1378a67cf0bab2e81cce7377a605bfdb8ac9a49b815680148b93b70a88: could not launch chaincode mycc2:6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454: connection to mycc2:6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454 failed: error cannot create connection for mycc2:6be5ebad4dabfda4e055b936658bf29a28079a2a4188915013998591426ac454: error creating grpc connection to 192.168.2.101:9999: failed to create new connection: connection error: desc = \"transport: error while dialing: dial tcp 192.168.2.101:9999: connect: no route to host\""``

> 就是节点连不上您外部启动的合约服务，确认合约是否正常启动，再者telnet一下看看ip:port是不是通的，改到通为止。



# 4.总结
对实现外部构建启动合约总结流程如下： 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200311155533903.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

其中构建外部合约对于docker形式来说是构建合约镜像，如果是本地跑的可以不需要这一步。

实践完整一个外部构建启动合约的过程，其实比较痛苦，因为官方文档确实描述有点乱，像我这种一般人很容易就会各种迷，但是经过大家讨论以及源码分析后，除了解决问题之外，对fabric这个新特性也有了更深的认识，通过外部构建方式，合约不再是只能是一个容器形式存在，他可以IDE运行debug，也可以编译成二进制运行，对于不同环境不同开发者来说有了灵活的选择，虽然暂时只支持go合约。





源码： [https://github.com/llzz9595/fabric-learning](https://github.com/llzz9595/fabric-learning)

