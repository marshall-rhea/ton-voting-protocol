# Vote Saver Protocol.

This repository implements [SAVER](https://eprint.iacr.org/2019/1270) voting protocol.
Virtualized part is implemented in TVM-specific Solidity dialect.

## Description

Full and formal protocol description with formal security proofs is available within
the [original SAVER voting protocol paper](https://eprint.iacr.org/2019/1270).

Basic workflow description is as follows.

### Before election.

Our voting system let each user publish his own pk to the public, where pk is generated from user’s secret value. For
example, a simple way is to `let pk = H(sk)` for collision-resistant hash `H`. Without knowing `sk`, no one can make a
valid ballot.

### Initiating election.

First, to open an election, the administrator makes the `pklist` of the voters, which prescribes the selection of
eligible voters who participate in the election. Then the administrator generates a secret key `SK`, a public key `PK`,
and a verification key `VK` for the occasion, to publish `PK`, `VK` inside a fault-tolerant replication-enabled cluster
along with the `pklist` and its Merkle root `rt`. This set of `PK`, `VK` and `pklist`, `rt` defines each election; a new
election can be initiated with a different set of `PK′`, `VK′` and `pklist′`, `rt′`.

### Casting votes

After the election is initiated, voters who are selected in the list can cast a vote. Each voter must encrypt the vote
and prove the relation (i.e. membership test and message validity) at the same time, via generic verifiable encryption
from zk-SNARK. Similar to the membership test in Zerocash, the zk-SNARK circuit outputs a Merkle root `rt` to prove the
belonging within the `pklist`, and a serial number `sn` to prevent the duplication. Note that the `sn` does not reveal
the identity; it is only used for checking the duplication. As a ballot, a set of serial number `sn`, proof `π` and
ciphertext `CT` is sent to the cluster as a transaction. The cluster node checks if sn already exists in the cluster's
database (then abort). If `sn` is unique, it first verifies the proof, rerandomizes the vote from `π`, `CT`
to `π′`, `CT′` (in case there is no submitted proof and `sn` uniqueness requirement), and publishes the renewed vote 
`sn`, `π′`, `CT′` to the public cluster. The voter verifies `π′`, `CT′` for his `sn` within the verifiable 
encryption, to be convinced that his vote is included. This satisfies the individual verifiability, but the voter 
can only check the existence of his vote; `π′`, `CT′` is unlinkable from `π`, `CT`, which also achieves the 
receipt-freeness.

### Tallying results.

After all the votes from participants are posted within the public database's cluster, the administrator closes the  
vote by declaring the tally result. Since the encryption scheme is additively-homomorphic, anyone can get the merged
ciphertext `CT` sum. The administrator is responsible for decrypting the `CT` sum with her own `SK`, and publishing the
corresponding vote result `M_sum` along with the decryption proof `ν`. By verifying `M_sum`, `ν` with the verifiable
decryption, anyone can be convinced that the result is tallied correctly (universal verifiability).

## Requirements

* Boost >= 1.76.

## Building

### Building CLI

Cli is used to generate R1CS, CRS, zk-SNARK proofs, ElGamal keys, execute encryption, decryption, proofs creation, (de)
serialize generated data and other operations, required for the voting protocol execution. To build cli it is required
to run the following:

```shell
git clone --recursive git@github.com:NilFoundation/ton-voting-protocol.git contest && cd contest
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make cli
```

## Usage

Let's consider voting session consisting of the session administrator and 4 voters.

Consider that their accounts have enough balances to proceed with this protocol.

### Generation

First phase, processed by the administrator, executed using cli with `encrypted_input_mode` flag:

```sh
./cli --mode encrypted_input
```

At the moment cli generates data both for administrator and voters. It's not the case for real application, and could be
used only for demonstration purposes.

After executing this command multiple binary files will be generated. To process voting protocol following files are
needed:

- ```verification_key0.bin``` - zk-SNARK verification key, used by administrator to initialize its in-TVM part.
- ```verification_input{i}.bin``` - contain constructed ballots of the voters (where ```i``` - voter's index). It
  consists of zk-SNARK proof, zk-SNARK verification key, encrypted ballot, session id, serial number of the ballot and
  merkle tree root hash, built upon voters public keys. Again in real application voter would construct such blob
  manually, receiving zk-SNARK keys, session id and merkle tree root hash from the administrator, but for demonstration
  purposes generation was simplified.
- `decryption_proof.bin` - contain decryption proof from the tally phase of the protocol, generated by the
  administrator.

Voters' generated proofs will be saved into `proof{i}.bin` files, but there is no need to use is, as these proofs
already included into `verification_input{i}.bin` files as part of the voters' ballots.

Also voters' public key will be printed into the terminal, and could be copied to initialize voters' in-TVM part.

To execute requests to the FLD cluster generated binary data should be converted into the hex string representation. It
could be done using following python script:

```python
 with open(verification_input0.bin, 'rb') as f:
    hexdata = binascii.hexlify(f.read())
    hexdata_str = '01' + hexdata.decode("ascii")
    print(hexdata_str)

with open(verification_key0.bin, 'rb') as f:
    hexdata = binascii.hexlify(f.read())
    hexdata_str = hexdata.decode("ascii")
    print(hexdata_str)
```

First byte `01` is required for the cluster's TVM to be able to work with ballot blob file.

Why and what does it mean?

To factor out the vote encryption out of the circuit, it was required to make Groth16 proof system to work well with
encrypted primary input. So the first byte was made to switch `VERGRTH16` instruction modes. `0x00` means it would work
with a usual primary input, `0x01` means it would work with ElGamal-encrypted primary input.

### Building In-TVM Application

Following instruction will cover work with FLD cluster and interaction with in-TVM part. To execute these operation
following forks should be used:

- [`tonos-cli`](https://github.com/NilFoundation/ton-tonos-cli/tree/2-groth16-verification-encrypted-input-mode)
- [`solc`](https://github.com/NilFoundation/tvm-solidity)
- [`tvm-linker`](https://github.com/NilFoundation/tvm-linker/tree/1-vergrth16)

There are two in-TVM part implementing voting protocol logic:

- voting_admin.sol
- voting_voter.sol

Building them requires to run the following commands:

 ```sh
 cd ./share/tvm
 ./build.sh voting_admin
 ./build.sh voting_voter
 mkdir keys
 mkdir genaddr-output
 ./genaddr.sh voting_admin
 ./genaddr.sh voting_voter 0
 ./genaddr.sh voting_voter 1
 # and so on, how much voters you need
 ```

Following interaction with FLD cluster considered to be executed from the same directory.

### Admin deployment:

```sh
tonos-cli deploy --abi voting_admin.abi.json --sign keys/voting_admin.keys.json voting_admin.tvc '{}'
```

### zk-SNARK pre-initialization

Admin pre-initialize voting context, namely upload zk-SNARK proving and verification keys. These keys could be large, so
several calls of the ```update_crs``` function may be required. There is also a function ```reset_crs``` allowing to
begin uploading again.

```sh
tonos-cli call 0:4eb04909a3d8e451a9ff30ad7f9659a1d4fb04eb0e85f8fca58905aa181fd450 update_crs '{"pk":"<crs_proving_key>", "vk":"<crs_verification_key>"}' --abi voting_admin.abi.json --sign keys/voting_admin.keys.json

tonos-cli call 0:4eb04909a3d8e451a9ff30ad7f9659a1d4fb04eb0e85f8fca58905aa181fd450 reset_crs '{}' --abi voting_admin.abi.json --sign keys/voting_admin.keys.json
```

### Admin initialize voting session:

```sh
tonos-cli call 0:4eb04909a3d8e451a9ff30ad7f9659a1d4fb04eb0e85f8fca58905aa181fd450 init_voting_session '{"eid":"<session id>","pk_eid":"<ElGamal public key>","vk_eid":"<ElGamal verification key>","voters_addresses":["0:38e18f490682cc199f97bec66c716508ae6379dcbb903f2976f89eed1b92a9a5","0:a72953bd268862c86648bd299bc4e0566176035128a490d8551df52ece40ec84","0:8dc2d5ae85de8f1da509f7ed190409c06cb9d191ab2a23c59a74219b16db8f52","0:b8e25c3ebbcfc28af45938ba580ab3a82f426331e67a0237079fe72d314cbfe3"],"rt":"<root hash of the merkle tree constructed upon voters public keys>"}' --abi voting_admin.abi.json --sign keys/voting_admin.keys.json
```

Each call to the function ```init_voting_session``` will initialize a new session completing the previous.

### Voters deployment:

```sh
tonos-cli deploy --abi voting_voter.abi.json --sign keys/voting_voter0.keys.json voting_voter.tvc '{"pk":"010203", "admin":"0:4eb04909a3d8e451a9ff30ad7f9659a1d4fb04eb0e85f8fca58905aa181fd450"}'
# ...the same for other voters
```

During deployment voters using constructor argument ```admin``` specify administrator address, which hold voting
session. Also they specify their public keys via ```pk``` (this is not deployment key from ```voting_voter0.keys.json```
, it's abstract public key required by the Saver protocol and it's used to construct merkle tree).

### Votes uploading and committing

Voters upload their encrypted, proved and rerandomized ballots:

```sh
tonos-cli call 0:38e18f490682cc199f97bec66c716508ae6379dcbb903f2976f89eed1b92a9a5 update_ballot '{"vi":"<ballot blob which consists of: proof, crs vkey, ElGamal pubkey, encrypted ballot, session id, serial number and merkle tree root hash>"}' --abi voting_voter.abi.json --sign keys/voting_voter0.keys.json
# ...the same for other voters
```

Input blob could be large, so several calls of the ```update_ballot``` function may be required. There is also a
function ```reset_ballot``` allowing to begin uploading again.

Voters commit their ballots:

```sh
tonos-cli call 0:38e18f490682cc199f97bec66c716508ae6379dcbb903f2976f89eed1b92a9a5 commit_ballot '{"proof_end":193,"ct_begin":35273,"eid_begin":35721,"sn_begin":37769,"sn_end":45929 }' --abi voting_voter.abi.json --sign keys/voting_voter0.keys.json
```

During this step zk-SNARK verification of the input blob is processed. Input parameters index input blob, which should
have strict format, mentioned above, otherwise zk-SNARK verification will return false result. When all checks
in ```commit_ballot``` finish successfully, internal message to admin is sent, containing session id and serial number
from the ballot blob. When all checks on the administrator side finish another internal message to the voter's callback
function will be sent, containing statis of the administrator checks. If all checks are successful
voter's ```m_is_vote_accepted``` flag is set to ```true``` (it could be got by the  ```is_vote_accepted``` function call
to the voter's in-TVM part). Also this voter will be marked as committed on the administrator side. Warning: every
successful call to the functions ```reset_ballot``` or ```update_ballot``` will decommit voter's vote on the voter and
administrator sides, so ```commit_ballot``` should be called again.

After all voters commit their ballots then administrator downloads them by calling to the
```get_ct``` function of voters' in-TVM part, forms aggregated cipher text and decrypts it to receive voting result (
which in fact contains counted votes due to additively-homomorphic property of the protocol). Together with that
decryption procedure decryption proof will be generated which should be used by every participant to verify decryption.
To upload voting results and decryption proof administrator uses ```update_tally``` function. And finally administrator
call ```commit_tally``` to make it possible for the voters to get this data by calling to the ```get_m_sum```
, ```get_dec_proof```. After voters download it they also download encrypted ballots of other voters just like
administrator using ```get_ct``` function (to get other voters' in-TVM addresses they should call
to `get_voters_addresses` function of admin' in-TVM part). Then they aggregate downloaded ballots and using aggregated
encrypted ballot and decryption proof they could verify voting results.

Operations described in previous paragraph (except calls to in-TVM functions) are executes by the admin and voters
locally (not in-TVM). To familiarize with its implementation see bin/cli/src/main.cpp where all protocol specified
actions were modeled (to see results in readable format just run cli with the `--mode encrypted_input` flag).

Administrator checks commit statuses of the voters calling to the ```get_voters_statuses```. Example call:

```sh
tonos-cli call 0:4eb04909a3d8e451a9ff30ad7f9659a1d4fb04eb0e85f8fca58905aa181fd450 get_voters_statuses '{}' --abi voting_admin.abi.json --sign keys/voting_admin.keys.json
```

Example answer:

```sh
#...
Succeeded.
Result: {
  "value0": {
    "0:b8e25c3ebbcfc28af45938ba580ab3a82f426331e67a0237079fe72d314cbfe3": false,
    "0:8dc2d5ae85de8f1da509f7ed190409c06cb9d191ab2a23c59a74219b16db8f52": false,
    "0:a72953bd268862c86648bd299bc4e0566176035128a490d8551df52ece40ec84": false,
    "0:38e18f490682cc199f97bec66c716508ae6379dcbb903f2976f89eed1b92a9a5": true
  }
}
```
