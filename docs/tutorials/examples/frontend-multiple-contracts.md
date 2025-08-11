---
id: frontend-multiple-contracts
title: Frontend Interacting with Multiple Contracts
sidebar_label: Interact with Multiple Contracts
description: "Learn how to build a frontend that interacts with multiple NEAR smart contracts simultaneously, including querying data and dispatching multiple transactions."
---
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import {CodeTabs, Language, Github} from "@site/src/components/codetabs"

This example showcases how to interact with multiple contracts from a single frontend.

Particularly, this example shows how to:

1. Query data from multiple contracts.
2. Call methods in multiple contracts simultaneously.

---

## Query Data from Multiple Contracts

To query multiple contracts simply perform multiple `view` calls:

<Language value="js" language="ts">
  <Github fname="index.js"
        url="https://github.com/near-examples/frontend-multiple-contracts/blob/main/frontend/index.js"
        start="70" end="76" />
</Language>

---

## Dispatching Multiple Transactions

The `wallet` object enables to dispatch multiple transactions simultaneously. However, please notice that the transactions execute independently.

Dispatching multiple transactions at once is just a nice way to improve UX, because the user interacts with the wallet only once.

<Language value="js" language="ts">
  <Github fname="index.js"
          url="https://github.com/near-examples/frontend-multiple-contracts/blob/main/frontend/index.js"
          start="35" end="62" />
</Language>

In this example, the user signs two independent transactions:

1. A transaction to call `set_greeting` in our [Hello NEAR example](https://github.com/near-examples/hello-near-examples)
2. A transaction to call `add_message` in our [GuestBook example](https://github.com/near-examples/guest-book-examples)

:::caution
Even when the user accepts signing the transactions at the same time, the
transactions remain **independent**. This is, if one fails, the other is **NOT** rolled back.
:::

---

## Batch Actions

You can aggregate multiple [actions](../../smart-contracts/anatomy/actions.md) directed towards a same contract into a single transaction. Batched actions execute **sequentially**, with the added benefit that, if **one fails** then they **all** get reverted.

```js
  // Register a user and transfer them FT on a single take
  const REGISTER_DEPOSIT = "1250000000000000000000";

  const ftTx = {
    receiverId: FT_ADDRESS,
    actions: [
      {
        type: 'FunctionCall',
        params: {
          methodName: 'storage_deposit',
          args: { account_id: "<receiver-account>" },
          gas: THIRTY_TGAS, deposit: REGISTER_DEPOSIT
        }
      },
      {
        type: 'FunctionCall',
        params: {
          methodName: 'ft_transfer',
          args: { receiver_id: "<receiver-account>", amount: amount_in_yocto },
          gas: THIRTY_TGAS, deposit: 1 }
      }
    ]
  }

  // Ask the wallet to sign and send the transaction
  await wallet.signAndSendTransactions({ transactions: [ ftTx ] })
```
---

## 📚 Tutorial: Multi-Contract Flow in a Real dApp

In many dApps (especially DeFi), you need to:

1. **Read** data from multiple contracts (balances, app state, staking info).
2. **Perform** multiple independent transactions in one wallet prompt.
3. **Batch** several actions into a single transaction when they target the same contract.

Here’s a real-world style example.

---

### Step 1 — Query Multiple Contracts in Parallel

```js
const view = (contractId, method, args = {}) =>
  wallet.viewMethod({ contractId, method, args });

async function loadDashboard(accountId) {
  const [storageBalance, storageBounds, messages] = await Promise.all([
    view('ft.example.near', 'storage_balance_of', { account_id: accountId }),
    view('ft.example.near', 'storage_balance_bounds'),
    view('guest-book.testnet', 'getMessages', { from_index: 0, limit: 10 }),
  ]);

  return {
    isRegistered: !!storageBalance,
    storageMin: storageBounds?.min,
    messages,
  };
}
```

**Why:** View calls are cheap, parallelizable, and safe, they never change blockchain state.

---

### Step 2 — Batch Actions for Atomic Execution

If the user isn’t registered on an FT contract, register them **and** transfer tokens in one atomic transaction:

```js
const THIRTY_TGAS = '30000000000000';

async function registerAndTransferFT({ receiverId, amountYocto, registerDepositYocto }) {
  const tx = {
    receiverId: 'ft.example.near',
    actions: [
      {
        type: 'FunctionCall',
        params: {
          methodName: 'storage_deposit',
          args: { account_id: receiverId },
          gas: THIRTY_TGAS,
          deposit: registerDepositYocto
        },
      },
      {
        type: 'FunctionCall',
        params: {
          methodName: 'ft_transfer',
          args: { receiver_id: receiverId, amount: amountYocto },
          gas: THIRTY_TGAS,
          deposit: '1'
        },
      },
    ],
  };

  return wallet.signAndSendTransactions({ transactions: [tx] });
}
```

**Why:** All-or-nothing execution means no half-registered users without tokens.

---

### Step 3 — Multiple Independent Transactions in One Prompt

You can send **two or more transactions** to different contracts in the same wallet flow:

```js
async function postGuestBookMessage(text) {
  return {
    receiverId: 'guest-book.testnet',
    actions: [
      {
        type: 'FunctionCall',
        params: {
          methodName: 'addMessage',
          args: { text },
          gas: THIRTY_TGAS,
          deposit: '10000000000000000000000'
        },
      },
    ],
  };
}

async function combinedFlow(receiverId, amountYocto, registerDepositYocto, message) {
  const ftTx = {
    receiverId: 'ft.example.near',
    actions: [
      {
        type: 'FunctionCall',
        params: {
          methodName: 'storage_deposit',
          args: { account_id: receiverId },
          gas: THIRTY_TGAS,
          deposit: registerDepositYocto,
        },
      },
      {
        type: 'FunctionCall',
        params: {
          methodName: 'ft_transfer',
          args: { receiver_id: receiverId, amount: amountYocto },
          gas: THIRTY_TGAS,
          deposit: '1',
        },
      },
    ],
  };

  const guestBookTx = await postGuestBookMessage(message);

  return wallet.signAndSendTransactions({ transactions: [ftTx, guestBookTx] });
}
```

**Why:** Reduces prompts for better UX.

❗ **Note:** These transactions are **independent** if one fails, the other still runs.

---

## Rust Contract Snippets

A **Fungible Token contract** might implement:

```rust
#[near_bindgen]
impl FungibleTokenContract {
    #[payable]
    pub fn storage_deposit(&mut self, account_id: Option<AccountId>) {
        let id = account_id.unwrap_or(env::predecessor_account_id());
        assert!(
            self.accounts.insert(&id, &0u128).is_none(),
            "Already registered"
        );
    }

    #[payable]
    pub fn ft_transfer(&mut self, receiver_id: AccountId, amount: U128) {
        assert_one_yocto();
        let sender_id = env::predecessor_account_id();
        self.internal_transfer(&sender_id, &receiver_id, amount.into());
    }
}
```

A **Guest Book contract** might implement:

```rust
#[near_bindgen]
impl GuestBook {
    #[payable]
    pub fn add_message(&mut self, text: String) {
        let sender = env::predecessor_account_id();
        self.messages.push(Message { sender, text });
    }
}
```

---

## Note

* **Multiple view calls** → query many contracts quickly in parallel.
* **Batch actions** → atomic operations on the same contract.
* **Multiple transactions in one wallet prompt** → independent execution across contracts.
* **Storage staking** → always check `storage_balance_of` and attach the required deposit.

---


