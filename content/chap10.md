

由于智能合约的一般运行环境是在docker容器里面，因此在调试智能合约的代码逻辑上面可能不是那么的方便，特别是对于智能合约开发人员来说，因此下面将基于vscode编辑器演示如何调试go合约

## 1. 环境准备
|系统工具|  版本|
|--|--|
| window| 10 |
| go| 1.14 |
| vscode| 1.42.1 |

### 1.1 下载vscode go插件
打开vscode的插件管理，下载go插件如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200302183246323.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)

## 2.编辑合约
### 2.1 创建合约
在本地gopath创建mychaincode01/test目录，目录名称可自定义。
执行go mod init命令初始化go.mod 文件


### 2.1 编辑合约内容
我们先直接复制例子中的 abstore.go合约。

## 3. 编写合约测试代码

### 3.1 新建测试文件
在 abstore.go同目录下，新建 abstore_test.go

引入以下依赖包：


```go
import (
	"fmt"
	"testing"

	"github.com/hyperledger/fabric-chaincode-go/shim"
	"github.com/hyperledger/fabric-chaincode-go/shimtest"
)
```

我们主要使用shim以及shimtest去帮助我们模拟环境达到调试合约代码的效果，因此上面的包是最基本的引入。

### 3.2 编写测试内容并进行测试

#### 3.2.1 state 查询状态
编写测试查询状态方法如下：

```go
func checkState(t *testing.T, stub *shimtest.MockStub, name string, value string) {
	bytes := stub.State[name]
	if bytes == nil {
		fmt.Println("State", name, "failed to get value")
		t.FailNow()
	}
	if string(bytes) != value {
		fmt.Println("State value", name, "was not", value, "as expected")
		t.FailNow()
	}
}
```
#### 3.2.2 init 初始化
编写测试init方法

```go
func checkInit(t *testing.T, stub *shimtest.MockStub, args [][]byte) {
	res := stub.MockInit("1", args)
	if res.Status != shim.OK {
		fmt.Println("Init failed", string(res.Message))
		t.FailNow()
	}
}
```
测试方法如下：

```go

func TestExample02_Init(t *testing.T) {
	scc := new(ABstore)
	stub := shimtest.NewMockStub("ex02", scc)

	// Init A=123 B=234
	checkInit(t, stub, [][]byte{[]byte("init"), []byte("A"), []byte("123"), []byte("B"), []byte("234")})

	checkState(t, stub, "A", "123")
	checkState(t, stub, "B", "234")
}
```

运行前要进行``go mod vendor``下载依赖，添加断点，运行调试
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200302185312276.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI4NTQwNDQz,size_16,color_FFFFFF,t_70)
右边可以看到变量值

控制台debug console输出：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200302185443430.png)
看到Pass成功即测试通过。


#### 3.2.3 invoke交易
invoke调用MockInvoke
```go
func checkInvoke(t *testing.T, stub *shimtest.MockStub, args [][]byte) {
	res := stub.MockInvoke("1", args)
	if res.Status != shim.OK {
		fmt.Println("Invoke", args, "failed", string(res.Message))
		t.FailNow()
	}
}
```

编写测试代码

```go
func TestExample02_Invoke(t *testing.T) {
	scc := new(ABstore)
	stub := shimtest.NewMockStub("ex02", scc)

	// Init A=567 B=678
	checkInit(t, stub, [][]byte{[]byte("init"), []byte("A"), []byte("567"), []byte("B"), []byte("678")})

	// Invoke A->B for 123
	checkInvoke(t, stub, [][]byte{[]byte("invoke"), []byte("A"), []byte("B"), []byte("123")})
	checkQuery(t, stub, "A", "444")
	checkQuery(t, stub, "B", "801")

	// Invoke B->A for 234
	checkInvoke(t, stub, [][]byte{[]byte("invoke"), []byte("B"), []byte("A"), []byte("234")})
	checkQuery(t, stub, "A", "678")
	checkQuery(t, stub, "B", "567")
	checkQuery(t, stub, "A", "678")
	checkQuery(t, stub, "B", "567")
}
```

调试过程与3.2.2一致


## 4.总结
项目总体目录如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200302185920188.png)

通过shim包提供的mock进行调试确实是比较方便实际，不需要去另外部署容器，比较推荐。

