# Accounts&Permissions

账户是人类可读的，存在区块链中的标识符。每一次transaction都根据account配置的authority对自身的permission进行评估。每个命名的permission都有一个阈值，一个transaction的签署authority满足该阈值时才是合法的。Transaction通过使用一个加载了解锁钱包（wallet）的客户端进行签署。Wallet是一个保护、使用你的密钥（keys）的软件，这些keys可能被授予一个acconut在blockchain上的authority的permission，也可能不会被授予。

## Wallets

Wallets 是存储着 keys 的客户端，这些keys可能与一个或多个accounts的permission相关。理想情况下，钱包具有受高熵密码保护的锁定（加密）和解锁（解密）状态。 EOSIO / eos存储库捆绑了一个名为cleos的命令行界面客户端，它与一个名为keosd的lite客户端进行交互，并且共同展示了这种模式。

## Accounts

帐户（Accounts）是存储在区块链中的人类可读的名称。它可以由个人或一组个人拥有，具体取决于权限配置。需要account才能将交易转移或以其他方式推送到区块链。

## Authorities and Permissions

Authorities 决定一个action是否合法。

每个账户都具有两个native named permission。

- owner authority：所有者权限象征着账户的所有权，只有少数交易需要该authority，但是需要注意对owner authority的任何修改。一般而言，建议owner authority使用冷存储并且不与任何人共享。owner authority可用于恢复可能泄漏的其他permission。
- active authority：主动权限用于转移资金，投票给生产者并进行其他高级账户更改。

除了native permission以外，一个account可以拥有 custom named permission（自定义权限）用于进一步扩展帐户管理。Custom permission非常灵活，并且在实现时可以对应很多实例。这很大程度上取决于开发人员如何使用他们以及采取什么约定方式。

对于任何给定authority的permission可以分配给一个或多个public keys或有效的account name。

## Putting it all Together

 以下是所有上述概念的组合以及它们如何实际应用的一些简单的例子。

### Default Account Configuration (Single-Sig)

这是帐户在创建后的配置方式，它拥有owner authority和active authority两个keys，这两个keys的权重均为1，权限阈值均为1。默认配置需要一个签名（signature）来授权（authorize）针对native permission的action。

#### **@bob account authorities**

| Permission | Account                                               | Weight | Threshold |
| ---------- | ----------------------------------------------------- | ------ | --------- |
| owner      |                                                       |        | 1         |
|            | EOS5EzTZZQQxdrDaJAPD9pDzGJZ5bj34HaAb8yuvjFHGWzqV25Dch | 1      |           |
| active     |                                                       |        | 1         |
|            | EOS61chK8GbH4ukWcbom8HgK95AeUfP8MBPn7XRq8FeMBYYTgwmcX | 1      |           |

在上述@bob account  authorities的例子中，以上的表格表示：bob账户的owner key的permission权重（weigths）是1，并且，想要在该authority下push一个transaction（到blockchain上）需要的threshold是1。

想要在owner authority下push一个transaction，只需要@bob（的account）用自己的owner key签名（signature），就可以使该交易合法。这个key会被存储在一个wallet内，通过cleos调用。

### Multi-sig Account & Custom Permissions

多账户与自定义权限

下面的例子是名为@multisig虚拟账户（account）的authorities。在该情形下，两个accounts被授权为@multisig account的 owner 和 active permission。三个accounts被授予权重不同的自定义权限（custom permission），publish。

#### **@multisig account authorities**

| Permission | Account                                | Weight | Threshold |
| ---------- | -------------------------------------- | ------ | --------- |
| owner      |                                        |        | 2         |
|            | @bob                                   | 1      |           |
|            | @stacy                                 | 1      |           |
| active     |                                        |        | 1         |
|            | @bob                                   | 1      |           |
|            | @stacy                                 | 1      |           |
| publish    |                                        |        | 2         |
|            | @bob                                   | 2      |           |
|            | @stacy                                 | 2      |           |
|            | EOS7Hnv4iBWo1pcEpP8JyFYCJLRUzYcXSqt... | 1      |           |

在这种情况下，需要权重阈值2来更改所有者权限级别（owner autority）。由于所有方都具有权重1，因此所有用户都必须签署，交易才能获得完全授权。

要push需要active permission的transaction，阈值设置为1。这意味着只需要1个签名即可授权来自active permission的操作。

还有一个名为publish的自定义命名权限（custom named permission）。这个示例下，publish permission用于使用博客dApp将帖子发布到@ multisig的博客。发布权限的阈值为2，@ bob和@stacy的权重均为2，公钥的权重为1。这意味着@bob和@stacy可以在没有额外签名的情况下发布，而公众密钥（public key）需要额外签名（来自@bob或者@stacy）才能在公共许可下进行授权操作。

因此，上述权限表意味着作为帐户所有者的@ bob和@stacy提升了与主持人或编辑者类似的权限。尽管这个原始示例在可扩展性方面有特别的限制，并不一定是一个好的设计，但它充分证明了EOSIO权限系统的灵活性。

此外，请注意上表中的permission是使用account name和key设置的。乍一看这可能看起来微不足道，但它确实提出了一些额外的灵活性。

**Observations**

- @ bob和@stacy可以明确标识为此帐户的所有者
- public key 不能在没有来自@bob或@stacy的signatue的情况下以publish permission push一个transaction。
- @bob 和 @stacy 不需要任何额外的signature就可以在publish permission下 push 一个transaction。