# "Unpacking" VMProtect

On my recent oncall rotation I was hit with a VMProtect packed ffdroid sample in T92023736. The teardown report for the sample can be found here, but this was my first time being faced with a VMProtect packed sample so I figured I'd write up the details of my methodology for getting the sample to a largely "debuggable" state which allowed me to glean the behavior of the binary. Here's the deets on the file if anyone is interested in following along:

f415405d413c1ad16b85c003f2ec6cda83d24518c4fb8a6e4aaaafc58dbb1254

**Note:** I'm not going to cover what VMProtect is or how it works, as it's complicated, and Rolf Rolles has already done a way better job than I could ever pretend to do here. I'd recommend if you want more context on what this note is about to start with his blog series. In this note I'm going to go through how do we relatively quickly get our VMProtected sample to a reverse-able state. As such I gloss over some of the technical nuances or details and focus more on recovering the packed/virtualized binary.

Googling around for "How to unpack VMProtect" will get you quite a few videos that offering suggestions, like this one. They all seem to come back to a similar theme, which is to start by breaking on VirtualProtect and follow the .text segment in a raw memory dump until you see the executable code, so let's start there.

For this walk through I'll be using x64dbg, as I continually forget how to use WinDBG.

// Image 1: Breakpoint set on VirtualProtect, Memory Dump 1 is our .text segment

// Image 2: Continue 3 or 4 times, we see the .text segment has machine code

As we can see from the image chain above, this seems to work. At this point you could dump the .text segment and start investigating around the sample in IDA, and for sufficiently simple samples this _may_ be enough to recover most of the functionality of the program.

However if your program is more than a "Hello World" there'll be quite a few things missing. Specifically we don't know where OEP (Original Entry Point) is, and without it IDA will have a tough time knowing where to start the disassembly, especially if the entry point isn't at the start of the segment. You'd likely get quite a bit of structure, but some functions may not show up or may be wonky. Moreover we don't have an IAT for all of our library imports for our user code, only those imports used throughout the VMProtect code. As such, it behoves us to try and get our sample to unfold itself a bit more before dumping the sample and starting our reversing.

Coincidentally, a short time back I had been chatting with Wesley about reversing VMProtect samples. He mentioned there was a talk recently at a conference (I've misplaced the link but will track it down and link to it in the hints section below), given by an RE who was attempting exactly this, unpacking a VMProtected binary. Specifically they mentioned to break on RtlExitUserThread, as at some point, the threads gotta exit, and see what you can derive by tracing stack frames. This has the added benefit of hopefully any encrypted strings or resources might still be available in raw memory in the clear, as opposed to just the hopefully unpacked user code. So let's set a breakpoint on RtlExitUserThread and see what we get:

// Image 3: Break on RtlExitUserThread, on the stack we see residual frames

Neat! We at least hit our breakpoint. Now it's simply a matter of trying to understand "who" is trying to exit. If we "walk" down the stack towards higher memory addresses, we should be able to trace through frames and find our way (hopefully) into user code somewhere, as we see in the picture above. Following the 0x40d86e address in our disassembler shows us a function prologue, which means we've at least found some workable user code.

At this point, if you take a dump of the program and try using 0x40d86e as the OEP, you'll find you don't get too terribly much. IDA will disassemble some of the program, but you find that this doesn't look like it's the OEP. We could keep trying to find the final RtlExitUserThread call, but the program is pretty long running, so we might be better to come up with another approach.

At this point I made some very lucky guesses, that the program might be doing some thread creation, and if we're hitting on RtlExitUserThread with some user-land code references in our stack, it's very likely that if I break on CreateThread I might catch some more workable OEP-candidates. So let's give that a shot:

// Image 4

It seems we're correct as we see in this screenshot. The 0x40d86e address has been pushed onto the stack as a parameter to CreateThread and we have a new user-land code address, 0x40d823, where we'll return from after kicking off our new thread.

// Image 5

Great, we see a function prologue, and it looks like we might be somewhere in the depths of our primary routines. Now that we've found a good pivot point let's see how far down the stack we can trace return addresses to try and get back closer to OEP. This process gets us (or at least got me) to 0x40ebe7 which based off of the following picture, looks pretty promising for some early code execution.

// Image 6: "better" candidate for OEP

To get a feel for how early in the code execution we are, I set a hardware breakpoint here, and re-ran the program:

Alright, a lot happening in this picture, let's break down a couple of important bits:

1. Looking at the stack layout from the function that called into 0x40ebe7, we're at a much higher memory address than where we jumped to (0x490cdc), and there are a couple of unresolved addresses on the stack, as well as some higher 0x5b addresses which is where our VMProtect code lives.
2. We see at the code right before our OEP-candidate, the 0x400000 address is being pushed onto the stack, which is our .text segment.
3. The MZ header is clearly apparent at our 0x400000 address in 2

This is some pretty good signal that we're likely at OEP, or at least closer to it, as the feel of the stack is that we've just gotten to an unpacked state, ready to start executing some user-written code.

At this point let's take our memory dump with 0x40ebe7 as our OEP. Opening the program up in IDA we now have a binary we can start working with.

Reconstructing a partial IAT
It wont take long of reversing this dumped binary, before you notice sub_40CB3E seems to have a bunch of encoded/encrypted strings in it. As such it's a great candidate to spend some time on.

// Image 7

One can spot that all of the strings are being passed to sub_470d68, for which there's a decryption routine in the report I wrote here, and decrypting all the strings we see there's some dll and API names being stored in data offsets. A good hint that some pieces of the IAT may be decrypted early in the program execution for use later.

// Image 8

We could spend more time annotating in IDA and trying to fix all of the references to the newly discovered IAT components, but let's just have our debugger do the hard work for us. To do this I set a break point after this identified decryption routine, at 0x40ec40 for me, and let the program run. After the decryption function returns you can check the dword_50fab0 region where the decrypted API references were being stored in the IDA screen cap above, and then use Scylla dump to again generate a clear binary dump with a decrypted IAT at 0x50fab0, and in the screen cap below we see that Scylla has recovered an IAT that our user-code actually uses.

// Image 9

Checking out our dump in disassembly/decompilation we get a bit more functionality derived by IDAs auto-analysis. In the following IDA-caps (Download the image, as it's hard to tell from the Note preview), the left hand picture is after we've dumped the binary with our decrypted imports, the right hand is before.

// Image 10: Decrypted Imports vs Non-Decrypted Imports

## Interlude on translating API calls that were not decrypted/recovered

While we were able to recover some of the decrypted API calls, there's a lot left obfuscated by the virtualization layers of VMProtect. At least for this sample, translating the calls wasn't too horrible it just took a bit of time. The following movie shows the general process:

// Movie: 0:00 / 1:18

1. Identify one of the obfuscated API calls. Typically it'll be a function call followed immediate by a \x90, or NOP, instruction.
2. Break on the function call and the NOP right after, and trace the execution all the way until you jump/return to the API call itself.
3. I try to slow down my "step into" calls when I get close to a ret or jmp instruction as the API call is resolved and then immediately jumped/returned to after.
4. Once you've identified the API call continue execution to your NOP-breakpoint and annotate as desired.
With this methodology, and time, and elbow-grease, and patience, you'll eventually be able to tell the broader story of what the sample is up to.

## Recap/Tips and Tricks for REing VMProtected Bins

* Try to identify some of the behavior your program may be using from Windows APIs, like CreateThread/RtlExitUserThread or maybe ReadFile/WriteFile or something. Break on these functions and try to trace through the stack frames until you get back into user code, in an attempt to derive the OEP.
* The IAT found by Scylla initially is probably wrong, and referencing the IAP used by the VMProtect code. You may need to generate a couple of dumps after the program has unpacked a bit more in order to get something working, and this is assuming the sample you're investigating is doing additional obfuscation/encryption on the imports.
* Identify where VMProtect is doing the function translation. In my sample subroutines being translated looked like a call followed by a single \x90 instruction, and when disassembled with IDA looked like hot garbage. Once found, step-into-debug until you hit the API call itself. This process helped annotate my IDB enough to understand generically what the code was doing.
* **Honorable Mentions:**
  * There's a project called vmpdump, which seemed like it would attempt to reconstruct the IAT for you, but I wasn't able to get it running. I took a small stab at fixing up the code, but it seemed like more trouble than it was worth as I didn't have to translate too many imports to get a loose understanding of what the program was doing.
  * JSB did a mediocre job of identifying some of the programs behaviors, so one might be able to detonate in JSB to identify what API functions might be good candidates for breaking during user-code execution to find OEP.
