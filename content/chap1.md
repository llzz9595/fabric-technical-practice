## 1.环境准备
|系统工具  |版本  |备注
|--|--|--|
| CentOS | 7 |
| Docker |   18.09.4|
| Docker-compose |   1.23.2| 参考：[CentOS7安装Docker-compose推荐方案](https://blog.csdn.net/qq_28540443/article/details/104262822)|
| GO |   1.13.4| 参考：[CentOS7安装Go](https://blog.csdn.net/qq_28540443/article/details/104265094)


## 2.下载源码
1.创建go工作目录
```bash
mkdir go
mkdir go/src
mkdir go/pkg 
mkdir go/bin
export GOPATH=xx/go
```

2.创建hyperledger目录

```bash
mkdir go/src/github.com/hyperledger
```
3.下载fabric release-2.0源码

```bash
cd go/src/github.com/hyperledger
git clone https://github.com/hyperledger/fabric.git
cd fabric && git checkout release-2.0
```

假如觉得git clone慢可参考 [如何快速clone github代码库](https://blog.csdn.net/qq_28540443/article/details/104264141)

## 3. 编译二进制文件以及docker镜像
当前在fabric目录
打开控制台进入fabric目录，执行以下命令
```bash
make all
```
中间可能提示没有安装gcc，此时只需要安装即可

```bash
yum install gcc
```

执行完成后，查看编译二进制文件如下：

```bash
ll build/bin
```
控制台输出如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200211171641259.png)


执行完成后，查看编译Docker镜像如下：

```bash
docker images |grep 2.0|grep fabric
```

控制台输出如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200211171800905.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
## 4.运行测试网络
1.将编译完成的二进制文件复制到 fabric-samples 目录

```bash
cp -r build/bin fabric-samples
```

2.网络准备

```bash
cd  fabric-samples/first-network 
./byfn.sh generate
```
生成证书文件以及通道文件如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020021117292335.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

3.运行测试网络

```bash
./byfn.sh up
```
控制台输出如下提示即运行成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200211173055678.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

查看docker状态如下：

```bash
docker ps |grep fabric
```
执行结果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200211173206200.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

## 5.END
官方测试网络运行结束，接下来将对2.0的部署配置、合约以及raft共识进行继续学习，请持续关注。


