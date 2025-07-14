<div style="text-align:center">

<p>

# Introduction

SolanaNet is a comprehensive, high-performance .NET SDK for building applications on the Solana blockchain. Designed with developers in mind, SolanaNet offers robust support for account management, transactions, token programs, staking, DeFi protocols, and more—making it ideal for everything from dApps and wallets to automated services and backend infrastructure.

</p>

</div>


## Features
- Full JSON RPC API coverage
- Full Streaming JSON RPC API coverage
- Wallet and accounts (sollet and solana-keygen compatible)
- Keystore (sollet and solana-keygen compatible)
- Transaction decoding from base64 and wire format and encoding back into wire format
- Message decoding from base64 and wire format and encoding back into wire format
- Instruction decompilation
- TokenWallet object to send SPL tokens and JIT provisioning of [Associated Token Accounts](https://spl.solana.com/associated-token-account)
- Programs
    - Native Programs
      - System Program
      - Stake Program
      - Solana Program Library (SPL)
      - Compute Budget Program
      - Account Compression Program
      - Governance Program
      - StakePool Program
      - Address Lookup Table Program
      - Memo Program
      - Token Program
      - Token Swap Program
      - Associated Token Account Program
      - Name Service Program
      - Shared Memory Program


## Examples

The [SolanaNet.Examples](https://github.com/jackfreemancoder/SolanaNet/tree/main/src/SolanaNet.Examples) contains some code examples,



#### Mini Examples
```c#
// -----------------------------
// Initialize a keypair from a secret key
// -----------------------------
var account = Account.FromSecretKey("your_secret_key_here");

// -----------------------------
// Initialize a Sollet-compatible wallet using mnemonic
// -----------------------------
var sollet = new Wallet("your mnemonic words here", WordList.English);
var accountAtIndex = sollet.GetAccount(10); // Derive account using index

// -----------------------------
// Initialize wallet compatible with solana-keygen (BIP39 + passphrase)
// -----------------------------
var wallet = new Wallet("your mnemonic words here", WordList.English, "optional_passphrase", SeedMode.Bip39);
var mainAccount = wallet.Account; // solana-keygen doesn't support indexed derivation

// -----------------------------
// Generate a new wallet and mnemonic
// -----------------------------
var newMnemonic = new Mnemonic(WordList.English, WordCount.Twelve);
var newWallet = new Wallet(newMnemonic);

// -----------------------------
// Encrypt and decrypt secret key, seed, or mnemonic using KeyStore
// -----------------------------
var keyStoreService = new SecretKeyStoreService();
string encryptedJson = keyStoreService.EncryptAndGenerateDefaultKeyStoreAsJson(password, data, address);
try {
    byte[] decryptedData = KeyStore.DecryptKeyStoreFromJson(password, encryptedJson);
} catch (Exception) {
    Console.WriteLine("Invalid password provided!");
}

// -----------------------------
// Restore a wallet using SolanaKeyStoreService and JSON keystore
// -----------------------------
var solanaKeyStoreService = new SolanaKeyStoreService();
var restoredWallet = solanaKeyStoreService.RestoreKeystore(filePath, passphrase);

// -----------------------------
// Using ClientFactory to get RPC clients with logging support
// -----------------------------
var mainnetClient = ClientFactory.GetClient(Cluster.MainNet, logger);
var paidRpcClient = ClientFactory.GetClient("YOUR_RPC_URL", logger);
var streamingClient = ClientFactory.GetStreamingClient(Cluster.MainNet, logger);

// -----------------------------
// Query blockchain using RPC
// -----------------------------
var accountInfo = paidRpcClient.GetAccountInfo("AccountPublicKeyHere");
var allTokenAccounts = paidRpcClient.GetTokenAccountsByOwner("OwnerPublicKeyHere");
var specificTokenAccounts = paidRpcClient.GetTokenAccountsByOwner("OwnerPublicKeyHere", "TokenMintAddressHere");
var programId = "ProgramIdPublicKeyHere";
var programAccounts = paidRpcClient.GetProgramAccounts(programId);
var filters = new List<MemCmp> { new MemCmp { Offset = 45, Bytes = "OwnerAddressHere" } };
var filteredAccounts = paidRpcClient.GetProgramAccounts(programId, memCmpList: filters);

// -----------------------------
// Using the streaming RPC for real-time transaction updates
// -----------------------------
var txSig = paidRpcClient.SendTransaction(tx);
var subscription = streamingClient.SubscribeSignature(txSig.Result, (state, response) => {
    Console.WriteLine("Transaction finalized!");
}, Commitment.Finalized);

// -----------------------------
// Sending a transaction with compute budget and memo
// -----------------------------
var rpcClient = ClientFactory.GetClient(Cluster.MainNet);
var wallet = new Wallet();
var from = wallet.GetAccount(0);
var to = wallet.GetAccount(1);
var blockHash = rpcClient.GetLatestBlockHash();
var tx = new TransactionBuilder()
    .SetRecentBlockHash(blockHash.Result.Value.Blockhash)
    .SetFeePayer(from)
    .AddInstruction(ComputeBudgetProgram.SetComputeUnitLimit(30000))
    .AddInstruction(ComputeBudgetProgram.SetComputeUnitPrice(1000000))
    .AddInstruction(MemoProgram.NewMemo(from, "Hello from Sol.Net :)"))
    .AddInstruction(SystemProgram.Transfer(from, to.GetPublicKey, 100000))
    .Build(from);
var signature = rpcClient.SendTransaction(tx);

// -----------------------------
// Create, initialize, and mint new SPL token
// -----------------------------
var mintAccount = wallet.GetAccount(21);
var owner = wallet.GetAccount(10);
var initialHolder = wallet.GetAccount(22);
var rentExemptionMint = rpcClient.GetMinimumBalanceForRentExemption(TokenProgram.MintAccountDataSize).Result;
var rentExemptionToken = rpcClient.GetMinimumBalanceForRentExemption(TokenProgram.TokenAccountDataSize).Result;
var mintTx = new TransactionBuilder()
    .SetRecentBlockHash(blockHash.Result.Value.Blockhash)
    .SetFeePayer(owner)
    .AddInstruction(SystemProgram.CreateAccount(owner, mintAccount, rentExemptionMint, TokenProgram.MintAccountDataSize, TokenProgram.ProgramIdKey))
    .AddInstruction(TokenProgram.InitializeMint(mintAccount.PublicKey, 2, owner.PublicKey, owner.PublicKey))
    .AddInstruction(SystemProgram.CreateAccount(owner, initialHolder, rentExemptionToken, TokenProgram.TokenAccountDataSize, TokenProgram.ProgramIdKey))
    .AddInstruction(TokenProgram.InitializeAccount(initialHolder.PublicKey, mintAccount.PublicKey, owner.PublicKey))
    .AddInstruction(TokenProgram.MintTo(mintAccount.PublicKey, initialHolder.PublicKey, 25000, owner))
    .AddInstruction(MemoProgram.NewMemo(initialHolder, "Token initialized!"))
    .Build(new List<Account> { owner, mintAccount, initialHolder });

// -----------------------------
// Transfer token to a new token account
// -----------------------------
var newHolder = wallet.GetAccount(23);
var transferTx = new TransactionBuilder()
    .SetRecentBlockHash(blockHash.Result.Value.Blockhash)
    .SetFeePayer(owner)
    .AddInstruction(SystemProgram.CreateAccount(owner, newHolder, rentExemptionToken, TokenProgram.TokenAccountDataSize, TokenProgram.ProgramIdKey))
    .AddInstruction(TokenProgram.InitializeAccount(newHolder.PublicKey, mintAccount.PublicKey, owner.PublicKey))
    .AddInstruction(TokenProgram.Transfer(initialHolder.PublicKey, newHolder.PublicKey, 25000, owner))
    .AddInstruction(MemoProgram.NewMemo(newHolder, "Token transferred"))
    .Build(new List<Account> { owner, newHolder });

// -----------------------------
// Decode transaction or message from base64 or byte array
// -----------------------------
var decodedTx = Transaction.Deserialize(txData);
var decodedMsg = Message.Deserialize(msgData);
var signedTx = Transaction.Populate(decodedMsg, new List<byte[]> { account.Sign(msgData) });

// -----------------------------
// Hello Solana - Send memo onchain
// -----------------------------
var helloTx = new TransactionBuilder()
    .SetFeePayer(wallet.Account)
    .AddInstruction(ComputeBudgetProgram.SetComputeUnitLimit(30000))
    .AddInstruction(ComputeBudgetProgram.SetComputeUnitPrice(1000000))
    .AddInstruction(MemoProgram.NewMemo(wallet.Account, "Hello Solana World, using SolanaNet :)"))
    .SetRecentBlockHash(rpcClient.GetRecentBlockHash().Result.Value.Blockhash)
    .Build(wallet.Account);

// -----------------------------
// Create and fund an Associated Token Account (ATA)
// -----------------------------
PublicKey tokenOwner = new PublicKey("65EoWs57dkMEWbK4TJkPDM76rnbumq7r3fiZJnxggj2G");
PublicKey ata = AssociatedTokenAccountProgram.DeriveAssociatedTokenAccount(tokenOwner, mintAccount);
var ataTx = new TransactionBuilder()
    .SetRecentBlockHash(recentHash.Result.Value.Blockhash)
    .SetFeePayer(owner)
    .AddInstruction(AssociatedTokenAccountProgram.CreateAssociatedTokenAccount(owner, tokenOwner, mintAccount))
    .AddInstruction(TokenProgram.Transfer(initialHolder, ata, 25000, owner))
    .AddInstruction(MemoProgram.NewMemo(owner, "ATA funded"))
    .Build(new List<Account> { owner });
var ataSig = rpcClient.SendTransaction(ataTx);

// -----------------------------
// Decode instructions from a transaction message
// -----------------------------
var message = Message.Deserialize(msgBase64);
var decodedInstructions = InstructionDecoder.DecodeInstructions(message);

// -----------------------------
// Display all token balances of a wallet
// -----------------------------
var tokens = TokenMintResolver.Load();
var tokenWallet = TokenWallet.Load(client, tokens, owner);
var balances = tokenWallet.Balances();
foreach (var token in tokenWallet.TokenAccounts()) {
    Console.WriteLine($"{token.Symbol} {token.BalanceDecimal} {token.TokenName} {token.PublicKey} {(token.IsAssociatedTokenAccount ? "[ATA]" : "")}");
}

// -----------------------------
// Send SPL token using TokenWallet
// -----------------------------
var sourceToken = tokenWallet.TokenAccounts().ForToken(WellKnownTokens.Serum).WithAtLeast(12.75M).FirstOrDefault();
var sendSig = tokenWallet.Send(sourceToken, 12.75M, target, txBuilder => txBuilder.Build(owner));
Console.WriteLine($"Transaction sent: {sendSig}");

```

