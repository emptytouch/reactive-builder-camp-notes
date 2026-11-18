# D3｜第一个 Reactive 合约：订阅 Sepolia 的代币转账（中文精讲）

> 今日平台任务：**第一次部署：监控 Sepolia 上的代币转账（+80 分，核心任务，剩余 18 天）**
> 官方原文：<https://dev.reactive.network/reactive-contracts>
> 本地模板：`D:/Web3/reactive-builder-camp/starter`（官方仓库的 clone，**不要动它作为 upstream 的 origin**）

目标是拿到三笔真实链上交易，这是黑客松 Execution 维度最硬的证据：

```
① Sepolia：ERC-20 转账 emit Transfer(from, to, value)
        ↓ 订阅匹配
② Lasna：CampReactive.react(log) 判断金额 ≥ 阈值，请求回调
        ↓ Callback Proxy 投递
③ Sepolia：CampCallback.onEvent(...) emit Handled  ← 这笔哈希就是证据
```

---

## 一、先背术语（英文文档看不懂就看这张）

| 英文 | 中文 | 一句话 |
|---|---|---|
| subscription | 订阅 | 六项条件：chainId + 合约地址 + 4 个 topic |
| `REACTIVE_IGNORE` | 忽略该位 | 这个 topic 位置不限制 |
| `opCode` | topic 数量 | LOG3 = 3，用来判断该不该解码 |
| `LogRecord` | 日志结构 | 系统塞给 `react()` 的那条日志 |
| `data` | 数据段 | 只有**非 indexed** 参数的 ABI 编码 |
| `indexed` | 索引参数 | 被放进 topic1/2/3，不在 data 里 |
| wildcard | 通配 | `chainId=0` 或 `contract=address(0)` 表示任意 |
| `onlySystem` | 仅系统可调用 | 防止别人伪造日志调你的 `react()` |

---

## 二、订阅的六个条件（今天最重要的一段）

```solidity
SYSTEM.subscribe(
    chainId_,     // uint256，0 = 任意链
    contract_,    // address，address(0) = 该链任意合约
    topic0_,      // 事件签名 hash
    topic1_,      // indexed 参数 1；REACTIVE_IGNORE = 任意
    topic2_,      // indexed 参数 2
    topic3_       // indexed 参数 3
);
```

四条必须记住的规则：

1. **只支持精确相等**，没有大于、小于、范围、位运算过滤。「金额大于 X」只能在 `react()` 里判断。
2. **单个订阅内部没有 OR**。要听多个合约或多个事件，就调用多次 `subscribe()`。
3. **至少一项要具体**，不能「所有链 + 所有合约 + 所有事件」全通配。
4. **订阅存在节点里，不在合约 storage 里**。所以：① 你在链上读不到自己的订阅列表，得自己在合约里记；② `unsubscribe()` 必须六个参数**完全一致**，否则静默失败，老订阅还在继续扣钱。

### ERC-20 Transfer 的订阅参数（本次任务直接用）

```solidity
uint256 TRANSFER_TOPIC0 = uint256(keccak256("Transfer(address,address,uint256)"));
// 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef

SYSTEM.subscribe(
    11155111,          // Sepolia
    tokenAddress,      // 你监控的 ERC-20 合约
    TRANSFER_TOPIC0,
    REACTIVE_IGNORE,   // topic1 = from，先不限
    REACTIVE_IGNORE,   // topic2 = to
    REACTIVE_IGNORE    // topic3
);
```

> 想只听「某个地址转出」，就把 `topic1_` 填成 `uint256(uint160(fromAddress))`——它是 indexed 参数，可以做精确匹配。

---

## 三、`react()` 收到的 `LogRecord` 长什么样

```solidity
struct LogRecord {
    uint256 chainId;
    address contractAddress;
    uint256 topic0;
    uint256 topic1;
    uint256 topic2;
    uint256 topic3;
    bytes   data;
    uint256 blockNumber;
    uint256 opCode;      // ← 原始日志的 topic 数量，LOG3 = 3
    uint256 blockHash;   // 注意是 uint256，不是 bytes32
    uint256 txHash;      // 同上
    uint256 logIndex;
}
```

ERC-20 `Transfer(address indexed from, address indexed to, uint256 value)` 的日志：

| 字段 | 内容 |
|---|---|
| `topic0` | `keccak256("Transfer(address,address,uint256)")` |
| `topic1` | `from`（补足 32 字节的地址） |
| `topic2` | `to` |
| `topic3` | 0（这条事件只有 3 个 topic） |
| `opCode` | **3** |
| `data` | 只有 `value` 的 ABI 编码，32 字节 |

所以解码只需要一行：

```solidity
function react(LogRecord calldata log_) external onlySystem {
    if (log_.topic0 != TRANSFER_TOPIC0 || log_.opCode != 3) return; // 不做事用 return
    uint256 amount = abi.decode(log_.data, (uint256));
    if (amount < threshold) return;
    // ……请求回调
}
```

三个坑：

- **`blockHash` / `txHash` 是 `uint256`**，跟 `bytes32` 比较要先 cast。
- **不要用 `revert` 过滤不匹配的事件**。系统合约会捕获 revert、发 `ReactiveContractReverted`，然后**照样收费**（D2 学过）。用 `return`。
- topic 位置为 0 不代表「没有这个 topic」——ERC-20 mint 的 `from` 就是 `address(0)`。判断 topic 数量看 `opCode`。

### `onlySystem` 为什么必须有

没有它，任何人都能手工构造一条假 `LogRecord` 调你的 `react()`，伪造源链事件、骗出回调。加上后 `msg.sender` 必须是 `SYSTEM`（`0x8888…8888`），否则 `NotAuthorized`。

---

## 四、请求回调：payload 第一个参数必须是占位 address

```solidity
bytes memory payload = abi.encodeWithSignature(
    "onEvent(address,uint256,uint256,uint256)",
    address(0),        // ← 占位！网络会把它替换成你的 Reactive 合约地址
    log_.chainId,
    log_.txHash,
    amount
);

SYSTEM.requestCallbackV_1_0(ISystemContract.CallbackConfiguration_V_1_0({
    chainId:  DEST_CHAIN_ID,
    recipient: recipient,
    gasLimit: 1_000_000,   // 目标链执行 gas
    payload:  payload
}));
```

目标链合约必须长这样（两个校验缺一不可）：

```solidity
function onEvent(address reactive_, uint256 chainId_, uint256 txHash_, uint256 amount_)
    external
    onlyServiceProvider                 // 调用者必须是本链 Callback Proxy
    onlyCallbackSender(reactive_)       // payload 注入的 sender 必须是我认可的 Reactive 合约
{
    ...
}
```

**部署顺序不能乱**（这是官方反复强调的）：

```
① 源链部署事件源 → 把地址填回 .env
② Lasna 部署 Reactive（构造时订阅 + 带 lREACT 预付款）→ 填 REACTIVE_ADDR
③ 目标链部署 Callback（带 ETH 付回调 gas，构造参数用②的地址）→ 填 CALLBACK_ADDR
④ 回 Lasna 调 setRecipient(CALLBACK_ADDR)   ← 少这一步，回调永远不发
⑤ 源链触发转账
```

---

## 五、本次任务的两种做法（我建议先 A 后 B）

### A. 先用原模板跑通三步走（30 分钟，保底）

模板的 `CampOrigin.ping(value)` 就是最小事件源，订阅、回调、核验脚本全现成：

```bash
cd /d/Web3/reactive-builder-camp/starter
source .env                 # 你已填好 PRIVATE_KEY / ORIGIN_ADDR 等

forge script script/Deploy.s.sol:DeployReactive --rpc-url $LASNA_RPC_URL --broadcast
# 输出 REACTIVE_ADDR → 填回 .env，重新 source

forge script script/Deploy.s.sol:DeployCallback --rpc-url $SEPOLIA_RPC_URL --broadcast
# 输出 CALLBACK_ADDR → 填回 .env，重新 source

forge script script/Deploy.s.sol:SetRecipient --rpc-url $LASNA_RPC_URL --broadcast
PING_VALUE=500 forge script script/Deploy.s.sol:Ping --rpc-url $SEPOLIA_RPC_URL --broadcast

sleep 30 && bash script/verify.sh
```

跑通后你会拿到三笔哈希。但**任务标题写的是「监控代币转账」**，Ping 事件严格说不算——所以接着做 B。

### B. 换成真正的 ERC-20 Transfer（对齐任务标题）

**第 1 步**：在 `src/` 加一个最小测试币（只用于测试网，别抄去主网）：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.29;

contract TestToken {
    event Transfer(address indexed from, address indexed to, uint256 value);

    mapping(address => uint256) public balanceOf;
    uint256 public totalSupply = 1_000_000 * 10 ** 18;

    constructor() {
        balanceOf[msg.sender] = totalSupply;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "insufficient");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
}
```

**第 2 步**：部署脚本加两个 contract（放进 `script/Deploy.s.sol`）：

```solidity
contract DeployToken is Script {
    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        TestToken t = new TestToken();
        vm.stopBroadcast();
        console.log("ORIGIN_ADDR=", address(t));   // 覆盖 .env 里的 ORIGIN_ADDR
    }
}

contract SendToken is Script {
    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        TestToken(vm.envAddress("ORIGIN_ADDR")).transfer(
            vm.envAddress("WALLET_ADDR"),
            vm.envUint("TRANSFER_AMOUNT")
        );
        vm.stopBroadcast();
    }
}
```

**第 3 步**：改 `DeployReactive` 里的 topic0：

```diff
- uint256 topic0 = uint256(keccak256("Ping(address,uint256)"));
+ uint256 topic0 = uint256(keccak256("Transfer(address,address,uint256)"));
```

**第 4 步**：重新走部署顺序 ①–⑤，触发时用 `TRANSFER_AMOUNT=500000000000000000000`（500 token，18 位小数）。

> **最长踩的坑：`threshold` 是最小单位**。token 是 18 位小数时，阈值 100 表示 100 wei（≈0），不是 100 个币。设阈值时写 `500 ether`（= 500×10^18）这种写法最不容易错。

> 另一个坑：`verify.sh` 里写死了 `Ping(...)` 事件签名，改成 Transfer 后那段日志查询会查不到——不影响另外两段（Lasna 的 `lastReactBlock`、目标链的 `Handled`），但你需要手动用 etherscan 查 Transfer 那笔哈希。

---

## 六、提交前自检（照着勾）

- [ ] `forge test -vv` 本地 8 个测试通过
- [ ] `lastReactBlock` 不是 0（说明系统真的调了 `react()`）
- [ ] 目标链回调合约 `count` ≥ 1，且有 `Handled` 事件
- [ ] `verify.sh` 里 `debt` 是 0，`remaining` 还有余量
- [ ] 三笔哈希都记进 `notes/2026-11-18.md` 的交易表
- [ ] 没有把私钥、`.env`、助记词写进笔记（仓库是公开的）
- [ ] 笔记里写明用了 AI 的哪些部分、你怎么验证的（D12 会再查一次，现在写省事）

## 七、自测（先答再看）

1. 你订阅了「Sepolia 上某 ERC-20 的 `Transfer`，`topic1 = 你的地址`」，结果一笔 `from = address(0)` 的 mint 没触发。为什么？
2. `react()` 里收到一笔 `value` 低于阈值的 Transfer，你想「过滤掉」，应该怎么写？
3. 部署顺序里为什么必须先部署 Reactive、再部署 Callback，最后还要 `setRecipient`？

<details>
<summary>答案</summary>

1. 因为订阅是**精确相等**匹配：`from = address(0)` 的 topic1 是 0，不等于你的地址。想同时听普通转账和 mint，要分别 `subscribe()` 两次。
2. 用 `return`，不能用 `revert`。revert 会被系统捕获、记事件、照样收费。
3. Callback 合约的构造参数需要 Reactive 合约地址来做 `onlyCallbackSender` 鉴权，所以 Reactive 必须先部署；而 Reactive 侧一开始不知道回调地址，所以最后要用 owner-only 的 `setRecipient` 补上。少最后一步，`recipient == address(0)`，`react()` 直接 return。

</details>
