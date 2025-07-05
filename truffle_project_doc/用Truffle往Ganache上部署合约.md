# 用Truffle往Ganache上部署合约
## 部署合约
1. 进入项目根目录，执行`truffle init`，初始化一个Trffule项目。    
2. 修改truffle-config.js    
```json  
module.exports = {
  networks: {
    development: {
     host: "127.0.0.1",     
     port: 7545,            // 这里改成7545
     network_id: "*",      
    },
  },

  compilers: {
    solc: {
      version: "0.8.21",    
      docker: false,        // 这里改成false
      settings: {          
       optimizer: {
         enabled: false,
         runs: 200
       },
       evmVersion: "byzantium"
      }
    }
  },
};
```  
3. 编译合约。执行`truffle compile`将编译contracts目录下的所有合约，生成的json文件会放到build/contracts目录下。该json文件用于网络客户端和区块链服务器之间通信，JSON-RPC是一种应用层协议。 该json文件被称为应用二进制接口（Application Binary Interface, ABI），因为它 定义了应用程序调用智能合约的接口。ABI文件里包含了EVM可以解释执行的字节码(bytecode)。  
4. 在migrations目录下新建文件1_deploy_DataStore.js，内容如下：  
```json
var myContract = artifacts.require("DataStore");   // 把合约名称传给require

module.exports = function(developer){
    // 此处可以deploy多个智能合约
    developer.deploy(myContract);  //部署智能合约时会调用构造函数
}
```  
5. 部署合约。确保Ganache处于启动状态。`truffle migrate --reset`，reset强制重新部署所有合约，没有reset则已经部署过的不再部署。部署合约要花费一些gas。  