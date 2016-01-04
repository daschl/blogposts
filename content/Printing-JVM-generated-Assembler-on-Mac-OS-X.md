+++
date = "2013-06-24"
draft = false
title = "Printing JVM generated Assembler on Mac OS X"
description = "A quick tutorial on how to get a handle on the assembly code produced by the JVM on OS X."
tags = ["java","jvm","assembler"]
slug = "Printing-JVM-generated-Assembler-on-Mac-OS-X"
+++

Thankfully, the JVM abstracts all of the nitty gritty details from us. Sometimes though, we need to peel off the first layers and see what's going on underneath. If you are curious (and here may be dragons) and want to learn about the actual [assembler](http://en.wikipedia.org/wiki/Assembly_language) that your code is generating, the JVM provides mechanisms to inspect it.

Since I wanted to make it work on my development machine and didn't find something comprehensive for Mac, here is how to do it.

First, make sure to have a more or less recent JDK installed. Mac ships with Java 6, but I think you want to upgrade to 7. You can grab the JDK packages from [here](http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html) if you haven't already.

	~ $ java -version
	java version "1.7.0_17"
	Java(TM) SE Runtime Environment (build 1.7.0_17-b02)
	Java HotSpot(TM) 64-Bit Server VM (build 23.7-b01, mixed mode)

Now to enable the ASM output, you need to pass in two flags, namely `UnlockDiagnosticVMOptions` and `PrintAssembly`. Because the generated ASM is different for each runtime, you need to pass it to the `java` command and not `javac`.

Create a very simple script like this and name it `Main.java`:

	public class Main {
		public static void main(String[] args) {
			System.out.println("Hello World");
		}
	}

Now, we're going to compile and run it with those options:

	michael@daschlbook ~/Downloads/java $ javac Main.java && java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly Main
	Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output
	Could not load hsdis-amd64.dylib; library not loadable; PrintAssembly is disabled
	Hello World

Woops, not what we expected. The code did compile properly, but HotSpot complains about `hsdis-amd64.dylib`. I had to google a bit to find it, but you can download [the file](https://kenai.com/projects/base-hsdis/downloads/download/gnu-versions/hsdis-amd64.dylib) from [here](https://kenai.com/projects/base-hsdis/downloads/directory/gnu-versions).

Now we need to put it somewhere to make it loadable, and the easiest thing I found is to put it onto `LD_LIBRARY_PATH`. Make sure to not override any other settings, but in my case the variable was empty so its straightforward.

	export LD_LIBRARY_PATH=~/PathToFile/

If you run our command again from before, you should now see "beautiful" ASM code generated:

	michael@daschlbook ~/Downloads/java $ javac Main.java && java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly Main
	Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output
	Loaded disassembler from hsdis-amd64.dylib
	Decoding compiled method 0x000000010ac74150:
	  0x000000010ac742ba: add	[eax], al
	[Disassembling for mach='i386(base-hsdis)']
	[Entry Point]
	[Constants]
	  # {method} 'hashCode' '()I' in 'java/lang/String'
	  #           [sp+0x30]  (sp of caller)
	  0x000000010ac742a0: inc	esp
	  0x000000010ac742a1: mov	edx, [esi+0x8]
	  0x000000010ac742a4: dec	ecx
	  0x000000010ac742a5: shl	edx, 3
	  0x000000010ac742a8: dec	ecx
	  0x000000010ac742a9: cmp	eax, edx
	  0x000000010ac742ab: jnz	0x000000000ac4ba60  ;   {runtime_call}
	  0x000000010ac742b1: nop
	  0x000000010ac742b4: invalid	0x0f #size=0
	  0x000000010ac742b5: pop	ds
	  0x000000010ac742b6: test	[eax], al
	  0x000000010ac742b8: add	[eax], al
	  0x000000010ac742ba: add	[eax], al
	  0x000000010ac742bc: nop
	[Verified Entry Point]
	  0x000000010ac742c0: mov	[esp-0x14000], eax
	  0x000000010ac742c7: push	ebp
	  0x000000010ac742c8: dec	eax
	  0x000000010ac742c9: sub	esp, 0x0000000000000020
	                                                ;*synchronization entry
	                                                ; - java.lang.String::hashCode@-1 (line 1446)
	...

Now I guess this is were the real fun starts, happy debugging!