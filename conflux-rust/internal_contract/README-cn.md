---
id: internal_contract
title: Internal Contract
custom_edit_url: https://github.com/Conflux-Chain/conflux-rust/edit/master/internal_contract/README.md
keywords:
  - conflux
  - contract
---

Conflux引入了一些内置的内部合约，以便更好地进行系统维护和链上治理。而本文档将介绍如何使用这些内部合约。

后续文档使用 [js-conflux-sdk](https://github.com/Conflux-Chain/js-conflux-sdk) 作为案例

## 通过赞助使用合约的方法

Conflux实现了一种赞助机制以补贴用户使用智能合约。因此，一个账户余额为0的新用户，只要执行赞助（通常由Dapps的运营商赞助），就能够直接调用智能合约。 通过引入内置的 SponsorControl 合约已记录对智能合约的赞助信息。

**SponsorControl** 智能合约为每个用户建立的合约维护以下信息:
+ `sponsor_for_gas`: 为提供燃料费用补贴的账户；
+ `sponsor_for_collateral`: 为存放抵押提供补贴的账户；
+ `sponsor_balance_for_gas`: 可用于燃料费用补贴的余额；
+ `sponsor_balance_for_collateral`: 可用于存放抵押补贴的余额；
+ `sponsor_limit_for_gas_fee`: 赞助者愿意为每笔交易提供的资助上限；
+ `whitelist`: 为合约愿意资助的正常账户清单，特殊的全0地址特指所有的正常账户。只有合约本身才有权更改该清单。

有两类资源可以被赞助：燃料费用及存储抵押物。

+ *对于燃料费用*: 如果一笔交易使用非空的 `sponsor_for_gas` 调用智能合约且交易发送者处于合约的 `whitelist` 列表内，且交易指定的燃料费用在 `sponsor_limit_for_gas_fee` 范围内，交易的燃料消耗将从合约的 `sponsor_balance_for_gas` 中支付（如果足够的话），而不是由交易发送者的账户余额支付，如果 `sponsor_balance_for_gas` 无法承担燃料消耗，则交易失败。否则，交易发送者应支付燃料费用。 
+ *For storage collateral*: If a transaction calls a contract with non-empty `sponsor_balance_for_collateral` and the sender is in the `whitelist` of the contract,  the collateral for storage incurred in the execution of the transaction is deducted from `sponsor_balance_for_collateral` of the contract, and the owner of those modified storage entries is set to the contract address accordingly. Otherwise, the sender should pay for the collateral for storage incurred in the execution.

### Sponsorship Update

Both sponsorship for gas and for collateral can be updated by calling the SponsorControl contract. The current sponsors can call this contract to transfer funds to increase the sponsor balances directly, and the current sponsor for gas is also allowed to increase the `sponsor_limit_for_gas_fee` without transferring new funds. Other normal accounts can replace the current sponsors by calling this contract and providing more funds for sponsorship.

To replace the `sponsor_for_gas` of a contract, the new sponsor should transfer to the contract a fund more than the current `sponsor_balance_for_gas` of the contract and set a new value for `sponsor_limit_for_gas_fee`. The new value of `sponsor_limit_for_gas_fee` should be no less than the old sponsor’s limit unless the old `sponsor_limit_for_gas_fee` cannot afford the old limit. Moreover, the transferred fund should be >= 1000 times of the new limit, so that it is sufficient to subsidize at least `1000` transactions calling C. If the above conditions are satisfied, the remaining `sponsor_balance_for_gas` will be refunded to the old `sponsor_for_gas`, and then `sponsor_balance_for_gas`, `sponsor_for_gas` and `sponsor_limit_for_gas_fee` will be updated according to the new sponsor’s
specification.

The replacement of `sponsor_for_collateral` is similar except that there is no analog of the limit for gas fee. The new sponsor should transfer to C a fund more than the fund provided by the current sponsor for collateral of the contract. Then the current `sponsor_for_collateral` will be fully refunded, i.e. the sum of `sponsor_balance_for_collateral` and the total collateral for storage used by the contract, and both collateral sponsorship fields are changed as the new sponsor’s request accordingly. A contract account is also allowed to be a sponsor.

### The Interfaces

The built-in contract address is `0x8ad036480160591706c831f0da19d1a424e39469`. The abi for the internal contract could be found [here](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/metadata/SponsorWhitelistControl.json) and [here](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/contracts/SponsorWhitelistControl.sol).

+ `set_sponsor_for_gas(address contract, uint upper_bound)`: If someone wants to sponsor the gas fee for a contract with address `contract`, he/she (it can be a contract account) should call this function and in the meantime transfer some tokens to the address `0x8ad036480160591706c831f0da19d1a424e39469`. The parameter `upper_bound` is the upper bound of the gas fee the sponsor will pay for a single transaction. The number of transfered tokens should be at least 1000 times of the `upper_bound`. The sponsor could be replaced if the new sponsor transfers more tokens and sets a larger upper bound. The current sponsor can also call the function to transfer more tokens to sponsor the contract. The `upper_bound` can be changed to a smaller one if current sponsor balance is less than the `upper_bound`.
+ `set_sponsor_for_collateral(address contract_addr)`: If someone wants to sponsor the CFS (collateral for storage) for a contract with address `contract`, he/she (it can be a contract account) should call this function and in the meantime transfer some tokens to the address `0x8ad036480160591706c831f0da19d1a424e39469`. The sponsor could be replaced if the new sponsor transfers more tokens. The current sponsor can also call the function to transfer more tokens to sponsor the contract.
+ `add_privilege(address[] memory)`: A contract can call this function to add some normal account address to the whitelist. It means that if the `sponsor_for_gas` is set, the contract will pay the gas fee for the accounts in the whitelist, and if the `sponsor_for_collateral` is set, the contract will pay the CFS (collateral for storage) for the accounts in the whitelist. A special address `0x0000000000000000000000000000000000000000` could be used if the contract wants to add all account to the whitelist.
+ `remove_privilege(address[] memory)`: A contract can call this function to remove some normal account address from the whitelist.

The transferred value when calling function `set_sponsor_for_gas` and `set_sponsor_for_collateral` represents the amount of tokens that the sender (new sponsor) is willing to pay. Every contract maintains its `whitelist` by calling `add_privilege` and `remove_privilege`.

### Examples

Suppose you have a simple contract like this.
```solidity
pragma solidity >=0.4.15;

import "https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/contracts/SponsorWhitelistControl.sol";

contract CommissionPrivilegeTest {
    mapping(uint => uint) public ss;

    function add(address account) public payable {
        SponsorWhitelistControl cpc = SponsorWhitelistControl(0x8ad036480160591706c831f0DA19D1a424e39469);
        address[] memory a = new address[](1);
        a[0] = account;
        cpc.add_privilege(a);
    }

    function remove(address account) public payable {
        SponsorWhitelistControl cpc = SponsorWhitelistControl(0x8ad036480160591706c831f0DA19D1a424e39469);
        address[] memory a = new address[](1);
        a[0] = account;
        cpc.remove_privilege(a);
    }

    function foo() public payable {
    }

    function par_add(uint start, uint end) public payable {
        for (uint i = start; i < end; i++) {
            ss[i] = 1;
        }
    }
}
```

After deploying the contract and the address is `contract_addr`, if someone wants to sponsor the gas consumption, he/she can send a transaction like below:
```javascript
const PRIVATE_KEY = '0xxxxxxx';
const cfx = new Conflux({
  url: 'http://testnet-jsonrpc.conflux-chain.org:12537',
  defaultGasPrice: 100,
  defaultGas: 1000000,
  logger: console,
});
const account = cfx.Account(PRIVATE_KEY); // create account instance

const sponsor_contract_addr = '0x8ad036480160591706c831f0da19d1a424e39469';
const sponsor_contract = cfx.Contract({
  abi: require('./contracts/sponsor.abi.json'),
  address: sponsor_contract_addr,
});
sponsor_contract.set_sponsor_for_gas(contract_addr, your_upper_bound).sendTransaction({
  from: account,
  value: your_sponsor_value
}).confirmed();
```

As for sponsor the storage collateral, you can simply replace the function `set_sponsor_for_gas(contract_addr, your_upper_bound)` to `set_sponsor_for_collateral(contract_addr)`.

After that you can maintain the `whitelist` for your contract using `add_privilege` and `remove_privilege`. The special address `0x0000000000000000000000000000000000000000` with all zeros means everyone is in the `whitelist`. You need to use it carefully.

```javascript
you_contract.add(white_list_addr).sendTransaction({
  from: account,
})

you_contract.remove(white_list_addr).sendTransaction({
  from: account,
})
```

After that the accounts in `whiltelist` will pay nothing while calling `you_contract.foo()` or `you_contract.par_add(1, 10)`.

## Admin Management

The **AdminControl** contract is introduced for better maintenance of other smart contracts, espeically which are generated tentatively without a proper destruction routine: it records the administrator of every user-established smart contract and handles the destruction on request of corresponding administrators.

The default administrator of a smart contract is the creator of the contract, i.e. the sender α of the transaction that causes the creation of the contract. The current administrator of a smart contract can transfer its authority to another normal account by sending a request to the AdminControl contract. Contract accounts are not allowed to be the administrator of other contracts, since this mechanism is mainly for tentative maintenance. Any long term administration with customized authorization rules should be implemented inside the contract, i.e. as a specific function that handles destruction requests.

At any time, the administrator `addr` of an existing contract has the right to request destruction of the contract by calling AdminControl. However, the request would be rejected if the collateral for storage of contract is not zero, or `addr` is not the current administrator of the contract. If `addr` is the current administrator of the contract and the collateral for storage of contract is zero, then the destruction request is accepted and
processed as follows:
1. the balance of the contract will be refunded to `addr`;
2. the `sponsor_balance_for_gas` of the contract will be refunded to `sponsor_for_gas`;
3. the `sponsor_balance_for_collateral` of the contract will be refunded to `sponsor_for_collateral`;
4. the internal state in the contract will be released and the corresponding collateral for storage refunded to owners;
5. the contract is deleted from world-state.

### The Interfaces

The contract address is `0x8060de9e1568e69811c4a398f92c3d10949dc891`. The abi for the internal contract could be found [here](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/metadata/AdminControl.json) and [here](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/contracts/AdminControl.sol).

+ `set_admin(address contract, address admin)`: Set the administrator of contract `contract` to `admin`. The caller should be the administrator of `contract` and it should be a normal account. Caller should make sure that `contract` should be an address of a contract and `admin` should be a normal account address. Otherwise, the call will fail.

+ `destroy(address contract)`: Perform a suicide of the contract `contract`. The caller should be the administrator of `contract` and it should be a normal account. If the collateral for storage of the contract is not zero, the suicide will fail. Otherwise, the `balance` of `contract` will be refunded to the current administrator, the `sponsor_balance_for_gas` will be refunded to `sponsor_for_gas`, the `sponsor_balance_for_collateral` will be refunded to `sponsor_for_collateral`.

### Examples

Consider you have deployed a contract whose address is `contract_addr`. The administrator can call `AdminControl.set_admin(contract_addr, new_admin)` to change the administrator and call `AdminControl.destroy(contract_addr)` to kill the contract. 

```javascript
const PRIVATE_KEY = '0xxxxxxx';
const cfx = new Conflux({
  url: 'http://testnet-jsonrpc.conflux-chain.org:12537',
  defaultGasPrice: 100,
  defaultGas: 1000000,
  logger: console,
});
const account = cfx.Account(PRIVATE_KEY); // create account instance

const admin_contract_addr = '0x8060de9e1568e69811c4a398f92c3d10949dc891';
const admin_contract = cfx.Contract({
  abi: require('./contracts/admin.abi.json'),
  address: admin_contract_addr,
});
// to change administrator
admin_contract.set_admin(contract_addr, new_admin).sendTransaction({
  from: account,
}).confirmed();

// to kill the contract
admin_contract.destroy(contract_addr).sendTransaction({
  from: account,
}).confirmed();
```

## Staking Mechanism

Conflux introduces the staking mechanism for two reasons: first, staking mechanism provides a better way to charge the occupation of storage space (comparing to “pay once, occupy forever”); and second, this mechanism also helps in defining the voting power in decentralized governance.

At a high level, Conflux implements a built-in **Staking** contract to record the staking information of all accounts. By sending a transaction to this contract, users (both external users and smart contracts) can deposit/withdraw funds, which is also called stakes in the contract. The interest of staked funds is issued at withdrawal, and depends on both the amount and staking period of the fund being withdrawn

### Interest Rate

The staking interest rate is currently set to 4% per year. Compound interest is implemented in the granularity of blocks.

When executing a transaction sent by account `addr` at block `B` to withdraw a fund of value `v` deposited at block `B'`, the interest is calculated as follows:

```
interest issued = v * (4% / 63072000)^T
```

where `T = BlockNo(B)−BlockNo(B')` is the staking period measured by number of blocks, and `63072000` is the expected number of blocks generated in `365` days with the target block time `0.5` seconds. 

### Staking for Voting Power

See the details in [Conflux Protocol Specification](https://confluxnetwork.org/developer/).

### The Interfaces

The contract address is `0x843c409373ffd5c0bec1dddb7bec830856757b65`. The abi for the internal contract could be found [here](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/metadata/Staking.json) and [here](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/contracts/Staking.sol).

+ `deposit(uint amount)`: The caller can call this function to deposit some tokens to Conflux Internal Staking Contract. The current annual interest rate is 4%.
+ `withdraw(uint amount)`: The caller can call this function to withdraw some tokens to Conflux Internal Staking Contract. It will trigger a interest settlement. The staking capital and staking interest will be transferred to the user's balance in time. All the withdrawal applications will be processed on a first-come-first-served basis according to the sequence of staking orders.
+ `vote_lock(uint amount, uint unlock_block_number)`: This function is related with Voting Rights in Conflux. Staking users can choose the voting amount and locking maturity by locking a certain amount of CFX in a certain maturity from staking. The `unlock_block_number` is measured in the number of blocks since genesis block.

### Examples

```javascript
const PRIVATE_KEY = '0xxxxxxx';
const cfx = new Conflux({
  url: 'http://testnet-jsonrpc.conflux-chain.org:12537',
  defaultGasPrice: 100,
  defaultGas: 1000000,
  logger: console,
});
const account = cfx.Account(PRIVATE_KEY); // create account instance

const staking_contract_addr = '0x843c409373ffd5c0bec1dddb7bec830856757b65';
const staking_contract = cfx.Contract({
  abi: require('./contracts/staking.abi.json'),
  address: staking_contract_addr,
});
// deposit some amount of tokens
staking_contract.deposit(your_number_of_tokens).sendTransaction({
  from: account,
}).confirmed();

// withdraw some amount of tokens
staking_contract.withdraw(your_number_of_tokens).sendTransaction({
  from: account,
}).confirmed();

// lock some tokens until some block number
staking_contract.vote_lock(your_number_of_tokens, your_unlock_block_number).sendTransaction({
  from: account,
}).confirmed();
```