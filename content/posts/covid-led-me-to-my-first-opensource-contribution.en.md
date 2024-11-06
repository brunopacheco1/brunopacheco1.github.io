+++ 
date = 2023-01-04T00:34:00+01:00
title = "Covid led me to my first open-source contribution"
+++

It's been a while since I've shared anything here. Mostly because my interests and priorities have changed. But now I'm back, and I hope to produce more content.

Let's go back to 2020, when I stopped writing. The Covid era, baby!

There were people going crazy because they couldn't see other people. There were people baking bread and brewing beer at home. And I was becoming a first-time parent, or at least starting the journey.

I'm an engineer by love and a developer to the bone. Like everyone else, I was looking for something to focus my thoughts on. And an article about a COBOL system that was overloaded in the US caught my attention. Of course, I had heard of COBOL in university, but the most "hardcore" language I had briefly touched was C/C++.

I was bored, and I didn't want to see more of REST, more of C-like languages, or more of architectural problems. I wanted something more basic, easy to start and easy to stop. Why not COBOL? I thought it might be useful in the future.

So, like any developer would do, I started a web course. If you're interested, you can check it out on [LinkedIn Learning](https://www.linkedin.com/learning/learning-cobol-2015).

What did I find out? COBOL is an elitist and segregating language. You have to have access to mainframes to code, and you have to have access to paid tools to code and debug. There is only one open-source compiler for COBOL, [GnuCOBOL](https://gnucobol.sourceforge.io/). And when I started my journey, there were zero open-source debuggers.

I was totally disappointed. How could that be? COBOL is a f**** old language. So I started digging around forums and I found an IBM learning platform that was oriented towards COBOL. If I remember correctly, the students got access to mainframes. But one thing caught my attention: there was another frustrated person complaining about the lack of open-source tools to code on a regular computer. They shared that they were writing a [COBOL debugger](https://marketplace.visualstudio.com/items?itemName=OlegKunitsyn.gnucobol-debug) for VSCode.

I tried out that tool and I thought it was a great idea. But soon I realized it wasn't mature enough to debug even one of the simplest tasks I did in the web course I mentioned.

Well, as we say in Rio, I had the cheese and the knife in my hands. I'm a developer, and I wanted to learn COBOL. So why not contribute to that tool? That little comment turned my course in a divergent direction. Soon I forgot about "how to read a file in COBOL" and started asking "how GnuCOBOL transpiles code into C", "how to debug C in the command line", and "how gdb integrates with VSCode."

In the middle of the project, I was already in touch with GnuCOBOL developers, asking them for new compiler features to help debug float and binary data types supported by that compiler. I was also adding support for JavaScript expressions in the debugger inspection view. And in my latest contributions, I was working on debugging remote processes and SSH tunneling.

All of this, all of those long nights coding, just because of a comment on a forum about COBOL.

And you know what? It was an amazing experience. I learned a lot of things, perhaps nothing directly useful for my current daily duties, but at least I contributed to an open-source project, impacting the lives of thousands of other anonymous developers.

**See you | Bis geschwënn | Até mais | À bientôt**
