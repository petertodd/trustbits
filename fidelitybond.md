# Fidelity Bonds

## Assumptions

1) Fidelity-bond using clients will be SPV clients, and thus will only have
access to the block header data.

2) However there will some method by which these clients can determine if a
given tx hash is or is not part of the UTXO set for the current block. For
example a future UTXO tree proposal that was a protocol rule or cheating a bit
and having the "SPV" clients either computer the UTXO set themselves, or using
a trusted validating node as an oracle.

Note that for simplicity versioning has not been included yet in the proposed
data structures. Versioning will be added later.


## Common data structures

Proof that a Bitcoin transaction exists and is in the block chain. This is the
same data that the CMerkleTx class contains:

    message MerkleBitcoinTx {
        required bytes raw = 1; // the serialized transaction
        required uint32 block = 2; // block number
        required uint32 index = 3; // index
        repeated bytes branches = 4; // list of partial branches
    }

Note that a secure implementation *must* check that every element of branches
is exactly 32 bytes long.


## Contracts

At the protocol level a contract is just some data that can be hashed. What
that data actually represents is undefined, but will presumably be some sort of
promise like "I will not spam on foo-wiki" or "I will run this banking service
honestly" or even a conventional bond such as "This tx output is eligable for
interest from Foo Bank for issue $some_bond"

Of course, here we are assuming the contracts have a value, denoted in some
amount of Bitcoins, and thus trading them on the blockchain is desirable, so we
will tie the hash of the contract to specially formatted Bitcoin transactions.
Note that this is not a new idea; Mike Hearn, jgarzik and others on the forums
developed much of the ground work with various "smartcoin" and "colored coin"
concepts that this proposal builds upon.


### Off-chain transactions

Having said that a contract does not have to be traded on the blockchain: one
perfectly valid contract would be "The Bank of Foo will keep authoritive
records of all further contracts related to this portion of the bond." and in
that circumstance all further contract trading would be done off-chain,
guaranteed by The Bank of Foo. This construct would be appropriate for
low-value contracts, such as online forum anti-spam pledges, where minimum
transaction fees interfere with the granularity of the contract value desired.

The actual data representing contract movement will still be valid Bitcoin
transactions, however their existence will be guaranteed by some trusted
centralized ledger service rather than the distributed blockchain. Note how
this means that modulo transaction hash mutability the transactions can be
eventually put into the blockchain proper if appropriate fees are paid.

A future extension using this concept may, as an example, find it convenient to
add a field to the MerkleBitcoinTx with the identity of some trusted ledger to
be consulted.


### Contract Hash Calculation

For a given contract C using protocol identifier P the hash of the contract
shall use the SHA256 HMAC as defined in RFC2104:

    H(C) = SHA256(P ^ opad | SHA256(C ^ ipad))

The contract can be any sequence of bytes. The protocol identifier must be a
UUID that is unique to the protocol by which the contract is decoded. A
properly generated 16-byte UUID, represented in binary, is recommended.

The purpose of the protocol identifier is to ensure that under no circumstance
can a contract be crafted in such a way that it can be interpreted by two
different protocols at once. This is particularly important with fidelity
bonds, which must ensure that the value of the bond can only be used for one
purpose.

As an example the hash of the contract 'Hello World!', protocol
4870ac42-6eba-11e2-8f13-b12283093f41 is
b1a8da950fa270e57d2cb7cd0381e9044a97acafb11f31bd71d14e61170dd889



### Contract Blockchain Transactions

Like any transaction there are one or more txins, and one or more txouts. Zero
or more txins will represent existing contracts, and one or more txouts will
form new contracts. Contract txouts and change txouts are distinguished by the
LSB of the output value, with contract txouts having an even number of satoshis
(LSB=0) and change txouts an odd number (LSB=1)

A contract output will always have a P2SH scriptPubKey of the following form:

    OP_HASH160 [20-byte-hash-value] OP_EQUAL

The redeemScript is a standard multisig scriptPubKey:

    m {pubkey}...{pubkey} n OP_CHECKMULTISIG

The pubkeys for a contract txout have the contract hash encoded into them, as
suggested by Stefan Thomas.
(https://bitcointalk.org/index.php?topic=108423.msg1178438#msg1178438) The base
public ECDSA point P is multiplied by the contract hash H to generate P_H = P *
H Each original pubkey used is provided as part of the proof. The security of
the scheme is that given P_H and H calculating P is intractable by the
difficulty of the discrete logorithm, and additionally given H calculating C is
also intractable by the difficulty of generating SHA256 collisions.

Since the special txouts are marked by the LSB of the txout value, providing
extra txins is possible to pay for miner fees, as well as to securely transfer
ownership of a contract in exchange for Bitcoins using well-known multi-party
transaction signing mechanisms.


### Proving the existence of a contract

Given transaction T_n it must be possible to prove that the transaction is a
valid decendent of the transactions that created the contract. Additionally
contracts may be combined, and may even have multiple "sources" creating them,
as is the case with fidelity bonds.

    message ContractProofTxOut {
        required MerkleBitcoinTx tx = 1;
        required uint32 n_out = 2; // the txout number in question
        required bytes contract = 3;
        required bytes redeemScript = 4;
        repeated bytes root_pubkeys = 5;
    }

    message ContractProof {
        required bytes protocol = 1;
        repeated ContractProofTxOut txouts = 2; // in topological order, parents before children
    }

For each txout in txouts the SPV node does the following:

1) Check that tx is a valid transaction with a valid merkle path to the
blockchain.

2) Check that the redeemScript hash matches the scriptPubKey of the txout.

3) Check that the redeemScript is of the proper CHECKMULTISIG format.

4) Check that the pubkeys in the redeemScript are derived from the root pubkeys
and contract hash.

5) Check that the conditions of the contract are met. For instance a bond txout
must either spend a previous valid bond txout or be the creation of the bond
itself.

Finally for the remaining, unspent txout(s), the SPV node must check that the
transaction hash is in the UXTO set. Note that if the node simply wants to
purchase a contract output with a transaction tied to the txout, checking this
explicitly may not actually be required; the purchase going through implies
membership in the UXTO set. Equally the ContractProof structure may need to be
extended with UXTO proofs when such a protocol rule exists.

Note that in many scenarios the above data structure will have a lot of
redundency. The actual implementation approach to minimizing that redundency is
yet to be determined.


### Contract Value Accounting

For many applications the value of a particular contract output should be some
division of the value the contract chain was initialized with. Fidelity bonds
and financial bonds are two examples. The following method is suggested, which
allows for indefinite divisibility without tying up funds:

For each transaction, sum the value of all contract txouts, and the value of
all contract txins. The contract value of any given txout can be calculated as
Vout_n = SUM(Vin) * vout_n/SUM(vout) with V for contract value, and v for
nominal value. Or in other words, the fraction of the value in is just the
nominal value proportioned among all contract outputs. Using the odd-even rule
the outputs that are and are not contract outputs are easy to determine even
without knowing the root pubkeys for every output.


### Small-data OP_DROP contract output designation

As an alternative the contract hash can be included in the output directly with
OP_DROP; for instance using jgarzik's "Add small-data OP_DROP transactions as
standard transactions." pull request. (#1809) However this has the disadvantage
of using extra space in the blockchain, and making OP_DROP a standard
transaction type appears to be unpopular with many core developers. (see the
discussion: https://bitcointalk.org/index.php?topic=108423)'



## Fidelity Bonds

One use of the above "contract smartcoin" protocol is in making identity
acquisition expensive. For this we need a way to sacrifice Bitcoins in a manner
that can be both proven later, and in addition is ideally costs 100% of the
Bitcoins sacrificed. (that is, there is no mechanism to recover the sacrificed
value)


### Sacrifice via unspendable outputs

The most simple mechanism is to just assign coins to an output that can-not be
spent, for instance a txout with the scriptPubKey "OP_FALSE". The advantage of
a special scriptPubKey, as opposed to an address whose private keys should not
exist, is prunability: the UXTO set is costly to maintain so we want to prune
as many outputs as possible. OP_FALSE will never be spendable under any
circumstance, so we can drop such transactions immediately. On the other hand
it is possible that a flaw in the OpenSSL library, or even a break in ECC
encryption, may make other outputs spendable, so (highly) conservative
programming would not prune those outputs. Additionally OP_FALSE takes up less
space in the full chain.

This type of sacrifice has the advantage that it is guaranteed to be a 100%
value sacrifice. On the other hand it would be nice to direct the value of the
sacrifice to something more "socially optimal" than simple deflationary
pressures.


### Sacrifice via charity

It has been suggested multiple times that coins could be sacrified to charity,
either in a single payment to a well-known charity, or a payment to multiple
well-known charities in a single transaction.
(https://bitcointalk.org/index.php?topic=134827.msg1492305#msg1492305)

Regardless this type of sacrifice has problems in that the list of charities,
and their public keys, needs to be kept up to date, revocations of those keys
needs to be handled, and insiders are able to purchase bonds without actually
sacrificing any value.


### Sacrifice via mining fees

The naive mining fee sacrifice is to simply create a transaction with large
fees and get it confirmed. However this does not meet the "true sacrifice"
criteria because a miner can mine the transaction, and collect the fees
themselves.

Multiple, sequential, transactions are a more sophisticated version of this
mechanism that relies on the difficulty in capturing a large fraction of the
hashing power. However the number of transactions needed is large as long "runs
of good luck" are common, so a miner can simply wait until they get lucky and
have a few blocks in a row. Since for practical reasons small gaps would have
to be allowed - even with fees miners do not always mine transactions in the
mempool - the amount of hashing power required is relatively small if the miner
is willing to wait, and the cost is only a single transaction and the small
risk of an orphaned block.

One subtle aspect of mining fee sacrifices is that the proof of sacrifice has
to include the txin providing the funds to the tx that spent them in fees.
Transactions by themselves do not contain any information about the value of
the txins, only the value assigned to the txouts, thus the full txins must be
provided to prove the value of the fee.


### Sacrifice via a trivially spendable output

A txout with an empty scriptPubKey is trivially spendable with the scriptSig
OP_TRUE. Since miners control the order of transactions provided that they have
implemented code to recognize trivially spendable transactions they can always
add an additional transaction in the block they mine directing the output to
themselves; such txouts are roughly equivalent to mining fees. The same
argument applies to txouts spending to addresses with known secret keys, albeit
those txouts are IsStandard() and thus anyone can easilly spend them.

Note that in the case of a trivially spendable output the very existance if the
output in the blockchain is proof enough - additional proof that the txout has
actually been spent is not required, although if possible, asserting spent
status is a prudent additional check. Limiting the proof to the txout itself is
desirable with regards to padding griefing, as we will see in the next section.


### Sacrifice via replacable output, SIGHASH_(NONE|SINGLE)

A varient on the trivially spendable output proposed by gmaxwell
(https://bitcointalk.org/index.php?topic=140711.msg1498806#msg1498806) is the
replacable output, achieved by signing the txin with either SIGHASH_NONE or
SIGHASH_SINGLE. Both methods are fatally flawed however in that a malicious
party can replace the desired, short, txout with any number of txouts, thus
making the proof much larger, potentially even up to the 1MiB block size. A
similar problem exists with the ANYONECANPAY flag.

Of course miners can pad SIGHASH_ANY-using transactions, however that limits
the damage to no more than 10KiB of additional data, as EvalScript() has a hard
10KiB limit. If there was a way of specifying that a single txout was
replacable the additional 10KiB in the scriptPubKey might be acceptable,
however SIGHASH_NONE, SIGHASH_SINGLE and the ANYONECANPAY flag all allow for an
unlimited number of extra txins or txouts to be added to the transaction.

A possible hard-fork change that would remove this problem is a proposal by
Peter Todd to extend the merkle tree to within transaction inputs themselves
(https://bitcointalk.org/index.php?topic=106449.msg1469608#msg1469608) but at
minimum it would be a few years before the proposal could possibly be adopted.


### Two-step sacrifice

nLockTime allows the creation of transactions that are not valid now, but will
be in the future. By publishing such a transaction publicly in the blockchain
we can prove at a later time that everyone had an equal chance at redeeming the
transaction as the transaction was public knowledge, meeting the "true
sacrifice" criteria.

To start a transaction with a bare CHECKMULTISIG txout
of the desired value is created . The scriptPubKey of that txout then has the following
form:

    n {pubkey}...{pubkey} m OP_CHECKMULTISIG

Spending the txout in the sacrifice transaction is done by a particularly
simple scriptSig:

    {signature}...{signature}

The contract output is assigned to a P2SH output with the usual scriptPubKey.
To avoid the problem of having to provide an additional proof of the txin value
the "fee" is sacrificed with an empty scriptPubKey output. All inputs and
outputs are signed for with SIGHASH_ALL avoiding the padding griefing attack
explained above.

The total size of the sacrifice transaction is fairly small, 32+4 bytes for the
txin hash+index, 71 bytes per signature required, 22 bytes for the P2SH
scriptPubKey, and another 32 bytes of various header values, 97 bytes in total
minimum - almost small enough to fit in the 80 bytes "small-data" proposal from
jgarzik.

Publishing the sacrifice transaction is accomplished by including it in a prior
transaction scriptPubKey with OP_DROP. The scriptPubKey rather than the
scriptSig is chosen because the latter is not signed - a malicious miner could
modify or remove the published data and still collect the transaction fee. Note
that the requirement for the data to be in the scriptPubKey precludes a
P2SH-style output; P2SH has a fixed scriptPubKey format.

For proof size the publish transaction should have a single input and output.
(particularly with regard to the padding griefing attack) For assuring a "true
sacrifice" the format should be standardized as much as possible to allow easy
detection. Finally we want to ensure that implementations do not add to the
UTXO set. Thus the following scriptPubKey for publication is proposed:

    <serialized transaction> OP_RETURN

The advantage is that the txout is guaranteed prunable; lazy implementors might
not bother to implement the code required to implement creating the signature
to spend a non-standard but spendable output; the author has not gotten
around to doing so with his first demonstration of a a two-step sacrifice...


#### Proving a two-step sacrifice

Three transaction existence proofs are required:

    message TwoStepSacrifice {
        required MerkleBitcoinTx sacrifice_txin = 1;
        required MerkleBitcoinTx sacrifice_tx = 2;

        required MerkleBitcoinTx publish_tx = 3;
        required uint32 publish_txout_n = 4;
    }

Checking the proof consists of the following steps:

1) Check that all transactions are valid.

2) Check that the sacrifice tx prototype embedded in the publish transaction is
valid and can successfully spend the sacrifice txin. Note that this is
complicated by transaction mutability: either we provide the txin explicitly,
or find all ways that the signatures in the publish_tx can be mutated while
still remaining valid. Additionally if more signatures are provided than
required, the spender could drop a redundent signature, again mutating the tx.

3) Verify that the sacrifice_tx spent the same txin as the prototype.

4) Verify that the nLockTime of the sacrifice prototype is before or equal to
the block in which the actual sacrifice transaction occured.
