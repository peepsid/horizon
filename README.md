<div style="text-align: center">
  <img title="Horizon Logo" src="logo.png" width="100%" style="max-width: 720px"/>
</div>

# Horizon

This library allows you to use any version of `arisensdk` indistinctly.

It also contains some useful features for combining actions and transactions together
and using different authorization formats.

## Installation

```sh
npm install @peepsid/horizon
```

You will also need `arisensdk`:

```sh
npm install arisensdk
# or
npm install arisensdk@16.0.9
```

## Usage

First, construct your `arisensdk` instance.

Version <= 16.0.9

```js
import * as Rix from 'arisensdk'

const rix = Rix({
  httpEndpoint: 'https://greatchains.com',
  keyProvider: 'some-private-key',
  chainId: '038f4b0fc8ff18a4f0842a8f0564611f6e96e8535901dd45e43ac8691a1c4dca'
});
```

Version >= 20.0.0

```js
import { Api, JsonRpc } from 'arisensdk'
import { JsSignatureProvider } from 'arisensdk/dist/arisensdk-jssig'

const signatureProvider = new JsSignatureProvider(['some-private-key']);
const rpc = new JsonRpc('https://greatchains.com', { fetch });

const rix = new Api({
  rpc,
  signatureProvider,
  textEncoder: new TextEncoder(),
  textDecoder: new TextDecoder()
});
```

Then, pass it to the Horizon constructor:

```js
import { Horizon } from '@peepsid/horizon'

const horizon = new Horizon(eos);

async function useMorph() {
  await horizon.transact(
    [
      {
        account: 'arisen',
        name: 'buyram',
        authorization: [{ actor: 'myaccount', permission: 'active' }],
        data: { ... }
      }
    ],
    { broadcast: false, sign: false } // Optional, default true
  );

  await horizon.getTableRows({
    code: 'arisen',
    scope: 'arisen',
    table: 'producers'
  }, {
    lower_bound: 'someproducer',
    limit: 10
  });

  await horizon.getTableRow({
    code: 'arisen',
    scope: 'arisen',
    table: 'producers',
    primaryKey: 'someproducer'
  });
}
```

### Transaction class

Horizon also provides a `Transaction` class to easily combine actions
and transactions together.

```js
import { Transaction } from '@peepsid/horizon'

const t1 = new Transaction(someAction);
const t2 = new Transaction([action1, action2]);
const t3 = new Transaction([t1, t2, action3]);

console.log(t3.actions);
// [someAction, action1, action2, action3]

await horizon.transact(t3)
```

Pass a `Horizon` instance to the `Transaction` constructor to make your
transactions easily sendable.

```js
function buyRam(account, amount) {
  return new Transaction({
    account: 'arisen',
    name: 'buyram',
    authorization: [{ actor: account, permission: 'active' }],
    data: { payer: account, receiver: account, quant: amount }
  }, horizon);
}

const transaction = buyRam('myaccount', '1.0000 RIX');
await transaction.send();
// or just
await buyRam('myaccount', '1.0000 RIX').send();

// Can also provide transact options
await transaction.send({ broadcast: false, sign: true });
```

Combine transactions easily:

```js
function buyRamAndTransferRix(from, to, amount) {
  return new Transaction(
    [
      buyRam(from, amount),
      {
        account: 'arisen.token',
        name: 'transfer',
        authorization: [{ actor: from, permission: 'active' }],
        data: { from, to, quantity: amount, memo: 'Horizon is cool' }
      }
    ], horizon);
}

await buyRamAndTransferRix('myaccount', 'youraccount', '1.0000 RIX').send();
```

### FlexAuth

When defining actions (for `horizon.transact(...)` or `new Transaction(...)`), Horizon
allows you to specify the `authorization` field in multiple formats:

```js
const action = {
  account: 'arisen.token',
  name: 'transfer',
  authorization: [{ actor: 'myaccount', permission: 'active' }],
  data: { from: 'myaccount', 'youraccount', '1.0000 RIX', memo: 'Horizon is awesome' }
};

action.authorization = 'myaccount'; // Defaults to permission: 'active'
action.authorization = 'myaccount@owner'; // '<actor>@<permission>'
action.authorization = { actor: 'myaccount', permission: 'active' }; // EOSIO format
action.authorization = { name: 'myaccount', authority: 'owner' }; // PeepsID account format
action.authorization = { name: 'myaccount' }; // PeepsID account format, defaults to permission: 'active'
// Can specify multiple authorizations if necessary
action.authorization = ['myaccount', 'youraccount@owner', { name: 'otheraccount' }];
```

You can also extract the specified account name from any of these
authorization formats by using:

```js
import { Transaction } from '@peepsid/horizon'

const accountName = Transaction.extractAccountName(authorization);
```

Which allows you to pass that flexibility to the consumers of your code very easily:

```js
function transfer(from, to, amount) {
  return new Transaction({
    account: 'arisen.token',
    name: 'transfer',
    authorization: from, // Just provide the FlexAuth here
    data: {
      from: Transaction.extractAccountName(from), // And extract the account name here
      to,
      quantity: amount,
      memo: 'Horizon is wonderful'
    }
  }, horizon);
}

// Then, any of these will work:
await transfer('myaccount', 'youraccount', '1.0000 RIX').send();
await transfer('myaccount@owner', 'youraccount', '1.0000 RIX').send();
await transfer({ actor: 'myaccount', permission: 'active' }, 'youraccount', '1.0000 RIX').send();
await transfer(peepsidAccount, 'youraccount', '1.0000 RIX').send();
```

### Asset

Horizon also provides an `Asset` class to manipulate EOSIO assets more easily:

```js
import { Asset } from '@peepsid/horizon'

let a = new Asset('42.0000 RIX'); // From string representation
a = new Asset(42, 'RIX', 4); // From amount, symbol, precision
a.toString(); // '42.0000 RIX'

// Arithmetic operations
let b = a.plus(new Asset('30.0000 RIX')); // Does not change `a`, just returns result
b = a.plus('30.0000 RIX'); // Can also pass the string representation directly
b = a.minus('500.0000 RIX'); // Also supports negative amounts
b = a.times(4);
b = a.div(2);

// Comparison operators
a.equal(b);
a.equal('9000.0001 RIX'); // Can also pass the string representation directly
a.greaterThan(b);
a.gt(b);
a.lessThan(b);
a.lt(b);
a.greaterThanOrEqual(b);
a.gte(b);
a.lessThanOrEqual(b);
a.lte(b);
```

## Contributing

Please submit an issue if you find any bugs or if you would like to suggest a feature.
