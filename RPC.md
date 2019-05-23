# Alice sends through Bob through Bitcoin RPC

When developing on Bitcoin, it is critical to be able to easily run your code to verify it works properly.
This is what we will use in all our tests. This is the goal of the `NBitcoin.TestFramework` package.

This library allows you to create a network of node and connect them to one another.
Let's see an example, it may takes a while to run at first because Bitcoin Core binaries, please feel free to debug step by step:

`Miner` give some money to `Alice` which gives 1 BTC to `Bob`.

```csharp
using System;
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

                var alice = env.CreateNode();
                var bob = env.CreateNode();
                var miner = env.CreateNode();
                env.StartAll();
                Console.WriteLine("Created 3 nodes (alice, bob, miner)");

                Console.WriteLine("Connect nodes to each other");
                miner.Sync(alice, true);
                miner.Sync(bob, true);

                Console.WriteLine("Generate 101 blocks so miner can spend money");
                var minerRPC = miner.CreateRPCClient();
                miner.Generate(101);
                
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

                Console.WriteLine("Alice send 1 BTC to bob");
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

This will output the following:

```
Created 3 nodes (alice, bob, miner)
Connect nodes to each other
Generate 101 blocks so miner can spend money
Alice gets money from miner
Mine a block and check that alice is now synched with the miner (same block height)
Alice Balance: 20.00000000
Alice send 1 BTC to bob
Alice mine her own transaction
Alice Balance: 18.99996680
Bob Balance: 1.00000000
```

We encourage you to run this example several time, try to play with the code.
One trick you will often see is that when `Alice` send money to Bob, she mines her own transaction and make sure Bob is aware of this block.

This is done this way because when Alice creates a transaction, it would take time to propagate to `Bob`, so we would need to put some sleep in the code. Instead, we just get `Alice` to mine her own transaction and make sure `Bob` received it. Then we can be sure the balance has been updated.

You will use this trick a lot during your development.