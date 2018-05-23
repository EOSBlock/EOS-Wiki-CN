| 分类术语                  | 同义词     | Block.one 定义                             |
| --------------------- | ------- | ---------------------------------------- |
| Account               |         | 一个由 本机许可 和/或 自定义权限 组成的 链上标识符，分配有一个一个或多个 keys 或 accounts |
| Authority             |         | 权限的抽象，表示权限实际上如何组合，并对应到一个或一群人。            |
| Block                 | Blk     | 区块链的 可确认单位。一个Block有一个或多个Transaction，以及与先前所有块的链接。<br />当一个区块变得“不可逆转地确认”时，这是因为绝大多数区块生产者已经互相同意并确认区块包含正确的Transaction。<br />一旦Block得到不可逆转的确认，它就成为不可变Blockchain的永久部分。 |
| DAC                   |         | 去中心化自治集体、去中心化自治公司。<br /> Decentralized Autonomous Collective, or Decentralized Autonomous Corporation. |
| DAO                   |         | Decentralized Autonomous Organization.<br />去中心化自治公司。 |
| Deferred  Transaction | defTx   | 一个由智能合约（Smart Contract）创建的Transaction，以便在之后的某个特定时间点执行。 |
| DLTs                  |         | Distributed Ledger Technologies.<br />分布式分类账户技术，一个分布式的分类账户（ledger）是指在地理上分布于多个站点、国家或机构，并且被复制、共享和同步的数据共识。<br />https://en.wikipedia.org/wiki/Distributed_ledger |
| DPoS                  |         | Delegated Proof of Stake. 授权证明。<br />共识算法集合之一。 |
| Key pair              | keys    | 公钥和私钥                                    |
| larimer               |         | 0.0001 EOS                               |
| Master Password       |         | 用来解锁一个钱包文件的密码                            |
| Action                |         | 对Blockchain的一个改动。一个Transaction由一个或多个Action组成。 |
| Non-Producing Node    |         | 一个运行nodeos的完整节点，只监视和验证每个块，并维护自己本地完整的区块链副本。 处于“备用池”中的非生产节点可以通过投票进程成为生产节点。 生产节点如果被票出，将成为非生产节点。 大多数非生产节点不在“待机池”中。 |
| Oracle                |         | 在区块链和智能合约的背景下，Oracle是一个代理，用于查找并验证真实世界的事件并将此信息提交给智能合约使用的区块链。 |
| peer-2-peer           | p2p     | 点对点计算或联网是一种分布式应用程序体系结构，可在同级之间分配任务或工作负载。 同级的点同样享有特权，在应用程序中等同参与者。 它们形成了一个点对点的节点网络。 |
| Permission            |         | 加权安全机制，通过评估其签名权限来确定消息是否得到了正确授权。          |
| Private Key           |         | A secret key used to sign transactions   |
| Public Key            | pub key | A publicly available key that is transmitted alongside a transaction. |
| Scope                 |         | 范围，是Contract中的一个数据区域。Contracts只能对自己的数据区域写入，但是可以读取人意Contract的数据区域。合适的Scoping允许Contracts并行运行，只要他们不写入相同的区域。Scope不与帐户名称（Account Name）相混淆，但为了方便起见，Contract可以使用相同的值。 |
| Smart Contract        |         | 智能合约是旨在促进，验证或执行谈判或履行Contract的计算机协议。      |
| Standby Pool          |         | 一组大致100个完整节点，等待被选为区块生成者（这些节点都有能力成为区块生成者）。一旦主链需要用一个新的Block Producer来替换现有的，新的Block Producer就会从Standby Pool中挑选出来。 |
| Transaction           | Tx,Txn  | 对区块链的一个 完全的 不全则无 的修改。 一个或多个消息的组合。 通常，由智能合约执行。 |
| Wallet                |         | 由客户端（例如cleos）生成和/或管理的加密文件，用于管理私钥并便于以安全方式签署事务。 钱包可能处于锁定或解锁状态。 |
| Block Producer        | BP      | 目前被轮到生产“当前”区块的节点，或者是该组中的成员之一。            |