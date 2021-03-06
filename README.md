# How to use

## Prerequisites

First you will need to have [ckb](https://github.com/nervosnetwork/ckb) compiled of course. Feel free to just following the official build steps in the README. We will customize configs later.

You will also need [mruby-contracts](https://github.com/nervosnetwork/mruby-contracts). Follow the steps in the README to build it, you will need the generated mruby contract file at `build/argv_source_entry`.

If you don't want to build mruby-contracts yourself, we have a prebuilt binary at [here](https://github.com/nervosnetwork/binary/raw/master/contracts/mruby/argv_source_entry).

## Configure CKB

Note this demo is developed together with CKB, and it is designed mainly for experimental purposes. So When you are running the demo here, it's best to compile CKB binary from source so you can tweak CKB source as you want. In order to keep your CKB safe, it's also best to launch your own dev chain locally instead of using testnet/mainnet when using this demo.

First, follow the [README](https://github.com/nervosnetwork/ckb/blob/develop/README.md) steps to make sure CKB is properly built with dev chain used.

There're optional steps here which would help you when you are using the demo but not required:

### Use Dummy POW mode

By default, CKB is running Cuckoo POW algorithm, depending on the computing power your machine has, this might slow things down.

To change to Dummy POW mode, which merely sleeps randomly for a few seconds before issuing a block, please locate `resource/specs/dev.toml` file in your CKB repo directory, navigate to `pow` section, and change the config to the following:

```toml
[pow]
func = "Dummy"

# Delay offset (in milliseconds)
[pow.params.delay]
type = "constant"
value = 5000
```

Then delete `[pow.params]`

This way you will be using Dummy POW mode, note that if you have run CKB before, you need to clean data directory (which is `data` by default in the directory where you run CKB), rebuild your CKB binary, then launch CKB again.

### Enlarge miner reward

By default, CKB issues 50000 capacities to a block, however, since we will need to install a binary which is roughly 1.6MB here, it might take quite a while for CKB to miner enough capacities. So you might want to enlarge miner reward to speedup this process.

To do this, locate `resources/specs/dev.toml` file in your config directory, navigate to `params` section, and adjust `initial_block_reward` field to the following:

```toml
initial_block_reward = 500_000_000_000_000
```

Note that if you have run CKB before, you need to clean data directory (which is `data` by default in the directory where you run CKB), rebuild your CKB binary, then launch CKB again.

### Custom log config

By default CKB doesn't emit any debug log entries, but when you are playing with the demo, chances are you will be interested in certain debug logs.

To change this, locate `ckb.toml` file in the directory where you initialize and run CKB, navigate to `logger` section, and adjust `filter` field to the following:

```toml
filter = "info,chain=debug,script=debug"
```

Now when you restart your CKB main process, you will have debug log entries from `chain` and `script` modules, which will be quite useful when you play with this demo.

## Running demo

Capacity unit in SDK is shannon (1 byte = 10**8 shannon)

Now we can setup the Ruby demo:

```bash
$ git clone --recursive https://github.com/nervosnetwork/ckb-demo-ruby
$ cd ckb-demo-ruby
$ bundle
$ bundle exec rake console
[1] pry(main)> api = Ckb::Api.new
[2] pry(main)> api.get_tip_number
28
```

Please be noted that the demo depends on the [bitcoin-secp256k1](https://github.com/cryptape/ruby-bitcoin-secp256k1) gem and the [rbnacl](https://github.com/crypto-rb/rbnacl) gem, which require manual install of secp256k1 and libsodium library. Follow [this](https://github.com/cryptape/ruby-bitcoin-secp256k1#prerequisite) and [this](https://github.com/crypto-rb/rbnacl#installation) to install them locally.

In the Ruby shell, we can start playing with the demo.

### Install mruby contract

We will need the `argv_source_entry` file as mentioned in `Prerequisite` section, then we can install this mruby contract into CKB:

```ruby
[1] pry(main)> asw = Ckb::AlwaysSuccessWallet.new(api)
[2] pry(main)> conf = asw.install_mruby_cell!("/path/to/argv_source_entry")
=> {:out_point=>{:tx_hash=>"0x20b849ffe67eb5872eca0d68fff1de193f07354ea903948ade6a3c170d89e282", :index=>0},
 :cell_hash=>"0x03dba46071a6702b39c1e626f469b4ed9460ed0ad92cf2e21456c34e1e2b04fd"}
[3] pry(main)> asw.configuration_installed?(conf)
=> false
[3] pry(main)> # Wait a while till this becomes true
[4] pry(main)> asw.configuration_installed?(conf)
=> true
```

Now you have the mruby contract installed in CKB, and the relevant configuration in `conf` structure. You can inform `api` object to use this configuration:

```ruby
[1] pry(main)> api.set_and_save_default_configuration!(conf)
```

Notice this line also saves the configuration to a local file, so next time when you are opening a `pry` console, you only need to load the save configuration:

```ruby
[1] pry(main)> api = Ckb::Api.new
[2] pry(main)> api.load_default_configuration!
```

Only when you clear the data directory in the CKB node, or switch to a different CKB node, will you need to perform the above installations again.

### Basic wallet

To play with wallets, first we need to add some capacities to a wallet:

```bash
[1] pry(main)> bob = Ckb::Wallet.from_hex(api, "0xe79f3207ea4980b7fed79956d5934249ceac4751a4fae01a0f7c4a96884bc4e3")
[2] pry(main)> bob.get_balance
=> 0
[3] pry(main)> asw.send_capacity(bob.lock, 300000 * 10 ** 8)
[4] pry(main)> # wait a while
[5] pry(main)> bob.get_balance
=> 30000000000000
```

Now we can perform normal transfers between wallets:

```bash
[1] pry(main)> bob = Ckb::Wallet.from_hex(api, "0xe79f3207ea4980b7fed79956d5934249ceac4751a4fae01a0f7c4a96884bc4e3")
[2] pry(main)> alice = Ckb::Wallet.from_hex(api, "0x76e853efa8245389e33f6fe49dcbd359eb56be2f6c3594e12521d2a806d32156")
[3] pry(main)> bob.get_balance
=> 100000
[4] pry(main)> alice.get_balance
=> 0
[5] pry(main)> bob.send_capacity(alice.lock, 100000 * 10 ** 8)
=> "0xd7abc1407eb07d334fea86ef0e9b12b2273833137327c2a53f2d8ba1be1e4d85"
[6] pry(main)> # wait for some time
[7] pry(main)> alice.get_balance
=> 10000000000000
[8] pry(main)> bob.get_balance
=> 20000000000000
```

### User defined token

We can also create user defined token that's separate from CKB. A new user defined token is made of 2 parts:

* A token name
* Token's admin pubkey, only token's admin can issue new tokens. Other user can only transfer already created tokens to others.

Ruby demo here provides an easy way to create a token from an existing wallet

```bash
[1] pry(main)> bob = Ckb::Wallet.from_hex(api, "0xe79f3207ea4980b7fed79956d5934249ceac4751a4fae01a0f7c4a96884bc4e3")
[2] pry(main)> alice = Ckb::Wallet.from_hex(api, "0x76e853efa8245389e33f6fe49dcbd359eb56be2f6c3594e12521d2a806d32156")
```

Now we can create this token from a user with CKB capacities(since the cell used to hold the tokens will take some capacity):

```bash
[9] pry(main)> bob.get_balance
=> 20000000000000
[10] pry(main)> # here we are creating 10000000 tokens for "Token 1", we put those tokens in a cell with 10000 * 10**8 CKB capacity
[11] pry(main)> result = bob.create_udt_token(10000 * 10**8, "Token 1", 10000000)
[12] pry(main)> token_info = result.token_info
=> #<Ckb::TokenInfo:0x0000561fee8cf550 @name="Token 1", @pubkey="0x024a501efd328e062c8675f2365970728c859c592beeefd6be8ead3d901330bc01">
[13] pry(main)> # token info represents the meta data for a token
[14] pry(main)> # we can assemble a wallet for user defined token with token info structure
[15] pry(main)> bob_token1 = bob.udt_wallet(token_info)
[16] pry(main)> alice_token1 = alice.udt_wallet(token_info)
[17] pry(main)> bob_token1.get_balance
=> 10000000
[18] pry(main)> alice_token1.get_balance
=> 0
```

Now that the token is created, we can implement a token transfer process between CKB capacities and user defined tokens. Specifically, we are demostrating the following process:

* Alice signs signatures providing a certain number of CKB capacities in exchange of some user defined tokens. Notice CKB contracts here can ensure that no one can spend alice's signed capacities without providing tokens for Alice
* Then bob provides user defined tokens for Alice in exchange for Alice's capacities.

Notice CKB is flexible to implement many other types of transaction for this problem, here we are simply listing one solution here. You are not limited to only this solution.

The following code fulfills this step:

```bash
[15] pry(main)> # Alice is paying 10999 * 10**8 CKB capacities for 12345 token 1, alice will also spare 3010 * 10**8 CKB capacities to hold the returned token 1.
[15] pry(main)> partial_tx = alice_token1.generate_partial_tx_for_udt_cell(12345, 10999 * 10**8, 3010 * 10**8)
[18] pry(main)> bob_token1.send_amount(12345, partial_tx)
[19] pry(main)> bob_token1.get_balance
=> 9987655
[20] pry(main)> alice_token1.get_balance
=> 12345
```

### User Defined Token which uses only one cell per wallet:

```bash
[1] pry(main)> bob = Ckb::Wallet.from_hex(api, "0xe79f3207ea4980b7fed79956d5934249ceac4751a4fae01a0f7c4a96884bc4e3")
[2] pry(main)> alice = Ckb::Wallet.from_hex(api, "0x76e853efa8245389e33f6fe49dcbd359eb56be2f6c3594e12521d2a806d32156")
[3] pry(main)> result = bob.create_udt_token(10000 * 10**8, "Token 2", 10000000, account_wallet: true)
[4] pry(main)> token_info2 = result.token_info
[5] pry(main)> bob_cell_token2 = bob.udt_account_wallet(token_info2)
[6] pry(main)> alice_cell_token2 = alice.udt_account_wallet(token_info2)
[7] pry(main)> alice.create_udt_account_wallet_cell(20000 * 10 ** 8, token_info2)
[8] pry(main)> bob_cell_token2.send_tokens(12345, alice_cell_token2)
[9] pry(main)> bob_cell_token2.get_balance
=> 9987655
[10] pry(main)> alice_cell_token2.get_balance
=> 12345
```

NOTE: While it might be possible to mix the 2 ways of using user defined token above in one token, we don't really recommend that since it could be the source of a lot of confusions.

### User Defined Token with a fixed upper cap

We have also designed a user defined token with a fixed upper cap. For this type of token, the token amount is set when it is initially created, there's no way to create more tokens after that.

```bash
[1] pry(main)> bob = Ckb::Wallet.from_hex(api, "0xe79f3207ea4980b7fed79956d5934249ceac4751a4fae01a0f7c4a96884bc4e3")
[2] pry(main)> alice = Ckb::Wallet.from_hex(api, "0x76e853efa8245389e33f6fe49dcbd359eb56be2f6c3594e12521d2a806d32156")
 # Create a genesis UDT cell with 20000 * 10**8 capacity, the UDT has a fixed amount of 10000000 * 10**8.
# The initial exchange rate is 1 capacity for 5 tokens.
[3] pry(main)> result = bob.create_fixed_amount_token(20000 * 10**8, 10000000 * 10**8, 5)
[4] pry(main)> fixed_token_info = result.token_info
# Creates a UDT wallet that uses only one cell, the cell has a capacity of 11111 * 10**8
[5] pry(main)> alice.create_udt_account_wallet_cell(11111 * 10**8, fixed_token_info)

 # Purchase 50000 * 10**8 UDT tokens
[6] pry(main)> alice.purchase_fixed_amount_token(50000 * 10**8, fixed_token_info)
# Wait for a while here...
[7] pry(main)> alice.udt_account_wallet(fixed_token_info).get_balance
```
