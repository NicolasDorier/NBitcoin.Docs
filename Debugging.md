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
   "version": "0.2.0",
   "configurations": [
        {
            "name": ".NET Core Launch (console)",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "build",
            "program": "${workspaceFolder}/bin/Debug/netcoreapp2.2/NBitcoinTraining.dll",
            "args": [],
            "cwd": "${workspaceFolder}",
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
