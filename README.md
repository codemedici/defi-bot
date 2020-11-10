## Project Structure

The entire bot is made up of just two core files basically:

* `index.js` : a node.js app that continuously fetches crypto prices on exchanges looking for arbitrage opportunities, trying to guaranteee that the trade is possible before even attempting to execute it.
* `TradingBot.sol` : a smart contract that gets called by the node app only when a profitable arbitrage is found, it will borrow funds with a flashloan and execute trades on decentralized exchanges.

## Basic Setup

### Deploy TradingBot.sol

In order to interact with the decentralized exchanges' smart contracts, users need to deploy a smart contract of their own which handles all of the logic, from flashloans to trading.

We assume that readers are already familiar with deploying a smart contract using the Remix IDE, but we have summed up the steps below:

* Install Metamask browser extension
* Create an Ethereum Mainnet account with some ETH for paying gas fees.
    * Don't use your personal account for this, create a new account for the bot in order to limit accidental losses (more on that later).
* Go to [Remix](https://remix.ethereum.org/) online IDE
    * Paste the smart contract solidity code
    * Compile the code using compiler version 0.5.17.
    * Deploy with an initial 100 wei, which is enough for 100 flashloans on dYdX.

A note on gas limit; when deploying the smart contract on Mainnet, users must be aware that the gas limit required will be very high due to the complexity of the contract, i.e. it will probably cost $100 or more worth of ETH. Use the recommended gas limit from metamask, if you decrease the recommended amount the contract will run out of gas and you will lose whatever amount of gas was used.
In order to pay less, users could try decreasing the gas price instead, and just wait longer for the transaction to get mined.

In order to deploy the contract on a testnet like Ropsten or Rinkeby, users should change all contract addresses like the ones of exchanges and erc20 tokens from both the node.js bot and the smart contract.

### Environment Setup

By cloning the project's code repository, users will find a file called ```.env.example``` inside the ```/src``` folder:

```
RPC_URL="https://mainnet.infura.io/v3/YOUR_API_KEY_HERE"
ADDRESS="0x..."
PRIVATE_KEY="0x..."
CONTRACT_ADDRESS="0x..."
GAS_LIMIT=3000000
GAS_PRICE=200
ESTIMATED_GAS=1700000
```

Fill in the missing fields:

* `RPC_URL` : the public address of an Ethereum node, the easiest one to set up is the Infura RPC provider, register an account in order to get an API key.
* `ADDRESS`  and `PRIVATE_KEY`: fill in the public Ethereum address of the bot account, and its corresponding private key.
* `CONTRACT_ADDRESS` : paste in the smart contract's address that was returned from Remix after the deployment step.
* `GAS_LIMIT` : how much gas the contract is allowed to use, leave as 3000000 or decrease to 2000000 which should be fine
* `GAS_PRICE` : change this depending on how fast you want the transaction to be mined, see https://ethgasstation.info/ for info.
* `ESTIMATED_GAS` : leave as is

Now the bot is ready to be run! From the project's root directory use the node command:

```
node src/index.js
```

## TradingBot.sol

We will begin by explaining how the smart contract works first. 

The actual contract for the trading bot starts on line 199 where the `TradingBot` contract is created inheriting from the `DyDxFloashLoan` contract, anything above that are just libraries, for example, we declare the interface for the 1inch contract called `IOneSplit`, which includes two functions that will be used by the bot. ```swap``` function is where you actually make a trade. ```getExpectedReturn``` is for [querying the interface](https://etherscan.io/address/0xc586bef4a0992c495cf22e1aeee4e446cecdee0e#readContract) in order to find out the expected return rate.

### Constructor

The contract's ```constructor``` function which will get executed during the contract deployment.
Inside of the constructor there are two things happening. First, ```getWeth``` turns any ETH that is sent to it into WETH, i.e. just calls the WETH contract.

```
(bool success, ) = WETH.call.value(_amount)("");
```

```approveWeth``` approves the 0x staking proxy, the proxy is the fee collector for 0x, i.e. we will use WETH in order to pay for trading fees, remember that erc20 tokens must be approved by the owner in order for the smart contract to spend them.

```
IERC20(WETH).approve(ZRX_STAKING_PROXY, _amount);
```

Finally, the constructor sets the owner so that the deployer will be the only person who is able to call the functions in this smart contract.

```
OWNER = msg.sender;
```

### Flashloan

The entrypoint of the entire arbitrage is the ```getFlashloan``` function, which is what the node.js app will call whenever a profitable arbitrage is found. All of the parameters needed will be passed in to that function which will in turn call the trade functions once the flashloan is received and so on. All of the parameters for this main function will be constructed and passed in from the client script:

```
address flashToken, // token address that you want to get loan for, i.e. WETH
uint256 flashAmount, // amount of flashloan, however much you need to do the arb, this value is currently set to 10,000 WETH in the client script
address arbToken, // token you want to arbitrage, i.e. DAI
bytes calldata zrxData, // raw data for 0x order which the bot compiles
uint256 oneSplitMinReturn, // minimum amount that you want to get back from 1inch, also generated by the bot code
uint256[] calldata oneSplitDistribution // distribution on 1inch exchange
```

```callFunction``` gets called from the dYdX smart contract; our smart contract has to deploy the ```callFunction``` in order to receive a flashloan from dYdX.

```
uint256 balanceAfter = IERC20(flashToken).balanceOf(address(this));
```

```balanceAfter``` is your token balance after you got the loan. If the loan was successful, the ```_arb``` function gets called

```
function _arb(address _fromToken, address _toToken, uint256 _fromAmount, bytes memory _0xData, uint256 _1SplitMinReturn, uint256[] memory _1SplitDistribution) internal {
    // Track original balance
    uint256 _startBalance = IERC20(_fromToken).balanceOf(address(this));

    // Perform the arb trade
    _trade(_fromToken, _toToken, _fromAmount, _0xData, _1SplitMinReturn, _1SplitDistribution);

    // Track result balance
    uint256 _endBalance = IERC20(_fromToken).balanceOf(address(this));

    // Require that arbitrage is profitable
    require(_endBalance > _startBalance, "End balance must exceed start balance.");
    }
```

The above code tracks the balance of our smart contract before and after the ```_trade``` function, which is where the entire arbitrage happens. If the end balance is not greater than the start balance, the operation will revert.

```_trade``` has two functions, one for the 0x trade and one for the 1inch trade. It    tracks ```_beforeBalance```, performs the trade on 0x with ```_zrxSwap```, checks ```_afterBalance```, takes that balance and swaps it on 1inch with  ```_oneSplitSwap```.

```
function _trade(address _fromToken, address _toToken, uint256 _fromAmount, bytes memory _0xData, uint256 _1SplitMinReturn, uint256[] memory _1SplitDistribution) internal {
        // Track the balance of the token RECEIVED from the trade
        uint256 _beforeBalance = IERC20(_toToken).balanceOf(address(this));

        // Swap on 0x: give _fromToken, receive _toToken
        _zrxSwap(_fromToken, _fromAmount, _0xData);

        // Calculate the how much of the token we received
        uint256 _afterBalance = IERC20(_toToken).balanceOf(address(this));

        // Read _toToken balance after swap
        uint256 _toAmount = _afterBalance - _beforeBalance;

        // Swap on 1Split: give _toToken, receive _fromToken
        _oneSplitSwap(_toToken, _fromToken, _toAmount, _1SplitMinReturn, _1SplitDistribution);
    }
```

### 0x swap

```zrxSwap``` first approves 0x to spend the erc20 tokens, fills the order, then resets approval.

```
address(ZRX_EXCHANGE_ADDRESS).call.value(msg.value)(_calldataHexString);
```

```_callDataHexString``` is just some data encoded into a raw byte string (tuple) that gets sent to the 0x contract as a single argument.
The 0x smart contract interface is so big and you don't need to keep it all inside your smart contract just to make a trade, so instead we generate all the data on the client side and then pass to the function.

### 1Inch  swap

The same logic applies to the 1inch contract; approve tokens, trade, reset approval. We use the swap function defined in the 1inch ocontract interface:

```
_oneSplitContract.swap(_fromIERC20, _toIERC20, _amount, _minReturn, _distribution, FLAGS);
```

### Profit!

After that, execution returns to the ```_arb``` function, which requires that a profit was made in order to repay the flashloan, if no profit was made the entire call sequence will revert.

```
require(_endBalance > _startBalance, "End balance must exceed start balance.");
```

Finally, the contract owner can withdraw tokens in the contract by passing in the token address to the ```withdrawToken``` function.

A ```withdrawEther``` function also exists, in case ETH is sent to this contract by mistake.


## index.js

The bulk of the entire bot is inside the ```index.js```, the bot is basically a node.js server that continuously runs, you can put it on a server like AWS or Heroku which supports express.js out of the box.

### Basic setup

At the very top of the file, we begin by importing the required libraries, the most important ones are:

* 'dotenv' to read the ```.env``` file containing sensitive information
* 'express' to run the server
* 'player' to play a sound when a profitable arb is found
* 'web3' to talk to the smart contracts
* 'axios' to make web requests for the 0x api
* 'moment' for working with time
* 'lodash' for doing javascript operations.

Next, Express server is set up to run on port 5000 and Web3 is inatantiated with the private key defined in the .env file.

Next all of the contract's abis and public addresses are declared, there is no need to change them unless for adding new exchanges to the bot. Abis are just JSON representations of the smart contracts, needed so that the web3.js can create raw byte strings for calling the smart contract functions. The contract is created with ABI plus the contract's address and is basically a JS version of the smart contract.

* The ERC20 abi is just for basic erc20 operations.
* TRADER_ABI is the trading bot abi. the contract address comes from the .env file as it should be private.
* FILL_ORDER abi is for 0x, it contains literally just the ```fillOrder``` function from 0x.

After that, there are just some JavaScript helper functions that will be useful later on.

### App Entry Point

The app actually starts at the very end of the file:

```
const marketChecker = setInterval(async () => { await checkMarkets() }, POLLING_INTERVAL)
```

 ```setInterval``` is just a JS function that will loop and check the orders on 0x to see if any of those could be dumped on 1inch. The ```POLLING_INTERVAL``` is set to 3 seconds and can be hardcoded:

```
const POLLING_INTERVAL = process.env.POLLING_INTERVAL || 3000 // 3 seconds
```

At every loop, the ```checkMarkets``` function will be called, which is the starting point of all the bot's logic.

Inside the ```checkMarkets``` function, there are a couple of checks to avoid checking again if another operation is already in progress.

First, if it's already checking then stop doing that, for example should the function take longer than 3 seconds to execute.

```
if(checkingMarkets) {
    return
}
```

If profitable arbitrage is found it is going to stop checking in order to alow for the smart contract calls to execute.

```
if(profitableArbFound) {
    clearInterval(marketChecker)
}
```

This is really just a safety measure, for example, imagine that a profitable arbitrage is found and execution goes to the smart contract, however the transaction gets reverted because somebody else filled the 0x order before you which will be the case if everybody reading this article was running the bot; in this case you will lose the gas fees, which given the complexity of the smart contract times the gas price could easily cost $30 worth of ETH.
Now imagine that the above safety check wasn't in place; the bot would keep executing the same (partially filled) order over and over until all of the account's money is spent in gas fees!
This was just an example; the bot is currently not production ready and for this reason the safety check was put there to avoid readers losing all their funds.

Next, the smart contract calls the main function where all the arbitrage logic unfolds, passing in the symbols for the desired trading pair, i.e. WETH and DAI.

```
await checkOrderBook(WETH, DAI)
```

This is the point in the code where users could add more trading pairs, by simply calling the ```checkOrderBook``` function again with another trading pair. The order of the pair matters, in this example WETH will be that base asset for the arbitrage, while DAI will be the quote asset, i.e. :

* get a WETH flashloan from dYdX
* trade WETH for DAI on 0x
* trade DAI back to WETH on 1inch

### Fetching 0x orderbook

Inside the ```checkOrderBook```, two constants are assigned to the addresses of the base asset symbol (WETH) and the quote asset (DAI). These addresses are dynamically added to the [0x API url](https://api.0x.org/sra/v3/orderbook?baseAssetData=0xf47261b0000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2&quoteAssetData=0xf47261b00000000000000000000000006b175474e89094c44da98b954eedeac495271d0f&perPage=1000) (click to see data returned by 0x) in order to retrieve the orderbook using the Axios library:

```
await axios.get(`https://api.0x.org/sra/v3/orderbook?baseAssetData=...&quoteAssetData=...`)
```

Next, the JavaScript `map` function iterates through all of the bids to see if any of them (the json object corresponding to an order) has an opportunity, by calling the `checkArb` function.

```
bids.map((o) => {
    checkArb({ zrxOrder: o.order, assetOrder: [baseAssetSymbol, quoteAssetSymbol, baseAssetSymbol] })
  })
```

The `checkArb` function will be called for each order retrieved from the 0x orderbook and determine whether it could be turned around for profit passing in the order details to 1inch.

### Skipping orders

Before getting the expected return rate from the 1inch smart contract there is some housekeeping to do with the 0x order.

####Skip if order was already checked
First, by keeping a list called `checkedOrders` of order IDs in order not to check them twice three seconds later. Users could remove this check but should add some logic in order to avoid submitting the same order multiple times and so forth.

####Skip if order has taker fee
Next, there is another simplistic check, i.e. to skip the 0x order if the order has a maker or taker fee. Factoring in taker fees into the profitability calculation would require additional logic to be coded and therefore we've added this check in order to avoid filling orders that have already almost completely being filled, i.e. trying to buy more DAI than than what it's left in that order, resulting in an EVM exception.
In order to be more competitive with other users, add custom logic to factor in fees into the profitability calculation so that 0x orders with fees are not simply skipped.

####Skip if order is partially filled
Finally, the bot checks whether orders have been partially filled, if so it will skip the order. Again, custom logic can be added to handle this case and remove this check.

```
const orderTuple = [
    zrxOrder.makerAddress,
    zrxOrder.takerAddress,
    zrxOrder.feeRecipientAddress,
    zrxOrder.senderAddress,
    zrxOrder.makerAssetAmount,
    zrxOrder.takerAssetAmount,
    zrxOrder.makerFee,
    zrxOrder.takerFee,
    zrxOrder.expirationTimeSeconds,
    zrxOrder.salt,
    zrxOrder.makerAssetData,
    zrxOrder.takerAssetData,
    zrxOrder.makerFeeAssetData,
    zrxOrder.takerFeeAssetData
 ]

const orderInfo = await zrxExchangeContract.methods.getOrderInfo(orderTuple).call()

if(orderInfo.orderTakerAssetFilledAmount.toString() !== '0') {
  console.log('Order partially filled')
  return
}
```

The order tuple contains the data from the 0x order, see the [order message format](https://0x.org/docs/guides/v2-specification#order-message-format) on the 0x docs for more information. A tuple is a collection as arguments that gets passed in as one argument to the function.

The tuple is passed to the 0x `getOrderInfo` function, check the [0x Docs](https://0x.org/docs/guides/v3-specification#orderinfo) to see the data types returned by calling `getOrderInfo`, which includes the `uint` amount of order filled.

If the order from 0x has been partially filled we want to skip it too, again this can be optimized order to account for edge cases where there might be a filled order already.

### Fetching 1inch Expected Rate

So far the bot knows the 0x taker asset (WETH) amount for a specific order, but still needs the expected output amount in WETH that it would get back from 1inch.
This is where we pass in the order info to 1inch to see whether it could be a profitable arbitrage.

```
const oneSplitData = await fetchOneSplitData({
  fromToken: ASSET_ADDRESSES[assetOrder[1]],
  toToken: ASSET_ADDRESSES[assetOrder[2]],
  amount: zrxOrder.makerAssetAmount,
})

const outputAssetAmount = oneSplitData.returnAmount
```

This function takes three arguments, the addresses of the maker and taker assets, as well as the maker asset amount.
Remember that the `checkArb` function was called with the following asset order: `[baseAssetSymbol, quoteAssetSymbol, baseAssetSymbol]`, i.e. WETH, DAI, WETH. Therefore, the `fromToken` i.e. `assetOrder[1]` corresponds to DAI, which is our base asset for the 1inch trade, while `toToken` will be WETH.
`amount` is the amount of DAI that would be returned by the 0x order, i.e. the makerAssetAmount on 0x; the maker was the person who created the order, the taker would be us, but after the order is filled our smart contract will receive the makerAssetAmount worth of DAI from 0x.

For more information about 1inch `getExpectedReturn` function see the [Github page](https://github.com/CryptoManiacsZone/1inchProtocol#getexpectedreturn). For a visual representation of this function see our [previous article](https://medium.com/@extropy_io/coding-a-defi-arbitrage-bot-45e550d85089), where we queried the 1inch contract directly using the Etherscan frontend interface.

`outputAssetAmount` is the amount you get back from 1split, i.e. the final amount you get from the arbitrage, from this amount we need to subtract any other costs in order to determine whether the arbitrage will be profitable.

### Determining Arbitrage Profitability

The net profit calculation is as follows:

`
let netProfit = outputAssetAmount - inputAssetAmount - estimatedGasFee
`

Where `outputAssetAmount` is the amount returned by 1inch, `inputAssetAmount` is the amount returned by 0x and `estimatedGasFee` is calculated as `ESTIMATED_GAS` * `GAS_PRICE` variables defined in the `.env` file.

Finally, if net profit is greater than 0, the bot will log the arbitrage information to the console and call the smart contract with all the parameters needed.

```
const profitable = netProfit.toString() > '0'
if(profitable) {
    ...
    await trade(assetOrder[0], ASSET_ADDRESSES[assetOrder[0]], ASSET_ADDRESSES[assetOrder[1]], zrxOrder, inputAssetAmount, oneSplitData)
  }
```

### Calling TradingBot.sol

In this function, the bot formats all the parameters needed for calling the smart contract as described at the beginning of this article.

We begin by setting a few constant values, including the flashloan amount which is hardcoded to 10,000 WETH, which is well over 3 million dollars at the current price. Note that the flashloan fee doesn't scale with the amount, hence we just ask for a large amount just in case without necessarily using it all.
    
Next, the bot factors in slippage. Slippage refers to the difference between the trade's expected price and the actual price of the trade. Slippage is due to inefficiency in the DEX trades, e.g. if you're going to trade 1 ETH for 300 USDC with a 1% slippage means you'd get back 297 instead of 300, basically just take 1% off. The bot currently takes into account a 1% slippage.

The next bit of code generates the endoded parameters for filling the 0x order

```
const orderTuple = [
    orderJson....
  ]

const takerAssetFillAmount = FROM_AMOUNT
const signature = orderJson.signature
const data = web3.eth.abi.encodeFunctionCall(FILL_ORDER_ABI, [orderTuple, takerAssetFillAmount, signature])
```

The encoded `data` is basically just the function call for 0x exchange, in particular for the `fillOrder` for which we imported the `FILL_ORDER_ABI`.

```
address(ZRX_EXCHANGE_ADDRESS).call.value(msg.value)(_calldataHexString);
```

The order signature is the signature of the account who put the order in 0x and signed it, i.e. the market maker.

Next, we get the minimum return and distribution from 0x which are required values when making a trade on 1inch. Then we apply  some math on `minReturn` in order to factor in slippage.

Finally, actually do the trade on the smart contract!

```
receipt = await traderContract.methods.getFlashloan(
  flashTokenAddress, // address flashToken,
  FLASH_AMOUNT, // uint256 flashAmount,
  arbTokenAddress, // address arbToken,
  data, // bytes calldata zrxData,
  minReturnWtihSplippage.toString(), // uint256 oneSplitMinReturn,
  distribution, // uint256[] calldata oneSplitDistribution
).send({
  from: process.env.ADDRESS,
  gas: process.env.GAS_LIMIT,
  gasPrice: web3.utils.toWei(process.env.GAS_PRICE, 'Gwei')
})
console.log(receipt)
}
```

## Customizing The Bot

As explained before, if everyone turned the bot on right now on this particular pair there will be competition. This is why it is essential to customize this bot and not just run it as is. Here are a few things that readers can do in order to make their bot more competitive and actually make money:

### Add New trading pairs

Be it from the 0x order book or from other DEXes.

* Update asset symbols and addresses, asset addresses could be found online in the project's docs or on etherscan.
* For each new pair, call the `checkOrderbook` function again, e.g. `checkOrderbook(USDC, WETH)`
* See [https://info.uniswap.org/pairs](https://info.uniswap.org/pairs) for ideas on trading pairs available
* Currently we are limited by the currencies available for flashloan on the dYdX platform, these are also listed in the smart contract imported in TradingBot.sol:

```
contract DyDxFlashLoan is Structs {
    DyDxPool pool = DyDxPool(0x1E0447b19BB6EcFdAe1e4AE1694b0C3659614e4e);

    address public WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public SAI = 0x89d24A6b4CcB1B6fAA2625fE562bDD9a23260359;
    address public USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address public DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
```
    
### Optimize The Bot

This means handling all of the edge cases where we previously just skipped the 0x orders instead for simplicity, for example:

* Taker fees
* Partial fills
* Check orders again
* Handling failures to continue execution
* Execute multiple orders simultaneously
* Dynamically calculating gas fees

Doing this will allow users to be more competitive by filloing even smaller orders, or by filling all orders faster than their competitors.

### Add Exchanges

Unfortunately adding new exchanges means changing the smart contract, the same is true if you wanted to take flashloans from different providers other than dYdX.
Check out [defiprime](https://defiprime.com/exchanges) for a list of exchanges.

### Add Strategies

It is up to the user which strategy to use, whether they'd prefer to arbitrage between 0x and AMMs, between AMMs themselves, or even from AMMs back to 0x. You could use 1inch or any other DEX aggregator in order to do this.
