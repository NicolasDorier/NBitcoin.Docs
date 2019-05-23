# Introduction

What a long road. I started writing NBitcoin in 2014 in order to learn about Bitcoin, this was being done by copy and pasting code from BitcoinJ and Bitcoin Core, along with their test suite and making sure it worked in .NET.

Along the road I learned about Bitcoin. I am the kind of programmer who can't learn something without coding it first.
As I learned about Bitcoin, I started writing [Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/) on a word document. Along the road, nopara73, joined me and transfered it to Github and GitBook and rewrote lot's of content.

Since then lot's of community member improved the book, that's the beauty of github.

If you want to learn conceptually about Bitcoin, I advise you to read [Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/) first before reading this documentation.

[Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/) introduces you primitive concept of Bitcoin which are useful whichever language you end up writing bitcoin.

This book is about programming with NBitcoin you will learn additional concept, not covered in [Programming The Blockchain in C#](https://programmingblockchain.gitbooks.io/programmingblockchain/content/), and mainly how to use them in NBitcoin.

NBitcoin is a tool I use everyday for other projects, be it closed source project or open source project (like [BTCPay Server](https://docs.btcpayserver.org)). This means that there is a very tight feedback loop.

The process is about:
* Add a new feature on NBitcoin
* Use such feature for another project
* Finding the pain points of using this feature
* Fixing NBitcoin to make the use case easier to code

This feedback loop sometimes take 10 min. This is why NBitcoin has today 604 releases.

This means that you can be sure that NBitcoin is the easiest library to use for bitcoin development. If it was not, I would shoot myself in the foot.

This book will allow me to explain the design of NBitcoin which has been shaped by my direct experience into using NBitcoin.

This book is about practical development, so I expect you to open visual studio, understand concepts, run and debug the code snippets.
Once you understand, you need to apply the concept on whatever project you want to work on.
