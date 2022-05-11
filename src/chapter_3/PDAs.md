# PDAs程序导出地址

知道如果使用PDAs是Solana编程最重要的技巧之一。
他们可以让变成模式变得更简单，更安全。那什么是PDAs呢？

PDAs (program derived addresses) 是有特殊属性的address，也就是地址。

和普通地址不同的是, PDAs不是真的公钥(public keys)，因此他们也没有相对应的私钥(private keys)。PDAs有两个使用场景。 第一，他们提供一种在链上实现类似hashmap的结构（key有开发者定义，value是地址）；第二，他们可以实现让程序为instruction签名。

## 如何生成一个PDA

在深入了解怎么在Anchor中使用PDA之前，我们先简单介绍一下什么是PDAs。

PDAs是通过对几个用户定义的seeds值和program的id一起进行哈希运算生成的:
```rust,ignore
// pseudo code
let pda = hash(seeds, program_id);
```

seeds值可以是随意定义。 可以是公钥, 一个string, 或者一个数字的数组等等。

但又50%的概率这个哈希函数还是会生成一个合法的public key(但PDAs不能是public keys), 所以我们可以搜一个bump值到一起哈希直到我们找到一个不是合法的public key作为PDA:
```rust,ignore
// pseudo code
fn find_pda(seeds, program_id) {
  for bump in 0..256 {
    let potential_pda = hash(seeds, bump, program_id);
    if is_pubkey(potential_pda) {
      continue;
    }
    return (potential_pda, bump);
  }
  panic!("Could not find pda after 256 tries.");
}
```

理论上有可能256次轮询之后还是找不bump，但概率很小可以忽略。
如果你对具体PDA怎么算的感兴趣，可以看[`solana_program` source code](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.find_program_address).

找到的第一个可以生成PDA的bump通常被称为“标准bump”。其他的bump值也可以生成合法的PDA，但推荐的做法是只用标准bump来避免误会。

## 使用PDAs

接下来我们会展示PDAs的功能和如何在Anchor中实现这些功能。

### 用PDAs实现类似Hashmap的结构 

在我们深入了解如果在Anchor中使用PDAs来创建hashmaps之前，我们先了解一下如果不用Anchor怎么实现。

#### 用PDAs构建hashmaps

PDAs是通过对bump，program id，用户所选择的seed进行哈希计算得到的。
这些seed可以被用来构建链上的类似的hashmap的结构.

例如,假设我们想开发一个网页游戏,并且需要储存一些用户的统计数据. 比如用户的等级和他们在游戏中的名字. 那么你可以创建一个大概这样的account:

```rust,ignore
pub struct UserStats {
  level: u16,
  name: String,
  authority: Pubkey
}
```

`authority`就是这个struct所属的用户.

这个方法会有这样一个问题, 它可以很容易的通过用户的stats账户拿到用户的account地址(读取`authority`字段), 但是如果你开始只有用户的account地址(更常见的情况, 比如用户刚打开页面), 我们怎么能找到所对应的stats 账号呢?
这是做不到的. 这就成问题了, 因为你的游戏很可能有同时需要用户的stats账号和它的authority的instruction, 这意味着我们需要把这些账号同时传入instruction (例如, 用来改名字的`ChangeName` instruction). 也许前端可以在localStorage存一个映射用户地址到用户stats的mapping. 但这个办法一旦在用户不小心清空localStorage的时候就会出问题.

如果用PDAs, 那我们可以用下面的结构:
```rust,ignore
pub struct UserStats {
  level: u16,
  name: String,
  bump: u8
}
```
然后把用户和用户的stats account的对应关系编码到用户stats account的地址本身中(通过使用用户address作为seed的一部分).

复用上面的伪代码:

```rust,ignore
// pseudo code
let seeds = [b"user-stats", authority];
let (pda, bump) = find_pda(seeds, game_program_id);
```

当一个用户connect到你的站点之后, 这个PDA的计算可以在前端完成, 只需要用户的account地址作为`authority`. 生成的PDA然后就可以作为用户stats account的地址. `b"user-stats"`是作为命名域加到seed中的, 因为用户也许还有其他的PDAs.
如果用户还需要定义一个表示库存的账号, 那这个库存账号的PDA可以用下面的seeds导出:

```rust,ignore
let seeds = [b"inventory", authority];
```

总结一下, 我们用PDAs实现了一个用户到用户的stats account的mapping. 这里并没有一个单独的hashmap例可以通过`get`函数来调用. 这种映射是隐含的. 每个用户的stats account可以通过特定的seeds ("user-stats" 加上用户的地址)作为输入参数, 由`find_pda`函数来找到.

#### Anchor中如果用PDA搭建hashmaps 

继续上一小节中的例子, 创建一个新的workspace
```
anchor init game
```

然后复制下面的代码

```rust,ignore
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod game {
    use super::*;
    // handler function
    pub fn create_user_stats(ctx: Context<CreateUserStats>, name: String) -> Result<()> {
        let user_stats = &mut ctx.accounts.user_stats;
        user_stats.level = 0;
        if name.as_bytes().len() > 200 {
            // proper error handling omitted for brevity
            panic!();
        }
        user_stats.name = name;
        user_stats.bump = *ctx.bumps.get("user_stats").unwrap();
        Ok(())
    }
}

#[account]
pub struct UserStats {
    level: u16,
    name: String,
    bump: u8,
}

// validation struct
#[derive(Accounts)]
pub struct CreateUserStats<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    // space: 8 discriminator + 2 level + 4 name length + 200 name + 1 bump
    #[account(
        init,
        payer = user,
        space = 8 + 2 + 4 + 200 + 1, seeds = [b"user-stats", user.key().as_ref()], bump
    )]
    pub user_stats: Account<'info, UserStats>,
    pub system_program: Program<'info, System>,
}
```

在account validation struct中, 我们将`seeds`和`init`一起使用来创建PDA.
额外的, 我们还加了一个空的`bump`限制条件来告诉Anchor我们要使用标准bump.
然后, 在handler方法里面, 我们可以调用`ctx.bumps.get("user_stats")`来获取anchor找到的bump, 然后把它存到user stats account里面.
如果在这之后我们想要在另外一个不同的instruction中使用这里创建的PDA, 我们可以用一个新的validation struct (这会通用调用`hash(seeds, user_stats.bump, game_program_id)`验证`user_stats` account确实是我们这里创建的):

```rust,ignore
// validation struct
#[derive(Accounts)]
pub struct ChangeUserName<'info> {
    pub user: Signer<'info>,
    #[account(mut, seeds = [b"user-stats", user.key().as_ref()], bump = user_stats.bump)]
    pub user_stats: Account<'info, UserStats>,
}
```
然后是另一个handler function:
```rust,ignore
// handler function (add this next to the create_user_stats function in the game module)
pub fn change_user_name(ctx: Context<ChangeUserName>, new_name: String) -> Result<()> {
    if new_name.as_bytes().len() > 200 {
        // proper error handling omitted for brevity
        panic!();
    }
    ctx.accounts.user_stats.name = new_name;
    Ok(())
}
```

最后, 我们来添加一个测试用例. 复制下面的代码到`game.ts`.

```ts
import * as anchor from '@project-serum/anchor';
import { Program } from '@project-serum/anchor';
import { PublicKey, SystemProgram } from '@solana/web3.js';
import { Game } from '../target/types/game';
import { expect } from 'chai';

describe('game', async() => {

anchor.setProvider(anchor.Provider.env());

  const program = anchor.workspace.Game as Program<Game>;

  it('Sets and changes name!', async () => {
    const [userStatsPDA, _] = await PublicKey
      .findProgramAddress(
        [
          anchor.utils.bytes.utf8.encode("user-stats"),
          anchor.getProvider().wallet.publicKey.toBuffer()
        ],
        program.programId
      );

    await program.rpc.createUserStats("brian", {
      accounts: {
        user: anchor.getProvider().wallet.publicKey,
        userStats: userStatsPDA,
        systemProgram: SystemProgram.programId
      }
    });

    expect((await program.account.userStats.fetch(userStatsPDA)).name).to.equal("brian");

    await program.rpc.changeUserName("tom", {
      accounts: {
        user: anchor.getProvider().wallet.publicKey,
        userStats: userStatsPDA
      }
    })

    expect((await program.account.userStats.fetch(userStatsPDA)).name).to.equal("tom");
  });
});
```

正如我们在之前的段落中描述的, 我们用了一个`find`来查找PDA. 然后我们就可以像对待普通address一样使用这个PDA(几乎). 当我们调用 `createUserStats`的时候, 我们不需要把PDA添加到`[signers]`, 即使创建新账号需要一个签名.
这是因为作为PDA是无法在程序外部对transaction进行签名的(因为PDA并不是真的public key, 所以也就没有对应的private key来签名). 作为替代方案, 签名是在对system program进行CPI调用的时候添加的.
这具体是什么原理, 我们会在[让程序作为Signers签名](#让程序作为Signers签名)小节介绍.

#### 保证唯一性

这个hashmap的结构的一个侧面的结果是保证了唯一性. 当`init`和`seeds`还有`bump`一起使用的时候, 它总会先搜索标准bump. 这意味着它只能被调用一次(因为第二次调用的时候, 同样地址的PDA已经被初始化了). 为演示保证唯一性有多强大, 我们拿一个区中心化交易所的应用来举例子. 在这个交易所的程序中, 任何人都可以用两种资产创建一个新的市场. 但是, 程序的创建者希望流动性更加集中并且每两个资产的组合应该只有一个对应的市场存在. 这个不需要PDA也可以实现, 但会需要一个表示全局的状态的来保存所有不同的市场. 然后在市场创建的时候, 程序会查资产的组后是否已经在全局市场列表中存在. 使用PDA的化就容易的多了. 任何市场可以直接通过对应的两个资产导出的PDA. 程序只需要检查导出的两个PDAs(因为两个地址作为seed的前后次序不同, 可以导出两个地址如[asset1, asset2] 和 [asset2, asset1])是否存在.

### 让程序作为Signers签名

创建PDAs需要PDAs对system program的`createAccount`CPI签名. 这怎么实现呢?

PDAs不是public keys, 所以他们是不可以对任何数据签名的. 但是, PDAs还是可以对CPI进行"伪"签名.
在Anchor中,要对一个PDA签名, 你必须把`CpiContext::new(cpi_program, cpi_accounts)`改为`CpiContext::new_with_signer(cpi_program, cpi_accounts, seeds)`, 这里`seeds`参数是指seeds _还有_ PDA创建的时候产生的bump.
当CPI被调用的时候, 对每个在`cpi_accounts`中的account, Solana的runtime都会检查`hash(seeds, current_program_id) == account address`是否是true. 如果是, 那个account的`is_signer`flag就会被设为true.
这意味着, 一个有某程序program X导出的PDA, 只可以被用来由progrom X产生的CPI调用. 这也意味着, 宏观看, PDA的签名可以被认作程序的签名.

这显然是好消息, 因为对于很多程序来说, 程序本身需要作为一些资产的Authority(程序需要能够签名).
比如说, 借贷协议的程序需要管理被用户存如的抵押物, 而自动做市商(AMM)需要管理流动性池子里面的代币.

让我们在回到puppet workspace, 然后加一个PDA的签名.

首先, 更新puppet-master的代码:
```rust,ignore
use anchor_lang::prelude::*;
use puppet::cpi::accounts::SetData;
use puppet::program::Puppet;
use puppet::{self, Data};

declare_id!("HmbTLCmaGvZhKnn1Zfa1JVnp7vkMV4DYVxPLWBVoN65L");

#[program]
mod puppet_master {
    use super::*;
    pub fn pull_strings(ctx: Context<PullStrings>, bump: u8, data: u64) -> Result<()> {
        let bump = &[bump][..];
        puppet::cpi::set_data(
            ctx.accounts.set_data_ctx().with_signer(&[&[bump][..]]),
            data,
        )
    }
}

#[derive(Accounts)]
pub struct PullStrings<'info> {
    #[account(mut)]
    pub puppet: Account<'info, Data>,
    pub puppet_program: Program<'info, Puppet>,
    /// CHECK: only used as a signing PDA
    pub authority: UncheckedAccount<'info>,
}

impl<'info> PullStrings<'info> {
    pub fn set_data_ctx(&self) -> CpiContext<'_, '_, '_, 'info, SetData<'info>> {
        let cpi_program = self.puppet_program.to_account_info();
        let cpi_accounts = SetData {
            puppet: self.puppet.to_account_info(),
            authority: self.authority.to_account_info(),
        };
        CpiContext::new(cpi_program, cpi_accounts)
    }
}
```

`authority` account现在是一个`UncheckedAccount`而非`Signer`. 当puppet-master被调用的时候, `authority` pda还不是一个signer, 所以我们肯定不能用Signer检查. 我们只在乎puppet-master可以签名, 所以我们不需要加任何seeds. 只有一个在链下计算的bump来传入function 就可以了.

最后, 这是更新后的`puppet.ts`:
```ts
import * as anchor from '@project-serum/anchor';
import { Program } from '@project-serum/anchor';
import { Keypair, PublicKey, SystemProgram } from '@solana/web3.js';
import { Puppet } from '../target/types/puppet';
import { PuppetMaster } from '../target/types/puppet_master';
import { expect } from 'chai';

describe('puppet', () => {
  anchor.setProvider(anchor.Provider.env());

  const puppetProgram = anchor.workspace.Puppet as Program<Puppet>;
  const puppetMasterProgram = anchor.workspace.PuppetMaster as Program<PuppetMaster>;

  const puppetKeypair = Keypair.generate();

  it('Does CPI!', async () => {
    const [puppetMasterPDA, puppetMasterBump] = await PublicKey
      .findProgramAddress([], puppetMasterProgram.programId);

    await puppetProgram.rpc.initialize(puppetMasterPDA, {
      accounts: {
        puppet: puppetKeypair.publicKey,
        user: anchor.getProvider().wallet.publicKey,
        systemProgram: SystemProgram.programId,
      },
      signers: [puppetKeypair]
    });

    await puppetMasterProgram.rpc.pullStrings(puppetMasterBump, new anchor.BN(42),{
      accounts: {
        puppetProgram: puppetProgram.programId,
        puppet: puppetKeypair.publicKey,
        authority: puppetMasterPDA
      ,
    }});

    expect((await puppetProgram.account.data
      .fetch(puppetKeypair.publicKey)).data.toNumber()).to.equal(42);
  });
});
```

`authority`已经不在是一个随机生成的公私钥对了, 而是一个由puppet-master program导出的PDA. 这意味着puppet-master可以用它来签名, 正如在`pullStrings`中所看到的. 值得注意我们的实现也允许使用非标准bump, 但这里我们只想验证我们可以签名, 所以我们并不在乎用的是否是标准bump.

> 在很多情况下, 我们可以通过让一个存数据的PDA, 也用来签名, 而不是额外定义一个PDA, 并以此来减少所需要accounts的数量.

## PDAs: 总结

这小节我们简单回顾一下PDA能实现的不同功能.

首先, 你可以用PDA来创建hashmaps. 我们创建了一个由用户地址导出的PDA来储存user stats. 这个导出方法把用户的地址和用户的stats account联系到了一起, 这样只要有用户地址, 就很容易导出stats account.

Hashmaps还保证了唯一性, 这点可以在很多场景中运用, 比如, 在一个区中心化交易所应用中, 每两种资产只允许一个市场存在.

第二, PDAs可以用来让程序能够为CPI签名. 这意味着程序可以代码的逻辑和规则控制和管理资产和数据.

你甚至可以结合这两个功能, 让一个instruction中的PDA即作为一个表示状态的account, 也为CPI签名.

坦白说, 对PDAs的使用是使用Solana进行合约开发中最有挑战的技能.
因此我们准备了一些额外的资料来对我们的内容进行补充.

- [Pencilflips's twitter thread on PDAs](https://twitter.com/pencilflip/status/1455948263853600768?s=20&t=J2JXCwv395D7MNkX7a9LGw)
- [jarry xiao's talk on PDAs and CPIs](https://www.youtube.com/watch?v=iMWaQRyjpl4)
- [paulx's guide on everything Solana (covers much more than PDAs)](https://paulx.dev/blog/2021/01/14/programming-on-solana-an-introduction/)
