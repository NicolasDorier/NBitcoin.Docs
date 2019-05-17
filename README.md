# Introduction

What a long road. I started writing NBitcoin in 2014 in order to learn about Bitcoin, this was being done by copy and pasting code from BitcoinJ and Bitcoin Core, along with their test suite and making sure it worked in .NET.

Along the road I learned about Bitcoin. I am the kind of programmer who can't learn something without coding it first.
As I learned about Bitcoin, I started writing [Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/) on a word document. Along the road, nopara73, joined me and transfered it to Github and GitBook and rewrote lot's of content.

Since then lot's of community member improved the book, that's the beauty of github.

If you want to learn conceptually about Bitcoin, I advice you to read [Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/) first before reading this documentation.

[Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/) introduces you primitive concept of Bitcoin which are useful whichever language you end writing bitcoin.

This book is about programming with NBitcoin you will learn additional concept, not covered in [Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/), and mainly how to use them in NBitcoin.

NBitcoin is a tool I use everyday for other projects, be it closed source project or open source project (like [BTCPay Server](https://docs.btcpayserver.org)). This mean that there is a very tight feedback loop.

The process is about:
* Add a new feature on NBitcoin
* Use such feature for another project
* Finding what is hard to use on NBitcoin
* Fixing NBitcoin to have better idea

This feedback loop sometimes take 10 min. This is why NBitcoin has today 604 releases.

This mean that you can be sure that NBitcoin is the easiest library to use for bitcoin development. If it was not, I would shot myself in the foot.

This book will allow me to explain the design of NBitcoin which has been shaped by my direct experience into using NBitcoin.

This book is about practical development, so I expect you to open visual studio, understand concepts, run and debug the code snippets.
Once you understand, you need to apply the concept on whatever project you want to work on.

# Development setup

Because this book is meant to be used for people using Windows, Linux or Mac OS, we advise you to use [Visual studio Code](https://code.visualstudio.com/).

Download and install the last version of [.NET Core SDK](https://dotnet.microsoft.com/download).

Then create your first project and add `NBitcoin` and `NBitcoin.TestFramework`.
```bash
mkdir NBitcoinTraining
cd NBitcoinTraining
dotnet new console
dotnet add package NBitcoin
dotnet add package NBitcoin.TestFramework
```

Then open the folder with Visual Studio Code.

```bash
code .
```

You can then put breakpoints and debug from inside Visual studio code. You can hit F5 to run your code.
This might ask you to install the extension `C# for Visual Studio Code (powered by OmniSharp)`.

# NBitcoin's test framework

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

                Console.WriteLine("Alice get money from miner");
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
Alice get money from miner
Mine a block and check that alice is now synched with the miner (same block height)
Alice Balance: 20.00000000
Alice send 1 BTC to bob
Alice mine her own transaction
Alice Balance: 18.99996680
Bob Balance: 1.00000000
```

We encourage you to run this example several time, try to play with the code.
One trick you will often see is that when `Alice` send money to Bob, she mines her own transaction and make sure Bob is aware of this block.

This is done this way because when Alice create a transaction, it would take time to propagate to `Bob`, so we would need to put some sleep in the code. Instead, we just get `Alice` to mine her own transaction and make sure `Bob` received it. Then we can be sure the balance has been updated.

You will use this trick a lot during your development.

# Debugging inside NBitcoin

NBitcoin is published with symbols so that Visual Studio Code can fetch sources from github and you can easily debug inside NBitcoin library easily.

## In Visual Studio Code

Inside your `.vscode/launch.json`, add the following to .NET Core Launch (console) configuration:
```json
"justMyCode": false,
"symbolOptions": {
    "searchPaths": [ "https://symbols.nuget.org/download/symbols" ],
    "searchMicrosoftSymbolServer": true
},
```

For example, here is my own `launch.json`:

```json
{
   // Use IntelliSense to find out which attributes exist for C# debugging
   // Use hover for the description of the existing attributes
   // For further information visit https://github.com/OmniSharp/omnisharp-vscode/blob/master/debugger-launchjson.md
   "version": "0.2.0",
   "configurations": [
        {
            "name": ".NET Core Launch (console)",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "build",
            // If you have changed target frameworks, make sure to update the program path.
            "program": "${workspaceFolder}/bin/Debug/netcoreapp2.2/NBitcoinTraining.dll",
            "args": [],
            "cwd": "${workspaceFolder}",
            // For more information about the 'console' field, see https://github.com/OmniSharp/omnisharp-vscode/blob/master/debugger-launchjson.md#console-terminal-window
            "console": "internalConsole",
            "stopAtEntry": false,
            "justMyCode": false,
            "symbolOptions": {
                "searchPaths": [ "https://symbols.nuget.org/download/symbols" ],
                "searchMicrosoftSymbolServer": true
            }
        },
        {
            "name": ".NET Core Attach",
            "type": "coreclr",
            "request": "attach",
            "processId": "${command:pickProcess}"
        }
    ]
}
```

Now, when I debug into Visual Studio Code, I can step inside NBitcoin's source code. (with `F11` while breaking)

We advise you to re-enable `Enable Just My Code` when you are done. Because Loading symbols can be quite time consuming.

## In Visual Studio

You need to run at least Visual Studio 15.9. Then, you need to:

* `Go in Tools / Options / Debugging / General` and turn **off** `Enable Just My Code`.
* `Go in Tools / Options / Debugging / Symbols` and add `https://symbols.nuget.org/download/symbols` to the `Symbol file (.pdb) locations`, make sure it is checked.

You should also check `Microsoft Symbol Server` or your debugging experience in visual studio will be slowed down.

Now you can Debug your project and step inside any call to NBitcoin under visual studio.

We advise you to re-enable `Enable Just My Code` when you are done. Because Loading symbols can be quite time consuming.