cardano-node run --topology /home/sergio/github/cardano-node/ff-topology.json \
                 --database-path /home/sergio/github/cardano-node/db \
                     --socket-path /home/sergio/github/cardano-node/db/node.socket \
		 --host-addr 81.149.31.222 \
		 --port 3001 \
		 --config /home/sergio/github/cardano-node/ff-config.json


 cardano-node run \
     --topology block-producing/ff-topology.json \
     --database-path block-producing/db \
     --socket-path block-producing/db/node.socket \
     --host-addr 81.149.31.222 \
     --port 8080 \
     --config block-producing/ff-config.json

 cardano-node run \
     --topology block-producing/ff-topology.json \
     --database-path block-producing/db \
     --socket-path block-producing/db/node.socket \
     --host-addr 81.149.31.222 \
     --port 8080 \
     --config block-producing/ff-config.json

 cardano-node run \
     --topology relay/ff-topology.json \
     --database-path relay/db \
     --socket-path relay/db/node.socket \
     --host-addr 81.149.31.222 \
     --port 8081 \
     --config relay/ff-config.json



[sergio-C:cardano.node.DnsSubscription:Error:738]
[sergio-C:cardano.node.IpSubscription:Error:44]
[sergio-C:cardano.node.DnsSubscription:Error:757]
[sergio-C:cardano.node.DnsSubscription:Error:751]
[sergio-C:cardano.node.DnsSubscription:Error:65]
needed to change the addr: link in topology.json for relay so to connect to the right network address (eg mainnet ff-testnet)

$ export CARDANO_NODE_SOCKET_PATH=/home/sergio/github/cardano-node/relay/db/node.socket

# check balance
 cardano-cli shelley query utxo \
     --address 00d7e29f8bb92b0dc04d346181abf30f830f2eb4995e31b6bd20d431988f3db315ed78d87afd72f31c3c4b53c5f9d34175fb53ffbf981b3b011b05a538bd39ae0b \
     --testnet-magic 42

 cardano-cli shelley query utxo \
--address $(cat payment.addr) \
--testnet-magic 42
# /check balance

curl -v -XPOST "https://faucet.ff.dev.cardano.org/send-money/00d7e29f8bb92b0dc04d346181abf30f830f2eb4995e31b6bd20d431988f3db315ed78d87afd72f31c3c4b53c5f9d34175fb53ffbf981b3b011b05a538bd39ae0b


## Create Key-Gen
     cardano-cli shelley address key-gen \
     --verification-key-file payment2.vkey \
     --signing-key-file payment2.skey
## Create Address
     cardano-cli shelley address build \
     --payment-verification-key-file payment2.vkey \
     --stake-verification-key-file stake.vkey \
     --out-file payment2.addr \
     --testnet-magic 42
## Get Protocol Parameters
     cardano-cli shelley query protocol-parameters \
     --testnet-magic 42 \
     --out-file protocol.json
## Get Current TIP
     cardano-cli shelley query tip --testnet-magic 42
## Specify TTL (Time To Live)]
<current tip (int)> + 1200 = <TTL (int)>
795346 + 1200 = 796546. So our TTL is 796546.
## Calculate The FEE
The transaction needs one (1) input: a valid UTXO from payment.addr, and two (2) outputs: The receiving address payment2.addr and an address to send the change back, in this case we use payment.addr

     cardano-cli shelley transaction calculate-min-fee \
     --tx-in-count 1 \
     --tx-out-count 2 \
     --ttl 1094160 \
     --testnet-magic 42 \
     --signing-key-file payment.skey \
     --protocol-params-file protocol.json

     > 167965
## Calculate Change
So we need to pay 167965 lovelace fee to create this transaction.

Assuming we want to send 100 ADA to payment2.addr spending a UTXO containing 1,000,000 ada (1,000,000,000,000 lovelace), now we need to calculate how much is the change to send back to payment.addr

     expr 1000000000000 - 100000000 - 167965
     > 999899832035
## Get Transaction Hash
We need the transaction hash and index of the UTXO we want to spend:

     cardano-cli shelley query utxo \
         --address $(cat payment.addr) \
         --testnet-magic 42

>                            TxHash                                 TxIx        Lovelace
> ----------------------------------------------------------------------------------------
> 4e3a6e7fdcb0d0efa17bf79c13aed2b4cb9baf37fb1aa2e39553d5bd720c5c99     4     1000000000000
## Build the Transaction
We write the transaction in a file, we will name it tx.raw.

Note that for --tx-in we use the following syntax: TxId#TxIx where TxId is the transaction hash and TxIx is the index and for --tx-out we use: TxOut+Lovelace where TxOut is the hex encoded address followed by the amount in Lovelace.

     cardano-cli shelley transaction build-raw \
     --tx-in 5f7fbf4c968577e487c818b20e1eee389a91e3f921bda913c56530bf2734438f#0 \
     --tx-out $(cat payment2.addr)+100000000 \
     --tx-out $(cat payment.addr)+999899832035 \
     --ttl 1094160 \
     --fee 167965 \
     --out-file tx.raw
## Sign Transaction
Sign the transaction with the signing key payment.skey and save the signed transaction in tx.signed

     cardano-cli shelley transaction sign \
     --tx-body-file tx.raw \
     --signing-key-file payment.skey \
     --testnet-magic 42 \
     --out-file tx.signed