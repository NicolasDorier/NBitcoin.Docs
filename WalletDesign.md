# Alice sends through Bob without RPC

In the previous section, we saw how `Alice` can send to `Bob` with the `NBitcoin.TestFramework` library.
This is all well and good, but in many cases, Bitcoin RPC can be very limited and inflexible in term of feature.

So let's try to achieve the same goal as in the previous chapter, but this time, without using RPC on Alice's side.

Let's change the diff and the whole source code:

```diff
diff --git a/Program.cs b/Program.cs
index 144a0a3..b9b4983 100644
--- a/Program.cs
+++ b/Program.cs
@@ -1,4 +1,6 @@
 ﻿using System;
+using System.Linq;
+using System.Collections.Generic;
 using System.Threading;
 using NBitcoin;
 using NBitcoin.Tests;
@@ -15,42 +17,68 @@ namespace NBitcoinTraining
                 // Removing node logs from output
                 env.ConfigParameters.Add("printtoconsole", "0");

-                var alice = env.CreateNode();
                 var bob = env.CreateNode();
                 var miner = env.CreateNode();
+                miner.ConfigParameters.Add("txindex", "1"); // So we can query a tx from txid
                 env.StartAll();
                 Console.WriteLine("Created 3 nodes (alice, bob, miner)");

                 Console.WriteLine("Connect nodes to each other");
-                miner.Sync(alice, true);
                 miner.Sync(bob, true);

                 Console.WriteLine("Generate 101 blocks so miner can spend money");
                 var minerRPC = miner.CreateRPCClient();
                 miner.Generate(101);

-                var aliceRPC = alice.CreateRPCClient();
                 var bobRPC = bob.CreateRPCClient();
                 var bobAddress = bobRPC.GetNewAddress();

                 Console.WriteLine("Alice get money from miner");
-                var aliceAddress = aliceRPC.GetNewAddress();
-                minerRPC.SendToAddress(aliceAddress, Money.Coins(20m));
+                var aliceKey = new Key();
+                var aliceAddress = aliceKey.PubKey.GetAddress(Network.RegTest);
+                var minerToAliceTxId = minerRPC.SendToAddress(aliceAddress, Money.Coins(20m));

                 Console.WriteLine("Mine a block and check that alice is now synched with the miner (same block height)");
                 minerRPC.Generate(1);
-                alice.Sync(miner);
+                
+                var minerToAliceTx = minerRPC.GetRawTransaction(minerToAliceTxId);
+                var aliceUnspentCoins = minerToAliceTx.Outputs.AsCoins()
+                                .Where(c => c.ScriptPubKey == aliceAddress.ScriptPubKey)
+                                .ToDictionary(c => c.Outpoint, c => c);

-                Console.WriteLine($"Alice Balance: {aliceRPC.GetBalance()}");
+                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()}");

                 Console.WriteLine("Alice send 1 BTC to bob");
-                aliceRPC.SendToAddress(bobAddress, Money.Coins(1.0m));
-                Console.WriteLine($"Alice mine her own transaction");
-                aliceRPC.Generate(1);

-                alice.Sync(bob);
+                var txBuilder = Network.RegTest.CreateTransactionBuilder();
+                var aliceToBobTx = txBuilder.AddKeys(aliceKey)
+                         .AddCoins(aliceUnspentCoins.Values)
+                         .Send(bobAddress, Money.Coins(1.0m))
+                         .SetChange(aliceAddress)
+                         .SendFees(Money.Coins(0.00001m))
+                         .BuildTransaction(true);
+
+                Console.WriteLine($"Alice broadcast to miner");
+                minerRPC.SendRawTransaction(aliceToBobTx);
+
+                foreach (var input in aliceToBobTx.Inputs)
+                {
+                    // Let's remove what alice spent
+                    aliceUnspentCoins.Remove(input.PrevOut);
+                }
+                foreach (var output in aliceToBobTx.Outputs.AsCoins())
+                {
+                    if (output.ScriptPubKey == aliceAddress.ScriptPubKey)
+                    {
+                        // Let's add what alice received
+                        aliceUnspentCoins.Add(output.Outpoint, output);
+                    }
+                }
+
+                miner.Generate(1);
+                miner.Sync(bob);

-                Console.WriteLine($"Alice Balance: {aliceRPC.GetBalance()}");
+                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()}");
                 Console.WriteLine($"Bob Balance: {bobRPC.GetBalance()}");
             }
         }
```

The source code is now:

```csharp
using System;
using System.Linq;
using System.Collections.Generic;
using System.Threading;
using NBitcoin;
using NBitcoin.Tests;

namespace NBitcoinTraining
{
    class Program
    {
        static void Main(string[] args)
        {
            // During the first run, this will take time to run, as it download bitcoin core binaries (more than 40MB)
            using (var env = NodeBuilder.Create(NodeDownloadData.Bitcoin.v0_18_0, Network.RegTest))
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

                Console.WriteLine("Generate 101 blocks so miner can spend money");
                var minerRPC = miner.CreateRPCClient();
                miner.Generate(101);
                
                var bobRPC = bob.CreateRPCClient();
                var bobAddress = bobRPC.GetNewAddress();

                Console.WriteLine("Alice get money from miner");
                var aliceKey = new Key();
                var aliceAddress = aliceKey.PubKey.GetAddress(Network.RegTest);
                var minerToAliceTxId = minerRPC.SendToAddress(aliceAddress, Money.Coins(20m));

                Console.WriteLine("Mine a block and check that alice is now synched with the miner (same block height)");
                minerRPC.Generate(1);
                
                var minerToAliceTx = minerRPC.GetRawTransaction(minerToAliceTxId);
                var aliceUnspentCoins = minerToAliceTx.Outputs.AsCoins()
                                .Where(c => c.ScriptPubKey == aliceAddress.ScriptPubKey)
                                .ToDictionary(c => c.Outpoint, c => c);

                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()}");

                Console.WriteLine("Alice send 1 BTC to bob");

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

                Console.WriteLine($"Alice Balance: {aliceUnspentCoins.Select(c => c.Value.TxOut.Value).Sum()}");
                Console.WriteLine($"Bob Balance: {bobRPC.GetBalance()}");
            }
        }
    }
}
```

The output is basically the same as the previous chapter.

```
Created 3 nodes (alice, bob, miner)
Connect nodes to each other
Generate 101 blocks so miner can spend money
Alice get money from miner
Mine a block and check that alice is now synched with the miner (same block height)
Alice Balance: 20.00000000
Alice send 1 BTC to bob
Alice broadcast to miner
Alice Balance: 18.99999000
Bob Balance: 1.00000000
```

Now this solution is not an improvement over the previous [RPC example](RPC.md). The code is harder to read, and you need to keep track of Alice's unspent coins through `aliceUnspentCoins`, here is some problems:

* What if blocks reorgs?
* What if Alice transaction get malleated or double spent?
* How do your prevent address reuse?

However, can you feel the power of having signed a transaction without relying on any software?
