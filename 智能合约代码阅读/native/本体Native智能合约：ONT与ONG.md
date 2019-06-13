# 本体Native智能合约：ONT与ONG



​	本体区块链网络中包含两种虚拟货币：ONT与ONG，二者并非相互独立，而是相互联系的，本文记录阅读代码时的总结和思考。

------

### 1. ONT

​	ONT作为整个本体网络的基础token，地位相当于BTC之于比特币网络，总共发行10亿个，最小单位为1，最少要交易一个ONT，比如在交易所买9个ONT，提币都需要付一个ONT作为手续费(我哭了)。下面从ONT的智能合约出发，介绍ONT可以进行的操作。

​	首先简单介绍一下智能合约代码的执行过程。代码中会产生一个NativeService对象，它包含调用智能合约所需要的信息(如下)，在智能合约初始化时，各个合约会先注册自身可调用的函数，使得随后可以根据合约地址和函数名直接调用函数。

```go
type NativeService struct {
   CacheDB       *storage.CacheDB //数据库对象的引用，按照键值对的方式存储数据
   ServiceMap    map[string]Handler //存储函数名对应的智能合约函数
   Notifications []*event.NotifyEventInfo //调用智能合约后，包含在交易中的通知
   InvokeParam   sstates.ContractInvokeParam //调用的智能合约地址和函数名，版本号等
   Input         []byte //函数的参数
   Tx            *types.Transaction	//交易
   Height        uint32 //区块高度
   Time          uint32 //区块时间
   BlockHash     common.Uint256	//区块hash
   ContextRef    context.ContextRef	//函数执行的上下文，包含当前执行所需要的信息
}
```

​	ONT具备[OEP-4](https://github.com/ontio/OEPs/blob/master/OEPS/OEP-4.mediawiki)协议中所规定的功能，如下为ONT合约的注册函数，将合约中的函数注册到NativeService实例中，比如native.Register(INIT_NAME, <u>OntInit</u>)中的OntInit是ONT的初始化函数：

```go
func RegisterOntContract(native *native.NativeService) {
	native.Register(INIT_NAME, OntInit) //初始化ONT合约，完成ONT的分配，并在DB中存储ONT的总发行量，智能调用一次
	native.Register(TRANSFER_NAME, OntTransfer) //转账，可实现多笔交易，比如A转x个ONT给B，实现方式是先扣掉A的x个ONT，更改DB数据，然后再给B加上x个ONT，然后对A和B提取当前的ONG，转账一定会触发ONG的提取
	native.Register(APPROVE_NAME, OntApprove) //A允许B通过transferFrom使用自己x个ONT
	native.Register(TRANSFERFROM_NAME, OntTransferFrom) //在A approve之后，B可以使用A的ONT转账
	native.Register(NAME_NAME, OntName) //返回“ONT Token”
	native.Register(SYMBOL_NAME, OntSymbol)	//返回“ONT“
	native.Register(DECIMALS_NAME, OntDecimals)	//返回精度
	native.Register(TOTALSUPPLY_NAME, OntTotalSupply)	//返回总发行数
	native.Register(BALANCEOF_NAME, OntBalanceOf) //查看A的账户余额
	native.Register(ALLOWANCE_NAME, OntAllowance)	//查看A借给B的余额
}
```

​	目前可知CacheDB为K-V存储，通过构造特殊的key，存储value，ONT账户余额，ONG账户余额都是如此，后面继续深入去看本体在数据存储上的实现。

------

### 2. ONG

​	ONG是本体双Token中的另一个，也具备转账等功能，初始发行10^18，最小单位为10^-9，在本体网络中，ONG更多承担gas的角色，即调用智能合约的"汽油费"，它会根据ONT持有量和时间不停增长。

​	依然是根据合约注册函数来了解ONG的具体功能：

```go
func RegisterOngContract(native *native.NativeService) {
	native.Register(ont.INIT_NAME, OngInit) //合约初始函数，将ONG总发行量存入DB中，同时将所有ONG发给ONT合约地址，其余函数和ONT基本一致
	native.Register(ont.TRANSFER_NAME, OngTransfer)
	native.Register(ont.APPROVE_NAME, OngApprove)
	native.Register(ont.TRANSFERFROM_NAME, OngTransferFrom)
	native.Register(ont.NAME_NAME, OngName)
	native.Register(ont.SYMBOL_NAME, OngSymbol)
	native.Register(ont.DECIMALS_NAME, OngDecimals)
	native.Register(ont.TOTALSUPPLY_NAME, OngTotalSupply)
	native.Register(ont.BALANCEOF_NAME, OngBalanceOf)
	native.Register(ont.ALLOWANCE_NAME, OngAllowance)
}
```

​	ONG的转账不会触发ONG的提取，只有ONT的transfer和transferFrom可以触发，同时transferFrom只会触发"借款人"和"收款人"的ONG提取，不会触发sender(被借款人)的ONG提取。

​	ONG的提取流程：1. 触发grantONG => 2. 根据offset和balance计算ONG的值 => 3. 调用ONG合约的approve方法，将ONT合约的ONG使用权授予某个地址

​	所以用户想要将ONG转到自己的地址下，还要再手动转出approve的ONG。

------

### 3. 对ONT与ONG的思考

​	本体和主流的BTC、以太坊网络不同，选择设定两个token，ONT最小单位为1，ONG则是10^-9，ONT作为主要货币，ONG则能承担gas的功能，并根据ONT数目随时间增加，二者是相辅相成的关系。

以下为疑问：

- 但是为什么要选择双币模式呢？

- ONT最小单位为1，是否会影响币价？

  