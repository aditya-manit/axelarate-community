# axelarate-community
Tools to join the axelar network


## Disclaimer 
Axelar Network is a work in progress. At no point in time should you transfer any real assets using Axelar. Only use testnet tokens that you're not afraid to lose. Axelar is not responsible for any assets lost, frozen, or unrecoverable in any state or condition. If you find a problem, please submit an issue to this repository following the template. 


## Prerequisites
- Docker (https://docs.docker.com/engine/install/)

## Useful links
- Axelar faucet: http://faucet.testnet.axelar.network/
- Latest docker images: https://hub.docker.com/repository/docker/axelarnet/axelar-core, 
  https://hub.docker.com/repository/docker/axelarnet/tofnd 

## What you need
- Bitcoin testnet faucet to send some test BTC: https://testnet-faucet.mempool.co/
- Metamask 
- Ethereum Ropsten address (generate via Metamask) 


## Joining the Axelar testnet

Run the script `join/joinTestnet.sh`. 
```
Usage: joinTestnet.sh [flags]

Mandatory flags:

--axelar-core       Version of axelar-core docker image to run (Format: vX.Y.Z)
--tofnd             Version of tofnd docker image to run (Format: vX.Y.Z)

Optional flags:
-r, --root          Local directory to store testnet data in (IMPORTANT: this directory is removed and recreated if --reset-chain is set)

--reset-chain       Delete local data to do a clean connect to the testnet

```
See https://hub.docker.com/repository/docker/axelarnet/axelar-core and https://hub.docker.com/repository/docker/axelarnet/tofnd for the latest available versions of the docker images.

Once you join, at the terminal you should see blocks produced:

```
I[2021-03-17|02:56:53.933] Executed block                               module=state height=2737 validTxs=0 invalidTxs=0 
I[2021-03-17|02:56:53.945] Committed state                              module=state height=2737 txs=0 appHash=DCFEB4C1574D6ADC1CC61CEBA8B119CBC0BBB87EB16B94507F19A10305D453CD
I[2021-03-17|02:56:59.682] Executed block                               module=state height=2738 validTxs=0 invalidTxs=0
I[2021-03-17|02:56:59.691] Committed state                              module=state height=2738 txs=0 appHash=5867EC297F83BB40F419EEBF7EB1FD4405
...
```

## Instructions to mint and burn tokens
These instructions are a step by step guide to run commands to move an asset from a source to a destination chain and back. The assets are minted as wrapped ERC-20 assets on the destination chain. The commands are submitted to the Axelar Network that's responsible for (a) generating deposit/withdrawal addresses, (b) routing and finalizing transactions, and (c) minting/burning the corresponding assets. 

To perform these tests, you'll need some test Bitcoins on the Bitcoin testnet, and a destination Ethereum address on the Ethereum Ropsten Testnet. 

### Generating a key on Axelar 
1. Enter Axelar node: 

```
docker exec -it axelar-core sh
```

2. By default, the node has an account named validator. Find its address: 

```
axelarcli keys show validator -a
```

3. Go to axelar faucet and get some coins on your validator's address (Your node is not yet a validator for the purpose of this ceremony; it's just the name of the account). http://faucet.testnet.axelar.network/

4. Check that you received the funds

```
axelarcli q account {validator_addr}
```


### Mint ERC20 Bitcoin tokens on Ethereum

1. Create a deposit address on Bitcoin (to which you'll deposit coins later)

  ```
  axelarcli tx bitcoin link ethereum {ethereum Ropsten dst addr} --from validator -y -b block
-> returns deposit address
  ```
  
  e.g.,
  
  ```
  axelarcli tx bitcoin link ethereum 0xc1c0c8D2131cC866834C6382096EaDFEf1af2F52 --from validator -y -b block
  ```
  
  Look for `chain: Bitcoin, address: {btcaddress}`
  
2. External: send a TEST BTC on Bitcoin testnet to the deposit address specific above, and wait for 6 confirmations (i.e. the transaction is 6 blocks deep in the Bitcoin chain). 
  - ALERT: DO NOT SEND ANY REAL ASSETS
  - (https://testnet-faucet.mempool.co/ 
  - You can monitor the status of your deposit using the testnet explorer: https://blockstream.info/testnet/
 
3. Create verification json object for Axelar
  
  ```
  axelarcli q bitcoin txInfo {blockhash} {txID}:{voutIdx} 
  -> returns json of verification info for the given outpoint (copy the escaped string)
  ```
  
  e.g.,
  
  ```
  axelarcli q bitcoin txInfo 4ac9dc50dc1b952cb1efca1e634216da2f5e3a12b4a4a802ce0f6b1271876bd2  da5b2e8037ce4b95f40ada01c6c2cd3ccb806d0a952906130eb9b806f7887590:1
  ```
  
4. Verify the Bitcoin outpoint

  ```
  axelarcli tx bitcoin verifyTx {"verification info" (first line of output of previous cmd)} --from validator -y -b block
  ```
  
  e.g.,
  
  ```
  axelarcli tx bitcoin verifyTx "{\"OutPoint\":{\"Hash\":\"NxF/hGLGQZ6mTNyMWkYHPJ21E+2PMb1DV/beV7R9Gpk=\",\"Index\":1},\"Amount\":\"100000000\",\"BlockHash\":\"tPqsKekDOp5lW6QUl+YwlaD/3cmbQJwuUqgiNkqloQM=\",\"Address\":\"bcrt1qrnc097fuepeyrchganj4jzl2yuf5c0fg800uenr5h0d58emztxasusnk7p\",\"Confirmations\":\"21\"}"} --from validator -y -b block
  ```
  
  Wait for verification to be confirmed (~10 Axelar blocks, ~50 secs). 
  Eventually, you'll see something like this in the node terminal: 
  
  `threshold of 2/3 has been met for bitcoinVerifyTx7d097730bbeba835e21dc0d953d4b1c3e42a6bf0da03e70f01a6bb0c1b71183c:1:`
  
5. Trigger signing of the transfers to Ethereum

  ```
  axelarcli tx ethereum sign-pending-transfers --from validator -y -b block
  -> returns commandID of signed tx
  -> wait for sign protocol to complete (~10 blocks)
  ```
  
  Look for commandID and its valud in the output: `"key": "commandID",
    "value": "d5e993e407ff399cf2770a1d42bc2baf5308f46632fcbe209318acb09776599f"`
    
6. Send the previous command to Ethereum
  ```
  axelarcli q ethereum sendCommand {commandID} {address of account that should send this tx}
  ```
  e.g., for the testnet, we allow you to use our address as the sender = `0xE3deF8C6b7E357bf38eC701Ce631f78F2532987A` 
  ```
  axelarcli q ethereum sendCommand 96c8dba428dbb0ce94ebd49eb342a13e8844630d28f80b8d8708324f0642cb3d 0xE3deF8C6b7E357bf38eC701Ce631f78F2532987A
  ```
  So use the above command, and just replace the `commandID` with your own. 
   
You can now open Metamask, add the custom asset (Bitcoin) with contract address (ask Axelar on discord if you can't find it) and see the minted Bitcoin tokens appear in it. 

### Burn ERC20 wrapped Bitcoin tokens and obtain native Satoshi

To send wrapped Bitcoin back to Bitcoin, run the following commands:

1. Create a deposit address on Ethereum 

  ```
  axelarcli tx ethereum link bitcoin {bitcoin addr} satoshi --from validator -y -b block
  -> returns deposit address
  ```
  
  e.g., 
  ```
  axelarcli tx ethereum link bitcoin tb1qg2z5jatp22zg7wyhpthhgwvn0un05mdwmqgjln satoshi --from validator -y -b block
  ```
  
  Look for the Ethereum deposit address as the first outout in this line (`0x5CFE...`): 
  
  ```
  "successfully linked {0x5CFEcE3b659e657E02e31d864ef0adE028a42a8E} and {tb1qq8wnre6rzctec9wycrl2dq00m3avravslahc8v}"
  ```
  
2. External: send wrapped tokens to deposit address (e.g. with Metamask). You need to have some Ropsten testnet Ether on the address to send transactions. Wait for 30 Ethereum block confirmations. 

3. Verify the Ethereum transaction

  ```
  axelarcli tx ethereum verify-erc20-deposit {txID} {amount} {deposit addr} --from validator -y -b block
  -> wait for verification to be confirmed (~10 Axelar blocks)
  ```
  
  Here, amount should be specific in Satoshi. (For instance, 0.0001BTC = 10000)
  e.g., 
  
  ```
  axelarcli tx ethereum verify-erc20-deposit 0x01b00d7ed8f66d558e749daf377ca30ed45f747bbf64f2fd268a6d1ea84f916a 10000  0x5CFEcE3b659e657E02e31d864ef0adE028a42a8E --from validator -y -b block
  -> wait for verification to be confirmed (~10 Axelar blocks)
  ```
  
4. Trigger signing of all pending transfers to Bitcoin

  ```
  axelarcli tx bitcoin sign-pending-transfers {tx fee} --from validator -b block -y
  -> wait for sign protocol to complete (~10 Axelar blocks)
  ```
  
 e.g., 
 
  ```
  axelarcli tx bitcoin sign-pending-transfers 0.0001btc --from validator -b block -y
  ```
5. Submit the transfer to Bitcoin

  ```
  axelarcli q bitcoin send
  -> returns tx ID
  ```
  
🛑 **IMPORTANT: Verify outpoints of previous withdrawal tx (repeat for each outpoint)**

Without this step, other users of the testnet will be unable to withdraw their wrapped tokens. Be a good citizen and verify the outpoints!

1. Create verification json object for Axelar

  ```
  axelarcli q bitcoin txInfo {blockhash} {txID}:{voutIdx}
  -> returns json of verification info for the given outpoint (copy the escaped string)
  ```
  
e.g., 
  
  ```
  axelarcli q bitcoin txInfo 4ac9dc50dc1b952cb1efca1e634216da2f5e3a12b4a4a802ce0f6b1271876bd2 da5b2e8037ce4b95f40ada01c6c2cd3ccb806d0a952906130eb9b806f7887590:1
  ```
  
2. Verify the Bitcoin outpoint

  ```
  axelarcli tx bitcoin verifyTx {"verification info" (output of previous cmd)} --from validator -y -b block
  ```
  
  e.g., 
  
  ```
  axelarcli tx bitcoin verifyTx "{\"OutPoint\":{\"Hash\":\"NxF/hGLGQZ6mTNyMWkYHPJ21E+2PMb1DV/beV7R9Gpk=\",\"Index\":1},\"Amount\":\"100000000\",\"BlockHash\":\"tPqsKekDOp5lW6QUl+YwlaD/3cmbQJwuUqgiNkqloQM=\",\"Address\":\"bcrt1qrnc097fuepeyrchganj4jzl2yuf5c0fg800uenr5h0d58emztxasusnk7p\",\"Confirmations\":\"21\"}" --from validator -y -b block
  ```
