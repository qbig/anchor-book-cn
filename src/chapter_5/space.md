# 占用空间参考

这里介绍你需要为每个account分配多少空间(通过`space=xxx`)。
这里的文档只适用于不用`zero-copy`的account。`zero-copy` 会用 `repr(C)` 作为一个指针的类型转换,
所以会用`C`的空间分配结构。

除了为account的data分配空间，你还要加`8`字节给`space`，作为Anchor的内部账号类型标记discriminatior (参考例子里用到额外`8`的地方)。

| Types           | Space in bytes                    | Details/Example    
| --------------- | --------------------              | ----------- 
| bool            | 1                                 | 只需要一个bit但仍然需要占用1 byte
| u8/i8           | 1                                 |
| u16/i16         | 2                                 |
| u32/i32         | 4                                 |
| u64/i64         | 8                                 |
| u128/i128       | 16                                |
| [T;amount]      | space(T) * amount                 | e.g. space([u16;32]) = 2 * 32 = 64
| Pubkey          | 32                                |
| Vec\<T>         | 4 + (space(T) * amount)           | 账号的大小是固定的，所以对于大小是变量的数据类型，一开始就要初始化足够大的空间
| String          | 4 + length of string in bytes     | 账号的大小是固定的，所以对于大小是变量的数据类型，一开始就要初始化足够大的空间
| Option<T>       | 1 + (space(T))                    | 
| Enum            | 1 + Largest Variant Size          | e.g. Enum { A, B { val: u8 }, C { val: u16 } } -> 1 + space(u16) = 3
| f32             | 4                                 | 对于NaN序列化会失败 
| f64             | 8                                 | 对于NaN序列化会失败

# Example
```rust,ignore
#[account]
pub struct MyData {
    pub val: u16,
    pub state: GameState,
    pub players: Vec<Pubkey> // we want to support up to 10 players
}

impl MyData {
    pub const MAX_SIZE: usize = 2 + (1 + 32) + (4 + 10 * 32);
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq, Eq)]
pub enum GameState {
    Active,
    Tie,
    Won { winner: Pubkey },
}

#[derive(Accounts)]
pub struct InitializeMyData<'info> {
    // Note that we have to add 8 to the space for the internal anchor
    #[account(init, payer = signer, space = 8 + MyData::MAX_SIZE)]
    pub acc: Account<'info, MyData>,
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>
}
```
