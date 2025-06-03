#### go-ethereum源码解读(一)：共识-工作量证明(PoW, Proof of Work)

go-ethereum 源码中工作量证明（PoW）的相关内容，包括其原理、代码实现、矿工收益获取方式以及交易执行与计算的时序。关键要点包括：

1. PoW 作用与目的：
   作为区块链最早的共识机制，保证区块链数据一致性，防止恶意攻击，如 double spending 攻击、服务滥用等。数据一致性包括账户余额、交易参数、交易顺序等。
2. PoW 原理：
   利用密码学哈希函数（以太坊常用 SHA-256 算法）生成区块哈希，矿工需找到小于或等于目标值的哈希，通过修改区块头 Nonce 值实现，找到后广播新区块，其他节点验证添加到区块链，矿工获交易消耗的 gas。
3. 代码实现：根据 CPU 核心数启动 goroutine 并发计算哈希，使用随机数作为 nonce 初始值。每个 goroutine 循环查找匹配 nonce，同时有计算当前区块难度值的相关代码。
4. 矿工收益：以太坊启动参数中通过 miner.etherbase 设置收益地址，创建新区块时设为 coinbase 地址，交易剩余 gas 作为奖励添加到该地址余额。
5. 交易执行与计算时序：区块执行完创建 task 到 Worker 的 taskCh 等待处理，taskLoop 中先对区块信息生成 Hash 缓存，再调用 Seal 方法计算 nonce 开始挖矿。

本文中工作量证明展示的代码为go-ethereum的1.11，仓库链接：https://github.com/ethereum/go-ethereum/tree/release/1.11

###### **原理**

工作量证明的主要作用是使用密码学哈希函数(以太坊中常用SHA-256算法计算哈希)生成**区块哈希**，矿工（以太坊中一个参与打包交易的以太坊节点）需要找到一个小于或等于**目标值**的哈希，这个目标值取决于**难度值**的大小。矿工通过不断修改区块头中的**Nonce值**，来改变计算得到的**结果值**大小。找到符合条件的哈希值后，矿工将新区块广播到网络，其他节点验证其有效性，并将其添加到区块链上。当新的区块被添加到区块链上，创建这个区块的矿工将会得到这个区块中所有交易消耗的gas，所以工作量证明的计算过程又被称为“挖矿”，参与计算哈希的以太坊节点被称为矿工。

工作量证明通过这种高难度的计算防止恶意节点攻击和服务的滥用。

而挖矿获得激励的机制，天然的鼓励更多的人参与构建去中心化的以太坊网络共识。

###### **工作量证明代码实现**

以太坊会根据CPU核心数启动对应数量的goroutine(Go语言中，需要并行执行的计算，需要创建一个goroutine)并发计算哈希，并且会使用随机数作为nonce的初始值，增加计算出哈希的效率：
代码源文件go-ethereum/consensus/ethash/sealer.go：

```
// Seal implements consensus.Engine, attempting to find a nonce that satisfies
// the block's difficulty requirements.
func (ethash *Ethash) Seal(chain consensus.ChainHeaderReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error {
	// If we're running a fake PoW, simply return a 0 nonce immediately
	if ethash.config.PowMode == ModeFake || ethash.config.PowMode == ModeFullFake {
		header := block.Header()
		header.Nonce, header.MixDigest = types.BlockNonce{}, common.Hash{}
		select {
		case results <- block.WithSeal(header):
		default:
			ethash.config.Log.Warn("Sealing result is not read by miner", "mode", "fake", "sealhash", ethash.SealHash(block.Header()))
		}
		return nil
	}
	// If we're running a shared PoW, delegate sealing to it
	if ethash.shared != nil {
		return ethash.shared.Seal(chain, block, results, stop)
	}
	// Create a runner and the multiple search threads it directs
	abort := make(chan struct{})

	ethash.lock.Lock()
	threads := ethash.threads
	if ethash.rand == nil {
		seed, err := crand.Int(crand.Reader, big.NewInt(math.MaxInt64))
		if err != nil {
			ethash.lock.Unlock()
			return err
		}
		ethash.rand = rand.New(rand.NewSource(seed.Int64()))
	}
	ethash.lock.Unlock()
	if threads == 0 {
		threads = runtime.NumCPU()
	}
	if threads < 0 {
		threads = 0 // Allows disabling local mining without extra logic around local/remote
	}
	// Push new work to remote sealer
	if ethash.remote != nil {
		ethash.remote.workCh <- &sealTask{block: block, results: results}
	}
	var (
		pend   sync.WaitGroup
		locals = make(chan *types.Block)
	)
	for i := 0; i < threads; i++ {
		pend.Add(1)
		go func(id int, nonce uint64) {
			defer pend.Done()
			ethash.mine(block, id, nonce, abort, locals)
		}(i, uint64(ethash.rand.Int63()))
	}
	// Wait until sealing is terminated or a nonce is found
	go func() {
		var result *types.Block
		select {
		case <-stop:
			// Outside abort, stop all miner threads
			close(abort)
		case result = <-locals:
			// One of the threads found a block, abort all others
			select {
			case results <- result:
			default:
				ethash.config.Log.Warn("Sealing result is not read by miner", "mode", "local", "sealhash", ethash.SealHash(block.Header()))
			}
			close(abort)
		case <-ethash.update:
			// Thread count was changed on user request, restart
			close(abort)
			if err := ethash.Seal(chain, block, results, stop); err != nil {
				ethash.config.Log.Error("Failed to restart sealing after update", "err", err)
			}
		}
		// Wait for all miners to terminate and return the block
		pend.Wait()
	}()
	return nil
}
```

每个goroutine都会循环查找匹配要求的nonce：

```
计算当前区块的难度值：
代码源文件：go-ethereum/consensus/ethash/consensus.go// mine is the actual proof-of-work miner that searches for a nonce starting from
// seed that results in correct final block difficulty.
func (ethash *Ethash) mine(block *types.Block, id int, seed uint64, abort chan struct{}, found chan *types.Block) {
	// Extract some data from the header
	var (
		header  = block.Header()
		hash    = ethash.SealHash(header).Bytes()
		target  = new(big.Int).Div(two256, header.Difficulty)
		number  = header.Number.Uint64()
		dataset = ethash.dataset(number, false)
	)
	// Start generating random nonces until we abort or find a good one
	var (
		attempts  = int64(0)
		nonce     = seed  // seed是个随机数，会作为nonce的初始值来计算结果值
		powBuffer = new(big.Int)
	)
	logger := ethash.config.Log.New("miner", id)
	logger.Trace("Started ethash search for new nonces", "seed", seed)
search:
	for {
		select {
		case <-abort:
			// Mining terminated, update stats and abort
			logger.Trace("Ethash nonce search aborted", "attempts", nonce-seed)
			ethash.hashrate.Mark(attempts)
			break search

		default:
			// We don't have to update hash rate on every nonce, so update after after 2^X nonces
			attempts++
			if (attempts % (1 << 15)) == 0 {
				ethash.hashrate.Mark(attempts)
				attempts = 0
			}
			// Compute the PoW value of this nonce
			digest, result := hashimotoFull(dataset.dataset, hash, nonce)
			if powBuffer.SetBytes(result).Cmp(target) <= 0 {
				// Correct nonce found, create a new header with it
				header = types.CopyHeader(header)
				header.Nonce = types.EncodeNonce(nonce)
				header.MixDigest = common.BytesToHash(digest)

				// Seal and return a block (if still needed)
				select {
				case found <- block.WithSeal(header):
					logger.Trace("Ethash nonce found and reported", "attempts", nonce-seed, "nonce", nonce)
				case <-abort:
					logger.Trace("Ethash nonce found but discarded", "attempts", nonce-seed, "nonce", nonce)
				}
				break search
			}
			nonce++
		}
	}
	// Datasets are unmapped in a finalizer. Ensure that the dataset stays live
	// during sealing so it's not unmapped while being read.
	runtime.KeepAlive(dataset)
}
```

计算当前区块的难度值：

代码源文件：go-ethereum/consensus/ethash/consensus.go

```
// Ethash proof-of-work protocol constants.
var (
	FrontierBlockReward           = big.NewInt(5e+18) // Block reward in wei for successfully mining a block
	ByzantiumBlockReward          = big.NewInt(3e+18) // Block reward in wei for successfully mining a block upward from Byzantium
	ConstantinopleBlockReward     = big.NewInt(2e+18) // Block reward in wei for successfully mining a block upward from Constantinople
	maxUncles                     = 2                 // Maximum number of uncles allowed in a single block
	allowedFutureBlockTimeSeconds = int64(15)         // Max seconds from current time allowed for blocks, before they're considered future blocks

	// calcDifficultyEip5133 is the difficulty adjustment algorithm as specified by EIP 5133.
	// It offsets the bomb a total of 11.4M blocks.
	// Specification EIP-5133: https://eips.ethereum.org/EIPS/eip-5133
	calcDifficultyEip5133 = makeDifficultyCalculator(big.NewInt(11_400_000))

	// calcDifficultyEip4345 is the difficulty adjustment algorithm as specified by EIP 4345.
	// It offsets the bomb a total of 10.7M blocks.
	// Specification EIP-4345: https://eips.ethereum.org/EIPS/eip-4345
	calcDifficultyEip4345 = makeDifficultyCalculator(big.NewInt(10_700_000))

	// calcDifficultyEip3554 is the difficulty adjustment algorithm as specified by EIP 3554.
	// It offsets the bomb a total of 9.7M blocks.
	// Specification EIP-3554: https://eips.ethereum.org/EIPS/eip-3554
	calcDifficultyEip3554 = makeDifficultyCalculator(big.NewInt(9700000))

	// calcDifficultyEip2384 is the difficulty adjustment algorithm as specified by EIP 2384.
	// It offsets the bomb 4M blocks from Constantinople, so in total 9M blocks.
	// Specification EIP-2384: https://eips.ethereum.org/EIPS/eip-2384
	calcDifficultyEip2384 = makeDifficultyCalculator(big.NewInt(9000000))

	// calcDifficultyConstantinople is the difficulty adjustment algorithm for Constantinople.
	// It returns the difficulty that a new block should have when created at time given the
	// parent block's time and difficulty. The calculation uses the Byzantium rules, but with
	// bomb offset 5M.
	// Specification EIP-1234: https://eips.ethereum.org/EIPS/eip-1234
	calcDifficultyConstantinople = makeDifficultyCalculator(big.NewInt(5000000))

	// calcDifficultyByzantium is the difficulty adjustment algorithm. It returns
	// the difficulty that a new block should have when created at time given the
	// parent block's time and difficulty. The calculation uses the Byzantium rules.
	// Specification EIP-649: https://eips.ethereum.org/EIPS/eip-649
	calcDifficultyByzantium = makeDifficultyCalculator(big.NewInt(3000000))
)
```

```
// CalcDifficulty is the difficulty adjustment algorithm. It returns
// the difficulty that a new block should have when created at time
// given the parent block's time and difficulty.
func CalcDifficulty(config *params.ChainConfig, time uint64, parent *types.Header) *big.Int {
	next := new(big.Int).Add(parent.Number, big1)
	switch {
	case config.IsGrayGlacier(next):
		return calcDifficultyEip5133(time, parent)
	case config.IsArrowGlacier(next):
		return calcDifficultyEip4345(time, parent)
	case config.IsLondon(next):
		return calcDifficultyEip3554(time, parent)
	case config.IsMuirGlacier(next):
		return calcDifficultyEip2384(time, parent)
	case config.IsConstantinople(next):
		return calcDifficultyConstantinople(time, parent)
	case config.IsByzantium(next):
		return calcDifficultyByzantium(time, parent)
	case config.IsHomestead(next):
		return calcDifficultyHomestead(time, parent)
	default:
		return calcDifficultyFrontier(time, parent)
	}
}
```

target=2**256/difficulty

nonce : nonce <=target

###### 工作量证明，矿工如何获得收益？

在以太坊启动的参数中有一个专门设置收益地址的参数，叫做`miner.etherbase`。

当本地以太坊执行层节点创建一个新区块时，会把这个参数设置为coinbase地址，在执行交易时把交易中剩余的gas作为奖励，直接添加到coinbase地址余额：

打包区块前，worker会把当前的coinbase赋值给generateParams，而generateParams中的coinbase属性会被新的区块头读取到：

代码源文件：go-ethereum/miner/worker.go

```
// commitWork generates several new sealing tasks based on the parent block
// and submit them to the sealer.
func (w *worker) commitWork(interrupt *atomic.Int32, noempty bool, timestamp int64) {
	start := time.Now()

	// Set the coinbase if the worker is running or it's required
	var coinbase common.Address
	if w.isRunning() {
		coinbase = w.etherbase()
		if coinbase == (common.Address{}) {
			log.Error("Refusing to mine without etherbase")
			return
		}
	}
	work, err := w.prepareWork(&generateParams{
		timestamp: uint64(timestamp),
		coinbase:  coinbase,
	})
	if err != nil {
		return
	}
	// Create an empty block based on temporary copied state for
	// sealing in advance without waiting block execution finished.
	if !noempty && !w.noempty.Load() {
		w.commit(work.copy(), nil, false, start)
	}
	// Fill pending transactions from the txpool into the block.
	err = w.fillTransactions(interrupt, work)
	switch {
	case err == nil:
		// The entire block is filled, decrease resubmit interval in case
		// of current interval is larger than the user-specified one.
		w.resubmitAdjustCh <- &intervalAdjust{inc: false}

	case errors.Is(err, errBlockInterruptedByRecommit):
		// Notify resubmit loop to increase resubmitting interval if the
		// interruption is due to frequent commits.
		gaslimit := work.header.GasLimit
		ratio := float64(gaslimit-work.gasPool.Gas()) / float64(gaslimit)
		if ratio < 0.1 {
			ratio = 0.1
		}
		w.resubmitAdjustCh <- &intervalAdjust{
			ratio: ratio,
			inc:   true,
		}

	case errors.Is(err, errBlockInterruptedByNewHead):
		// If the block building is interrupted by newhead event, discard it
		// totally. Committing the interrupted block introduces unnecessary
		// delay, and possibly causes miner to mine on the previous head,
		// which could result in higher uncle rate.
		work.discard()
		return
	}
	// Submit the generated block for consensus sealing.
	w.commit(work.copy(), w.fullTaskHook, true, start)

	// Swap out the old work with the new one, terminating any leftover
	// prefetcher processes in the mean time and starting a new one.
	if w.current != nil {
		w.current.discard()
	}
	w.current = work
}
```

在真正执行交易之前，会先取worker中预设的coinbase地址。

并赋值给新创建的generateParams，执行prepareWork方法时，新创建的Header会使用这个generateParams中的coinbase地址，作为Header的coinbase地址。

执行交易时，会在引用Header中的coinbase地址。

交易执行时，给coinbase发放奖励：

代码源文件：go-ethereum/core/state\_transition.go

```
func (st *StateTransition) TransitionDb() (*ExecutionResult, error) {
// First check this message satisfies all consensus rules before
// applying the message. The rules include these clauses
//
// 1. the nonce of the message caller is correct
// 2. caller has enough balance to cover transaction fee(gaslimit * gasprice)
// 3. the amount of gas required is available in the block
// 4. the purchased gas is enough to cover intrinsic usage
// 5. there is no overflow when calculating intrinsic gas
// 6. caller has enough balance to cover asset transfer for **topmost** call// Check clauses 1-3, buy gas if everything is correct
if err := st.preCheck(); err != nil {
	return nil, err
}

if tracer := st.evm.Config.Tracer; tracer != nil {
	tracer.CaptureTxStart(st.initialGas)
	defer func() {
		tracer.CaptureTxEnd(st.gasRemaining)
	}()
}

var (
	msg              = st.msg
	sender           = vm.AccountRef(msg.From)
	rules            = st.evm.ChainConfig().Rules(st.evm.Context.BlockNumber, st.evm.Context.Random != nil, st.evm.Context.Time)
	contractCreation = msg.To == nil
)

// Check clauses 4-5, subtract intrinsic gas if everything is correct
gas, err := IntrinsicGas(msg.Data, msg.AccessList, contractCreation, rules.IsHomestead, rules.IsIstanbul, rules.IsShanghai)
if err != nil {
	return nil, err
}
if st.gasRemaining < gas {
	return nil, fmt.Errorf("%w: have %d, want %d", ErrIntrinsicGas, st.gasRemaining, gas)
}
st.gasRemaining -= gas

// Check clause 6
if msg.Value.Sign() > 0 && !st.evm.Context.CanTransfer(st.state, msg.From, msg.Value) {
	return nil, fmt.Errorf("%w: address %v", ErrInsufficientFundsForTransfer, msg.From.Hex())
}

// Check whether the init code size has been exceeded.
if rules.IsShanghai && contractCreation && len(msg.Data) > params.MaxInitCodeSize {
	return nil, fmt.Errorf("%w: code size %v limit %v", ErrMaxInitCodeSizeExceeded, len(msg.Data), params.MaxInitCodeSize)
}

// Execute the preparatory steps for state transition which includes:
// - prepare accessList(post-berlin)
// - reset transient storage(eip 1153)
st.state.Prepare(rules, msg.From, st.evm.Context.Coinbase, msg.To, vm.ActivePrecompiles(rules), msg.AccessList)

var (
	ret   []byte
	vmerr error // vm errors do not effect consensus and are therefore not assigned to err
)
if contractCreation {
	ret, _, st.gasRemaining, vmerr = st.evm.Create(sender, msg.Data, st.gasRemaining, msg.Value)
} else {
	// Increment the nonce for the next transaction
	st.state.SetNonce(msg.From, st.state.GetNonce(sender.Address())+1)
	ret, st.gasRemaining, vmerr = st.evm.Call(sender, st.to(), msg.Data, st.gasRemaining, msg.Value)
}

if !rules.IsLondon {
	// Before EIP-3529: refunds were capped to gasUsed / 2
	st.refundGas(params.RefundQuotient)
} else {
	// After EIP-3529: refunds are capped to gasUsed / 5
	st.refundGas(params.RefundQuotientEIP3529)
}
effectiveTip := msg.GasPrice
if rules.IsLondon {
	effectiveTip = cmath.BigMin(msg.GasTipCap, new(big.Int).Sub(msg.GasFeeCap, st.evm.Context.BaseFee))
}

if st.evm.Config.NoBaseFee && msg.GasFeeCap.Sign() == 0 && msg.GasTipCap.Sign() == 0 {
	// Skip fee payment when NoBaseFee is set and the fee fields
	// are 0. This avoids a negative effectiveTip being applied to
	// the coinbase when simulating calls.
} else {
	fee := new(big.Int).SetUint64(st.gasUsed())
	fee.Mul(fee, effectiveTip)
	st.state.AddBalance(st.evm.Context.Coinbase, fee)
}

return &ExecutionResult{
	UsedGas:    st.gasUsed(),
	Err:        vmerr,
	ReturnData: ret,
}, nil
// Check clauses 1-3, buy gas if everything is correct
if err := st.preCheck(); err != nil {
	return nil, err
}

if tracer := st.evm.Config.Tracer; tracer != nil {
	tracer.CaptureTxStart(st.initialGas)
	defer func() {
		tracer.CaptureTxEnd(st.gasRemaining)
	}()
}

var (
	msg              = st.msg
	sender           = vm.AccountRef(msg.From)
	rules            = st.evm.ChainConfig().Rules(st.evm.Context.BlockNumber, st.evm.Context.Random != nil, st.evm.Context.Time)
	contractCreation = msg.To == nil
)

// Check clauses 4-5, subtract intrinsic gas if everything is correct
gas, err := IntrinsicGas(msg.Data, msg.AccessList, contractCreation, rules.IsHomestead, rules.IsIstanbul, rules.IsShanghai)
if err != nil {
	return nil, err
}
if st.gasRemaining < gas {
	return nil, fmt.Errorf("%w: have %d, want %d", ErrIntrinsicGas, st.gasRemaining, gas)
}
st.gasRemaining -= gas

// Check clause 6
if msg.Value.Sign() > 0 && !st.evm.Context.CanTransfer(st.state, msg.From, msg.Value) {
	return nil, fmt.Errorf("%w: address %v", ErrInsufficientFundsForTransfer, msg.From.Hex())
}

// Check whether the init code size has been exceeded.
if rules.IsShanghai && contractCreation && len(msg.Data) > params.MaxInitCodeSize {
	return nil, fmt.Errorf("%w: code size %v limit %v", ErrMaxInitCodeSizeExceeded, len(msg.Data), params.MaxInitCodeSize)
}

// Execute the preparatory steps for state transition which includes:
// - prepare accessList(post-berlin)
// - reset transient storage(eip 1153)
st.state.Prepare(rules, msg.From, st.evm.Context.Coinbase, msg.To, vm.ActivePrecompiles(rules), msg.AccessList)

var (
	ret   []byte
	vmerr error // vm errors do not effect consensus and are therefore not assigned to err
)
if contractCreation {
	ret, _, st.gasRemaining, vmerr = st.evm.Create(sender, msg.Data, st.gasRemaining, msg.Value)
} else {
	// Increment the nonce for the next transaction
	st.state.SetNonce(msg.From, st.state.GetNonce(sender.Address())+1)
	ret, st.gasRemaining, vmerr = st.evm.Call(sender, st.to(), msg.Data, st.gasRemaining, msg.Value)
}

if !rules.IsLondon {
	// Before EIP-3529: refunds were capped to gasUsed / 2
	st.refundGas(params.RefundQuotient)
} else {
	// After EIP-3529: refunds are capped to gasUsed / 5
	st.refundGas(params.RefundQuotientEIP3529)
}
effectiveTip := msg.GasPrice
if rules.IsLondon {
	effectiveTip = cmath.BigMin(msg.GasTipCap, new(big.Int).Sub(msg.GasFeeCap, st.evm.Context.BaseFee))
}

if st.evm.Config.NoBaseFee && msg.GasFeeCap.Sign() == 0 && msg.GasTipCap.Sign() == 0 {
	// Skip fee payment when NoBaseFee is set and the fee fields
	// are 0. This avoids a negative effectiveTip being applied to
	// the coinbase when simulating calls.
} else {
	fee := new(big.Int).SetUint64(st.gasUsed())
	fee.Mul(fee, effectiveTip)
	st.state.AddBalance(st.evm.Context.Coinbase, fee)
}

return &ExecutionResult{
	UsedGas:    st.gasUsed(),
	Err:        vmerr,
	ReturnData: ret,
}, nil
}
```

执行交易之后，交易所消耗的Gas都会先乘以一个GasPrice系数，然后添加到Header的Coinbase地址的余额。

注：交易在执行st.evm.Call方法之后，即使交易执行失败，gas也必定会被消耗。
