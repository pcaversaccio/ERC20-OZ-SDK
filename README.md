# ERC20-OZ-SDK
We will learn how to deploy an ERC20 smart contract using the [OpenZeppelin SDK](https://github.com/OpenZeppelin/openzeppelin-sdk). We will also write a `TokenExchange` contract that will allow any user to purchase at a fixed exchange rate an ERC20 token in exchange for ETH. We will write the `TokenExchange` smart contract ourselves, but leverage the [ERC20 implementation](https://docs.openzeppelin.com/contracts/3.x/erc20) from [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/3.x/).

## Install
First, install [Node.js](http://nodejs.org/) and [npm](https://npmjs.com/). Then, install the OpenZeppelin SDK (globally) running:

```
npm install -g @openzeppelin/cli
```

> If you get an `EACCESS permission denied` error while installing, please refer to the [npm documentation on global installs permission errors](https://docs.npmjs.com/resolving-eacces-permissions-errors-when-installing-packages-globally). Alternatively, you may run `sudo npm install --unsafe-perm --global @openzeppelin/cli`, but this is highly discouraged and you should rather either use a node version manager or manually change npm's default directory.

## Setup
We recommend using the OpenZeppelin SDK through the `openzeppelin SDK` command line interface.

To start, create a directory for the project and access it:

```
mkdir TestERC20Token
cd TestERC20Token
```

Use `npm` to create a `package.json` file:

```
npm init -y
```

And initialize the OpenZeppelin SDK project:

```
npx oz init
```

Now it is possible to use `npx oz deploy` to create instances for these contracts that 
later can be upgraded and many more things.

Run `npx oz --help` for more details about thes and all the other functions of the
OpenZeppelin CLI.

## ERC20 Smart Contract
We will first get ourselves an ERC20 token. Instead of coding one from scratch, we will use the one provided by the [OpenZeppelin Contracts Ethereum Package](https://github.com/OpenZeppelin/openzeppelin-contracts-ethereum-package). An Ethereum Package is a set of contracts set up to be easily included in an OpenZeppelin project, with the added bonus that the contracts' code *is already deployed in the Ethereum network*. This is a more secure code distribution mechanism and also helps you save gas upon deployment.

To link the OpenZeppelin Contracts Ethereum Package into your project, simply run the following:
```
npx oz link @openzeppelin/contracts-ethereum-package
```

This command will download the Ethereum Package (bundled as a regular npm package) and connect it to your OpenZeppelin project. We now have all of OpenZeppelin Contracts at our disposal, so let us create an ERC20 token!
> Make sure you install `@openzeppelin/contracts-ethereum-package` and not the vanilla `@openzeppelin/contracts`. The latter is set up for general usage, while `@openzeppelin/contracts-ethereum-package` is tailored for being used with [OpenZeppelin Upgrades](https://docs.openzeppelin.com/upgrades/2.8/). This means that its contracts are [already set up to be upgradeable](https://docs.openzeppelin.com/upgrades/2.8/writing-upgradeable#use-upgradeable-packages).

Let us deploy an ERC20 token contract to our `development` network. Make sure to have a [Ganache](https://www.trufflesuite.com/ganache) instance running, or start one by running:

```
 npx ganache-cli --deterministic
```
For setting up the token, we will be using the [ERC20PresetMinterPauser](https://github.com/OpenZeppelin/openzeppelin-contracts-ethereum-package/blob/master/contracts/presets/ERC20PresetMinterPauser.sol) implementation provided by the OpenZeppelin package. We will *initialize* the instance with the token metadata (name, symbol) and then mint a large initial supply for one of our accounts.

Check the RPC server for your Ganache environment and adjust the correct port in the `network.js` file.
> Usually you have to adjust the port from 8545 to 7545.

![](images/ERC20_deployment.png)

Let us break down what we did in the command above. We first chose to create an instance of the `ERC20PresetMinterPauserUpgradeSafe` contract from the `@openzeppelin/contracts-ethereum-package` package we had linked before, and to create it in the local `development` network. We are then instructing the CLI to *initialize* it with the initial values needed to set up our token. This requires us to choose the appropriate `initialize` function, and input all the required arguments. The OpenZeppelin CLI will then atomically deploy and initialize the new instance in a single transaction.

We now have a working ERC20 token contract in our `development` network.

Next we get the accounts we have setup:

![](images/default_account.png)

Then we mint 100 TERC20 to our default account:

![](images/minting.png)
> The standard ERC20 smart contract has 18 decimals, i.e. 1 token = 10^18.

We can check that the initial supply was properly allocated by using the `balance` command. Make sure to use the address where your ERC20 token instance was created.

![](images/check_account.png)

Great! We can now write an exchange contract and connect it to this token when we deploy it.

## Token Exchange Smart Contract
In order to transfer an amount of tokens every time it receives ETH, our exchange contract will need to store the token contract address and the exchange rate in its state. We will set these two values during initialization, when we deploy the instance with `npx oz deploy`.

Because we are writing upgradeable contracts we cannot use [Solidity constructors](https://docs.openzeppelin.com/upgrades/2.8/proxies#the-constructor-caveat). Instead, we need to use *initializers*. An initializer is just a regular Solidity function, with an additional check to ensure that it can be called only once.

To make coding initializers easy, [OpenZeppelin Upgrades](https://docs.openzeppelin.com/upgrades/2.8/) provides a base `Initializable` contract, that includes an `initializer` modifier that takes care of this. You will first need to install it:

```
npm i @openzeppelin/upgrades
```
Now, let us write our exchange contract in `contracts/TokenExchange.sol`, using an *initializer* to set its initial state:

![](images/tokenexchange_contract.png)
> Solidity 0.6.8 introduces SPDX license identifiers so developers can specify the [license](https://spdx.org/licenses/) the contract uses. E.g. OpenZeppelin Contracts use the MIT license. SPDX license identifiers should be added to the top of contract files. The following identifier should be added to the top of your contract (example uses MIT license):
```
// SPDX-License-Identifier: MIT
```

Note the usage of the `initializer` modifier in the `initialize` method. This guarantees that once we have deployed our contract, no one can call into that function again to alter the token or the rate.

Let us now create and initialize our new `TokenExchange` contract:

![](images/tokenexchange_deployment.png)
> For Visual Studio Code users, if you get an `File import callback not supported` error due to the imported packages, consider adding the following to your VS Code settings:
```
"solidity.packageDefaultDependenciesContractsDirectory": "",
"solidity.packageDefaultDependenciesDirectory": "node_modules"
```

Our exchange is almost ready! We only need to fund it, so it can send tokens to purchasers. Let us do that using the `npx oz send-tx` command, to transfer the full token balance from our own account to the exchange contract. Make sure to replace the recipient of the transfer with the `TokenExchange` address you got from the previous command.

![](images/send_tokens_to_exchange.png)
All set! We can start playing with our brand new token exchange.

## Using Our Exchange
Now that we have initialized our exchange contract and seeded it with funds, we can test it out by purchasing tokens. Our exchange contract will send tokens back automatically when we send ETH to it, so let us test it by using the `npx oz transfer` command. This command allows us to send funds to any address; in this case, we will use it to send ETH to our `TokenExchange` instance:

![](images/send_eth.png)

> Make sure you replace the receiver account with the corresponding address where your `TokenExchange` was created.

We can now use `npx oz balance` again, to check the token balance of the address that made the purchase. Since we sent 0.1 ETH, and we used a 1:10 exchange rate, we should see a balance of 1 TERC20 (TestERC20Token).

![](images/balance_tokens.png)

Success! We have our exchange up and running, gathering ETH in exchange for our tokens.

## Upgrading the Exchange
We forgot to add a method to withdraw the funds from the token exchange contract! While this would typically mean that the funds are locked in there forever, we can upgrade the contract with the OpenZeppelin CLI to add a way to collect those funds.
> While upgrading a contract is certainly useful in situations like this, where you need to fix a bug or add a missing feature, it could still be used to change the rules of the game. For instance, you could upgrade the token exchange contract to alter the rate at any time. Because of this, it is important to have appropriate project governance in place.

Let us modify the `TokenExchange` contract to add a withdraw method, only callable by an `owner`.

![](images/tokenexchange_upgrade.png)

When modifying your contract, you will have to place the `owner` variable **after** the other variables ([learn more](https://docs.openzeppelin.com/upgrades/2.8/writing-upgradeable#modifying-your-contracts) about this restriction). Don not worry if you forget about it, the CLI will check this for you when you try to upgrade.
> If you are familiar with **[OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/3.x/)**, you may be wondering why we did not simply extend from `Ownable` and used the `onlyOwner` modifier. The issue is OpenZeppelin Upgrades does not support extending from now contracts in an upgrade (if they declare their own state variables). Again, the CLI will alert you if you attempt to do this. Refer to the [Upgrades documentation](https://docs.openzeppelin.com/upgrades/2.8/writing-upgradeable#modifying-your-contracts) for more info.

The only thing missing is actually *setting* the `owner` of the contract. To do this, we can add another function that we will call when upgrading, making sure it can only be called once:

![](images/tokenexchange_upgrade_2.png)

First, we compile the contract using
```
npx oz compile
```

We can now upgrade our token exchange contract to this new version, and call `setOwner` during the upgrade process. The OpenZeppelin CLI will take care of making the upgrade and the call atomically in a single transaction.

![](images/tokenexchange_upgrade_deploy.png)

Yes! We can now call withdraw from our default address to extract all ETH sent to the exchange.

![](images/tokenexchange_upgrade_withdraw.png)
>You can also upgrade dependencies from an Ethereum Package. Upon a new release of `@openzeppelin/contracts-ethereum-package`, if you want to update your ERC20 to include the latest fixes, you can just `oz link` the new version and use `npx oz upgrade` to get your instance to the newest code.

## Summary
We have built a more complex setup in this tutorial, and learned several concepts along the way. We introduced [Ethereum Packages](https://blog.openzeppelin.com/open-source-collaboration-in-the-blockchain-era-evm-packages/) as dependencies for our projects, allowing us to spin up a new token with little effort. 

We also presented some [limitations](https://docs.openzeppelin.com/upgrades/2.8/writing-upgradeable) of [how Upgrades works](https://docs.openzeppelin.com/upgrades/2.8/proxies), such as [initializer methods](https://docs.openzeppelin.com/upgrades/2.8/writing-upgradeable#initializers) as a replacement for constructors, and [preserving the storage layout](https://docs.openzeppelin.com/upgrades/2.8/writing-upgradeable#modifying-your-contracts) when modifying our source code. We also learned how to run a function as a migration when upgrading a contract.

# Appendix
## Connecting to the Rinkeby Test Network
Since we are using public nodes, we will need to sign all our transactions locally. We will use `@truffle/hdwallet-provider` to do this, setting it up with our mnemonic. We will also tell the provider how to connect to the test network by using the [Infura](https://infura.io/) endpoint.

Let us start by installing the provider:
```
npm i @truffle/hdwallet-provider
```

Then, we will update our `networks.js` file with a new connection to the test network. Here we will use Rinkeby, but you can use whichever you want:
```
const { projectId, mnemonic } = require('./secrets.json');
const HDWalletProvider = require('@truffle/hdwallet-provider');

 module.exports = {
   networks: {
     development: {
      ...
     },
    rinkeby: {
      provider: () => new HDWalletProvider(
        mnemonic, `https://rinkeby.infura.io/v3/${projectId}`
      ),
      networkId: 4,
      gasPrice: 10e9
    }
   },
 };
```
> See the `HDWalletProvider` documentation for information on [configuration](https://github.com/trufflesuite/truffle/tree/master/packages/hdwallet-provider) options.

Note in the first line that we are loading the project id and mnemonic from a `secrets.json` file, which should look like the following, but using your own values. Make sure to `.gitignore` it!
```
{
  "mnemonic": "pioneer tent curve wild ...",
  "projectId": "305c13705054a8d918ad77549e402c72"
}
```

We can now test out that this configuration is working by listing the accounts we have available for the Rinkeby network. Remember that yours will be different, as they depend on the mnemonic you used.

![](images/rinkeby_account.png)

We can also test the connection to the Infura node, by querying our account balance.

![](images/rinkeby_account_balance.png)

Since we have a non-zero balance, we are ready to deploy our smart contract on the Rinkeby test network:

![](images/rinkeby_deploy.png)

You can see your (already verified) contract on a block explorer such as [Etherscan](https://rinkeby.etherscan.io/address/0x72C1f72a98EF6adc092195725245BB7D546917AF).