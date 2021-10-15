+++
date = 2020-02-03
title = "Patching Cargo Dependencies"
description = "Using Cargo Patch"
+++

_Recommened Rust Skill_: **Intermediate** [^1]

### Summary
Here we cover Cargo's patch mechanism for temporarily changing and easily changing a dependency. We motivate it, show various examples, and point out edge-cases and footguns beyond what the Cargo book covers. I found the official documentation quite terse and took me a long time to handle some cases not mentioned in the documentation.

### Motivation
Cargo does a great job at managing Rust dependencies and most of the time _it just works_. There will come a time when you might need to modify the code for a dependency. The Cargo Book's section on [Overriding Dependencies](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html) mentions several use cases. For example, your application relies on the `nix` crate but there is a feature you need that has not been added to the lastest cratio.io version[^2]. `nix`'s code is hosted on github and the lastest commit on master includes the bug fix, how can you use this github's version until the changes make it to the next version on crates.io?

### Cargo Patch
Cargo has a mechanism to handle these type of cases, Cargo patch. Patching subsumes the older mechanism [Cargo replace](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html#the-replace-section)[^3]. You can find [the official documentation](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html) on Cargo patch on the official Rust Book. Namely, Cargo patch allows you to temporarily[^4] use an alternative version of a crate dependency (we will call this _patched dependency_). This will be usually be a crates.io dependency specified in your `Cargo.toml` (the manifest):
```toml
# Your Cargo.toml
# ...

[dependencies]
nix = "0.17"
libc = "0.2"
# ...
```
These dependencies are automatically downloaded from crates.io when your project is built. We will also look at examples when not using crates.io. The patched dependency can be a local version or a github version (others are supported as well).

### Patching Dependency With a Local Version
For the simple case, patching a dependency is as simple as adding a `[patch]` section in your `Cargo.toml` file. Here we will be patching the great `Nix` crate:
```toml
# Your Cargo.toml
# ...

[dependencies]
nix = "0.17"
libc = "0.2"

[patch.crates-io]
nix = { path = "/path/to/local/version/nix" }
```
Even in the simplest case there are a few things to unpack. I had never seen the `.crate-io` or `{ path = .. }` syntax before. I just accept the syntax as-is. This syntax comes from the TOML file format. See here for a quick [overview](https://learnxinyminutes.com/docs/toml/). Depending on your specific need, your local version of nix must match the version
on the manifest or be higher.

Notice it is not enough to simply `git clone` the Nix source code from [github.com/nix-rust/nix](https://github.com/nix-rust/nix) as this will pull the latest version on master. Some projects provide releases or tag commits corresponding to crate-io versions, but in general there is no easy way to find which commit belongs to the specific version. Event the [cargo clone](https://github.com/JanLikar/cargo-clone/) subcommand [seems to have problems](https://github.com/JanLikar/cargo-clone/issues/17) if you want an exact version. Cargo fetches the source code of all dependencies when building a package, so my current solution is to copy this downloaded version of the crate. My IDE tells me the source code for Nix is stored in `~/.cargo/registry/src/github.com-1ecc6299db9ec823/nix-0.17.0/`[^5]. I am then free to modify this copy and `patch` to its location.

Patching with local versions is not very portable. If you push `my-crate` to GitHub so others may also work off your patch version, they must use exact path for this to work. A better approach is to use TODO
#### Cargo Update

#### 🦀 Sharp Edge: Picking the Correct Package Version 
A dependency entry in the `Cargo.toml` file (e.g. `nix = "0.17"`) is generally not enough to tell what version of a package cargo is using as Cargo has some choice when picking version. See [Dependency Resolution](https://doc.rust-lang.org/cargo/reference/resolver.html) for more information. Instead, consult the `Cargo.lock` file see the exact version the resolver picked for you code:
```toml
# Cargo.lock file
...
[[package]]
name = "nix"
version = "0.17.0"
...
```

### Patching Dependency With a Github Version
  We can instead push modified dependency to github and have `cargo patch` use this version. On `cargo build`, cargo will automatically clone the source code from the master branch of my GitHub repository:
```toml
# Your Cargo.toml
# ...

[dependencies]
nix = "0.17"
libc = "0.2"

[patch.crates-io]
nix = { git = "https://github.com/gatowololo/nix" }
```
This tells cargo to use my version on the nix library, living under my GitHub username: `gatowololo/nix`. This presents its own set of problems, if we update `gatowololo/nix`'s master branch what will happen next time someone does `cargo build`? This is once again handled by the `Cargo.lock` file. The first time you `cargo build` with a patch manifest section cargo will write relevant information to the projects `Cargo.lock` file:
```toml
[[patch.unused]]
name = "nix"
version = "0.17.0"
source = "git+https://github.com/gatoWololo/nix#3b8180c430fe838e4fd71b83e5f92db6386e5c57"
```
Here we can see Cargo specifies the commit hash `3b8180c` which was used for patching as well as the URL for my version of Nix. If you are ever unsure if Cargo is using your patch, you can check here.

#### Using Different Branch 
By default `Cargo` pulls your code from the default branch, you can specify what branch to use as well:
```toml
[patch.crates-io]
nix = { git = "https://github.com/gatowololo/nix", branch="my-dev" }
```

#### Specifying Commit Hash
You can also specify the commit to use:
```toml
[patch.crates-io]
nix = { git = "https://github.com/gatowololo/nix", rev="3b8180c" }
```

#### 🦀 Sharp Edge: `Cargo.lock` and Version Control
If your `my-crate` project [does not check `Cargo.lock` into version control](https://doc.rust-lang.org/cargo/faq.html#why-do-binaries-have-cargolock-in-version-control-but-not-libraries), `cargo build` will use the latest version on `master`. This irreproducible behavior is probably **not** what you want!

### Patching GitHub Dependencies
Not all cargo crates live on crates.io. What happens when a dependency is itself specified as a git dependency? Like:
```toml
# Cargo.toml
# ...
[dependencies]
servo-media = { git = "https://github.com/servo/media" }
```
We can patch such a dependency like so:
```toml
# Cargo.toml
# ...
[patch."https://github.com/servo/media"]
servo-media = {git = "https://github.com/gatoWololo/servo-media"}
```
We specify the URL in as `[patch."URL"]`. This patch replaces `servo/media` with my own version `gatowololo/servo-media` on Github.

#### 🦀 Sharp Edge: Patching with Different Branch
What happens if you want to patch a git dependency using the same repository but a different branch? Like so:
```toml
# Cargo.toml
# ...
[dependencies]
rr_channel = { git = "https://github.com/gatowololo/rr_channel"}
# ...
[patch."https://github.com/gatowololo/rr_channel"]
rr_channel = { git = "https://github.com/gatoWololo/rr_channel", branch = "servo_development"}
```
Here we attempt to test a change on our `servo_development` branch in the same GitHub repository. When running `cargo build` we get the following error however:
```
error: failed to resolve patches for `https://github.com/gatowololo/rr_channel`

Caused by:
  patch for `rr_channel` in `https://github.com/gatowololo/rr_channel` points to the same source, but patches must point to different sources
```
This is a known [issue](https://github.com/rust-lang/cargo/issues/5478). The current workaround is to add extra '/' to the URL of the patch so Cargo thinks it is the patch points to a different source but resolves to the same path:
```toml
# Cargo.toml
# ...
[patch."https://github.com/gatowololo/rr_channel"]
# Notice extra '/' on URL.
rr_channel = { git = "https:///github.com//gatoWololo/rr_channel", branch = "servo_development"}
```
This works but creates a new foot gun, see [Sharp Edge: Git Dependency with Extra `/`](@/blog/cargo_patch.md#crab-sharp-edge-git-dependency-with-extra) below.
#### 🦀 Sharp Edge: Virtual Workspaces
Cargo enables crates to be broken down into multiple crates using [Cargo Workspaces](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html). Patching crates with workspaces is slightly different.

Patching with the target being a git source is straightforward enough. Notice servo-media-gstreamer and servo-media-dummy are (sub?? TODO ) crates in servo media:
```toml
[patch."https://github.com/servo/media"]
servo-media = {git = "https://github.com/gatoWololo/servo-media"}
servo-media-gstreamer = { git = "https://github.com/gatoWololo/servo-media"}
servo-media-dummy = { git = "https://github.com/gatoWololo/servo-media"}
```
The behavior for patching `servo-media` with a local copy is different. When patching subcrates you must specify the path not to the room of the package, but to the subcrate:
```toml
[patch."https://github.com/servo/media"]
servo-media = {path = "../media/servo-media"}
servo-media-gstreamer = {path = "../media/backends/gstreamer"}
servo-media-dummy = { path = "../media/backends/dummy"}
```

#### 🦀 Sharp Edge: Git Dependency with Extra `/`
Using the extra `/` trick as shown above (TODO link) is helpful but can lead to a different error.

#### Footnotes
[^1] One common feedback about the Rust ecosystem is the lack of intermediate and advanced documentation and blogs. I have several half-finished blogs because they were ambitious, so this blog will assume familiarity with `Cargo`, the `Cargo.toml` file, and roughly how Rust deals with dependencies. In the future I may come back and do a `Cargo.toml` and `Cargo.lock` in depth blog.

[^2] [crates.io](https://crates.io/) is Rust's offical crate repository, and all crate dependencies are all downloaded from here by default on `cargo build`.

[^3] While Cargo replace is not technically replicated. The [docs](https://doc.rust-lang.org/edition-guide/rust-2018/cargo-and-crates-io/replacing-dependencies-with-patch.html) mention ""while we don't intend to deprecate or remove [replace], you should prefer [patch] in all circumstances". So you should always use Cargo patch.

[^4] There is nothing stopping you from leaving the `[patch]` permanently, but it does not seem to be good practice.

[^5] The structure of the `~/.cargo/` directory is unclear to me. So I omit atempting to explain this path.