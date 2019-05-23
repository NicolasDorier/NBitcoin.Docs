# Connect to your own node via the P2P protocol

NBitcoin allows you to connect to the P2P protocol. The documentation of the P2P protocol can be found spread accross many [Bitcoin Improvement Proposals (BIPs)](https://github.com/bitcoin/bips/).

You can easily test the P2P protocol with the test framework
```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using NBitcoin;
using NBitcoin.Altcoins;
using NBitcoin.Protocol;
using NBitcoin.Protocol.Behaviors;
using NBitcoin.Tests;

namespace NBitcoinTraining
{
    class Program
    {
        // For this to build, you need to set C# 7.1 in your csproj file
        // <PropertyGroup>
        //   ...
        //   ...
        //   <LangVersion>7.1</LangVersion>
        // </PropertyGroup>
        static async Task Main(string[] args)
        {
            // During the first run, this will take time to run, as it download bitcoin core binaries (more than 40MB)
            using (var env = NodeBuilder.Create(NodeDownloadData.Bitcoin.v0_18_0, Network.RegTest))
            {
                // Removing node logs from output
                env.ConfigParameters.Add("printtoconsole", "0");
                var testNode = env.CreateNode();
                env.StartAll();

                var p2p = testNode.CreateNodeClient();
                p2p.VersionHandshake();
                System.Console.WriteLine("Connected via P2P!");
                testNode.Generate(101);

                var headerChain = p2p.GetSlimChain();
                System.Console.WriteLine($"Header chain retrieved with {headerChain.Height} blocks");
            }
        }
    }
}
```

Output:
```
Connected via P2P!
Header chain retrieved with 101 blocks
```

Outside of the test framework you would this instead to connect to your node:

```csharp
var p2p = await Node.ConnectAsync(Network.Main, "mynode.com");
```

You will see how to use the P2P protocol to retrieve blocks and transaction in the [Advanced Wallet Design part](AdvancedWalletDesign.md).

# Using Tor

For using the P2P protocol via Tor, you need to properly setup your SOCKS5 proxy configuration before connecting your node.

In this example, we assume you are running the Tor Browser locally. 

Running the Tor Browser will create a SOCKS proxy on port `9150`, which NBitcoin can use to route `.onion` traffic. This proxy sends the traffic through Tor.

By default, clearnet destination will not use Tor. If you want to force all traffic to use tor, set `SocksSettingsBehavior.OnlyForOnionHosts` to `false`.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using NBitcoin;
using NBitcoin.Altcoins;
using NBitcoin.Protocol;
using NBitcoin.Protocol.Behaviors;
using NBitcoin.Tests;

namespace NBitcoinTraining
{
    class Program
    {
        // For this to build, you need to set C# 7.1 in your csproj file
        // <PropertyGroup>
        //   ...
        //   ...
        //   <LangVersion>7.1</LangVersion>
        // </PropertyGroup>
        static async Task Main(string[] args)
        {
            var connectionParameter = new NodeConnectionParameters();
            // Assuming you are running the Tor browser locally
            var socksConfig = new SocksSettingsBehavior(Utils.ParseEndpoint("localhost", 9150));
            connectionParameter.TemplateBehaviors.Add(socksConfig);

            using (var node = await Node.ConnectAsync(Network.Main, "7xnmrhmkvptbcvpl.onion", connectionParameter))
            {
                node.VersionHandshake();
                Console.WriteLine("Connected!");
            }
        }
    }
}
```

Assuming `7xnmrhmkvptbcvpl.onion` is still online when you try this code,

Output:
```
Connected!
```

# Node Behaviors

The concept of `NodeBehavior` in NBitcoin is a way to encapsulate easily some logic that should apply to a node.

NBitcoin has some pre-made behaviors in the namespace `NBitcoin.Protocol.Behaviors` that you can explore like:

* `AddressManagerBehavior`: this listen and broadcast peer information and save them into a file. This is useful if you connect to untrusted peers and want to speed up connection recovery when your program is restarting.
* `PingPongBehavior`: This behavior is attached by default. This make sure you handle Ping/Pong messages as described by [BIP31](https://github.com/bitcoin/bips/blob/master/bip-0031.mediawiki).
* `SlimChainBehavior/ChainBehavior`: This make sure that you keep a `SlimChain` or `ConcurrentChain` instance up-to-date with the headers that the peer you connect to know about.
* `SocksSettingsBehavior`: You saw this one previously, this behavior has no active behavior, but provide information to NBitcoin to connect via a SOCKS proxy.

Let's see an example of how you can use those behavior.
Imagine that you want to keep an instance of `SlimChain` (a chain of header's hashes) in sync with a node.

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using NBitcoin;
using NBitcoin.Altcoins;
using NBitcoin.Protocol;
using NBitcoin.Protocol.Behaviors;
using NBitcoin.Tests;

namespace NBitcoinTraining
{
    class Program
    {
        // For this to build, you need to set C# 7.1 in your csproj file
        // <PropertyGroup>
        //   ...
        //   ...
        //   <LangVersion>7.1</LangVersion>
        // </PropertyGroup>
        static async Task Main(string[] args)
        {
            // During the first run, this will take time to run, as it download bitcoin core binaries (more than 40MB)
            using (var env = NodeBuilder.Create(NodeDownloadData.Bitcoin.v0_18_0, Network.RegTest))
            {
                // Removing node logs from output
                env.ConfigParameters.Add("printtoconsole", "0");
                var alice = env.CreateNode();
                var miner = env.CreateNode();
                var minerRPC = miner.CreateRPCClient();
                var aliceRPC = alice.CreateRPCClient();
                env.StartAll();
                SlimChain chain = new SlimChain(env.Network.GenesisHash);
                var slimChainBehavior = new SlimChainBehavior(chain);
                var aliceNodeConnection = alice.CreateNodeClient();
                aliceNodeConnection.Behaviors.Add(slimChainBehavior);
                aliceNodeConnection.VersionHandshake();
                miner.Generate(101);
                alice.Sync(miner, true);

                await WaitInSync(alice, chain);
                System.Console.WriteLine($"SlimChain height: {chain.Height}");
            }
        }

        private static async Task WaitInSync(CoreNode node, SlimChain chain)
        {
            var rpc = node.CreateRPCClient();
            while (chain.Tip != await rpc.GetBestBlockHashAsync())
            {
                await Task.Delay(100);
            }
        }
    }
}
```

The `SlimChainBehavior` will manage to always keep your chain in sync with the chain of the remote node.

Output:
```
SlimChain height: 101
```

Encapsulating your logic into a Behavior can have advantages, as you can then leverage NBitcoin to automatically reconnect to an unreliable node as we will see in [Advanced Wallet Design](AdvancedWalletDesign.md), or to maintain connection to a group of node on which you want to attach the same behaviors.