---
title: Ideas and Resources for OS projects
nav_order: 1
parent: Alex's Random Docs
---


# Ideas and Resources for OS projects

This is far from an exhaustive list, but here are a bunch of ideas for
things to work on for Operating Systems projects, and some guidance
on how to approach them.  This is from a couple of semesters of experience
TAing Operating Systems, as well as the current project in the Multicore
Operating Systems class, where I've had a chance to work on a number of
these things in more depth.

## Core parts of kernel

- Scheduling
    - More efficient scheduling, prioritizing tasks
    - Linux's "Completely fair scheduler"
    - Keeping tasks on the same cores, for better cache usage
    - Minimizing contention -- per-core work queues and work-stealing?

- Inter-process communication
    - Unix domain sockets or something similar for local communication
    - The ability to send file descriptors is very useful, especially
      for creating shared memory between processes (see the section
      below on the display server).

- Inter-process shared memory
    - Shared memory between independent processes (ie. unrelated processes,
      not just preserving a shared mapping across fork)
    - Useful for transferring large amounts of data between processes,
      such as framebuffers.

- Signals
    - The first major use-case for signals that you'll find is when using
      a shell -- if you accidentally start a program that infinitely loops,
      you have no way of killing it to get back to your shell.
      On most systems, Ctrl-C is intercepted by the terminal driver (usually
      in the kernel), where depending on the mode of the terminal, it
      will send a signal to kill the current "foreground process group" --
      whatever the terminal things is currently the thing running in the
      foreground.  How you specify this is up to you, but you'll need
      some way for the shell to tell the kernel what's it should kill
      on Ctrl-C, and it should work with pipes (`cat | grep ...`), so
      it needs to handle multiple processes.

- Logging/Tracing
    - Having a proper logging system makes debugging much easier, and
      also makes your OS look more professional.
    - Allow for filtering the logging output, either by component
      (ie. different drivers) or importance (levels of info like trace,
      debug, info, warn, etc.)
    - Prints are slow, but logging to a ringbuffer can be fast.  Can you
      offload printing to another task, to improve performance of the rest
      of the system?
    - If you have a single shared data structure for logs, that can also
      end up being slow from contention and locking.  What if you had
      multiple logs, one for each core, and coallesce them into a single
      log when printing them?
    - (If you want, you can also add colors using [ANSI escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code).)
    - On the tracing side, if you have a fast logging system, you can also
      generate traces of what your kernel runs on each core, as well
      as when it enters/leaves functions;
    - [Chrome's trace event format](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/preview?tab=t.0#heading=h.yr4qxyxotyw) is a relatively simple and well supported format for traces,
    and there are a number of viewers for that format, like [Perfetto](https://ui.perfetto.dev/).
    - There are some even nicer systems like [Tracy](https://github.com/wolfpld/tracy) that also support Chrome's trace format, but Tracy is a pain to build.
    - Another thing to try is to generate [flame graphs](https://www.brendangregg.com/flamegraphs.html) of how time is spent in your kernel,
    so you know what's slowing it down and what to optimize.

- Backtraces
    - GDB will generally do this for you if you're running in QEMU, but
      it might be nice to have a better kernel panic format -- that prints
      out the state of registers, as well as the stack trace of functions
      that ended up calling the one that panicked.

## Drivers

- Audio
    - I don't know much about audio, but here are some resources for audio
      on the Raspberry Pi:
    - [An example using the audio jack on the rpi4b](https://www.rpi4os.com/part9-sound/)
    - [An example using audio over HDMI on rpi4b](https://github.com/CodexTextor/rpi-hdmi/tree/rpi4)

- Graphics (drivers)
    - Proper GPU drivers (supporting graphics APIs like OpenGL or Vulkan)
      are theoretically possible to port, but they're incredibly large
      code-bases and aren't feasible for any small-scale project.
    - (It's possible to get a subset OpenGL running slowly in software
      by using tools like llvmpipe or tinygl, if you really want it.)
    - However, getting an RGB framebuffer is relatively easy on most
      modern systems; on x86 (or at least QEMU's x86 emulator), there
      are VGA modes that expose framebuffers, and on the Raspberry Pi
      (at least on 3b/4b) the GPU can set up a framebuffer using simple
      commands through [a memory mapped "mailbox"](https://jsandler18.github.io/extra/prop-channel.html).
    - Once you have a framebuffer, exposing it to user code is relatively
      simple -- the cleanest way is to expose it as a `mmap`able file in
      a virtual filesystem, but you could also have syscalls to make the
      kernel copy user buffers into the proper framebuffer.

## Usermode

- "Server" processes
    - Operating systems typically have a number of local server processes
      that other user processes connect to interact with the rest of the
      system.
    - Applications connect to the windowing / display server to create
      windows, access the screen, and receive keyboard input.
    - Applications connect to the audio server to be able to play audio,
      (so the audio server can mix the incoming streams from each application
      into one output signal).
    - There needs to be some way for applications to discover and connect
      to these servers, without having to be a direct child of them; the
      most common mechanism to do this is with sockets, but you could
      have a custom system and it could work just as well.

- Display server
    - The goal of a display server is to allow multiple applications to
      access the display without conflicting with each other.  In the simplest
      case, this can be done by giving one application access to the full
      screen and ignoring the output of the others, but if you want to
      switch applications, you'll need some way of multiplexing the
      input (keyboard/mouse) and output (video) of multiple applications.
    - Instead of directly accessing the framebuffer for the display,
      graphical applications interact with a "display server" to get
      access to individual framebuffers; this allows the display server
      to act as an intermediary, and control which applications' framebuffers
      are copied to the display's framebuffer and drawn to the screen.
    - This needs some way of sharing large amounts of memory between
      "clients" (applications) and the display server, so the approach I
      took was to create a shared mapping of an anonymous file (a file
      not attached to the filesystem, created with a syscall like
      `memfd_create`).
    - If you have sockets or some other way of sending
      file descriptors between processes, either the client or server
      can create a buffer to store the framebuffer data, and then send
      the file descriptor to the other; both ends then mmap the buffer
      and have access to the same shared memory.
    - The other aspect of a display server (or sometimes window manager)
      is multiplexing input -- controlling which applications get keyboard
      or mouse input.  This could be done through sockets, or it could be
      done by adding a ringbuffer of events to the shared memory region used for
      framebuffer data.
    - There are many events that display clients need to deal with:
        - Indicating to the server when a frame is ready to present (and no longer being modified)
        - Indicating to the client when the server has finished reading from a frame
        - Client -> server that the client is exiting (to free resources, clean up state, close windows)
        - Server -> client for input events (mouse, keyboard)
        - Server -> client for requesting close (someone clicked the close button, or pressed Alt-f4, etc)
    - Depending on the implementation, the display server may also be the layer
      at which raw mouse inputs are turned into a virtual cursor location within
      the screen; this could also be done by the window manager, as the border
      between the two is flexible.
    - Some interesting (if low level) resources:
        - https://arcan-fe.com/2024/11/21/a-deeper-dive-into-the-shmif-ipc-system/

- Window Manager and Compositor
    - As an extension to the display server, a window manager allows for
      multiple applications to be displayed at the same time, by allocating
      regions of the screen to each application, then combining their images
      into a single image to write to the final framebuffer.
    - There are many ways of allocating screen space to applications, but the
      two general categories are "floating" window managers, where windows
      are individually movable and may overlap with some layering mechanism,
      and "tiling" window managers, where windows are arranged in some tiled
      pattern and do not overlap.
    - The process of merging the applications into a single framebuffer is
      called compositing; if windows can overlap, then you can handle the
      layering of windows by drawing them from back to front, so the "front"
      windows are drawn last and overwrite the image data from "background" windows.
    - The window manager also needs to track what applications are in focus,
      so that it can properly delegate input -- such as clicks on certain
      windows should only go to those windows, and not others.

- Text rendering (PCF fonts, etc.)
    - PSF - "PC Screen Font"; the [Wikipedia page gives enough information
      to implement it](https://en.wikipedia.org/wiki/PC_Screen_Font).
    - There's also the slightly newer format PCF ("Portable Compiled Format"),
      which is more complex but supports non-monospace fonts.
      [Official documentation](https://fontforge.org/docs/techref/pcf-format.html#pcf-format-metricsdata).
    - PCF has the minor advantage that you can theoretically convert other fonts
      to PCF, with the tools `otf2bdf` and `bdftopcf`.  (The second one has [a
      web version available](https://adafruit.github.io/web-bdftopcf/)).

- Image rendering: BMP is easy, QOI is straightforward and usably compressed
    - BMP images can store uncompressed raw image data, and supporting
      a basic subset is straightforward.  [Wikipedia has a good description
      of the file format](https://en.wikipedia.org/https://fontforge.org/docs/techref/pcf-format.html#pcf-format-metricsdatawiki/BMP_file_format), though pick one set of options and stick with it -- you likely want
      24 or 32 bits per pixel (RGB or RGBA) and want it to be uncompressed
      for simplicity.
    - QOI is a pretty good ("quite okay") image format with basic compression;
      the specification is [a single page](https://qoiformat.org/qoi-specification.pdf), and is straightforward to implement.
    - PPM is likely the simplest format to implement, though it's less
      supported and has no support for compression.  [Wikipedia has a
      description of the format](https://en.wikipedia.org/wiki/Netpbm#File_formats), but it's just an ASCII header with
      image data, either encoded in ASCII or packed bytes.
    - To convert files into these formats, you can use [ImageMagick's `convert`
      command](https://imagemagick.org/script/convert.php):
      ```
      convert image.png image.qoi
      ```
      (Though if you're trying this on the UT lab machines, it's unlikely
      the 6 year old version of ImageMagick they have supports QOI.)
    - Fun fact, you can use `convert` to convert a pdf into a series of
      images...  There are some interesting things you can do with this,
      but be aware you'll likely need to use QOI or another compressed
      format, since uncompressed images take a ton of space and are slow
      to read.
      ```sh
      convert -density 300 -scene 1 -alpha off +compress -background black -gravity center -resize 640x480 -extent 640x480 -scale 100% example.pdf example/img_%03d.qoi
      ```

- A nicer shell
    - It's easy to make a simple shell / REPL -- just a loop of reading from 
      serial input, then using fork/exec/wait to run commands.  However, a
      simple shell can be painful to use, as you'll soon find out when
      backspace doesn't work and you have to fully retype a command whenever
      you make a mistake...
      So here are some suggestions:
    - Make a line-editor, so that you can use backspace and arrow keys.  You'll
      need to look into [ANSI escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code) for the input, as well as for the ability to clear out
      old characters when you backspace.
    - Add support for basic line-editing shortcuts, like Ctrl-A (start of line),
      Ctrl-E (end of line), Ctrl-K (cut), and Ctrl-Y (paste)
    - Add support for history, so you can use the up/down arrows to navigate
      previous commands.  You can even save it to a file, if you have writing!
    - Allow searching the command history with Ctrl-R; try it in bash!
      (For inspiration, you should also try [the fish shell](https://fishshell.com/), which has a much more polished approach to history searching.)
    - Add tab completion for executables and file paths
    - Add a proper PATH and path searching (could be in the kernel)
    - Use ANSI escape codes for colors
    - Keep track of the current working directory, and display it. (This
      may be harder than it seems, mainly because of symbolic links; but
      also, what if you move the directory the shell is in?)

- Terminal programs
    - There are tons of different terminal applications that would be useful to have and are educational to implement.
    - Beyond the most basic commands (`ls`, `cd`, `cat`, `echo`), here are a few other suggestions of programs to implement:
        - A pager, like `less` or `more`, which show files a page at a time,
          or allow scrolling.
        - A basic editor, using terminal using [ANSI escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code); there are a number of guides online on how to write a terminal editor
    - There are a number of portable implementations of the core commandline
      utilities (commonly called `coreutils`); the most famous portable
      implementation is [BusyBox](https://en.wikipedia.org/wiki/BusyBox), which
      bundles the utilities into a single executable.  However, porting C
      programs can be challenging, so I would recommend setting up your own
      basic utilities first.

- Init system
    - Once you have a process running in usermode, you have to decide
      what that initial program should be.  In the simplest case, it could
      just be a shell, but what if you want to make the user log in first?
    - The general approach is to have an *init system* that does the usermode
      side of the system initialization -- setting up the system, starting
      any background processes that need to run, and then launching the login
      terminal or lock screen.
    - This could be hardcoded, but most systems are configurable, either
      via directories with shell scripts that are run on boot, or with
      configuration files.

- Login system
    - Setting up a login prompt seems easy, but there's a lot of complexity
      when you look in to it.
    - To make login work as a user process, you need to have some sort of
      user id tracked for each process, and some way of modifying that
      user id from usermode; on Unix-like systems the `setuid` family of
      syscalls (see `man setresuid`), which has a ton of semi-necessary
      complexity.
    - The simple explanation is that `login` is run as root (uid 0), and
      the kernel lets processes running as root set their uids to anything;
      after that point, the process can no longer change its uid.
      (`sudo` is similar, but it has a `setuid` permission bit on the
      executable, so the kernel runs it as root even if it was run
      by a non-root user.)
    - The login system also needs to support adding new users, so the
      list of users (and their passwords) needs to be stored on disk;
      for security, this should only be readable by root, and the passwords
      should be hashed so that they can't be determined from the stored values.
      ([Wikipedia has a page on Linux's format on disk: `/etc/passwd` and `/etc/shadow`](https://en.wikipedia.org/wiki/Passwd))

- Virtual filesystems
    - Virtual filesystems are an interesting tool, and aren't that hard
      to implement in the kernel.  They allow the kernel to expose non-file
      operations or information through normal filesystem APIs, so that tools
      like `ls`, `cat`, `grep`, etc. all work without changes.
    - The most well-known examples are:
        - `procfs` (see `man procfs`) - `/proc/{pid}/...` on Linux, exposes information about each running process on the system
        - `sysfs` (see `man 5 sysfs`) - `/sys/...` - exposes system information and configuration
        - `devfs` - `/dev/...` on Linux, includes things like `/dev/zero`, `/dev/null`, `/dev/random`, as well as files representing block devices (disk), serial ports, and most other devices.
    - If you're interested in more extensions of this, you should look into
      the [plan9 operating system from Bell Labs](https://en.wikipedia.org/wiki/Plan_9_from_Bell_Labs), which
      exposes almost everything through virtual filesystems and mount points,
      including network connections.

<!-- - Self-hosting - Can you get GCC running on your OS?

## Raspberry Pi basics

## A quick* guide to porting doom


- Porting newlib
- Porting doom
- What you need: an rgb framebuffer; an elf loader; sleep/get_time; keyboard input -->
