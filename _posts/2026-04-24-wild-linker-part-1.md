---
layout: "post"
title:  "Wild linker part - 1"
date:   "2052-04-24 14:23:35 +0530"
categories: linkers
--- 

## eh_frame support

These are necessary to order for stack unwinding to work. Without eh_frames, panics and backtraces won't work. Generally each function in your binary will have an entry in eh_frame that provides information about the stack for any instruction in the function. This is used both for the simple case of printing a stack trace and the more complicated case of unwinding, whcih requires running Drop for variables in each stack frame.

eh_frames requires a few special things from the linker.

* The linker needs to build an index of the frame entries so that the runtime can do a binary search to find the entry for a particular function.
* In order for the index to be valid and not confuse the runtime, its a good idea to drop any frame entries for functions that we've decided not to link. i.e, we don't want frame entries for dead code that the linker removed. That means that eh_frame handling needs to be integrated into the garbage collection pass of the linker, which figures out which sections of the input files to link and which to discard.

Now that Wild supports eh_frames, panic and backtraces work in Rust binaries that we produces.

## ifuncs

An ifunc is a type of function where the implementation is resolved at runtime during program startup. When libc starts, it goes through a list of ifuncs created by the linker and calls a "resolve function" for each ifunc. The resolver function gets passed information such as what CPU features are supported. It can then return the most optimal implementation for the current CPU. libc then stores the returned pointer for use whenever the function is called during program execution.

This is used for functions like memcpy which have faster implementations for some CPUs.

ifuncs are a GNU extension to ELF, but glibc uses them, so if you want to link against glibc, you need to support them.

## Dynamic relocation

Earlier versions of Wild only supported non-relocatable static binaries. This meant that the binary had to be loaded at a fixed address that was decided by the linker. This likely isn't a problem for development, however as a step towards implementing dynamic linking, I though it'd be good to add support for statically linked, relocatable executables.

When a linker runs, it takes the sections of the input files and decides where to put them in the output file. It then needs to apply the relocations required by each section. A relocation is an instruction written by the compiler to tell the linker to put an address of a section or a symbol at a particular location. There are many different types of relocations (43 in the ELP 64 bit spec). Some of these relocations are relative, which means that we write the relative offset to the requested symbol or section. However some things like vtable (used when you have trait objects) require absolute addresses.

When an absolute address is required and we're writing a relocatable binary, the linker doesn't yet know the address of the target. This means that it needs to write a dynamic relocation, which is an instruction to the runtime. These dynamic relocations are applied at startup. Sometimes there can be quite a lot of them. e.g rustc has over 32k dynamic relocations which get processed each time rustc starts.

As an aside, this made me think a bit about whether it would be possible to have vtables that contain only relative pointer. Indeed, there has been some research done on this in relation to LLVM.

Most of the work for dynamic relocation was finding all the places where we needed to apply it. One place that came as a bit of a surprise was the relocations for ifuncs, which I mentioned above. These ifunc relocations need to contain the absolute address of the resolve function and the absolute address where the relocation is to be applied. This meant that I needed to apply dynamic relocations to the ifunc relocation. Applying a relocation to a relocation felt a bit meta. It also made me realise how many layers of complexity Linux ELF (including GNU extensions) has build up over the years. Ideally the ifunc relocations would contain only relative pointers, then we wouldn't need to apply dynamic relocations to them.

Another chunk of work for dynamic relocation was implementing some additional linker micro-optimization. These are not LTO, which we don't yet support, they're small optimizations that the linker does when it's applying relocations to some compiled code. These optimisations are supposed  to be optional for the linker to implement, but as we'll see, this not always the case.

To explain this example, I first need to explain global offset table (GOT). It's a table, build by the linker, that contains pointers to some functions. For functions that the compiler wants to allow to be overriden at runtime by dynamic libraries, the compiler, instead of calling a function directly, requests that the linker create a GOT entry containing a pointer to the function.

Because the global offset table contains the absolute addresses of functions, if we're building a relocatable executable, we need to apply dynamic relocations to all the entries it contains.

Sometimes the linker knows, perhaps because we're statically linking that the function cannot be overriden at runtime, meaning that its free to bypass the GOT entry by transforming indirect calls to the function into direct calls to that same function. This means that we no longer need an entry in the global offset table and, perhaps more importantly, we no longer need to apply a dynamic relocation.

The following assembly code contains a call instruction (which calls a function). That call instruction has a relocation put there by the compiler that request a GOT entry for the function `__libc_start_main` function.

```rust
    1f: ff 15 00 00 00 00    call  *0x0(%rip)
                        21: R_X86_GOTCRELX __libc_start_main-0x4 
```

What's interesting here is that this assembly code is in the function `_start` provided by libc, which is where the program starts executing. It's trying to use the GOT, but dynamic relocations haven't been applied to the GOT yet, because that happens somewhere in `__libc_start_main` which is the function we're trying to call! That means that if we take the compiler's instructions as written, our binary will segfault.

So the "optional" optimization to eliminate the GOT entry turns out to be not so optional after all. This is perhaps not surprising. Once GNU ld implementated optimisations like this, libc would have at some point come to depend on it.

## comment section

I wanted a reliable way to identify whether wild was used to produce a particular binary file. The way this is generally done is by adding a text comment into the `.comment` section in the binary. The compiler rustc also puts a comment stating its version.

You can viwe the comments in a binary using `readelf` as follows:

```rust
>> readelf -p .comment my-binary
String dump of section '.comment':
  [     1]  GCC: (GNU) 9.4.0
  [    12]  rustc version 1.76.0 (07dca489a 2024-02-04)
  [    3e]  GCC: (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
  [    69]  Linker: Wild version 0.1.0
```

## String merging

This work was mostly motivated by the work on the comment section, since each object produced by the compiler has a separate compy of the comment. If we didn't do string merging, then out output binary would have the same comment repeated many times.

String merging is activated when a section of an input file is marked with the merge and strings flags.

You can see these flags with `readelf` by running:

```rust
>> readelf --sections my-binary
```

The comment section might look like:

```
  [16] .comment          PROGBITS         0000000000000000  00003264
       000000000000002d  0000000000000001  MS       0     0     1
```

Here MS are the flags for the `.comment` section and mean merge and strings.

Some linkers like GNU ld take string merging to an extreme. For example if they find a string "foobar" and another string with the same suffix such as "bar" they'll include only "foobar" the make references to "bar" point to the last three bytes of "foobar". This is a neat optimisation, but since we're trying to write a fast linker, we don't do this. Other fast linkers like lld and mold also don't do this - at least not by default.


## Performance

Wild doesn't support all the same features as the more mature linkers, so when computing, its import to try to make the comparison as fair as possible by only exercising features in the other linkers that Wild also supports.

* Linking debug info is very time consuming, so we pass `--strip-debug` to all linkers so that they don't need to spend time linking debug info.
* Wild doesn't yet support dynamic linking, so we ensure that we're building a statically linked executable.
* Wild doesn't yet support build IDs, so we omit those flags from the link command.

The size column is mostly there to check that none of the linkers are doing significantly more or less work than the others. The larger size for GOLD and LLD is mostly because they have twice as much data in their `.gcc_except_table` sections compared with the other linkers. `.gcc_except_table` is used by `.eh_frames` which I described above and contains stuff required for Drop to work when rust unwinds. I am not sure why those two linkers both have exactly 3132732 bytes in this section, while the other three have exactly 1259724. My guess would be that their garbage collection algorithm don't extend to these sections.

For the multithreaded linkers (lld, modl and wild), it's also interesting to look at total CPU time consumed. If we allow as many thread as there are CPUs, then hyperthreading inflates the CPU usage time when two threads are running on the same core. So we restrict the linkers to 4 threads, which how many CPU cores my laptop has.

We also set `--no-fork` for Mold, otherwise if forks a subprocess to do the work, which means we don't get to observe the CPU usage of the process doing the actual work.

Currently Wild has slightly higher CPU usage compared to LLD, but considerably lower than Mold’s. It has more parallelism than LLD’s, but not as much as Mold’s. I’ve got ideas for improving performance, but I’m trying to avoid working on those until I’ve improved correctness and completeness.

All we can really conclude from this benchmark is that Wild is currently reasonably efficient at non-incremental linking and reasonable at taking advantage of a few threads. I don’t think that adding the missing features should change this benchmark significantly. i.e. adding support for debug info really shouldn’t change our speed when linking with no debug info. I can’t be sure however until I implement these missing features.

The goal of Wild isn’t to be the fastest at a cold build, but rather to be very fast at incremental linking, once that’s implemented. But it’s nice to see that it’s currently faring pretty well at linking from scratch.

