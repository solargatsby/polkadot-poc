<h1>Substrate调研</h1>

<h2>1.基本架构</h2>

Substrate作为一个区块链开发框架，在架构上主要包括2各部分：Core和Runtime。 

Core部分对应到一般区块链系统中的基础模块部分，如共识，P2P网络，存储以及RPC和交易池等。 这一部分体现了当前区块链模型下都会具备的基础功能部分。

Runtime部分对应到了是一个区块链系统的业务部分，这一部分对于不同的区块链系统通常是独特的，体现出一个区块链系统在功能上有别于另一个区块链系统的部分。 与基础部分相比，这一部分差异比较大。Substrate作为一个区块链框架，认为所有的链都应该具备区块链系统的基础部分，而由开发者自由定制链的功能部分。

根据这样的划分，当开发者使用Substrate框架的时候，无需关心区块链基础功能也就是Core部分的工作，而只需关心自己链能够提供的功能，也就是Runtime部分的工作。 注意图中虚拟机EVM也是Runtime的一个组件。与以太坊的结构相比，相当于把以太坊的智能合约功能也能随意作为一个链的功能组件添加进入使用Substrate开发的链中。

<h2>Substrate的Runtime</h2>

Substrate的Runtime为了支持系统设计，做了一个特殊的设计。 Substrate的Runtime可以用同一份代码编译出两份可执行文件：
- 一份Rust的本地代码，我们一般称为native代码，native与其他代码无异，是这个执行文件中的二进制数据，直接运行。在Substrate的相关代码以native命名 
- 一份wasm的链上代码，我们一般成为wasm代码，wasm被部署到链上，所有人可获取，wasm通过构建一个wasm的运行时环境执行 。在Substrate的相关代码以wasm命名
  在节点启动的时候可以选择执行策略，使用native possible，wasm或者both。不同的执行策略会执行不同的执行文件。

由于这两份代码是由相同的代码编译出来的，所以其执行逻辑完全相同。其中wasm将会部署到链上，所有人都可以获取到，也就是说即使本地运行的不是最新版本的节点，只要同步了区块，一定可以获取到最新的wasm代码。
由于wasm代码的存在，可以保证即使节点没有更新到最新版本，仍然能够以最新的代码运行，保证不会因为代码的不同而分叉。同时在节点同步老数据的过程中也不会因为本地代码是最新的而导致同步出错。

<h2>2.Substrate的交易</h2>

在Substrate中的交易不再称为Transaction，而是称为了Extrinsic，中文翻译就是“外部的；外表的；外源性”，意味着被称为Extrinsic的概念，
对于区块链而言是外部的输入。这种定义脱离了本身“交易”的范畴（更像是转账的概念），而是在链的状态的角度下，认为交易及类似概念是一种改变状态的外部输入（意味着不止转账，只要是外部的操作都是）。

在Substrate中Extrinsic实际上具备极大的灵活性，能够允许开发者做出各种灵活的定制。 一个“外部输入”至少会具备以下两个条件：

发送者的证明
- 外部输入的行为
- 其中第一点是显然的，只要基于公私钥体系，就一定需要发送者对这个发送的内容进行签名，在链上通过这个签名验证合法性，解析出发送者的公钥识别身份。
  等价于互联网中客户端持有的token这类的。而第二点就是这个“输入”是到链上干嘛的，其相当于是用户发送到链上的指令行为。

这两点对应到Substrate的交易模板上即为 primitives/runtime/src/generic/unchecked_extrinsic.rs:L32：
```rust
/// A extrinsic right from the external world. This is unchecked and so
/// can contain a signature.
#[derive(PartialEq, Eq, Clone)]
pub struct UncheckedExtrinsic<Address, Call, Signature, Extra>
where
    Extra: SignedExtension
{
    /// The signature, address, number of extrinsics have come before from
    /// the same signer and an era describing the longevity of this transaction,
    /// if this is a signed extrinsic.
    pub signature: Option<(Address, Signature, Extra)>,  // 对应第一点
    /// The function that should be called.
    pub function: Call,  // 对应第二点
}
```

其中：

- signature: 就是发送者的身份标示与验证的信息
- function: 就是发送者的意图指令，类型为Call，用于调用链上的相应功能，例如转账transfer。
  这块即是一条链对外提供的功能，也是一条链的Runtime的入口组成部分。一个区块打包了所有的交易，执行区块的过程即是在Runtime中执行每一条交易的function的指令。

这个交易模板实现了trait primitives/runtime/src/traits.rs:L605：

```rust
`primitives/runtime/src/traits.rs:L605`：/// Something that acts like an `Extrinsic`.
pub trait Extrinsic: Sized {
    type Call;
    type SignaturePayload;
    fn is_signed(&self) -> Option<bool> { None }
    fn new(_call: Self::Call, _signed_data: Option<Self::SignaturePayload>) -> Option<Self> { None }
}
/// A "checkable" piece of information, used by the standard Substrate Executive in order to
/// check the validity of a piece of extrinsic information, usually by verifying the signature.
/// Implement for pieces of information that require some additional context `Context` in order to be
/// checked.
pub trait Checkable<Context>: Sized {
    type Checked;
    fn check(self, c: &Context) -> Result<Self::Checked, TransactionValidityError>;
}
```

<h2>3.状态管理</h2>
Substrate的状态管理与以太坊是一致的，都是基于MPT树的全历史世界状态模型。可以在任意一个区块上查看当前的所有世界状态。
在Runtime的接口层面，可以直接把MPT的trie树看成是一个Key/Value的数据库，支持get(key)->value；set(key, value) / remove(key)接口。

参考资料：

1、[Substrate 设计总览](https://zhuanlan.zhihu.com/p/56383616)

2、[Substrate 入门 - 具备状态的链](https://zhuanlan.zhihu.com/p/96866051)

3、[Substrate 入门 - 交易体](https://zhuanlan.zhihu.com/p/100770550)
