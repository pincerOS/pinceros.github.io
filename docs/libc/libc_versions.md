---
title: Picking an Implementation
nav_order: 1
parent: libc
---

## Overview

There are many different implementations of libc out there; OSDev provides a [fairly comprehensive list](https://wiki.osdev.org/C_Library) of them. The most well known of these is probably the [GNU C Library (glibc)](https://www.gnu.org/software/libc/) by the GNU Project. This comes standard on pretty much every GNU/Linux installation, and is generally regarded as being the most complete/featureful implementation of libc. A common alternative to glibc is [musl](https://musl.libc.org/), which is often used by those who prioritize having less bloat than glibc, while still maintaining a large enough feature set to work with most applications. Others include [Newlib](https://sourceware.org/newlib/), which is a much smaller implementation of libc, and [mlibc](https://github.com/managarm/mlibc), which is a newer implementation that aims to be flexible and easy to port, among many, many more options.

## Weighing my Options

Here was my thought process behind selecting between the 4 options from above:

1. **glibc**: This is the most feature-rich implementation, but it is also the largest and most complex. It is also not very portable, as it relies on many GNU-specific features and basically assumes you are running the Linux kernel. Plus, it is GPL licensed, while the rest of our project is MIT licensed. So glibc was the first option I ruled out.
2. **musl**: While smaller than glibc, musl is still quite large and complex. Because it strives for high compatibility with glibc, it makes a lot of similar assumptions about the environment it is running in, although I have heard it is a bit more portable than glibc. The MIT license was a step in the right direction, but ultimately I felt it would be too much to try to port to PincerOS.
3. **Newlib**: Newlib is a much smaller implementation of libc, with a very free license, and generally pretty portable. However, I know from people's experience trying to port Newlib as a part of 439H (sophomore-year Honors Operating Systems) final projects that its build system is outdated and difficult to work with. I decided not to go with Newlib, but I did keep it as a backup option just in case.
4. **mlibc**: mlibc is a newer implementation of libc that is designed to be portable and easy to work with. It provides several feature flags, so I would be able to disable the features I don't need (with the option to expand the scope if time permits), and it is MIT licensed. As a massive bonus, it is developed by the [Managarm](https://github.com/managarm/managarm) team, whose primary focus is the Managarm microkernel. Since we were aiming for a microkernel design, it seemed like a perfect fit. So I decided to go with mlibc.

## Deciding on a Version

Now that I had settled on mlibc, I needed to decide which version to use. One option would have been to fork directly off of master. But upon reading the [mlibc README](https://github.com/managarm/mlibc/blob/1183d85f914f3c7cea56961edbf472b49b22d92b/README.md), I saw that it was not recommended to use the master branch, as it is a work in progress and not guaranteed to be stable. Instead, I decided to use the latest release version, [release-5.0](https://github.com/managarm/mlibc/tree/release-5.0)
