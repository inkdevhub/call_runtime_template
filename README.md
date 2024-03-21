# Call-runtime example

This documentation serves as an example for educational purposes only.

Smart contracts, by themselves, are limited to manipulating their own storage and performing internal computations. Call-runtime allows them to interact with other parts of the blockchain system, provided by the runtime. This enables contracts to leverage functionalities like token transfers (using the Balances pallet), setting account balances (using the Balances pallet), or interacting with on-chain oracles (using a custom pallet).

Unlike chain extensions, you would not need to write any additional code in pallet-contracts API. That makes this method of calling a pallet's dispatchables more secure and easier on the development side. It does not require additional audition and tests as well.

## Requirements:

### Node configuration

Substrate node configured to run contracts with unstable features. For that, adjust pallet contracts configuration in your runtime with:

```Rust
// In your node's runtime configuration file (runtime.rs)
impl pallet_contracts::Config for Runtime {
  …
  // `Everything` or anything that will allow for the `Balances::transfer` extrinsic.
  type CallFilter = frame_support::traits::Everything; 
  type UnsafeUnstableInterface = ConstBool<true>;
  …
}
```

### Cargo.toml

In your `Cargo.toml` add the `call-runtime` feature and explicitly disable some of `sp-io` features, to avoid conflicts:
```
sp-io = { version = "18.0.0", default-features = false, features = ["disable_panic_handler", "disable_oom", "disable_allocator"] }
sp-runtime = { version = "19.0.0", default-features = false }
```
Also, enable the `"call-runtime"` feature for `ink!` dependancy.

### Contract's specifics

#### RuntimeCall
`RuntimeCall` enum is a part of the runtime dispatchables API. `Ink!` doesn't expose the real enum, so we need a partial definition matching our targets. Find the full definition with `construct_runtime!` macro or tools like `frame_support` crate.

With the `construct_runtime!`, you can count the index of a pallet, it is zero based. So for `Balances` pallet the index is `4`.
```Rust
construct_runtime!(
    pub enum Runtime where
        Block = Block,
        NodeBlock = opaque::Block,
        UncheckedExtrinsic = UncheckedExtrinsic
    {
        System: frame_system,
        Timestamp: pallet_timestamp,
        Aura: pallet_aura,
        Grandpa: pallet_grandpa,
        Balances: pallet_balances,
        TransactionPayment: pallet_transaction_payment,
        Sudo: pallet_sudo,
        // ...
    }
);
```

With `frame_support` you can call upon the snippet below, which would output the index:
```Rust
use frame_support::traits::PalletInfoAccess;

let my_pallet_index = MyPallet::index();
```

#### PalletCall

`PalletCall` enum is a part of a pallet dispatchables API and is named after a palet. For example, `Balances` pallet's enum would be `BalancesCall`. 

The indexes can be found in your pallet code's `#[pallet::call]` section and check `#[pallet::call_index(x)]` attribute of the call. If these attributes are missing, use source-code order (0-based).

#### Call-runtime

The example message `transfer_call_runtime` call will fail if:
- It's called outside the blockchain environment (e.g., during development or testing).
- Blockchain itself doesn't allow the "call-runtime" feature (controlled by a setting called `UnsafeUnstableInterface`).
- Chain restricts contracts from calling the `Balances::transfer` function (enforced by a `CallFilter`).
- Recipient account doesn't have enough funds to cover the existential deposit after the transfer.
- Contract itself doesn't have enough balance to complete the transfer.

## Chain extensions

Use chain-extensions instead of call-runtime when you need to:
- Return data.
- Provide functionality exclusively to contracts.
- Provide custom weights.
- Avoid the need to keep the Call data structure stable.
