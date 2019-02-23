+++
date = 2018-11-14
title = "Clang++ missing C++ header?"
description = "How I figured out where the missing headers were."
+++

(Hopefully someone will find this useful, since I was not able to find any other clear
resource online).

### Problem

In our Linux Ubuntu 18.04 server, I kept getting this error when trying to compile our
C++ project with clang++:

```bash
dettraceSystemCall.cpp:17:10: fatal error: 'limits' file not found
#include <limits>
        ^~~~~~~~
```

Looking online I was not able to find much help. Googling "clang missing c++ headers"
returned the only relevant search [clang doesn't see basic headers](https://stackoverflow.com/questions/26333823/clang-doesnt-see-basic-headers).

None of these answers were helpful to me... Compiling with g++ works, so I do have the correct C++ headers installed.


### Solution
Taking the failing command from `make` and running it manually, I could reproduce the issue:
`clang++ -Wall -c -o dettraceSystemCall.o dettraceSystemCall.cpp`

I added the `-E` (call only the preprocessor, for those that don't know, the C and C++
preprocessor is responsible for expanding macros and finding headers) and `-v` (verbose)
flags.

This prints out a lot of input, but shorted to the relevant parts:

```bash
clang version 6.0.0-1ubuntu2 (tags/RELEASE_600/final)
Target: x86_64-pc-linux-gnu
Thread model: posix
InstalledDir: /usr/bin
Found candidate GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/7
Found candidate GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/7.3.0
Found candidate GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/8
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7.3.0
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/8
Selected GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/8
...
#include "..." search starts here:
#include <...> search starts here:
../include
/usr/bin/../lib/gcc/x86_64-linux-gnu/8/../../../../include/c++
/usr/include/clang/6.0.0/include
/usr/local/include
/usr/include/x86_64-linux-gnu
/usr/include
End of search list.
...
```

So, the server has several gcc installations: 7, 7.3, and 8. Out of those clang++
is picking gcc version 8: `Selected GCC installation: /usr/bin/../lib/gcc/x86_64-linux-gnu/8`.

Since `g++` works I know the C++ headers have to be somewhere? Where are they located?
```bash
$> find /usr -name "iostream"
/usr/include/c++/7/iostream
```

Interesting... Only g++-7 seems to have the C++ header files, but it's picking gcc-8 as the installation to use!
Somehow, the server has g++-8 installed but not it's header files? I used `sudo apt install g++-8` to install all g++-8 headers.

Now:
```
$> find /usr -name "iostream"
/usr/include/c++/8/iostream
/usr/include/c++/7/iostream
```

And clang++ works :)

## Background
It seems the cause of the issue is that clang++ does not come with it's own headers or
runtime for C or C++. Instead it relies on other projects to providing these (usually gcc/g++
for most Linux systems, or an alternative implementation like [libc++](https://libcxx.llvm.org/)).
So you will often see online answers point to installing libc++ to solve this issue.

(Thanks to [this blog](https://cxwangyi.wordpress.com/2012/01/20/using-clang-cannot-find-standard-header/) for pointing me in the right direction!)
