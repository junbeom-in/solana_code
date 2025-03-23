# 사전에 해야할것

# Solana Code
## Basics
1. Smart contract = Programs
	1. BPF bytecode -> Blockchain
2. Program = Program ID(PUBKEY) 로 구분
3. STATELESS (Unlike Ethereum) 
	1. NO DATA STORED
4. Persistent STroage? Use accounts!
## Account
- Holds Data
- SOL Tokens (Lamports 1 SOL = 1 billion lamports)
- Executasble OR NOT
- Data modification? = Must be Owner
- Fund -> Create accounts -> Transfer Ownership to the program
## Sample Code
### There's no Serialization format in Solana!
1. 1 Account -> Argument
2. Greeting Count
3. Verfies the account owner!
4. Desirialize the data -> ``Greeting Account`` structure
5. Increment the counter
6. Re-Serialize
7. Print a message!
```rust
use borsh::{BorshDeserialize, BorshSerialize}; // 시리얼리제이션

// Solana
use solana_program::{
    account_info::{next_account_info, AccountInfo}, // READ - Account INFOs
    entrypoint, entrypoint::ProgramResult,          // ENTRY POINT
    msg,                                           // PRINT (LOG)
    program_error::ProgramError,
    pubkey::Pubkey,
};

#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct GreetingAccount {
//Unsigned 32bit
//# of greetings
//ITTRs
    pub counter: u32,
}

// Entrypoint
// Function called when the program is executed
entrypoint!(process_instruction);

// The entrypoint function signature 
// Solana runtime calls this when the program executes
pub fn process_instruction(
    program_id: &Pubkey,        // Public key of this program (공개키)
    accounts: &[AccountInfo],   // Accounts passed in to the program (목적지)
    _instruction_data: &[u8],   // Serialized instruction data (명령어)
) -> ProgramResult {
    msg!("JUN = Hello World program executed"); // Print a message to the program log (결과값)

    // Iterate through the accounts
    let accounts_iter = &mut accounts.iter();
    let account = next_account_info(accounts_iter)?; // 1st Account from the list

    // Owner 확인
    // Permission to write!!!!
    if account.owner != program_id {
        msg!("Account is not owned by this program");
        return Err(ProgramError::IncorrectProgramId);
    }

    // Deserialize (account data) -> GreetingAccount struct
    let mut greeting_data = GreetingAccount::try_from_slice(&account.data.borrow())?;
    // Increment the counter
    greeting_data.counter = greeting_data.counter + 1;
    // Serialize (modified data) -> account
    greeting_data.serialize(&mut &mut account.data.borrow_mut()[..])?;

    msg!("Greeted {} time(s)!", greeting_data.counter); 
    Ok(())
}
```

# Deployment
## DEVNET vs TESTNET
## 1. Compile
- Compile -> Deploy the PRORAM ON A NETWORK!
- Which network?
	- Test : DEVNET
		- **Devnet** need airdrops of SOL for free!
	- Real : MAINNET(-BETA)
		- **Mainnet** need REAL SOL for fees
### 1. Install the Solana Tools
```shell
https://solana.com/docs/intro/installation
```

### 2. Choose a cluster
```sh
#DEVNET
solana config set --url https://api.devnet.solana.com

#MAINNET
solana config set --url https://api.mainnet-beta.solana.com
```

### 3. Generate a keypair
#### For Deployment and paying fees
```sh
solana-keygen new --force
```
JSON -> Save into the file somewhere

### 4. Fund the Keyfair (TESTNET)
```sh
solana airdrop 2
```

### 5. Build the program
- RUST
	- Cargo is OK
```sh
# Cargo
cargo build-bpf --release

# Solana SDK
npm run build:program-rust
```

### 6. Deploy the program
#### Program ID (Pubkey) = My account!
```sh
solana program deploy Users/Jun/Documents/testcode.so
```

# Execution
- Typescript + Solana-web3.js
## Design
### 1. Connect to the cluster (RPC)
### 2. Generate or load keypairs (FEE-PAYING + Program-OWNED account)
### 3. Create the program-owned account
### 4. Build and send the transactions

## Code
```javascript
import { Connection, Keypair, PublicKey, SystemProgram, Transaction } from '@solana/web3.js';
import { BN } from 'bn.js'; // Big number library for handling large integers

// 1. Connect to a Solana cluster
const connection = new Connection('https://api.devnet.solana.com', 'confirmed'); 
// Use 'confirmed' or ERROR

// 2. Generate a new Keypair for the account that will store the greeting count
const greetedAccount = Keypair.generate(); 

// 3. Define the program ID
const programId = new PublicKey('Program Id');

// 4. Calculate the required space
// (rent for the data account)
const GREETING_SIZE = 4; // u32 = which is 4 bytes
const rentExemption = await connection.getMinimumBalanceForRentExemption(GREETING_SIZE);

// 5. Create a transaction to create the account and call the program
let tx = new Transaction();

// 6. Add instruction to create the account [system program create account]
tx.add(SystemProgram.createAccount({
    fromPubkey: payer.publicKey,        // 1. funds the creation
    newAccountPubkey: greetedAccount.publicKey, // 2. public key of the account TO BE CREATED
    lamports: rentExemption,            // 3. Amount of lamports ( fund rent exemption)
    space: GREETING_SIZE,              // 4. Space in bytes ( account data)
    programId: programId,               // 5. designate program as the owner of this account
}));

// 7. Add instruction to call the program's entrypoint
tx.add({
    keys: [
      { pubkey: greetedAccount.publicKey, isSigner: false, isWritable: true }
    ],
    programId: programId,
    data: Buffer.alloc(0) // no instruction data needed for this program (we're just incrementing counter)
});

// 8. Sign and send the transaction
tx.feePayer = payer.publicKey;
tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
tx.sign(payer, greetedAccount); // Sign with the payer
const txSignature = await connection.sendRawTransaction(tx.serialize());
console.log("Transaction sent, signature:", txSignature);
```

# JITO
## Keypoints
### PriorityFees vs Jito Tips
 - Priority Fee : Solana
	 - Add @ Compute Budget Instruction
 - Tips : Jito (Out of Protocol - Special Program)
	 - Minimum Tip (1000 lamports)
### Submission?
- Normal
	- RPC Nodes + gossip network -> Leader's transaction queue
- Jito
	- Jito Block Engine -> Current leader (Modified Client) - FASTER!!!

### Bundles
- Bundling transactions (1 , More)
- All-or-nothing
- Atomic multi-tx sequences NOT POSSIBLE w/o combining into 1 transaction
- Executions In order OR NOT

### Ordering
- Normal
	- First come first serve
	- NO Public Mempool
- Jito
	- Private MEMPOOL
	- Searchers submit transaction -> Jito Validator picks!
	- Flashbot vs Jito
		- Flashbot : All BLOCK AUCTION
		- Jito : Partial Block auction

### Protection
- **bundleOnly=true**
	- Only land if they can be includedvia the bundles
	- Will not simply fall back to the normal path
	- Revert Protection
		- Arbitrage -> Fail
			- UNLESS certain conditions are met
			- Not execute AT ALL
			- UNless it's in the intended bundle ordering!
		- Simulation
			- "PROFIT" calculated by jito block engine!


## Transaction @ Jito
```
1. I want to transfer 0.001 SOL to Bob
2. Prioritize via Jito
3. Attach a tip + send it to the Jito Block engine
```
### 1. Normal Transaction
#### Create a Transaction
- **SystemProgram.transfer**
- Construct Instructions
- Sign ...
### 2. Include a Priority Fee
#### **Compute Budget priority fee instruction**
- ComputeBudgetProgram.setComputeUnitPrice({ microLamports: X })
	- Higher Fee per compute unit
- Jito Validator accepts both normal fees and Jito tips.

### 3. Include a JITO TIP
- Include a SOL transfer to one of the Jito's tip accounts
	- Jito RPC - `getTIPAccounts`
- After the transfer -> `Tip Distribution Program`
	- Lamports -> Block Producer : MEV Rewards

### 3. Sign the Transaction
- **The fee payer** signs the transaction **as usual**.

### 4. Submit via JITO
- NOT USING **connection.sendTransaction**
- Use Jito Block Engine's endpoint (RPC)
```
https://<region>.block-engine.jito.wtf/api/v1/transactions
```
HAS TO BE JSON!

- **sendTransaction** with my transaction (in BASE64)
- Jito will foward it directly to a jito validator!
	- Bypass the normal gossip propagators
- Additionally **bundleOnly=true** can be enabled!


## Codes

```javascript
import {
  Transaction,
  SystemProgram,
  ComputeBudgetProgram,
  sendAndConfirmRawTransaction,
} from '@solana/web3.js';

const transaction = new Transaction();

// 1. Normal transfer instruction
transaction.add(SystemProgram.transfer({
  fromPubkey: me.publicKey,
  toPubkey: bob.publicKey,
  lamports: 1_000_000, // 0.001 SOL (1e6 lamports)
}));

// 2. Priority fee
// Pay 1000 micro-lamports per ComputingUnit for priority
transaction.add(ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 1000n }));

// 3. JITO TIP
// Transfer 10000 lamports (0.00001 SOL) to a Jito tip account
const tipAccount = new PublicKey("키키키KEYKEYKEY"); 

// ^ One of the known Jito tip accounts (getTipAccounts or from docs)
transaction.add(SystemProgram.transfer({
  fromPubkey: jun.publicKey,
  toPubkey: tipAccount,
  lamports: 10000,
}));

// 4. Sign the transaction as ME (fee payer and source of funds)
transaction.feePayer = jun.publicKey;
transaction.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
transaction.sign(jun);

// 5. Encode and send via Jito RPC
const serializedTx = transaction.serialize();
const base64Tx = serializedTx.toString('base64');

// 6. Use fetch to call Jito's sendTransaction RPC endpoint
const jitoUrl = "https://testnet.block-engine.jito.wtf"; 
const res = await fetch(`${jitoUrl}?bundleOnly=true`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    jsonrpc: "2.0",
    id: 1,
    method: "sendTransaction",
    params: [ base64Tx, "base64" ]  // sending the tx in base64 encoding
  })
});
const json = await res.json();
if(json.error) {
  console.error("Jito submission failed:", json.error);
} else {
  console.log("Jito transaction signature:", json.result);
}
```

# BUNDLE 보내기
## See
### APIs
- **getInflightBundleStatuses** : Last 5 minutes check , 5 MAX 
- **getBundleStatuses** : Final outcome
- **getTipAccounts** : Current tip accounts

### Code
```javascript
//JUNBEOM IN, 2025
//1. What to query
const bundleIds = ["BUNDLE ID_1", "BUNDLE ID_2"];

//2. Fetch!
const response = await fetch('https://mainnet.block-engine.jito.wtf/api/v1/bundles', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    jsonrpc: "2.0",
    id: 1,
    method: "getInflightBundleStatuses",
    params: [ bundleIds ]
  })
});

//3. Response
const statusInfo = await response.json();

//4. Print
console.log(statusInfo.result);
```

- Result
	- Pending
	- Landed
	- Failed
	- Dropped
### All transaction view?
- **NOT POSSIBLE**: Jito does not have full mempool
## Create
```javascript
import bs58 from 'bs58'; //base 58
import { Transaction, SystemProgram } from '@solana/web3.js';

//1. Add Transactions
const tx1 = new Transaction().add(
  SystemProgram.transfer({
    fromPubkey: searcher.publicKey,
    toPubkey: target.publicKey,
    lamports: 500000, // Arbitrary action
  })
);

//2. Sign tx1
tx1.feePayer = searcher.publicKey;
tx1.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
tx1.sign(searcher);

//3. Sign tx2
const tx2 = new Transaction().add(
  SystemProgram.transfer({
    fromPubkey: searcher.publicKey,
    toPubkey: tipAccount, // tipAccount from getTipAccounts
    lamports: 2000000,    // e.g., 0.002 SOL tip (2000000 lamports)
  })
);

tx2.feePayer = searcher.publicKey;
tx2.recentBlockhash = tx1.recentBlockhash; // use same blockhash to ensure bundle goes in same block if possible
tx2.sign(searcher);

// Prepare bundle for submission
const bundleTxs = [ bs58.encode(tx1.serialize()), bs58.encode(tx2.serialize()) ];

// Submit the bundle via Jito RPC
const res = await fetch('https://mainnet.block-engine.jito.wtf/api/v1/bundles', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    jsonrpc: "2.0",
    id: 1,
    method: "sendBundle",
    params: [ bundleTxs ]  // an array of base58 encoded transactions
  })
});

// SEND!!!!
const result = await res.json();
if(result.error) {
  console.error("Bundle submission failed:", result.error);
} else {
  const bundleId = result.result;
  console.log("Bundle submitted! ID:", bundleId);
}
```