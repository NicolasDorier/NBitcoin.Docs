# Connecting to P2P via Tor

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