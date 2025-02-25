The aim of this document is to provide basic explanation of configuration parameters of TON Blockchain, and to give step-by-step instructions for changing these parameters by a consensus of a majority of validators. We assume that the reader is already familiar with Fift and the Lite Client as explained in [LiteClient-HOWTO](https://toncoin.org/docs/#/howto/step-by-step), and with [FullNode-HOWTO](https://toncoin.org/docs/#/howto/full-node) and [Validator-HOWTO](https://toncoin.org/docs/#/howto/validator) in the sections where validators' voting for the configuration proposals is described.

#### 1. Configuration parameters
The **configuration parameters** are certain values that affect the behavior of validators and/or fundamental smart contracts of TON Blockchain. The current values of all configuration parameters are stored as a special part of the masterchain state, and are extracted from the current masterchain state when needed. Therefore, it makes sense to speak of the values of the configuration parameters with respect to a certain masterchain block. Each shardchain block contains a reference to the latest known masterchain block; the values from the corresponding masterchain state are assumed to be active for this shardchain block, and are used during its generation and validation. For masterchain blocks, the state of the previous masterchain block is used to extract the active configuration parameters. Therefore, even if one tries to change some configuration parameters inside a masterchain block, the changes will become active only for the next masterchain block.

Each configuration parameter is identified by a signed 32-bit integer index, called **configuration parameter index** or simply **index**. The value of a configuration parameter always is a Cell. Some configuration parameters may be missing; then it is sometimes assumed that the value of this parameter is `Null`. There also is a list of **mandatory** configuration parameters that must be always present; this list is stored in configuration parameter `#10`.

All configuration parameters are combined into a **configuration dictionary** - a Hashmap with signed 32-bit keys (configuration parameter indices) and values consisting of exactly one cell reference. In other words, a configuration dictionary is a value of TL-B type (`HashmapE 32 ^Cell`). In fact, the collection of all configuration parameters is stored in the masterchain state as a value of TL-B type `ConfigParams`:

    _ config_addr:bits256 config:^(Hashmap 32 ^Cell) = ConfigParams;

We see that, apart from the configuration dictionary, `ConfigParams` contains `config_addr` -- 256-bit address of the configuration smart contract in the masterchain. More details on the configuration smart contract will be provided later.

The configuration dictionary containing the active values of all configuration parameters is available via special TVM register `c7` to all smart contracts when their code is executed in a transaction. More precisely, when a smart contract is executed, `c7` is initialized by a Tuple, the only element of which is a Tuple with several "context" values useful for the execution of the smart contract, such as the current Unix time (as registered in the block header). The tenth entry of this Tuple (i.e., the one with zero-based index 9) contains a Cell representing the configuration dictionary. Therefore, it can be accesses by means of TVM instructions `PUSH c7; FIRST; INDEX 9`, or by equivalent instruction `CONFIGROOT`. In fact, special TVM instructions `CONFIGPARAM` and `CONFIGOPTPARAM` combine the previous actions with a dictionary lookup, returning any configuration parameter by its index. We refer to the TVM documentation for more details on these instructions. What is relevant here is that all configuration parameters are easily accessible from all smart contracts (masterchain or shardchain), and smart contracts may inspect them and use them to perform specific checks. For instance, a smart contract might extract workchain data storage prices from a configuration parameter to compute the price for storing a chunk of user-provided data.

The values of configuration parameters are not arbitrary. In fact, if the configuration parameter index `i` is non-negative, then the value of this parameter must be a valid value of TL-B type (`ConfigParam i`). This restriction is enforced by the validators, which will not accept changes to configuration parameters with non-negative indices unless they are valid values of the corresponding TL-B type.

Therefore, the structure of such parameters is determined in source file `crypto/block/block.tlb`, where (`ConfigParam i`) are defined for different values of `i`. For instance,

    _ config_addr:bits256 = ConfigParam 0;
    _ elector_addr:bits256 = ConfigParam 1;
    _ dns_root_addr:bits256 = ConfigParam 4;  // root TON DNS resolver

    capabilities#c4 version:uint32 capabilities:uint64 = GlobalVersion;
    _ GlobalVersion = ConfigParam 8;  // all zero if absent

We see that configuration parameter `#8` contains a Cell with no references and exactly 104 data bits. The first four bits must be `11000100`, then 32 bits with the currently enabled "global version" are stored, and 64-bit integer with flags corresponding to currently enabled capabilities follow. A more detailed description of all configuration parameters will be provided in an appendix to the TON Blockchain documentation; for now, one can inspect the TL-B scheme in `crypto/block/block.tlb` and check how different parameters are used in the validator sources.

In contrast with configuration parameters with non-negative indices, configuration parameters with negative indices can contain arbitrary values. At least, no restrictions on their values are enforced by the validators. Therefore, they can be used to store important information (such as the Unixtime when certain smart contracts must start operating) that is not crucial for the block generation, but is used by some of the fundamental smart contracts.

#### 2. Changing configuration parameters

We have already explained that the current values of configuration parameters are stored in a special portion of the masterchain state. How do they ever get changed?

In fact, there is a special smart contract residing in the masterchain, called the **configuration smart contract**. Its address is determined by the `config_addr` field in `ConfigParams`, which we have described before. The first cell reference in its data must contain an up-to-date copy of all configuration parameters. When a new masterchain block is generated, the configuration smart contract is looked up by its address `config_addr`, and the new configuration dictionary is extracted from the first cell reference of its data. After some validity checks (such as verifying that any value with non-negative 32-bit index `i` is indeed a valid value of TL-B type (`ConfigParam i`)) the validator copies this new configuration dictionary into the portion of masterchain containing ConfigParams. This is performed after all transactions have been created, so only the final version of the new configuration dictionary stored in the configuration smart contract is inspected. If the validity checks fail, then the "true" configuration dictionary is left unchanged. In this way the configuration smart contract cannot install invalid values of configuration parameters. If the new configuration dictionary coincides with the current configuration dictionary, then no checks are performed and no changes made.

In this way, all changes in configuration parameters are performed by the configuration smart contract, and it is its code that determines the rules for changing configuration parameters. Currently, the configuration smart contract supports two modes for changing configuration parameters:

1) By means of an external message signed by a specific private key, corresponding to a public key stored in the data of the configuration smart contract. This is the method employed in the public testnet and probably in the smaller private test networks, controlled by one entity, because it enables the operator to easily change the values of any configuration parameters. Note that this public key may be changed by a special external message signed by an old key, and that if it is changed to zero, then this mechanism is disabled. Therefore, one might use it for fine-tuning immediately after the launch, and then disable for good.
2) By means of creating "configuration proposals" that are subsequently voted for or against by validators. Typically, a configuration proposal has to collect votes from more than 3/4 of all validators (by weight), and not only in one round, but in several rounds (i.e., several consecutive sets of validators have to confirm the proposed parameter change). This is the distributed governance mechanism to be used by TON Blockchain mainnet.

We would like to describe the second way of changing configuration parameters in more detail.

#### 3. Creating configuration proposals

A new **configuration proposal** contains the following data:
- the index of the configuration parameter to be changed
- the new value of the configuration parameter (or Null, if it is to be deleted)
- the expiration Unixtime of the proposal
- a flag indicating whether the proposal is **critical** or not
- an optional **old value hash** with the cell hash of the current value (the proposal can be activated only if the current value has indicated hash)

Anybody with a wallet in the masterchain can create a new configuration proposal, provided he pays an adequate fee. However, only validators can vote for or against existing configuration proposals.

Note that there are **critical** and **ordinary** configuration proposals. A critical configuration proposal can change any configuration parameter, including one of the so-called critical ones (the list of critical configuration parameters is stored in configuration parameter `#10`, which is itself critical). However, creating critical configuration proposals is more expensive, and they usually need to collect more validator votes in more rounds (the precise voting requirements for ordinary and critical configuration proposals are stored in critical configuration parameter `#11`). On the other hand, ordinary configuration proposals are cheaper, but they cannot change the critical configuration parameters.

In order to create a new configuration proposal, one first has to generate a BoC (bag-of-cells) file containing the proposed new value. The exact way of doing this depends on the configuration parameter being changed. For instance, if we want to create parameter `-239` containing UTF-8 string "TEST" (i.e., `0x54455354`), we could create `config-param-239.boc` as follows: invoke Fift and then type

    <b "TEST" $, b> 2 boc+>B "config-param-239.boc" B>file
    bye

As a result, 21-byte file `config-param-239.boc` will be created, containing the serialization of the required value.

For more sophisticated cases, and especially for configuration parameters with non-negative indices this straightforward approach is not easily applicable. We recommend using `create-state` (available as `crypto/create-state` in the build directory) instead of `fift`, and copying and editing suitable portions from source files `crypto/smartcont/gen-zerostate.fif` and `crypto/smartcont/CreateState.fif`, which are usually employed to create the zero state (corresponding to the "genesis block" of other blockchain architectures) of the TON Blockchain.

Consider, for instance, configuration parameter `#8`, which contains the currently-enabled global blockchain version and capabilities:

    capabilities#c4 version:uint32 capabilities:uint64 = GlobalVersion;
    _ GlobalVersion = ConfigParam 8;

We can inspect its current value by running the lite-client and typing `getconfig 8`:

```
> getconfig 8
...
ConfigParam(8) = (
  (capabilities version:1 capabilities:6))

x{C4000000010000000000000006}
```

Now suppose that we want to enable the capability represented by bit `#3` (`+8`), which is `capReportVersion` (when enabled, this capability forces all collators to report their supported version and capabilities in the block headers of the blocks they generate). Therefore, we want to have `version=1` and `capabilities=14`. In this example, we still can guess the correct serialization and create the BoC file directly by typing in Fift

    x{C400000001000000000000000E} s>c 2 boc+>B "config-param8.boc" B>file

(A 30-byte file `config-param8.boc` containing the desired value is created as a result.)

However, in more complicated cases this might not be an option, so let's do this example differently. Namely, we can inspect source files `crypto/smartcont/gen-zerostate.fif` and `crypto/smartcont/CreateState.fif` for relevant portions:

    // version capabilities --
    { <b x{c4} s, rot 32 u, swap 64 u, b> 8 config! } : config.version!
    1 constant capIhr
    2 constant capCreateStats
    4 constant capBounceMsgBody
    8 constant capReportVersion
    16 constant capSplitMergeTransactions

and

    // version capabilities
    1 capCreateStats capBounceMsgBody or capReportVersion or config.version!

We see that `config.version!` without the last `8 config!` essentially does what we need, so we can create a temporary Fift script, say, `create-param8.fif`:
```
#!/usr/bin/fift -s
"TonUtil.fif" include

1 constant capIhr
2 constant capCreateStats
4 constant capBounceMsgBody
8 constant capReportVersion
16 constant capSplitMergeTransactions
{ <b x{c4} s, rot 32 u, swap 64 u, b> } : prepare-param8

// create new value for config param #8
1 capCreateStats capBounceMsgBody or capReportVersion or prepare-param8
// check the validity of this value
dup 8 is-valid-config? not abort"not a valid value for chosen configuration parameter"
// print
dup ."Serialized value = " <s csr.
// save into file provided as first command line argument
2 boc+>B $1 tuck B>file
."(Saved into file " type .")" cr
```

Now if we run `fift -s create-param8.fif config-param8.boc`, or, even better, `crypto/create-state -s create-param8.fif config-param8.boc` (from the build directory), we see the following output:

    Serialized value = x{C400000001000000000000000E}
    (Saved into file config-param8.boc)

and we obtain 30-byte file `config-param8.boc` with the same content as before.

Once we have a file with the desired value of the configuration parameter, we invoke script `create-config-proposal.fif` that can be found in directory `crypto/smartcont` of the source tree with suitable arguments. Again, we recommend using `create-state` (available as `crypto/create-state` from the build directory) instead of `fift`, because it is a special extended version of Fift that is able to do more blockchain-related validity checks:

```
$ crypto/create-state -s create-config-proposal.fif 8 config-param8.boc -x 1100000


Loading new value of configuration parameter 8 from file config-param8.boc
x{C400000001000000000000000E}

Non-critical configuration proposal will expire at 1586779536 (in 1100000 seconds)
Query id is 6810441749056454664 
resulting internal message body: x{6E5650525E838CB0000000085E9455904_}
 x{F300000008A_}
  x{C400000001000000000000000E}

B5EE9C7241010301002C0001216E5650525E838CB0000000085E9455904001010BF300000008A002001AC400000001000000000000000ECD441C3C
(a total of 104 data bits, 0 cell references -> 59 BoC data bytes)
(Saved to file config-msg-body.boc)
```

We have obtained the body of an internal message to be sent to the configuration smart contract with a suitable amount of TON Coins from any (wallet) smartcontract residing in the masterchain. The address of the configuration smart contract may be obtained by typing `getconfig 0` in the lite-client:
```
> getconfig 0
ConfigParam(0) = ( config_addr:x5555555555555555555555555555555555555555555555555555555555555555)
x{5555555555555555555555555555555555555555555555555555555555555555}
```
We see that the address of the configuration smart contract is `-1:5555...5555`. By running suitable get-methods of this smart contract, we can find out the required payment for creating this configuration proposal:

```
> runmethod -1:5555555555555555555555555555555555555555555555555555555555555555 proposal_storage_price 0 1100000 104 0

arguments:  [ 0 1100000 104 0 75077 ] 
result:  [ 2340800000 ] 
remote result (not to be trusted):  [ 2340800000 ] 
```

The parameters to get-method `proposal_storage_price` are the critical flag (0 in this case), the time interval during which this proposal will be active (1.1 Megaseconds), the total amount of bits (104) and cell references (0) in the data. The latter two quantities can be seen in the output of `create-config-proposal.fif`.

We see that one has to pay 2.3408 TON Coins to create this proposal. It is better to add at least 1.5 TON Coins to the message to pay for the processing fees, so we are going to send 4 TON Coins along with the request (all excess TON Coins will be returned back). Now we use `wallet.fif` (or the corresponding Fift script for the wallet we are using) to create a transfer from our wallet to the configuration smart contract carrying 4 TON Coins and the body from `config-msg-body.boc`. This usually looks like

```
$ fift -s wallet.fif my-wallet -1:5555555555555555555555555555555555555555555555555555555555555555 31 4. -B config-msg-body.boc

Transferring GR$4. to account kf9VVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVQft = -1:5555555555555555555555555555555555555555555555555555555555555555 seqno=0x1c bounce=-1 
Body of transfer message is x{6E5650525E835154000000085E9293944_}
 x{F300000008A_}
  x{C400000001000000000000000E}

signing message: x{0000001C03}
 x{627FAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA773594000000000000000000000000000006E5650525E835154000000085E9293944_}
  x{F300000008A_}
   x{C400000001000000000000000E}

resulting external message: x{89FE000000000000000000000000000000000000000000000000000000000000000007F0BAA08B4161640FF1F5AA5A748E480AFD16871E0A089F0F017826CDC368C118653B6B0CEBF7D3FA610A798D66522AD0F756DAEECE37394617E876EFB64E9800000000E01C_}
 x{627FAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA773594000000000000000000000000000006E5650525E835154000000085E9293944_}
  x{F300000008A_}
   x{C400000001000000000000000E}

B5EE9C724101040100CB0001CF89FE000000000000000000000000000000000000000000000000000000000000000007F0BAA08B4161640FF1F5AA5A748E480AFD16871E0A089F0F017826CDC368C118653B6B0CEBF7D3FA610A798D66522AD0F756DAEECE37394617E876EFB64E9800000000E01C010189627FAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA773594000000000000000000000000000006E5650525E835154000000085E9293944002010BF300000008A003001AC400000001000000000000000EE1F80CD3
(Saved to file wallet-query.boc)
```

Now we send the external message `wallet-query.boc` to the blockchain with the aid of the lite-client:

    > sendfile wallet-query.boc
    ....
    external message status is 1

After waiting for some time, we can inspect the incoming messages of our wallet to check for response messages from the configuration smart contract, or, if we feel lucky, simply inspect the list of all active configuration proposals by means of method `list_proposals` of the configuration smart contract:

```
> runmethod -1:5555555555555555555555555555555555555555555555555555555555555555 list_proposals
...
arguments:  [ 107394 ] 
result:  [ ([64654898543692093106630260209820256598623953458404398631153796624848083036321 [1586779536 0 [8 C{FDCD887EAF7ACB51DA592348E322BBC0BD3F40F9A801CB6792EFF655A7F43BBC} -1] 112474791597373109254579258586921297140142226044620228506108869216416853782998 () 864691128455135209 3 0 0]]) ] 
remote result (not to be trusted):  [ ([64654898543692093106630260209820256598623953458404398631153796624848083036321 [1586779536 0 [8 C{FDCD887EAF7ACB51DA592348E322BBC0BD3F40F9A801CB6792EFF655A7F43BBC} -1] 112474791597373109254579258586921297140142226044620228506108869216416853782998 () 864691128455135209 3 0 0]]) ] 
...	caching cell FDCD887EAF7ACB51DA592348E322BBC0BD3F40F9A801CB6792EFF655A7F43BBC
```

We see that the list of all active configuration proposals consists of exactly one entry, represented by pair
```
[6465...6321 [1586779536 0 [8 C{FDCD...} -1] 1124...2998 () 8646...209 3 0 0]]
```
Here the first number `6465..6321` is the unique identifier of the configuration proposal, equal to its 256-bit hash. The second component of this pair is a Tuple describing the status of this configuration proposal. The first component of this Tuple is the expiration Unixtime of the configuration proposal (`1586779546`). The second component (`0`) is the criticality flag. Next comes the configuration proposal proper, described by triple `[8 C{FDCD...} -1]`, where `8` is the index of the configuration parameter to be modified, `C{FDCD...}` is the cell with the new value (represented by the hash of this cell), and `-1` is the optional hash of the old value of this parameter (`-1` means that this hash has not been specified). Next we see a large number `1124...2998` representing the identifier of the current validator set, then an empty list `()` representing the set of all currently active validators that have voted for this proposal so far, then `weight_remaining` equal to `8646...209` - a number that is positive if the proposal has not yet collected enough validator votes in this round, and negative otherwise. Then we see three numbers `3 0 0`. These numbers are `rounds_remaining` (this proposal will survive at most three rounds, i.e., changes of the current validator set), `wins` (the count of rounds where the proposal collected votes of more than 3/4 of all validators by weight) and `losses` (the count of rounds where the proposal failed to collect 3/4 of all validator votes).

We can inspect the proposed value for configuration parameter `#8` by asking the lite-client to expand cell `C{FDCD...}` using its hash `FDCD...` or a sufficiently long prefix of this hash to uniquely identify the cell in question:
```
> dumpcell FDC
C{FDCD887EAF7ACB51DA592348E322BBC0BD3F40F9A801CB6792EFF655A7F43BBC} =
  x{C400000001000000000000000E}
```
We see that the value is `x{C400000001000000000000000E}`, which is indeed the value we have embedded into our configuration proposal. We can even ask the lite-client to display this Cell as a value of TL-B type (`ConfigParam 8`):
```
> dumpcellas ConfigParam8 FDC
dumping cells as values of TLB type (ConfigParam 8)
C{FDCD887EAF7ACB51DA592348E322BBC0BD3F40F9A801CB6792EFF655A7F43BBC} =
  x{C400000001000000000000000E}
(
    (capabilities version:1 capabilities:14))
```
This is especially useful when we consider configuration proposals created by other people.

Note that the configuration proposal is henceforth identified by its 256-bit hash -- the huge decimal number `6465...6321`. We can inspect the current status of a specific configuration proposal by running get-method `get_proposal` with the only argument equal to the identifier of the configuration proposal:
```
> runmethod -1:5555555555555555555555555555555555555555555555555555555555555555 get_proposal 64654898543692093106630260209820256598623953458404398631153796624848083036321
...
arguments:  [ 64654898543692093106630260209820256598623953458404398631153796624848083036321 94347 ] 
result:  [ [1586779536 0 [8 C{FDCD887EAF7ACB51DA592348E322BBC0BD3F40F9A801CB6792EFF655A7F43BBC} -1] 112474791597373109254579258586921297140142226044620228506108869216416853782998 () 864691128455135209 3 0 0] ] 
```
We obtain essentially the same result as before, but for only one configuration proposal, and without the identifier of the configuration proposal at the beginning.

#### 4. Voting for configuration proposals

Once a configuration proposal is created, it is supposed to collect votes from more than 3/4 of all current validators (by weight, i.e., by stake) in the current and maybe in several subsequent rounds (elected validator sets). In this way the decision to change a configuration parameters must be approved by a significant majority not only of the current set of validators, but also of several subsequent sets of validators.

Voting for a configuration proposal is possible only for current validators, listed (with their permanent public keys) in configuration parameter `#34`. The process is approximately the following:

- The operator of a validator looks up `val-idx`, the (0-based) index of his validator in the current set of validators as stored in configuration parameter `#34`.
- The operator invokes special Fift script `config-proposal-vote-req.fif`, found in directory `crypto/smartcont` of the source tree, indicating `val-idx` and `config-proposal-id` as its arguments:
```
    $ fift -s config-proposal-vote-req.fif -i 0 64654898543692093106630260209820256598623953458404398631153796624848083036321
    Creating a request to vote for configuration proposal 0x8ef1603180dad5b599fa854806991a7aa9f280dbdb81d67ce1bedff9d66128a1 on behalf of validator with index 0 
    566F744500008EF1603180DAD5B599FA854806991A7AA9F280DBDB81D67CE1BEDFF9D66128A1
    Vm90RQAAjvFgMYDa1bWZ-oVIBpkaeqnygNvbgdZ84b7f-dZhKKE=
    Saved to file validator-to-sign.req
```
- After that, the vote request has to be signed by the current validator private key, using `sign <validator-key-id> 566F744...28A1` in `validator-engine-console` connected to the validator. This process is similar to that described in [Validator-HOWTO](https://toncoin.org/docs/#/howto/validator) for participating in validator elections, but this time the currently active key has to be used.
- Next, another script `config-proposal-signed.fif` has to be invoked. It has similar arguments to `config-proposal-req.fif`, but it expects two extra arguments: the base64 representation of the public key used to sign the vote request, and the base64 representation of the signature itself. Again, this is quite similar to the process described in [Validator-HOWTO](https://toncoin.org/docs/#/howto/validator).
- In this way file `vote-msg-body.boc` containing the body of an internal message carrying a signed vote for this configuration proposal is created.
- After that, `vote-msg-body.boc` has to be carried in an internal message from any smart contract residing in the masterchain (typically the controlling smart contract of the validator will be used) along with a small amount of TON Coins for processing (typically 1.5 TON Coins should suffice). This is again completely similar to the procedure employed during validator elections. This is typically achieved by means of running
```
$ fift -s wallet.fif my_wallet_id -1:5555555555555555555555555555555555555555555555555555555555555555 1 1.5 -B vote-msg-body.boc
```
(if a simple wallet is used to control the validator) and then sending the resulting file `wallet-query.boc` from the lite-client:

```
> sendfile wallet-query.boc
```

You can monitor answer messages from the configuration smart contract to the controlling smart contract to learn the status of your voting queries. Alternatively, you can inspect the status of the configuration proposal by means of get-method `show_proposal` of the configuration smart contract:
```
> runmethod -1:5555555555555555555555555555555555555555555555555555555555555555 get_proposal 64654898543692093106630260209820256598623953458404398631153796624848083036321
...
arguments:  [ 64654898543692093106630260209820256598623953458404398631153796624848083036321 94347 ] 
result:  [ [1586779536 0 [8 C{FDCD887EAF7ACB51DA592348E322BBC0BD3F40F9A801CB6792EFF655A7F43BBC} -1] 112474791597373109254579258586921297140142226044620228506108869216416853782998 (0) 864691128455135209 3 0 0] ]
```
This time the list of indices of validators that voted for this configuration proposal should be non-empty, and it should contain the index of your validator. In this example, this list is (`0`), meaning that only the validator with index `0` in configuration parameter `#34` has voted. If the list becomes large enough, the last-but-one integer (the first zero in `3 0 0`) in the proposal status will increase by one, indicating a new win by this proposal. If the number of wins becomes greater than or equal to the value indicated in configuration parameter `#11`, then the configuration proposal is automatically accepted and the proposed changes become effective immediately. On the other hand, when the validator set changes, then the list of validators that have already voted becomes empty, the value of `rounds_remaining` (three in `3 0 0`) is decreased by one, and if it becomes negative, the configuration proposal is destroyed. If it is not destroyed, and if it did not win in this round, then the number of losses (the second zero in `3 0 0`) is increased. If it becomes larger than a value specified in configuration parameter `#11`, then the configuration proposal is discarded. In this way all validators that have abstained from voting in a round have implicitly voted against.

#### 5. An automated way for voting for configuration proposals

Similarly to the automation provided by command `createelectionbid` of `validator-engine-console` for participating in validator elections, `validator-engine` and `validator-engine-console` offer an automated way of performing most of the steps explained in the previous section, producing a `vote-msg-body.boc` ready to be used with the controlling wallet. In order to use this method, you must install Fift scripts `config-proposal-vote-req.fif` and `config-proposal-vote-signed.fif` into the same directory that the validator-engine uses to look up `validator-elect-req.fif` and `validator-elect-signed.fif` as explained in Section 5 of [Validator-HOWTO](https://toncoin.org/docs/#/howto/validator). After that, you simply run
```
    createproposalvote 64654898543692093106630260209820256598623953458404398631153796624848083036321 vote-msg-body.boc
```
in validator-engine-console to create `vote-msg-body.boc` with the body of the internal message to be sent to the configuration smart contract.

#### 6. Upgrading the code of configuration smart contract and the elector smart contract

It may happen that the code of the configuration smart contract itself or the code of the elector smart contract has to be upgraded. To this end, the same mechanism as described above is used. The new code is to be stored into the only reference of a value cell, and this value cell has to be proposed as the new value of configuration parameter `-1000` (for upgrading the configuration smart contract) or `-1001` (for upgrading the elector smart contract). These parameters pretend to be critical, so a lot of validator votes is needed to change the configuration smart contract (this is akin to adopting a new constitution). We expect that such changes will involve first testing them in a test network, and discussing the proposed changes in public forums before each validator operator decides to vote for or against proposed changes.

Alternatively, critical configuration parameters `0` (the address of the configuration smart contract) or `1` (the address of the elector smart contract) can be changed to other values, that must correspond to already existing and correctly initialized smart contracts. In particular, the new configuration smart contract must contain a valid configuration dictionary in the first reference of its persistent data. Since it is not so easy to correctly transfer changing data (such as the list of active configuration proposals, or the previous and current participant lists of validator elections) between different smart contracts, in most cases it is better to upgrade the code of existing smart contract rather than to change the configuration smart contract address.

There are two auxiliary scripts used to create such configuration proposals to upgrade the code of the configuration or elector smart contract. Namely, `create-config-upgrade-proposal.fif` loads a Fift assembler source file (`auto/config-code.fif` by default, corresponding to the code automatically generated by FunC compiler from `crypto/smartcont/config-code.fc`) and creates the corresponding configuration proposal (for configuration parameter `-1000`). Similarly, `create-elector-upgrade-proposal.fif` loads a Fift assembler source file (`auto/elector-code.fif` by default) and uses it to create a configuration proposal for configuration parameter `-1001`. In this way creating configuration proposals to upgrade one of these two smart contracts should be very simple. However, one should also publish the modified FunC source of the smart contract, the exact version of the FunC compiler used to compile it, so that all validators (or rather their operators) would be able to reproduce the code in the configuration proposal (and compare the hashes), and study and discuss the source code and the changes in this code before deciding to vote for or against proposed changes.
