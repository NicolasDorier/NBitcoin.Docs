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
