# 开发web3.0应用（dapp）
1. 在项目根目录下新建app目录，进入app目录，初始化go项目`go mod init storage`  
2. 安装abigen  
```Shell
go get -u github.com/ethereum/go-ethereum
cd $GOPATH/pkg/mod/github.com/ethereum/go-ethereum@vx.x.x/cmd/abigen    vx.x.x以实际安装的最新版本为准
go build  
```
把生成的abigen可执行文件（在Windows上叫abigen.exe）放到系统的$PATH目录下。    
查看abigen的版本 `abigen --version`     
3. 在vscode里安装插件solidity，发布者Juan Blanco。在合约代码上右键，点击Solidity: Compile Contract，在bin/contracts目录下会生成.abi、.bin和.json文件。    
4. 生成abi对应的go文件  
```Shell
abigen --abi=./bin/contracts/DataStore.abi --bin=./bin/contracts/DataStore.bin --pkg=main --out=./data_store.go
abigen --abi=./bin/contracts/Ballot.abi --bin=./bin/contracts/Ballot.bin --pkg=ballot --out=./bin/ballot/ballot.go
go mod tidy
```
--pkg指定go代码的package名称，  
5. 编写go代码storage.go，把合约函数转成go函数。  
6. 单元测试。storage_test.go  
