# ChainID和NetworkID 

网络ID(NetworkID)，主要用来在网络层标识当前的区块链网络。NetworkId 不一致的两个节点无法建立连接。  

区块链可能会出现分叉，通过在签名信息中加入Chain ID， 避免一个交易在签名之后被重复在不同的链上提交。  

Mainnet : 以太坊主网络
network_id: 1
chain_id: 1

Ropsten : 以太坊测试网络 (PoW)
network_id: 3
chain_id: 3

Rinkeby : 以太坊测试网络 (PoA)
network_id: 4
chain_id: 4

Goerli : 以太坊测试网络 (PoA)
network_id: 5
chain_id: 5

Kovan : 以太坊测试网络 (PoA)
network_id: 42
chain_id: 42

Ethereum Classic Mainnet : 以太坊经典主网络
network_id: 1
chain_id: 61

Morden : 以太坊经典测试网络
network_id: 2
chain_id: 62