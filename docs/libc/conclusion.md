---
title: Conclusion
nav_order: 5
parent: libc
---

## Future Work

Unfortunately, I spent at least half of the semester just trying to get mlibc to successfully build and link with a user program, that it did not leave me as much time as I would have thought to work on actually implementing the library. As such, there is a good deal of work that could still be done in the future to improve and complete the port of mlibc to PincerOS.

- Fixing the initialization issues
  - I still have not yet figured out the root cause of the issues I was having when setting `_start` as the entry point. Originally, I suspected memory corruption of some sort, because it seemed to be happening around the point where `libraryPaths.initialize` was *returning* rather than actually executing, so maybe the saved link register or other information was getting corrupted. Now, I have begun to suspect it might have something to do with not saving floating point registers on context switches. I began working on a patch to do this saving, but I was not able to get it working before the end of the semester.
- Implementing more syscalls
  - I focused primarily on the process management, virtual memory, and file I/O syscalls, but there are more syscalls needed internally by mlibc, notably those for futexes.
  - `read` and `write` are currently implemented in terms of `pread` and `pwrite`. However, this is not exactly correct, since `pread` and `pwrite` take an offset while `read` and `write` use the implicit offset of the descriptor head instead. For now, they just use an offset of 0, which is good enough for interacting with serial I/O like we have been using for the terminal, but they would not work for actual files.
  - Additionally, there are several functions which are not necessary for mlibc to work, but would be nice to have for more compatibility with real programs, such as in order to support more ANSI and POSIX features.
- Dynamic linking
  - Since I was mainly focused on static binaries, I did not really work on dynamic linking too much. I was able to get the dynamic linker to build, but I did not ever try getting it to run in the PincerOS.
- Implementing Rust std
  - From my understanding, once you have a working libc implementation, it is fairly simple to get the Rust standard library to work.
- Self hosting
  - Given a complete enough implementation of libc, you should be able to get a relatively unmodified C or Rust compiler to run on PincerOS. Then, you could use that to build a new version of the kernel, achieving a self-hosting system.
  - I recognize this is not really a realistic goal, but it would be very cool.

## Lessons Learned

I learned a lot from my adventures in porting mlibc to PincerOS.

1. Building libc is hard, but it helps to have a good build system.
    - It seems that every implementation of libc I've ever encountered is a massive pain to build in one way or another. While I did spend the majority of my time resolving build issues, I greatly appreciated the fact that mlibc uses Meson and Ninja as its build system. In contrast to what I have observed in the past with Newlib's system of Makefiles, mlibc's build system is much more modern and easier to work with.
    - A lot of my issues, at least early on, stemmed from the fact that I was trying to build mlibc for Aarch64 from an x86-64 system. I struggled with cross compilation, flipping back and forth between gcc and clang, dealing with various compiler and linker issues, plus a ton of problems with libgcc/compiler_rt. I have a feeling that if I used a native build system, I would have had a lot fewer issues and probably would have been able to get a lot further.
2. Porting libc is hard, but it helps to have a good implementation.
    - I have mixed feelings about the design of mlibc. On one hand, I really appreciate its modular design with feature flags and the small amount that is actually depended on by the library internally. On the other hand, I still do not completely understand the design decisions behind some of the features, such as the reuse of dynamic linker logic in static builds. I don't know why a static executable needs to call into `__dlapi_entrystack` to parse the stack and `__dlapi_enter` which calls `interpreterMain` to set things related to shared objects.
    - The modular design and small number of internal dependencies make mlibc theoretically a really good candidate for porting to a new system, especially one that has only a few syscalls implemented and therefore would not be able to support something like glibc or musl without a lot of work. Even within the feature flags, its use of [[gnu::weak]] with some special macros means that it can detect at runtime whether or not certain functions are defined, and either use them or set errno to ENOSYS dynamically. This is a really nice feature, and I wish I had more time to explore it and how it actually works.
3. Debugging build errors and weird runtime issues in a new kernel are hard, so it helps to be flexible and patient, and to have a good support system.
    - I spent so much time this semester staring at entire screens full of build errors, and at times it felt a little overwhelming. I became quite familiar with the [GCC Options Index](https://gcc.gnu.org/onlinedocs/gcc/Option-Index.html) and the [Link Options](https://gcc.gnu.org/onlinedocs/gcc/Link-Options.html) pages, and I learned a lot about object files and executables, which complemented my work early on in the semester with writing the ELF parsing crate. By staying flexible and patient, I was able to continue trying many different options, always maintaining hope that I would find something that worked.
    - Having people to talk to about my issues was also a huge help. Discussing compiling and linking the test program with Alex helped get me unstuck on that issue. And in working with Aaron on the stack setup on `exec`, I was able to realize the pretty major issues when running global constructors, that I'm sure had I not noticed then, I would had a seriously hard time figuring out later on whenever I tried to use something that relied on global constructors having being run first.
