


##Run Block-Producer
     cardano-node run \
     --topology block-producing/ff-topology.json \
     --database-path block-producing/db \
     --socket-path block-producing/db/node.socket \
     --host-addr 81.149.31.222 \
     --port 8080 \
     --config block-producing/ff-config.json
##Run Relay
     cardano-node run \
     --topology relay/ff-topology.json \
     --database-path relay/db \
     --socket-path relay/db/node.socket \
     --host-addr 81.149.31.222 \
     --port 8081 \
     --config relay/ff-config.json


# Assign SOCKET
     $ export CARDANO_NODE_SOCKET_PATH=/home/sergio/github/cardano-node/relay/db/node.socket

# Get Faucet Money
     curl -v -XPOST "https://faucet.ff.dev.cardano.org/send-money/00d7e29f8bb92b0dc04d346181abf30f830f2eb4995e31b6bd20d431988f3db315ed78d87afd72f31c3c4b53c5f9d34175fb53ffbf981b3b011b05a538bd39ae0b

# check balance
      cardano-cli shelley query utxo \
     --address $(cat payment.addr) \
     --testnet-magic 42

# Make Transaction
### Create Key-Gen
     cardano-cli shelley address key-gen \
     --verification-key-file payment2.vkey \
     --signing-key-file payment2.skey
### Create Address
     cardano-cli shelley address build \
     --payment-verification-key-file payment2.vkey \
     --stake-verification-key-file stake.vkey \
     --out-file payment2.addr \
     --testnet-magic 42
### Get Protocol Parameters
     cardano-cli shelley query protocol-parameters \
     --testnet-magic 42 \
     --out-file protocol.json
### Get Current TIP
     cardano-cli shelley query tip --testnet-magic 42
### Specify TTL (Time To Live)]
-current tip- + 1200 = -TTL-
795346 + 1200 = 796546. So our TTL is 796546.
### Calculate The FEE
The transaction needs one (1) input: a valid UTXO from payment.addr, and two (2) outputs: The receiving address payment2.addr and an address to send the change back, in this case we use payment.addr

     cardano-cli shelley transaction calculate-min-fee \
     --tx-in-count 1 \
     --tx-out-count 2 \
     --ttl 1010392 \
     --testnet-magic 42 \
     --signing-key-file payment.skey \
     --protocol-params-file protocol.json

     > 167965
### Calculate Change
So we need to pay 167965 lovelace fee to create this transaction.

Assuming we want to send 100 ADA to payment2.addr spending a UTXO containing 1,000,000 ada (1,000,000,000,000 lovelace), now we need to calculate how much is the change to send back to payment.addr

     expr 100000000000 - 100000000 - 167965
     > 99899832035
### Get Transaction Hash
We need the transaction hash and index of the UTXO we want to spend:

     cardano-cli shelley query utxo \
         --address $(cat payment.addr) \
         --testnet-magic 42

     >                            TxHash                                 TxIx        Lovelace
     > ----------------------------------------------------------------------------------------
     > 4e3a6e7fdcb0d0efa17bf79c13aed2b4cb9baf37fb1aa2e39553d5bd720c5c99     4     1000000000000
### Build the Transaction
We write the transaction in a file, we will name it tx.raw.

Note that for --tx-in we use the following syntax: TxId#TxIx where TxId is the transaction hash and TxIx is the index and for --tx-out we use: TxOut+Lovelace where TxOut is the hex encoded address followed by the amount in Lovelace.

     cardano-cli shelley transaction build-raw \
     --tx-in 5f7fbf4c968577e487c818b20e1eee389a91e3f921bda913c56530bf2734438f#0 \
     --tx-out $(cat payment2.addr)+100000000 \
     --tx-out $(cat payment.addr)+99899832035 \
     --ttl 1099590 \
     --fee 167965 \
     --out-file tx.raw
### Sign Transaction
Sign the transaction with the signing key payment.skey and save the signed transaction in tx.signed

     cardano-cli shelley transaction sign \
     --tx-body-file tx.raw \
     --signing-key-file payment.skey \
     --testnet-magic 42 \
     --out-file tx.signed
### Submit Transaction
     cardano-cli shelley transaction submit \
             --tx-file tx.signed \
             --testnet-magic 42

# Register Stake Address
### Registration Certificate
     cardano-cli shelley stake-address registration-certificate \
     --stake-verification-key-file stake.vkey \
     --out-file stake.cert

### Calculate fees and deposit
This transaction has only 1 input (the UTXO used to pay the transaction fee) and 1 output (our payment.addr to which we are sending the change). This transaction has to be signed by both the payment signing key payment.skey and the stake signing key stake.skey; and includes the certificate stake.cert:

     cardano-cli shelley transaction calculate-min-fee \
     --tx-in-count 1 \
     --tx-out-count 1 \
     --ttl 1104402 \
     --testnet-magic 42 \
     --signing-key-file payment.skey \
     --signing-key-file stake.skey \
     --certificate-file stake.cert \
     --protocol-params-file protocol.json

     > 171309
### Calculate Amounts
The deposit amount can be found in the protocol.json under keyDeposit:

    ...
    "keyDeposit": 400000,
    ...
Query the UTXO:

    cardano-cli shelley query utxo \
        --address $(cat payment.addr) \
        --testnet-magic 42

    >                            TxHash                                 TxIx        Lovelace
    > ----------------------------------------------------------------------------------------
    > 57eae78ecae94b11bc734327ad083d78317da6fa476a113f411ab5c0bd4f18f9     1      99899832035
So we have 1000 ada, calculate the change to send back to payment.addr:

     expr 99899832035 - 171309 - 400000

     > 99899260726

### Submit the certificate with a transaction:
Build the transaction:

     cardano-cli shelley transaction build-raw \
     --tx-in 57eae78ecae94b11bc734327ad083d78317da6fa476a113f411ab5c0bd4f18f9#1 \
     --tx-out $(cat payment.addr)+99899260726 \
     --ttl 1103993 \
     --fee 171309 \
     --out-file tx.raw \
     --certificate-file stake.cert
Sign it:

     cardano-cli shelley transaction sign \
     --tx-body-file tx.raw \
     --signing-key-file payment.skey \
     --signing-key-file stake.skey \
     --testnet-magic 42 \
     --out-file tx.signed
And submit it:

     cardano-cli shelley transaction submit \
     --tx-file tx.signed \
     --testnet-magic 42

# Generate stake pool keys
Create a directory on your local machine to store your keys:

    mkdir pool-keys
    cd pool-keys

### 1. Generate __Cold__ Keys and a __Cold_counter__:

    cardano-cli shelley node key-gen \
    --cold-verification-key-file cold.vkey \
    --cold-signing-key-file cold.skey \
    --operational-certificate-issue-counter-file cold.counter

### 2. Generate VRF Key pair

    cardano-cli shelley node key-gen-VRF \
    --verification-key-file vrf.vkey \
    --signing-key-file vrf.skey

### 3. Generate the KES Key pair

    cardano-cli shelley node key-gen-KES \
    --verification-key-file kes.vkey \
    --signing-key-file kes.skey

### 4. Generate the Operational Certificate

We need to know the slots per KES period, we get it from the genesis file:

    cat ff-genesis.json | grep KESPeriod
    > "slotsPerKESPeriod": 3600,

So one period lasts 3600 slots.

Then we need the current tip of the blockchain:

We can use your relay node to query the tip:

    cardano-cli shelley query tip --testnet-magic 42
    > Tip (SlotNo {unSlotNo = 432571}) ...

Look for Tip `unSlotNo` value. In this example we are on slot 432571. So we have KES period is 120:

    expr 432571 / 3600
    > 120

To generate the certificate:

    cardano-cli shelley node issue-op-cert \
    --kes-verification-key-file kes.vkey \
    --cold-signing-key-file cold.skey \
    --operational-certificate-issue-counter cold.counter \
    --kes-period 120 \
    --out-file node.cert

# Register a Stake Pool
### Generate Stake pool registration certificate
Create a stake pool registration certificate:

     cardano-cli shelley stake-pool registration-certificate \
     --cold-verification-key-file ./pool-keys/cold.vkey \
     --vrf-verification-key-file ./pool-keys/vrf.vkey \
     --pool-pledge 99899260726 \
     --pool-cost 10000000 \
     --pool-margin 0.05 \
     --pool-reward-account-verification-key-file stake.vkey \
     --pool-owner-stake-verification-key-file stake.vkey \
     --testnet-magic 42 \
     --out-file pool.cert

### Generate delegation certificate (pledge)
We have to honor our pledge by delegating at least the pledged amount to our pool, so we have to create a delegation certificate to achieve this:

     cardano-cli shelley stake-address delegation-certificate \
     --stake-verification-key-file stake.vkey \
     --cold-verification-key-file ./pool-keys/cold.vkey \
     --out-file delegation.cert
This creates a delegation certificate which delegates funds from all stake addresses associated with key stake.vkey to the pool belonging to cold key cold.vkey. If we had used different staking keys for the pool owners in the first step, we would need to create delegation certificates for all of them instead.

### Submit the pool certificate and delegation certificate to the blockchain
Finally we need to submit the pool registration certificate and the delegation certificate(s) to the blockchain by including them in one or more transactions. We can use one transaction for multiple certificates, the certificates will be applied in order.

#### calculating the fees:
     cardano-cli shelley transaction calculate-min-fee \
     --tx-in-count 1 \
     --tx-out-count 1 \
     --ttl 1116314 \
     --testnet-magic 42 \
     --signing-key-file payment.skey \
     --signing-key-file stake.skey \
     --signing-key-file ./pool-keys/cold.skey \
     --certificate-file pool.cert \
     --certificate-file delegation.cert \
     --protocol-params-file protocol.json

     > 185037

Note how we included the two certificates in the call to calculate-min-fee and that the transaction will have to be signed by the payment key corresponding to the address we use to pay for the transaction, the staking key(s) of the owner(s) and the cold key of the node.

We will also have to pay a deposit for the stake pool registration. The deposit amount is specified in the genesis file:

     "poolDeposit": 500000000

check wallet balance

     cardano-cli shelley query utxo \
     --address $(cat payment.addr) \
     --testnet-magic 42

     > ...addr ix ..9899260726

calculate

     expr 99899260726 - 500000000 - 185037    


     > 99399075689

Now we can build the transaction:

     cardano-cli shelley transaction build-raw \
     --tx-in 887192471e31625a368c9e6277f30178272e8f041fd225c6a053f38e8a449fb5#0 \
     --tx-out $(cat payment.addr)+99399075689 \
     --ttl 1116314 \
     --fee 185037 \
     --out-file tx.raw \
     --certificate-file pool.cert \
     --certificate-file delegation.cert

Sign it:

     cardano-cli shelley transaction sign \
     --tx-body-file tx.raw \
     --signing-key-file payment.skey \
     --signing-key-file stake.skey \
     --signing-key-file ./pool-keys/cold.skey \
     --testnet-magic 42 \
     --out-file tx.signed

And submit:

     cardano-cli shelley transaction submit \
     --tx-file tx.signed \
     --testnet-magic 42

### Verify stake pool registration was indeed successful
     cardano-cli shelley stake-pool id --verification-key-file ./pool-keys/cold.vkey

will output your poolID. You can then check for the presence of your poolID in the network ledger state, with the following command:

     cardano-cli shelley query ledger-state --testnet-magic 42 | grep poolPubKey | grep bb6878d686a7d4adedfc43f24885503ca775fc004b2469dd419b5aa4b7272c14

but you will have to wait that the TTL you selected has passed