# Overview

This repository contains a transaction manager contract, allowing its owner
to make multiple contract calls in one Ethereum transaction. This saves time,
but what's most important it allows things like multi-step arbitrage to be
executed in one transaction.

NOTE: this is great for testnets but it doesn't implement `auth` so anyone can invoke it. DO NOT USE ON MAINNET.

# Contract deployment

The `TxManager` contract takes no parameters.

Use Dapp (<https://github.com/dapphub/dapp>) to build and deploy
the contract:

```bash
dapp build
ETH_GAS=2000000 dapp create TxManager
```

# Contract usage

The contract has only one public function:

```
function execute(bytes script) { … }
```

The `script` parameter is a byte array representing the sequence of
contract calls to be made. It consists of multiple call records concatenated
together, whereas each record is built as follows:

```
+-----------------+---------------------+-----------------------...------+
|     address     |   calldata length   |         calldata               |
|                 |                     |                                |
|   (20 bytes)    |      (32 bytes)     |                                |
+-----------------+---------------------+-----------------------...------+

```

For example, if you want to make one call to `0x11111222223333344444333332222211111122222`,
the script may look like this `111112222233333444443333322222111111222220000000000000000000000000000000000000000000000000000000000000024a39f1c6c0000000000000000000000000000000000000000000000000000000000000064`,
the last part of it being calldata encoded with:

```bash
$ seth calldata 'cork(uint256)' 100
0xa39f1c6c0000000000000000000000000000000000000000000000000000000000000064
```

The script parameter should be a concatenation of records of all calls to be made.

## Typescript Example

```typescript
import Web3 from "web3";

function padBytes(web3: Web3, calldata: string) {
  return web3.utils.padRight(calldata, Math.ceil(calldata.length / 32) * 32);
}

/**
 * We need to align build string so web3 will convert it to bytes later. This might be avoided if we build whole calldata manually but since we want to avoid it this is the only way. Also, this forced us to add additional check in tx-manager to skip ill formatted calldata
 */
export function buildCalls(
  web3: Web3,
  calls: { address: string; calldata: any }[]
) {
  let finalCalldata = "";
  for (const call of calls) {
    const calldata = call.calldata.encodeABI().slice(2);

    finalCalldata +=
      //tslint:disable-next-line
      call.address.slice(2) +
      web3.eth.abi.encodeParameter("uint256", calldata.length / 2).slice(2) +
      calldata;
  }

  return "0x" + padBytes(web3, finalCalldata);
}

const calls = [
  { address: spot.address, calldata: spot.methods.poke(ilkCode) },
  { address: jug.address, calldata: jug.methods.init(ilkCode) }
];
const calldata = buildCalls(web3, calls);

await txManager.methods.execute(calldata).send();
```
