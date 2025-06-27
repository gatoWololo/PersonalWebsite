+++
date = 2019-02-22
title = "FUSE From Scratch"
description = "Building a FUSE based filesystem from scratch"
+++

**This is a work in progress entry! If you arrived here! Turn back. It is not ready.**

## Why this blog?

  While FUSE is an amazing piece of technology, I found it too difficult to get started
  with. Information on FUSE is spread across the web in a combination of blogs,
  mailing list questions, and source code header documentation. [libfuse](https://github.com/libfuse/libfuse "Github Repo") contains "documentation" in the form of doxygen generated
  docs for their headers, which I consider just slightly better than reading the raw header
  files. To their credit, getting help is extremely easy! ([Their mailing list](https://lists.sourceforge.net/lists/listinfo/fuse-devel) is very welcoming and helped
  me when I got _really_ stuck in a flash!). Most blogs I came across were short and failed
  to provide details, or anything beyond implementing a single function. (Probably a better
  justification for this blog to exist) other blogs use the high-level FUSE API. While
  convenient the FUSE high-level API, is not adequate to implement some POSIX filesystem semantics that applications running on top of our filesystem will want.

  The FUSE github repository does provide many [examples implementations](https://github.com/libfuse/libfuse/tree/master/example) and use cases. However,
  these are poorly (if at all) documented or commented. It's
  hard to understand _how_ they work, and more importantly the _why_ in their design choices.
  Many low-level C details bog down reading their code. I do think they're easy
  to understand now (once I already know how FUSE works and implemented my own FUSE layer!
  Defeating the purpose of needing examples...) Here I attempt to present a
  comprehensive introduction to FUSE.

  I will be using the excellent rust-fuse library for Rust. For those interested in using
  libfuse, the design is almost identical and all information transfers over.
  **Prerequistes:** This guide assumes familiarity of C and Rust. As wells as knowledge of
  libc system calls for interacting with the filesystem (open, close, mkdir, stat, etc),
  and basic knowledge of Linux filesystems (inodes, file descriptors).

# Introduction
  Most filesystems are written as part of the kernel (e.g. kernel module), this is allows
  for efficient implementation of custom filesystems, as the filesystem runs as part of the
  kernel. However, we lose portability, ease of programming, and security (see kernel modules
  for more information). Most of us are not kernel developers, and writing an incorrect
  kernel module may lead to OS security holes, or entire system crashes.

  FUSE stands for filesystem in userspace. Fuse is made of two parts:
  - A kernel component built into the Linux kernel.
  - A userspace library `libfuse` for communicating with the kernel.

  Instead, FUSE allows us to write our custom filesystem as a
  friendly userspace process. Once you have a compiled your FUSE process,
  you select a directory as a mount
  point for your filesystem. Any program which performs operations on files under the
  mount point will trigger an event in the FUSE process daemon. This allows us to create
  custom filesystem implementations. The fuse kernel module component already comes bundled up with Linux by default. Since we're switching between process, kernel and fuse process, we incur several context switches and we see a non-trivial slowdown. See this Usenix paper [To FUSE or Not FUSE](https://www.usenix.org/system/files/conference/fast17/fast17-vangoor.pdf) for in depth performance breakdown of FUSE.

  The libfuse API comes in two flavors:
  1) A high-level API which allows you to work with paths directly. All requests will
  include the path the user wants to do an operation on.
  2) A low-level API which works on inodes.

  The high-level API is implemented in terms of the low-level API with some extra
  bookkeeping done to keep track of paths. While the high-level API seems more appealing,
  the low-level API is more flexible: Not all POSIX filesystem operations can be given in
  terms of paths. We will see an example of this later.

# Passthrough Filesystem
  We will be implementing a FUSE layer almost identical to the [passthrough_ll.c](https://github.com/libfuse/libfuse/blob/master/example/passthrough_ll.c) libfuse example.
  Passthrough works as an intermediary FUSE layer between a user program the underlying OS.
  That is, for each filesystem request from the program makes, we will merely defer the
  operation to the actual filesystem "underneath" us. A passthrough layer allows us to later
  wrap or intercept any filesystem events we're interested in, while leaving all other events
  for the OS to handle.

  Roughly, when a user program wants to open a file `open("myFile.txt")` we will
  perform an equivalent `open("myFile.txt")` from our FUSE process, creating the illusion
  that the user program is doing the operations itself.

  Our passthrough program will take two arguments:
  ```bash
  USAGE:
      ./passthrough -t <file_tree> -m <mount_point>

OPTIONS:
    -t <file_tree> Real directory to do read and write operations through the real filesystem.
    -m <mount_point> Place to mount fuse filesystem.
  ```

  `mount_point` is the mounted directory where all operations will go to our FUSE process,
  `file_tree` is the "real" directory where we do all filesystem operations to. We require
  a separate `file_tree/` directory: if you tried using the same directory both as a mount
  point and for your FUSE process to do operations on, you end up recursing on your
  requests and the process hangs forever. So we do all operations to a separate directory. (This is
  different from the libfuse `passthrough_ll.c` example, which mirrors the entire filesystem
  tree stating at `/` to the mount point).

# Rust Fuse Library
  We will use the high-level, idiomatic library for Rust [rust-fuse](https://github.com/zargony/rust-fuse). This crate isn't just FUSE bindings, instead it is a from-scratch
  rewrite of the FUSE userspace protocol necessary to communicate with the kernel.

  rust-fuse provides a trait [FileSystem](https://docs.rs/fuse/0.3.1/fuse/trait.Filesystem.html), our job is to implement these trait's methods. The methods directly mirror
  those of the low-level API provided by libfuse. Consider:
  ```rust
  fn readlink(&mut self, req: &Request, ino: u64, reply: ReplyData);
  ```
  With an given an inode represents a symbolic link, `readlink` expects us to return the path   our symbolic link points to. We will see how to know what file this inode refers to later. The arguments are as followed:
  - self: This is a mutable reference to our user-defined type that implements the FileSystem trait. We can keep "global" filesystem state here as fields on our type.
  - req: A request provides information such as uid, gid, pid or requesting process, and
    unique request id (number).
  - ino: Inode number given to us by the kernel. Inodes are the way
    the filesystem uniquely [^1] identify files across the system.
  - reply: Object which we can use to reply to a request. We will reply with
    either `reply.data(&[u8])` where the argument is the path as a byte array, or `reply.error(errno)` which contains the error number code (errno) of why the request failed.

 FUSE's low-level API allow for asynchronous requests, allowing a FUSE process to be threaded
 for servicing multiple requests at once. We do not cover this feature in this guide.

 Great! You're all ready to implement a userspace filesystem now?! Ehh, almost. The
 devil is in the details. We will explain the high-level ideas, and implementation details
 in the upcoming sections.

# Passthrough.rs
  Here we discuss the full design of our filesystem.
## lookup
  The very first requests you will see when you start the FUSE daemon are lookup:
  ```rust
  fn lookup(&mut self, req: &Request, parent: u64, name: &OsStr, reply: ReplyEntry)
  ```
  `parent` is an inode to the directory where this request takes place, and `name` is
  the file[^2] inside the `parent` directory. The very first call to lookup will have `parent=1`.
  Through our starting command line arguments, we know what directory the user specified as our mountpoint so we can keep track of the fact that `inode 1 -> mount point directory`. We have a map `inode_data: HashMap<u64, FileRef>` which maps inodes to an open file descriptor for that inode. In the next section we will see why we use file descriptors instead of paths in our implementation.

  Lookup can fail most commonly if the file doesn't exist (as well as other reasons,
  e.g. insufficient permissions). The OS expects us to remember every file we looked up
  successfully. This is the only way to keep track of which inodes refer to which files
  (as we saw earlier for `readlink`). The OS will give us only an inode, and it's up to us to
  know/remember which path that inode belongs to.

  On lookup, we want to create a new file reference (Linux file descriptor) and add an entry
  for the key-value pair `(inode, file reference)` to our `inode_data` map.

  The first challenge is: what file does the `parent: u64` refer to? Here, `u64` should be
  ready as `inode`. Well this inode must be looked up in our `inode_data` map, but how do
  we know the entry will be there? Well, the entry _has_ to be there. Let's see the way
  FUSE works, given a small sample:

  `forget` does the exact opposite

      - Release
      - Flush
      - Mapping Inodes to Paths, What goes wrong?
      - Mapping inodes to file descriptors
      - Nix and libc
      - Permissions
  ## Details of implementation
  ## Look Up and Forget file descriptor references.

    [^1]: Actually, pairs of `(inode, device_number)` uniquely identify a file in a system. We do not worry about that here.

    [^2]: I use file here a generic term to mean: file, directory, fifo, device, etc. Since ?everything? in Linux is a file.
