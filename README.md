# Artis

The solution to execute decentralized margin trading on DEXes.

It works with:
- [Uniswap](https://uniswap.io/) (exchange)
- [MakerDAO](https://makerdao.com) (lending protocol, Mainnet only)
- [Compound V2](ttps://compound.finance/presidio) (lending protocol, Rinkeby only).

More DEXes and lending protocols are coming.

**THIS SOFTWARE IS NOT AUDITED AND NOT READY FOR PRODUCTION USAGE. USE IT AT YOUR OWN RISK.**

## How to open a leveraged long ETH position

In this guide, we will open a long ETH positions with DAI as an owed token.

Other token pairs will come once Compound v2 will be released to Mainnet.

### Prerequisites
- OS: Mac OS or Linux
- Ethereum account with initial deposit in ETH
- [Seth](https://dapp.tools/seth) command line tool, with access to the abovementioned account. Seth docs can be found [here](https://github.com/dapphub/dapptools/tree/master/src/seth)

You can install Seth with those commands:
```
curl https://dapp.tools/install | sh
. "$HOME/.nix-profile/etc/profile.d/nix.sh"
```

In order to enable seth to operate your account, you need to put a [json keystore file](https://medium.com/@julien.maffre/what-is-an-ethereum-keystore-file-86c8c5917b97) that corresponds to `ETH_FROM` address in `~/.ethereum/keystore` folder.

If you don't have this folder, you can create it with the next command:
```
mkdir -p ~/.ethereum/keystore
```

The easiest way to obtain json keystore file is to create a new account on https://www.myetherwallet.com/ or https://mycrypto.com/.

After you put a keystore file into this directory, you can run `seth accounts` command to ensure that `seth` managed to parse the file. If everything is ok, you will see `ETH_FROM` address in the output.

After the installation of seth, it's needed to specify default parameters in `.sethrc` file in your `$HOME` folder. If you don't have one, please create a new `.sethrc` file inside `$HOME` folder. You can do it in any text editor, like `nano`.

Here you can find an example of `.sethrc` file with Seth settings:
```
export ETH_FROM=<your address>
export ETH_GAS=3000000
export ETH_GAS_PRICE=5000000000 # 5 GWei, please adjust it if transactions are slow
export SETH_CHAIN=ethlive
export ETH_KEYSTORE=~/.ethereum/keystore
```

Please note that you need to set `SETH_CHAIN` variable according to the network that you are going to use (`ethlive` or `rinkeby`).
### Steps

All commands in a code block can be executed in bash shell

#### 1. Load predefined variables

##### 1.1 Predefined variables and functions can be loaded from `artis-makerdao-mainnet.sh` or `artis-compoundv2-rinkeby.sh` files, depending on protocol and network that you want to use. Those files can be found in the root folder of this repo.
```
$ source artis-makerdao-mainnet.sh
```
Or
```
$ source artis-compoundv2-rinkeby.sh
```

#### 2. Get proxy wallet

If you used [CDP portal](https://cdp.makerdao.com) before, you most probably already have a proxy wallet, and you can use it for Artis.
```
# check if you have proxy wallet, assossiated with your address
$ seth call $PROXY_REGISTRY "proxies(address)(address)" <your address>
```
If this command return non-zero code, it's a `DS_PROXY` address, that you can use for further actions.

If you don't have a proxy wallet, you can create a new one. Don't forget to transfer some ether to `ETH_FROM` address, so you can send transactions from it.
```
$ seth send $PROXY_REGISTRY "build()"
```
You will be promted to enter a password for the keystore file.

To to obtain newly created `DS_PROXY` address, you can execute the next command:

```
$ seth call $PROXY_REGISTRY "proxies(address)(address)" <your address>
```

#### 3. Define some variables in bash that will be used to create a transaction

##### 3.1 Specify address of `DS_PROXY` from the previous step
```
$ DS_PROXY=<address of proxy wallet from the previous step>
```

##### 3.2 Specify the amount of the initial deposit (in ETH):
Minimum supported value: 0.01 ETH
```
$ AMOUNT=1.1 # specify desired value
```

##### 3.3 Specify collateral ratio that the underlying position will have

Smaller values will give you more leverage, however, they give you more risk of liquidation: in case if ETH will go down relatively to DAI and your initial deposit becomes too small to cover minimal collateral ratio and your initial deposit will be seized.

Minimal amount for Mainnet: 1.5 (very risky). Recommended amount: 1.7

Minimal amount for Rinkeby: 2. Recommended amount: 2.1

Max theoretical leverage table (exchange fees and interest are not included in this calculation):

| Leverage | Collateral ratio |
| ----------- | ----------- |
| ~3 | 1.5 |
| ~2.6 | 1.6 |
| ~2.4 | 1.7 |
| ~2.2 | 1.8 |
| ~2.1 | 1.9 |
| ~2.0 | 2.0 |

```
$ COLL_RATIO=1.7 # specify desired value
```
#### 4. Send `openPosition` transaction
Transaction will go through your proxy wallet, so the calldata needs to be formed manually
```
# Calculate function signature
$ SIG=$(seth sig "openPosition(address[4],uint256[4])")

# Form calldata for Artis contract
$ CALLDATA="$SIG$(toArg $LENDER)$(toArg $EXCHANGE)$(toArg $HELD_ASSET)$(toArg $PRINCIPAL)$(toRawAmount 0 eth)$(toRawAmount $COLL_RATIO eth)$MAX_ITERATIONS$MIN_COLLATERAL_AMOUNT"

# Send transaction to your proxy wallet
$ seth send --value $(seth --to-wei $AMOUNT eth) "$DS_PROXY" "execute(address,bytes memory)(bytes32)" "$LEVERAGER" "$CALLDATA"
```
You will be promted to enter a password for the keystore file.

## How to close a long ETH position

### Prerequisites
You have an open long ETH position and executed all steps from the previous part

### Steps

#### 1. Define some variables in bash that will be used to create a transaction
##### 1.1 Get `POSITION_ID` that you want to close
Every position has its id that needs to be specified when you want to close it.

To obtain the ID you can go to https://etherscan.io, open the transaction that opened the position -> go to Event Logs -> search for **the last event** and take the first data field from this event. It will be `POSITION_ID`.

![PositionId](./img/position-id.png)

```
$ POSITION_ID=0000000000000000000000000000000000000000000000000000000000000001 # specify your ID
```


#### 2. Send `closePosition` transaction
Transaction will go through your proxy wallet, so the calldata needs to be formed manually
```
# Calculate function signature
$ SIG=$(seth sig "closePosition(address[2],uint256[3])")

# Form calldata for Artis contract
$ CALLDATA="$SIG$(toArg $LENDER)$(toArg $EXCHANGE)$POSITION_ID$COLL_RATIO_CLOSE$MAX_ITERATIONS"

# Send transaction to your proxy wallet
$ seth send "$DS_PROXY" "execute(address,bytes memory)(bytes32)" "$LEVERAGER" "$CALLDATA"
```
You will be promted to enter a password for the keystore file.

## How to top-up collateral and track the underlying loan

All assets and loan ownership stays on your proxy wallet, Artis smart contracts are used as bytecode and do not store any state or assets. Therefore you can manage the underlying loan with some 3rd party tools.
With MakerDAO as the underlying lending protocol, you can manage a CDP with the help of [CDP portal](https://cdp.makerdao.com). For example, you may top-up your collateral or view the liquidation price.

You can also use MakerDAO protocol explorers, like [LoanScan](https://loanscan.io) to see all the transactions that executed under the hood and see your CDP balance and collateral supplied.
