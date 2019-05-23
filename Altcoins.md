# Alice sends through Bob through Altcoin RPC

NBitcoin and the test framework support a bunch altcoins. You can find the [list here](https://github.com/MetacoSA/NBitcoin/blob/master/NBitcoin.Altcoins/README.md).

Those altcoins are mainly bitcoin clone and any bitcoin specific logic is in general applicable to altcoin as well.
The design of NBitcoin respect this, for example let's adapt [the RPC example](RPC.md) to use Litecoin instead.

You can go back to this project, then add `NBitcoin.Altcoins` package.

```bash
dotnet package add NBitcoin.Altcoins
```
Then, here is how you can adapt to litecoin:
```diff
diff --git a/NBitcoinTraining.csproj b/NBitcoinTraining.csproj
index 92b89fa..6dce647 100644
--- a/NBitcoinTraining.csproj
+++ b/NBitcoinTraining.csproj
@@ -7,6 +7,7 @@

   <ItemGroup>
     <PackageReference Include="NBitcoin" Version="4.1.2.31" />
+    <PackageReference Include="NBitcoin.Altcoins" Version="1.0.2.6" />
     <PackageReference Include="NBitcoin.TestFramework" Version="1.7.4" />
   </ItemGroup>

diff --git a/Program.cs b/Program.cs
index b9b4983..42d6ecd 100644
--- a/Program.cs
+++ b/Program.cs
@@ -3,6 +3,7 @@ using System.Linq;
 using System.Collections.Generic;
 using System.Threading;
 using NBitcoin;
+using NBitcoin.Altcoins;
 using NBitcoin.Tests;

 namespace NBitcoinTraining
@@ -12,7 +13,7 @@ namespace NBitcoinTraining
         static void Main(string[] args)
         {
             // During the first run, this will take time to run, as it download bitcoin core binaries (more than 40MB)
-            using (var env = NodeBuilder.Create(NodeDownloadData.Bitcoin.v0_18_0, Network.RegTest))
+            using (var env = NodeBuilder.Create(NodeDownloadData.Litecoin.v0_16_3, Litecoin.Instance.Regtest))
             {
                 // Removing node logs from output
                 env.ConfigParameters.Add("printtoconsole", "0");
@@ -26,9 +27,9 @@ namespace NBitcoinTraining
                 Console.WriteLine("Connect nodes to each other");
                 miner.Sync(bob, true);

-                Console.WriteLine("Generate 101 blocks so miner can spend money");
+                Console.WriteLine("Generate enough blocks so miner can spend money");
                 var minerRPC = miner.CreateRPCClient();
-                miner.Generate(101);
+                miner.Generate(env.Network.Consensus.CoinbaseMaturity + 1);

                 var bobRPC = bob.CreateRPCClient();
                 var bobAddress = bobRPC.GetNewAddress();
@@ -46,9 +47,9 @@ namespace NBitcoinTraining
                                 .Where(c => c.ScriptPubKey == aliceAddress.ScriptPubKey)
                                 .ToDictionary(c => c.Outpoint, c => c);

-                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()}");
+                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()} {env.Network.NetworkSet.CryptoCode}");

-                Console.WriteLine("Alice send 1 BTC to bob");
+                Console.WriteLine($"Alice send 1 {env.Network.NetworkSet.CryptoCode} to bob");

                 var txBuilder = Network.RegTest.CreateTransactionBuilder();
                 var aliceToBobTx = txBuilder.AddKeys(aliceKey)
@@ -78,8 +79,8 @@ namespace NBitcoinTraining
                 miner.Generate(1);
                 miner.Sync(bob);

-                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()}");
-                Console.WriteLine($"Bob Balance: {bobRPC.GetBalance()}");
+                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()} {env.Network.NetworkSet.CryptoCode}");
+                Console.WriteLine($"Bob Balance: {bobRPC.GetBalance()} {env.Network.NetworkSet.CryptoCode}");
             }
         }
     }
```

Full `Program.cs`:

```csharp
using System;
using System.Linq;
using System.Collections.Generic;
using System.Threading;
using NBitcoin;
using NBitcoin.Altcoins;
using NBitcoin.Tests;

namespace NBitcoinTraining
{
    class Program
    {
        static void Main(string[] args)
        {
            // During the first run, this will take time to run, as it download bitcoin core binaries (more than 40MB)
            using (var env = NodeBuilder.Create(NodeDownloadData.Litecoin.v0_16_3, Litecoin.Instance.Regtest))
            {
                // Removing node logs from output
                env.ConfigParameters.Add("printtoconsole", "0");

                var bob = env.CreateNode();
                var miner = env.CreateNode();
                miner.ConfigParameters.Add("txindex", "1"); // So we can query a tx from txid
                env.StartAll();
                Console.WriteLine("Created 3 nodes (alice, bob, miner)");

                Console.WriteLine("Connect nodes to each other");
                miner.Sync(bob, true);

                Console.WriteLine("Generate enough blocks so miner can spend money");
                var minerRPC = miner.CreateRPCClient();
                miner.Generate(env.Network.Consensus.CoinbaseMaturity + 1);
                
                var bobRPC = bob.CreateRPCClient();
                var bobAddress = bobRPC.GetNewAddress();

                Console.WriteLine("Alice gets money from miner");
                var aliceKey = new Key();
                var aliceAddress = aliceKey.PubKey.GetAddress(Network.RegTest);
                var minerToAliceTxId = minerRPC.SendToAddress(aliceAddress, Money.Coins(20m));

                Console.WriteLine("Mine a block and check that alice is now synched with the miner (same block height)");
                minerRPC.Generate(1);
                
                var minerToAliceTx = minerRPC.GetRawTransaction(minerToAliceTxId);
                var aliceUnspentCoins = minerToAliceTx.Outputs.AsCoins()
                                .Where(c => c.ScriptPubKey == aliceAddress.ScriptPubKey)
                                .ToDictionary(c => c.Outpoint, c => c);

                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()} {env.Network.NetworkSet.CryptoCode}");

                Console.WriteLine($"Alice send 1 {env.Network.NetworkSet.CryptoCode} to bob");

                var txBuilder = Network.RegTest.CreateTransactionBuilder();
                var aliceToBobTx = txBuilder.AddKeys(aliceKey)
                         .AddCoins(aliceUnspentCoins.Values)
                         .Send(bobAddress, Money.Coins(1.0m))
                         .SetChange(aliceAddress)
                         .SendFees(Money.Coins(0.00001m))
                         .BuildTransaction(true);

                Console.WriteLine($"Alice broadcast to miner");
                minerRPC.SendRawTransaction(aliceToBobTx);

                foreach (var input in aliceToBobTx.Inputs)
                {
                    // Let's remove what alice spent
                    aliceUnspentCoins.Remove(input.PrevOut);
                }
                foreach (var output in aliceToBobTx.Outputs.AsCoins())
                {
                    if (output.ScriptPubKey == aliceAddress.ScriptPubKey)
                    {
                        // Let's add what alice received
                        aliceUnspentCoins.Add(output.Outpoint, output);
                    }
                }

                miner.Generate(1);
                miner.Sync(bob);

                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()} {env.Network.NetworkSet.CryptoCode}");
                Console.WriteLine($"Bob Balance: {bobRPC.GetBalance()} {env.Network.NetworkSet.CryptoCode}");
            }
        }
    }
}
```

Output:

```
Created 3 nodes (alice, bob, miner)
Connect nodes to each other
Generate enough blocks so miner can spend money
Alice gets money from miner
Mine a block and check that alice is now synched with the miner (same block height)
Alice Balance: 20.00000000 LTC
Alice send 1 LTC to bob
Alice broadcast to miner
Alice Balance: 18.99999000 LTC
Bob Balance: 1.00000000 LTC
```

So when you develop on top of NBitcoin, a good practice is to never hard code `Network.Regtest/Mainnet/Testnet`. Instead, fetch the network based on user's input.

Here is an example of refactoring where I replace the hard coded `Network.Mainnet` by a code having the exact same result, but where `cryptoCode` and `useMainnet` could eventually come from configuration file or some other user input.

```diff
+ var cryptoCode = "BTC";
+ var useMainnet = true;

+ var networkSet = AltNetworkSets.GetAll().First(n => n.CryptoCode == cryptoCode);
+ Network network = useMainnet ? networkSet.Mainnet : networkSet.Testnet;
- Network network = Network.Mainnet;
// Continue working on network instance
```

The `Network` class contains network specific information about the crypto currency you are using. `network.Consensus` also gather useful information.

If you want to add support for your own currency, [read this](https://github.com/MetacoSA/NBitcoin/blob/master/NBitcoin.Altcoins/README.md).

In the rest of this book, we might hard code `Network.RegTest`, but keep in mind that you can substitute any example with another network instance from another crypto currency.
