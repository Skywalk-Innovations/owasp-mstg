# Tampering and Reverse Engineering

Reverse engineering and tampering techniques have long belonged into the realm of crackers, modders, malware analysts, and other more exotic professions. For "traditional" security testers and researchers, reverse engineering has been more of a complementary, nice-to-have-type skill that wasn't all that useful in 99% of day-to-day work. But the tides are turning: Mobile app black-box testing increasingly requires testers to disassemble compiled apps, apply patches, and tamper with binary code or even live processes. The fact that many mobile apps implement defenses against unwelcome tampering doesn't make things easier for us.

Mobile security testers should be able to understand basic reverse engineering concepts. It goes without saying that they should also know mobile devices and operating systems inside out: The processor architecture, executable format, programming language intricacies, and so forth.

Reverse engineering is an art, and describing every available facet of it would fill a whole library. The sheer range techniques and possible specializations is mind-blowing: One can spend years working on a very specific, isolated sub-problem, such as automating malware analysis or developing novel de-obfuscation methods. Security testers are generalists: To be effective reverse engineers, they must be able filter through the vast amount of information to build a workable methodology.

There is no generic reverse engineering process that always works. That said, we'll describe commonly used methods and tools later on, and give examples for tackling the most common defenses.

## Why Should You Even Bother?

To sum things up, mobile security testing requires at least basic reverse engineering skills for several reasons:

**1. To enable black-box testing of mobile apps.** Modern apps often employ technical controls that will hinder your ability to perform dynamic analysis. SSL pinning and E2E encryption could prevent you from intercepting or manipulating traffic with a proxy. Root detection could prevent the app from running on a rooted device, preventing you from using advanced testing tools. In this cases, you must be able to deactivate these defenses.

**2. To enhance static analysis in black-box security testing.** In a black-box test, static analysis of the app bytecode or binary code is helpful for getting a better understanding of what the app is doing. It also enables you to identify certain flaws, such as credentials hardcoded inside the app.

**3. To assess resiliency against reverse engineering.**  Apps that implement the software protection measures listed in MASVS-R should be resilient against reverse engineering to a certain degree. In this case, testing the reverse engineering defenses ("resiliency assessment") is part of the overall security test. In the resiliency assessment, the tester assumes the role of the reverse engineer and attempts to bypass the defenses.

## Before You Start

Before you dive into the world of mobile app reversing, we have some good news and some bad news for you. Let's start with the good news:

**Ultimately, the reverse engineer always wins.**

This is even more true in the mobile world, where less defensive options are available to developers: The way mobile apps are deployed and sandboxed is much more restrictive by design, and is simply not feasible to include the rootkit-like functionality that can often be found in Windows software (e.g. DRM systems). You - the reverse engineer - have a much higher degree of control over the mobile operating system, giving you easy auto-wins in many situation (assuming you know how to use that power).

What is the bad news you ask? Reverse engineering jobs can still be extremely difficult, backbreaking (or better: brain-frying) work in cases where the app does everything it can to make your life difficult. Dealing with multi-threaded anti-debugging controls, cryptographic white-boxes, stealthy anti-tampering features and highly complex control flow transformations is not for the faint-hearted. By nature, the best software protection schemes are highly proprietary, and while many tasks can be automated, the way to successful reversing is plastered with good amounts of thinking, coding, frustration, and - depending on your personality - sleepless nights and strained relationships.

(... TODO ...)

## Basic Tampering Techniques

Tampering is the process of making changes to a mobile app either the compiled app, or the running process) or its environment to affect changes in its behavior. For example, and app might refuse to running on your rooted test device, making it impossible to run some of your tests. In cases like that, you'll want to alter that particular behavior.

In the following section we'll give a high level overview of the techniques most commonly used in mobile app security testing. Later, we'll drill down into  OS-specific details for both Android and iOS.

### Binary Patching

Patching means making changes to the compiled app - e.g. changing code in a binary executable file(s), modifying Java bytecode, or tampering with resources. Patches can be applied in any number of ways, from decompiling and re-assembling an app, to editing binary files in a hex editor - anything goes (this rule applies to all of reverse engineering). We'll give some detailed examples for useful patches in later chapters.

One thing to keep in mind is that modern mobile OSes strictly enforce code signing, so running modified apps is not as straightforward as it used to be in traditional Desktop environments. Yep, security experts had a much easier life in the 90ies! Fortunately, this is not all that difficult to do if you work on your own device - it simply means that you need to re-sign the app, or disable the default code signing facilities to run modified code.

### Runtime Modifications

Code injection is a very powerful technique that allows you to explore and modify processes during runtime. The injection process can be implemented in various ways*, but you'll get by without knowing all the details thanks to freely available, well-documented tools that automate it. These tools give you direct access to process memory and important structures such as live objects instantiated by the app, and come with many useful utility functions for resolving loaded libraries, hooking methods and native functions, and more. Tampering with process memory is more difficult to detect than patching files, making in the preferred method in the majority of cases.

Substrate, Frida and XPosed are the most widely used code injection frameworks. The three frameworks differ in design philosophy and implementation details: Substrate and Xposed only focus on code injection and hooking, while Frida aims to be a full-blown "dynamic instrumentation framework" that incorporates both code injection and language bindings, as well as an injectable JavaScript VM and console. Substrate however does provide code injection support for Cycrypt, the programming environment (a.k.a. "Cycript-to-JavaScript" compiler) authored by Saurik of Cydia fame.

#### Substrate, Frida and Xposed

Cydia Substrate (formerly called MobileSubstrate) is the de-facto framework for developing run-time patches (“Cydia Substrate extensions”) on iOS. It comes with Cynject, a tool that provides code injection support for C. By injecting a JavaScriptCore VM into a running process on iOS, users can interface with C code, with support for primitive types, pointers, structs and C Strings, as well as Objective-C objects and data structures. It is also possible to access and instantiate Objective-C classes inside a running process. Some examples for the use of Cycript are listed in the iOS chapter.

Xposed is another popular code injection framework for Android. Installing Xposed on a rooted Android device allows you to apply runtime modifications to processes. The Xposed framework is described in more detail in the [Android Reverse Engineering](/Document/0x06a-Reverse-Engineering-and-Tampering-Android.md) chapter.

Frida is a dynamic instrumentation framework that lets the user you inject JavaScript into native apps on Windows, Mac, Linux, iOS, Android, and QNX. We'll cover Frida in a bit more detail below.

To complicate things, Frida's authors also created a fork of Cycript named ["frida-cycript"](https://github.com/nowsecure/frida-cycript) that replaces Cycript's runtime with a Frida-based runtime called Mjølner. This enables Cycript run on all the platforms and architectures maintained by frida-core. The release was accompanies by a blog post by Ole titled "Cycript on Steroids", which prompted a vitriolic response by Saurik on [Reddit](https://www.reddit.com/r/ReverseEngineering/comments/50uweq/cycript_on_steroids_pumping_up_portability_and/).

Ultimately, you can achieve many of the same goals with either framework. Frida is however the most versatile, as it can inject a Javascript VM on both Android and iOS, while Cycript injection with Substrate only works on iOS.

#### Dynamic Instrumentation with Frida

Code injection can be achieved in different ways. For example, Xposed makes some permanent modifications to the Android app loader that provide hooks to run your own code every time a new process is started. In contrast, Frida achieves code injection by writing code directly into process memory. The process is outlined in a bit more detail below.

When you "attach" Frida to a running app, it uses ptrace to hijack a thread in a running process. This thread is used to allocate a chunk of memory and populate it with a mini-bootstrapper. The bootstrapper starts a fresh thread, connects to the Frida debugging server running on the device, and loads a dynamically generated library file containing the Frida agent and instrumentation code. The original, hijacked thread is restored to its original state and resumed, and execution of the process continues as usual.

Frida injects a complete JavaScript runtime into the process, along with a powerful API that provides a wealth of useful functionality, including calling and hooking of native functions and injecting structured data into memory. It also supports interaction with the Android Java runtime, such as interacting with objects inside the VM.

![Frida](Images/Chapters/0x06/frida.png)

*FRIDA Architecture, source: http://www.frida.re/docs/hacking/*

(todo... add some Frida console examples and links)

## Static / Dynamic Binary Analysis - Old-Fashioned Way

Reverse engineering is the process of reconstructing the semantics of the original source code from a compiled program. In other words, you take the program apart, run it, simulate parts of it, and do other unspeakable things to in order to understand what exactly it is doing and how.

### Using Disassemblers and Decompilers

Disassemblers and decompilers allow you to translate an app's binary code or byte-code back into a more or less understandable format. In the case of native binaries, you'll usually obtain assembler code matching the architecture the app was compiled for. Android Java apps can be disassembled to Smali (an Assembler language for the dex format), and also quite easily converted back to Java code.

A wide range of tools and frameworks is available: From expensive, but convenient GUI tools, to open source disassembler engines and reverse engineering frameworks. Advanced usage instructions for any of these tools often easily fill a book on their own. We'll introduce some of the most widely used disassemblers in the following section. The best way to get started is simply pick the a that fits your needs and budget and buy a well-reviewed user guide along with it (some recommendations are listed below).


TODO: introduce a few standard tools, IDA Pro, Hopper, Radare2, JEB (?)

TODO: Talk about IDA Scripting and the many plugins developed by the community

### Debugging

### Execution Tracing


## Advanced Techniques

For more complicated tasks, such as de-obfuscating heavily obfuscated binaries, you'll won't get far without automating certain parts of the analysis. For example, understanding and simplifying a complex control flow graph manually in the disassembler would take you years (and most likely drive you made way before you're done). Instead, you can augment your workflow with custom made scripts or tools. Fortunately, modern disassemblers come with scripting and extension APIs, and many useful extensions are available for popular ones. Additionally, open-source disassembler engines and binary analysis frameworks exist to make your life easier.

Like always in hacking, the anything-goes-rule applies: Simply use whatever brings you closer to your goal most efficiently. Every binary is different, and every reverse engineer has their own style. Often, the best way to get to the goal is to combine different approaches, such as emulator-based tracing and symbolic execution, to fit the task as hand. To get started, pick a good disassembler and/or reverse engineering framework and start using them to get comfortable with their particular features and extension APIs. Ultimately,  the best way to get better is getting hands-on experience.

(... TODO ...)

### Dynamic Binary Instrumentation

Another useful method for dealing with native binaries is dynamic binary instrumentations (DBI). Instrumentation frameworks such as Valgrind and PIN support fine-grained instruction-level tracing of single processes. This is achieved by inserting dynamically generated code at runtime. Valgrind compiles fine on Android, and pre-built binaries are available for download. The [Valgrind README](http://valgrind.org/docs/manual/dist.readme-android.html) contains specific compilation instructions for Android.

### Emulation-based Dynamic Analysis

Running an app in the emulator gives you powerful ways to monitor and manipulate its environment. For some reverse engineering tasks, especially those that require low-level instruction tracing, emulation is the best (or only) choice.

(... TODO ...)

### Program Analysis Using Symbolic / Concolic Execution

TODO: Introduce RE frameworks

In the 2000s, symbolic-execution based testing has gained increased popularity as a means of identifying security vulnerabilities. Symbolic "execution" actually refers to the process of representing possible paths through a program as formulas in first-order logic, whereby variables are represented as symbols. So-called SMT solvers are used to check satisfiability of those formulas and provide a solution, including concrete values for the variables needed to reach a certain point of execution.

Typically, this approach is used in combination with other techniques such as dynamic execution to improve code coverage. However, it also comes in handy for supporting de-obfuscation tasks, such as simplifying control flow graphs. For example, Jonathan Salwan and Romain Thomas have shown how to reverse engineer VM-based software protections using Dynamic Symbolic Execution (i.e., using a mix of actual execution traces, simulation and symbolic execution) [1].

In the Android section, you'll find a walkthrough for cracking a simple license check in an Android application using symbolic execution.

### Domain-Specific De-Obfuscation Attacks

## References

[1] https://triton.quarkslab.com/files/csaw2016-sos-rthomas-jsalwan.pdf
