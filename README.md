# Constant-Product AMM

An educational automated market maker built on Solana with Anchor. The program manages a two-token liquidity pool, issues LP tokens to liquidity providers, executes swaps using the constant-product invariant, and returns proportional reserves when LP tokens are redeemed.

## AMM model

The pool follows the constant-product invariant:

```text
x * y = k
```

- `x` is the token X balance in `vault_x`.
- `y` is the token Y balance in `vault_y`.
- `k` is the product of both reserves.

A swap adds one token to the pool and removes the other. The output is calculated so that the pool remains on its constant-product curve, subject to integer rounding and fees.

Swap fees use an LP-fee model. Only the fee-adjusted input determines the output, while the complete input amount is transferred into the input vault. The difference remains in the pool and increases the value represented by existing LP tokens. There is no separate treasury account in this implementation.

## Instructions

| Instruction | Purpose |
| --- | --- |
| `initialize` | Creates the config PDA, LP mint, and token X/Y vaults. |
| `deposit` | Transfers proportional X and Y liquidity into the vaults and mints LP tokens. |
| `swap` | Exchanges X for Y or Y for X using fee-adjusted constant-product pricing. |
| `withdraw` | Burns LP tokens and returns the corresponding share of both reserves. |

## Program flow

```mermaid
flowchart TD
    User([User])

    User -->|initialize| Initialize[Create AMM]
    Initialize --> Config[(Config PDA<br/>mints, fee, bumps)]
    Initialize --> LP[(LP mint<br/>authority: Config PDA)]
    Initialize --> VaultX[(Vault X)]
    Initialize --> VaultY[(Vault Y)]

    User -->|deposit amount, max X, max Y| Deposit[Deposit liquidity]
    VaultX -. current X reserve .-> Deposit
    VaultY -. current Y reserve .-> Deposit
    LP -. current LP supply .-> Deposit
    Deposit -->|user X| VaultX
    Deposit -->|user Y| VaultY
    Deposit -->|mint ownership tokens| UserLP[(User LP account)]

    User -->|swap is_x, amount_in, minimum_out| Direction{Swap direction}
    Direction -->|is_x = true| XtoY[X enters; Y leaves]
    Direction -->|is_x = false| YtoX[Y enters; X leaves]
    XtoY --> Normalize[Arrange input and output reserves]
    YtoX --> Normalize
    Normalize --> Fee[Apply fee to pricing input]
    Fee --> Curve[Calculate output with x * y = k]
    Curve --> Slippage{Output meets minimum?}
    Slippage -->|No| Revert[Revert transaction]
    Slippage -->|Yes| Transfers[Transfer full input in<br/>and calculated output out]
    Transfers --> LPFee[Fee remains in the pool<br/>for LP holders]

    UserLP -->|withdraw LP amount, min X, min Y| Withdraw[Calculate proportional reserves]
    Withdraw --> Burn[Burn LP tokens]
    Burn -->|return X| User
    Burn -->|return Y| User
```

## Accounts

The `Config` PDA is derived from:

```text
["config", seed]
```

It records the two token mints, swap fee, authority, lock state, and PDA bumps. The LP mint is derived from:

```text
["lp", config]
```

The config PDA controls the LP mint and owns the associated token vaults, allowing the program to sign token transfers with deterministic PDA seeds.

## Build and test

The project uses Rust-based LiteSVM integration tests, so no external validator is required.

```bash
cargo build-sbf --manifest-path programs/amm-video/Cargo.toml
cargo test --workspace --all-targets
```

The test suite covers the happy path for every instruction:

- Pool initialization
- Liquidity deposit
- Fee-aware swap behavior
- Liquidity withdrawal

## Test result

All program and LiteSVM tests pass:

![All AMM tests passing](assets/tests-passing.png)

## Project structure

```text
programs/amm-video/
├── src/
│   ├── instructions/
│   │   ├── initialize.rs
│   │   ├── deposit.rs
│   │   ├── swap.rs
│   │   └── withdraw.rs
│   ├── error.rs
│   ├── lib.rs
│   └── state.rs
└── tests/
    ├── ix_handlers/
    └── tests.rs
```

## Program ID

```text
6KoUjko5kqLHaF31gdWGBihf8Pw8dUNte2hBpBEJveVe
```
