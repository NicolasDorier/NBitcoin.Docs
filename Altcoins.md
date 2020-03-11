# Alice sends through Bob through Altcoin RPC

NBitcoin and the test framework support a bunch altcoins. You can find the [list here](https://github.com/MetacoSA/NBitcoin/blob/master/NBitcoin.Altcoins/README.md).

Those altcoins are mainly bitcoin clone and any bitcoin specific logic is in general applicable to altcoin as well.
The design of NBitcoin respect this, for example let's adapt [the RPC example](RPC.md) to use Litecoin instead.

You can go back to this project, then add `NBitcoin.Altcoins` package.

```bash
dotnet add package NBitcoin.Altcoins
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
index 144a0a3..ac24447 100644
--- a/Program.cs
+++ b/Program.cs
@@ -1,6 +1,9 @@
 using System;
 using System.Threading;
 using NBitcoin;
+using NBitcoin.Altcoins;
 using NBitcoin.Tests;

 namespace NBitcoinTraining
@@ -9,8 +12,8 @@ namespace NBitcoinTraining
     {
         static void Main(string[] args)
         {
-            // During the first run, this will take time to run, as it download bitcoin core binaries (more than 40MB)
-            using (var env = NodeBuilder.Create(NodeDownloadData.Bitcoin.v0_18_0, Network.RegTest))
+            // During the first run, this will take time to run, as it download litecoin core binaries (more than 40MB)
+            using (var env = NodeBuilder.Create(NodeDownloadData.Litecoin.v0_16_3, Litecoin.Instance.Regtest))
             {
                 // Removing node logs from output
                 env.ConfigParameters.Add("printtoconsole", "0");
@@ -25,15 +28,15 @@ namespace NBitcoinTraining
                 miner.Sync(alice, true);
                 miner.Sync(bob, true);

-                Console.WriteLine("Generate 101 blocks so miner can spend money");
+                Console.WriteLine($"Generate {env.Network.Consensus.CoinbaseMaturity + 1} blocks so miner can spend money");
                 var minerRPC = miner.CreateRPCClient();
-                miner.Generate(101);
+                miner.Generate(env.Network.Consensus.CoinbaseMaturity + 1);

                 var aliceRPC = alice.CreateRPCClient();
                 var bobRPC = bob.CreateRPCClient();
                 var bobAddress = bobRPC.GetNewAddress();

-                Console.WriteLine("Alice get money from miner");
+                Console.WriteLine("Alice gets money from miner");
                 var aliceAddress = aliceRPC.GetNewAddress();
                 minerRPC.SendToAddress(aliceAddress, Money.Coins(20m));

@@ -43,7 +46,7 @@ namespace NBitcoinTraining

                 Console.WriteLine($"Alice Balance: {aliceRPC.GetBalance()}");

-                Console.WriteLine("Alice send 1 BTC to bob");
+                Console.WriteLine($"Alice send 1 {env.Network.NetworkSet.CryptoCode} to bob");
                 aliceRPC.SendToAddress(bobAddress, Money.Coins(1.0m));
                 Console.WriteLine($"Alice mine her own transaction");
                 aliceRPC.Generate(1);

```

Full `Program.cs`:

```csharp
using System;
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
            // During the first run, this will take time to run, as it download litecoin core binaries (more than 40MB)
            using (var env = NodeBuilder.Create(NodeDownloadData.Litecoin.v0_16_3, Litecoin.Instance.Regtest))
            {
                // Removing node logs from output
                env.ConfigParameters.Add("printtoconsole", "0");

                var alice = env.CreateNode();
                var bob = env.CreateNode();
                var miner = env.CreateNode();
                env.StartAll();
                Console.WriteLine("Created 3 nodes (alice, bob, miner)");

                Console.WriteLine("Connect nodes to each other");
                miner.Sync(alice, true);
                miner.Sync(bob, true);

                Console.WriteLine($"Generate {env.Network.Consensus.CoinbaseMaturity + 1} blocks so miner can spend money");
                var minerRPC = miner.CreateRPCClient();
                miner.Generate(env.Network.Consensus.CoinbaseMaturity + 1);
                
                var aliceRPC = alice.CreateRPCClient();
                var bobRPC = bob.CreateRPCClient();
                var bobAddress = bobRPC.GetNewAddress();

                Console.WriteLine("Alice gets money from miner");
                var aliceAddress = aliceRPC.GetNewAddress();
                minerRPC.SendToAddress(aliceAddress, Money.Coins(20m));

                Console.WriteLine("Mine a block and check that alice is now synched with the miner (same block height)");
                minerRPC.Generate(1);
                alice.Sync(miner);

                Console.WriteLine($"Alice Balance: {aliceRPC.GetBalance()}");

                Console.WriteLine($"Alice send 1 {env.Network.NetworkSet.CryptoCode} to bob");
                aliceRPC.SendToAddress(bobAddress, Money.Coins(1.0m));
                Console.WriteLine($"Alice mine her own transaction");
                aliceRPC.Generate(1);

                alice.Sync(bob);

                Console.WriteLine($"Alice Balance: {aliceRPC.GetBalance()}");
                Console.WriteLine($"Bob Balance: {bobRPC.GetBalance()}");
            }
        }
    }
}
```

Output:

```
Created 3 nodes (alice, bob, miner)
Connect nodes to each other
Generate 101 blocks so miner can spend money
Alice gets money from miner
Mine a block and check that alice is now synched with the miner (same block height)
Alice Balance: 20.00000000
Alice send 1 LTC to bob
Alice mine her own transaction
Alice Balance: 18.99668000
Bob Balance: 1.00000000
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

For example, in the above example, we use `network.Consensus.CoinbaseMaturity` to know how many blocks a miner need to mine before being able to spend his reward. While for bitcoin it is 101 blocks, for other altcoins it might be more or less.

If you want to add support for your own currency, [read this](https://github.com/MetacoSA/NBitcoin/blob/master/NBitcoin.Altcoins/README.md).

In the rest of this book, we might hard code `Network.RegTest`, but keep in mind that you can substitute any example with another network instance from another crypto currency.
