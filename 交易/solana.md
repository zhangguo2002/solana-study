# Solana 买卖交易实现

本文只总结纯链上交易部分，忽略订单、策略、风控、业务路由等外围逻辑。

对应源码：

- 买入：`service/solana_trade.go`

项目当前只实现了 Solana 上 `SOL -> wSOL -> 目标 SPL Token` 的买入流程，没有找到已实现的 Solana 卖出函数。下面先解释项目里的买入步骤，再给出同一 SPL Token-Swap 模型下的反向卖出模板。

## 程序和账户

项目使用 SPL Token-Swap 程序手动构造 swap 指令，核心配置来自环境变量：

```go
SolanaSwapProgramID // Token Swap 程序 ID
SolanaPoolState     // 池子状态账号
SolanaPoolAuth      // 池子 authority
SolanaPoolTokenWSOL // 池子里的 wSOL token account
SolanaPoolTokenOut  // 池子里的目标 token account
SolanaPoolMint      // LP token mint
SolanaPoolFeeAcc    // fee account
SolanaWsolMint      // wSOL mint
```

Token Program：

```go
const tokenProgramID = "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA"
```

Swap 指令 data：

```text
byte[0]      = 1，表示 Swap 指令
byte[1:9]    = amountIn，uint64 little endian
byte[9:17]   = minimumAmountOut，uint64 little endian
```

## 买入步骤：SOL 买目标 SPL Token

1. 连接 Solana RPC，加载 base58 私钥，得到 payer。
2. 解析 `amountIn`，单位 lamports。
3. 解析目标 token mint：`tokenOutMint`。
4. 读取池子配置账户：
   `poolState, poolAuthority, poolTokenWSOL, poolTokenOut, poolMint, poolFeeAccount, wsolMint`。
5. 计算用户目标 token ATA：`FindAssociatedTokenAddress(payer, tokenOutMint)`。
6. 读取池子 wSOL 和目标 token 储备，项目用恒定乘积公式粗略估算输出：
   `estimatedOut = outReserve - k / (wsolReserve + amountIn)`。
7. 按滑点计算 `amountOutMin`。
8. 获取 recent blockhash。
9. 计算用户 wSOL ATA。
10. 如果用户 wSOL ATA 不存在，添加创建 ATA 指令。
11. 添加系统转账指令：从 payer 向 wSOL ATA 转入 lamports。
12. 添加 `SyncNative` 指令，把原生 SOL 同步成 wSOL 余额。
13. 构造 Token-Swap 的 swap 指令：
    用户源账户是用户 wSOL ATA，池子源账户是 `poolTokenWSOL`，池子目标账户是 `poolTokenOut`，用户目标账户是用户目标 token ATA。
14. 添加关闭 wSOL ATA 指令，回收剩余 SOL。
15. 构建 transaction，payer 签名。
16. `SendTransactionWithOpts` 发送交易。
17. 查询用户目标 token ATA 余额确认。

## 买入模板代码

```go
package trade

import (
	"context"
	"encoding/binary"
	"math"
	"math/big"

	associatedtokenaccount "github.com/gagliardetto/solana-go/programs/associated-token-account"
	"github.com/gagliardetto/solana-go"
	"github.com/gagliardetto/solana-go/programs/system"
	"github.com/gagliardetto/solana-go/programs/token"
	"github.com/gagliardetto/solana-go/rpc"
)

const splTokenProgramID = "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA"

type SolanaSwapPool struct {
	SwapProgramID solana.PublicKey
	PoolState     solana.PublicKey
	PoolAuthority solana.PublicKey
	PoolTokenWSOL solana.PublicKey
	PoolTokenOut  solana.PublicKey
	PoolMint      solana.PublicKey
	PoolFeeAcc    solana.PublicKey
	WSOLMint      solana.PublicKey
	TokenOutMint  solana.PublicKey
}

func BuildSwapData(amountIn, minOut uint64) []byte {
	data := make([]byte, 17)
	data[0] = 1
	binary.LittleEndian.PutUint64(data[1:9], amountIn)
	binary.LittleEndian.PutUint64(data[9:17], minOut)
	return data
}

func BuySolanaToken(
	ctx context.Context,
	client *rpc.Client,
	privateKeyBase58 string,
	pool SolanaSwapPool,
	amountInLamports uint64,
	slippagePercent float64,
) (solana.Signature, error) {
	owner, err := solana.PrivateKeyFromBase58(privateKeyBase58)
	if err != nil {
		return solana.Signature{}, err
	}
	payer := owner.PublicKey()

	userOutATA, _, err := solana.FindAssociatedTokenAddress(payer, pool.TokenOutMint)
	if err != nil {
		return solana.Signature{}, err
	}
	userWSOLATA, _, err := solana.FindAssociatedTokenAddress(payer, pool.WSOLMint)
	if err != nil {
		return solana.Signature{}, err
	}

	wsolInfo, err := client.GetAccountInfo(ctx, pool.PoolTokenWSOL)
	if err != nil {
		return solana.Signature{}, err
	}
	outInfo, err := client.GetAccountInfo(ctx, pool.PoolTokenOut)
	if err != nil {
		return solana.Signature{}, err
	}
	wsolReserve := binary.LittleEndian.Uint64(wsolInfo.Value.Data.GetBinary())
	outReserve := binary.LittleEndian.Uint64(outInfo.Value.Data.GetBinary())

	k := float64(wsolReserve) * float64(outReserve)
	estimatedOut := float64(outReserve) - (k / (float64(wsolReserve) + float64(amountInLamports)))
	minOut := uint64(math.Floor(estimatedOut * (1 - slippagePercent/100)))

	recent, err := client.GetRecentBlockhash(ctx, rpc.CommitmentFinalized)
	if err != nil {
		return solana.Signature{}, err
	}

	var instructions []solana.Instruction

	wsolATAInfo, _ := client.GetAccountInfo(ctx, userWSOLATA)
	if wsolATAInfo == nil || wsolATAInfo.Value == nil {
		instructions = append(instructions, associatedtokenaccount.NewCreateInstruction(
			payer,
			payer,
			pool.WSOLMint,
		).Build())
	}

	outATAInfo, _ := client.GetAccountInfo(ctx, userOutATA)
	if outATAInfo == nil || outATAInfo.Value == nil {
		instructions = append(instructions, associatedtokenaccount.NewCreateInstruction(
			payer,
			payer,
			pool.TokenOutMint,
		).Build())
	}

	instructions = append(instructions,
		system.NewTransferInstruction(amountInLamports, payer, userWSOLATA).Build(),
		token.NewSyncNativeInstruction(userWSOLATA).Build(),
	)

	swapInst := solana.NewInstruction(
		pool.SwapProgramID,
		[]*solana.AccountMeta{
			{PublicKey: pool.PoolState, IsSigner: false, IsWritable: true},
			{PublicKey: pool.PoolAuthority, IsSigner: false, IsWritable: false},
			{PublicKey: userWSOLATA, IsSigner: false, IsWritable: true},
			{PublicKey: pool.PoolTokenWSOL, IsSigner: false, IsWritable: true},
			{PublicKey: pool.PoolTokenOut, IsSigner: false, IsWritable: true},
			{PublicKey: userOutATA, IsSigner: false, IsWritable: true},
			{PublicKey: pool.PoolMint, IsSigner: false, IsWritable: true},
			{PublicKey: pool.PoolFeeAcc, IsSigner: false, IsWritable: true},
			{PublicKey: pool.WSOLMint, IsSigner: false, IsWritable: false},
			{PublicKey: pool.TokenOutMint, IsSigner: false, IsWritable: false},
			{PublicKey: solana.MustPublicKeyFromBase58(splTokenProgramID), IsSigner: false, IsWritable: false},
		},
		BuildSwapData(amountInLamports, minOut),
	)
	instructions = append(instructions, swapInst)

	instructions = append(instructions, token.NewCloseAccountInstruction(
		userWSOLATA,
		payer,
		payer,
		[]solana.PublicKey{},
	).Build())

	tx, err := solana.NewTransaction(
		instructions,
		recent.Value.Blockhash,
		solana.TransactionPayer(payer),
	)
	if err != nil {
		return solana.Signature{}, err
	}
	if _, err := tx.Sign(func(key solana.PublicKey) *solana.PrivateKey {
		if key.Equals(payer) {
			return &owner
		}
		return nil
	}); err != nil {
		return solana.Signature{}, err
	}

	return client.SendTransactionWithOpts(ctx, tx, rpc.TransactionOpts{
		SkipPreflight:       false,
		PreflightCommitment: rpc.CommitmentFinalized,
	})
}
```

## 卖出步骤：目标 SPL Token 卖回 wSOL/SOL

项目里没有实现此函数。按同一个 SPL Token-Swap 程序，卖出的核心就是把买入 swap 的方向反过来：

1. 用户源账户从 `userWSOLATA` 改成用户目标 token ATA。
2. 池子源账户从 `poolTokenWSOL` 改成 `poolTokenOut`。
3. 池子目标账户从 `poolTokenOut` 改成 `poolTokenWSOL`。
4. 用户目标账户从用户目标 token ATA 改成用户 wSOL ATA。
5. swap 前要确保用户 wSOL ATA 存在，用来接收 wSOL。
6. swap 后执行 `CloseAccount(userWSOLATA, payer, payer)`，把 wSOL 解包回 SOL。

账户顺序在项目当前写法中保持为：

```text
poolState
poolAuthority
userSource
poolSource
poolDestination
userDestination
poolMint
poolFeeAccount
sourceMint
destinationMint
tokenProgram
```

## 卖出模板代码

```go
func SellSolanaToken(
	ctx context.Context,
	client *rpc.Client,
	privateKeyBase58 string,
	pool SolanaSwapPool,
	amountInTokens uint64,
	minOutLamports uint64,
) (solana.Signature, error) {
	owner, err := solana.PrivateKeyFromBase58(privateKeyBase58)
	if err != nil {
		return solana.Signature{}, err
	}
	payer := owner.PublicKey()

	userTokenATA, _, err := solana.FindAssociatedTokenAddress(payer, pool.TokenOutMint)
	if err != nil {
		return solana.Signature{}, err
	}
	userWSOLATA, _, err := solana.FindAssociatedTokenAddress(payer, pool.WSOLMint)
	if err != nil {
		return solana.Signature{}, err
	}

	recent, err := client.GetRecentBlockhash(ctx, rpc.CommitmentFinalized)
	if err != nil {
		return solana.Signature{}, err
	}

	var instructions []solana.Instruction
	wsolATAInfo, _ := client.GetAccountInfo(ctx, userWSOLATA)
	if wsolATAInfo == nil || wsolATAInfo.Value == nil {
		instructions = append(instructions, associatedtokenaccount.NewCreateInstruction(
			payer,
			payer,
			pool.WSOLMint,
		).Build())
	}

	swapInst := solana.NewInstruction(
		pool.SwapProgramID,
		[]*solana.AccountMeta{
			{PublicKey: pool.PoolState, IsSigner: false, IsWritable: true},
			{PublicKey: pool.PoolAuthority, IsSigner: false, IsWritable: false},
			{PublicKey: userTokenATA, IsSigner: false, IsWritable: true},
			{PublicKey: pool.PoolTokenOut, IsSigner: false, IsWritable: true},
			{PublicKey: pool.PoolTokenWSOL, IsSigner: false, IsWritable: true},
			{PublicKey: userWSOLATA, IsSigner: false, IsWritable: true},
			{PublicKey: pool.PoolMint, IsSigner: false, IsWritable: true},
			{PublicKey: pool.PoolFeeAcc, IsSigner: false, IsWritable: true},
			{PublicKey: pool.TokenOutMint, IsSigner: false, IsWritable: false},
			{PublicKey: pool.WSOLMint, IsSigner: false, IsWritable: false},
			{PublicKey: solana.MustPublicKeyFromBase58(splTokenProgramID), IsSigner: false, IsWritable: false},
		},
		BuildSwapData(amountInTokens, minOutLamports),
	)
	instructions = append(instructions, swapInst)

	instructions = append(instructions, token.NewCloseAccountInstruction(
		userWSOLATA,
		payer,
		payer,
		[]solana.PublicKey{},
	).Build())

	tx, err := solana.NewTransaction(
		instructions,
		recent.Value.Blockhash,
		solana.TransactionPayer(payer),
	)
	if err != nil {
		return solana.Signature{}, err
	}
	if _, err := tx.Sign(func(key solana.PublicKey) *solana.PrivateKey {
		if key.Equals(payer) {
			return &owner
		}
		return nil
	}); err != nil {
		return solana.Signature{}, err
	}

	return client.SendTransactionWithOpts(ctx, tx, rpc.TransactionOpts{
		SkipPreflight:       false,
		PreflightCommitment: rpc.CommitmentFinalized,
	})
}
```

## 注意点

- `service/solana_trade.go` 当前把 `privKeyBase58 := ""` 写死为空，实际运行前必须从配置或请求里读 Solana 私钥。
- 项目当前用 `GetAccountInfo(...).Data.GetBinary()` 直接按 token account 数据开头读储备，这不一定是 SPL Token Account amount 字段的正确偏移。生产代码建议用 SPL Token Account 解码结构读取 amount。
- 卖出模板里的 `minOutLamports` 应该通过池子公式、SDK 报价或链上模拟得到，不建议写死。

