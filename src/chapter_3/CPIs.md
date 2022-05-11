# Cross-Program Invocations跨程序调用

我们经常需要在一个程序中调用另外一个程序。在Solana里，这是通过跨程序调用实现的（CPIs）。

来看一下这个傀儡大师操控傀儡的例子。当然，这不是一个非常实际的例子，但是足够说明一些CPIs中需要注意的细节。在这章最后的里程碑项目会有一个更接近实际场景的有很多CPIs调用的例子。

## 搭建基础的CPI功能

创建一个新的workspace
```
anchor init puppet
```

然后复制下面的代码：

```rust,ignore
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod puppet {
    use super::*;
    pub fn initialize(_ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn set_data(ctx: Context<SetData>, data: u64) -> Result<()> {
        let puppet = &mut ctx.accounts.puppet;
        puppet.data = data;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub puppet: Account<'info, Data>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct SetData<'info> {
    #[account(mut)]
    pub puppet: Account<'info, Data>,
}

#[account]
pub struct Data {
    pub data: u64,
}
```

上面的代码并没什么特别的地方，只是一个非常简单的程序！有意思的地方是它将怎么和我们接下来要创建的程序交互。（通过跨程序调用，也就是CPI）

在Workspace中运行
```
anchor new puppet-master
```
inside the workspace and copy the following code:

```rust,ignore
use anchor_lang::prelude::*;
use puppet::cpi::accounts::SetData;
use puppet::program::Puppet;
use puppet::{self, Data};

declare_id!("HmbTLCmaGvZhKnn1Zfa1JVnp7vkMV4DYVxPLWBVoN65L");

#[program]
mod puppet_master {
    use super::*;
    pub fn pull_strings(ctx: Context<PullStrings>, data: u64) -> Result<()> {
        let cpi_program = ctx.accounts.puppet_program.to_account_info();
        let cpi_accounts = SetData {
            puppet: ctx.accounts.puppet.to_account_info(),
        };
        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);
        puppet::cpi::set_data(cpi_ctx, data)
    }
}

#[derive(Accounts)]
pub struct PullStrings<'info> {
    #[account(mut)]
    pub puppet: Account<'info, Data>,
    pub puppet_program: Program<'info, Puppet>,
}
```

然后加这行代码 `puppet_master = "HmbTLCmaGvZhKnn1Zfa1JVnp7vkMV4DYVxPLWBVoN65L"` 到你`Anchor.toml`文件的`[programs.localnet]`部分。 最后, 引入puppet program到puppet-master program，这需要在`puppet-master` 程序文件夹中的`Cargo.toml` 文件里，要把下面的代码加到`[dependencies]`部分:
```toml
puppet = { path = "../puppet", features = ["cpi"]}
```

`features = ["cpi"]`的配置是为了让我们不仅能够使用puppet的类，还能使用puppet的instruction builders还有cpi函数。 如果没有这些，我们就不得不用更低抽象层的solana syscalls。幸好anchor提供了这些接口的抽象，用起来会容易的多。 开启`cpi`功能之后, puppet-master程序就可以access到`puppet::cpi`模块。 Anchor 自动生成这个模块并且包含了专门的instructions builders还有对应的cpi helpers函数。

对于puppet程序, puppet-master使用`puppet::cpi::accounts` 模块提供的`SetData` instruction builder struct来提交puppet程序期望的`SetData` instruction的accounts. 然后，puppet-master创建一个新的cpi context 并把它传入`puppet::cpi::set_data`cpi函数。 这个函数的功能和puppet program中的`set_data` function完全一样，只是这里的要传的context是`CpiContext`而不是`Context`。

初始化CPI的代码和可能会分散业务逻辑代码的可读性，所以推荐的做法是把它放到instruction的`impl`代码。puppet-master程序会是这个样子:
```rust,ignore
use anchor_lang::prelude::*;
use puppet::cpi::accounts::SetData;
use puppet::program::Puppet;
use puppet::{self, Data};

declare_id!("HmbTLCmaGvZhKnn1Zfa1JVnp7vkMV4DYVxPLWBVoN65L");

#[program]
mod puppet_master {
    use super::*;
    pub fn pull_strings(ctx: Context<PullStrings>, data: u64) -> Result<()> {
        puppet::cpi::set_data(ctx.accounts.set_data_ctx(), data)
    }
}

#[derive(Accounts)]
pub struct PullStrings<'info> {
    #[account(mut)]
    pub puppet: Account<'info, Data>,
    pub puppet_program: Program<'info, Puppet>,
}

impl<'info> PullStrings<'info> {
    pub fn set_data_ctx(&self) -> CpiContext<'_, '_, '_, 'info, SetData<'info>> {
        let cpi_program = self.puppet_program.to_account_info();
        let cpi_accounts = SetData {
            puppet: self.puppet.to_account_info()
        };
        CpiContext::new(cpi_program, cpi_accounts)
    }
}
```

我们可以验证有所得步骤都按预期可以运行，这秩序把`puppet.ts`文件得内容换成如下:
```ts
import * as anchor from '@project-serum/anchor';
import { Program } from '@project-serum/anchor';
import { Keypair, SystemProgram } from '@solana/web3.js';
import { expect } from 'chai';
import { Puppet } from '../target/types/puppet';
import { PuppetMaster } from '../target/types/puppet_master';

describe('puppet', () => {
  anchor.setProvider(anchor.Provider.env());

  const puppetProgram = anchor.workspace.Puppet as Program<Puppet>;
  const puppetMasterProgram = anchor.workspace.PuppetMaster as Program<PuppetMaster>;

  const puppetKeypair = Keypair.generate();

  it('Does CPI!', async () => {
    await puppetProgram.rpc.initialize({
      accounts: {
        puppet: puppetKeypair.publicKey,
        user: anchor.getProvider().wallet.publicKey,
        systemProgram: SystemProgram.programId
      },
      signers: [puppetKeypair]
    });

    await puppetMasterProgram.rpc.pullStrings(new anchor.BN(42),{
      accounts: {
        puppetProgram: puppetProgram.programId,
        puppet: puppetKeypair.publicKey
      }
    })

    expect((await puppetProgram.account.data
      .fetch(puppetKeypair.publicKey)).data.toNumber()).to.equal(42);
  });
});
```

然后运行`anchor test`.

## Privilege Extension权限得延伸性

CPIs 会把权限由调用者延续给被调用的程序. puppet account是传给puppet-master的mutable account但它对于通过CPI调用的puppet程序依然是mutable的(否则测试中的`expect`会报错). 这同样是适用于用户签名(signatures).

如果你想验证这一点，可以在puppet程序加个`authority`字段给`Data`struct.
```rust,ignore
#[account]
pub struct Data {
    pub data: u64,
    pub authority: Pubkey
}
```

然后调整`initialize`函数:
```rust,ignore
pub fn initialize(ctx: Context<Initialize>, authority: Pubkey) -> Result<()> {
    ctx.accounts.puppet.authority = authority;
    Ok(())
}
```

加`32`给`puppet`的`space` constraint因为`Data` struct新加了`Pubkey`字段。
```rust,ignore
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 8 + 8 + 32)]
    pub puppet: Account<'info, Data>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

然后, 调整`SetData`的validation struct:
```rust,ignore
#[derive(Accounts)]
pub struct SetData<'info> {
    #[account(mut, has_one = authority)]
    pub puppet: Account<'info, Data>,
    pub authority: Signer<'info>
}
```

`has_one`这个限制条件检查`puppet.authority = authority.key()`.

puppet-master程序现在也需要调整:
```rust,ignore
use anchor_lang::prelude::*;
use puppet::cpi::accounts::SetData;
use puppet::program::Puppet;
use puppet::{self, Data};

declare_id!("HmbTLCmaGvZhKnn1Zfa1JVnp7vkMV4DYVxPLWBVoN65L");

#[program]
mod puppet_master {
    use super::*;
    pub fn pull_strings(ctx: Context<PullStrings>, data: u64) -> Result<()> {
        puppet::cpi::set_data(ctx.accounts.set_data_ctx(), data)
    }
}

#[derive(Accounts)]
pub struct PullStrings<'info> {
    #[account(mut)]
    pub puppet: Account<'info, Data>,
    pub puppet_program: Program<'info, Puppet>,
    // Even though the puppet program already checks that authority is a signer
    // using the Signer type here is still required because the anchor ts client
    // can not infer signers from programs called via CPIs
    pub authority: Signer<'info>
}

impl<'info> PullStrings<'info> {
    pub fn set_data_ctx(&self) -> CpiContext<'_, '_, '_, 'info, SetData<'info>> {
        let cpi_program = self.puppet_program.to_account_info();
        let cpi_accounts = SetData {
            puppet: self.puppet.to_account_info(),
            authority: self.authority.to_account_info()
        };
        CpiContext::new(cpi_program, cpi_accounts)
    }
}
```

最后, 改一下test:

```ts
import * as anchor from '@project-serum/anchor';
import { Program } from '@project-serum/anchor';
import { Keypair, SystemProgram } from '@solana/web3.js';
import { Puppet } from '../target/types/puppet';
import { PuppetMaster } from '../target/types/puppet_master';
import { expect } from 'chai';

describe('puppet', () => {
  anchor.setProvider(anchor.Provider.env());

  const puppetProgram = anchor.workspace.Puppet as Program<Puppet>;
  const puppetMasterProgram = anchor.workspace.PuppetMaster as Program<PuppetMaster>;

  const puppetKeypair = Keypair.generate();
  const authorityKeypair = Keypair.generate();

  it('Does CPI!', async () => {
    await puppetProgram.rpc.initialize(authorityKeypair.publicKey, {
      accounts: {
        puppet: puppetKeypair.publicKey,
        user: anchor.getProvider().wallet.publicKey,
        systemProgram: SystemProgram.programId,
      },
      signers: [puppetKeypair]
    });

    await puppetMasterProgram.rpc.pullStrings(new anchor.BN(42),{
      accounts: {
        puppetProgram: puppetProgram.programId,
        puppet: puppetKeypair.publicKey,
        authority: authorityKeypair.publicKey
      },
      signers: [authorityKeypair]
    })

    expect((await puppetProgram.account.data
      .fetch(puppetKeypair.publicKey)).data.toNumber()).to.equal(42);
  });
});
```

测试通过了，这是因为传给puppet-master的signature权限会被延申到puppet program，而puppet程序用signature来确认puppet account确实对交易进行了签名。

> Privilege extension特权延申算然很方便但是也很危险。如果一个对恶意程序的CPI被无意中调用了，就很危险了。
> 因为这个程序的调用中程序和调用者的权限一样，相当于你给了恶意程序权限。
> Anchor 有两个办法可以保护你无意中调用恶意程序的CPIs：
> 首先, `Program<'info, T>` 类型会检查所提供的account是应该传入的正确的程序`T`。
> 如果你忘了用`Program`类型, 自动生成的cpi function
> (比如上个例子中的`puppet::cpi::set_data`)
> 也会检查`cpi_program`的传入参数是应该传入的正确的程序。

## 重新加载一个Account

在puppet程序中, `Account<'info, T>` 类型被用来表示`puppet` account. 如果通过CPI更改了一个该类型的account，那么在调用程序（caller）中的这个account在调用之后是不会变化的。

你可以很容易自己验证这一点，可以把下面的代码加到`puppet::cpi::set_data(ctx.accounts.set_data_ctx(), data)` CPI调用之后。
```rust,ignore
puppet::cpi::set_data(ctx.accounts.set_data_ctx(), data)?;
if ctx.accounts.puppet.data != 42 {
    panic!();
}
Ok(())
```
现在你的测试会失败。为啥呢？之前的测试是可以跑通的，所以CPI确实把`data`字段更新为`42`了。

`data`字段没有在调用函数中更新的原因是，只有在instruction开始被处理的时候`Account<'info, T>`类型会反序列化输入的字节为一个新的struct。在这之后，这个struct就不再会随着链上account的数据更新而更新了。CPI更改了链上的account，但是因为调用程序中的struct和链上的account并没有联系，所以调用程序中的struct并不发生变化。

如果你需要读最新的刚刚被CPI更新过的account的状态，你可以调用`reload`方法，这样会重新反序列化account。如果你把`ctx.accounts.puppet.reload()?;`加到CPI调用之后，测试就可以通过了。

```rust,ignore
puppet::cpi::set_data(ctx.accounts.set_data_ctx(), data)?;
ctx.accounts.puppet.reload()?;
if ctx.accounts.puppet.data != 42 {
    panic!();
}
Ok(())
```

## 如何从CPI返回值

在Solana 1.8.12之后, `set_return_data`和`get_return_data` syscalls可以被用来写和读CPIs的返回值。 虽然你已经可以在用Anchor的程序中直接使用这些接口, Anchor暂时还没有提供搭更好的抽象来简化。

syscalls的返回值最大只可以有1024字节，所以我们依然有必要介绍老的获取CPI返回值的方法，这样可以应付大于1024字节的情况。

通过共同使用CPI和`reload`我们可以模拟返回值的功能。比如，一个例子就是，我们可能没有直接给`data`字段赋值为42，而是对`42`进行了一些计算，然后把结果保存到了`data`字段。然后puppet-master就可以在CPI之后调用`reload`，来使用计算的结果，实现返回值的效果。

## Programs as Signers由程序来签名

还有另外一个功能可以通过CPIs实现。但是，要了解的话，我们必须先学习什么是PDAs (由程序导出的地址)。这些我们会在下一个章讲解。