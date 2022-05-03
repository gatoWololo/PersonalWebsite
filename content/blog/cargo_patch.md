+++
date = 2022-05-03
title = "Patching Cargo Dependencies"
description = "Using Cargo Patch"
+++

_Recommened Rust Skill_: **Intermediate** [^1]

### Summary
This blog covers Cargo's patch mechanism. Cargo patch allows us to temporarily change the version of a crate our project depends on. This blog motivates, show various examples, and point out edge-cases and footguns beyond what the Cargo book covers. I found the official documentation quite terse and it took me a long time to handle some cases not mentioned there.

### Motivation
Cargo does a great job at managing Rust dependencies. Most of the time _it just works_. However, there will come a time when you need to modify the version of one of your dependencies. The Cargo Book's section on [Overriding Dependencies](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html) mentions several use cases. For example, your application relies on the `nix` crate and there is a bug that has only been fixed on the latest Github version. `nix`'s code is hosted on Github and the lastest commit on `master` includes the bug fix. How can you use this Github version until the changes make it to the next version on crates.io[^2]?

### Cargo Patch
Cargo has a mechanism to handle these type of cases: Cargo patch. Patching subsumes the older mechanism [Cargo replace](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html#the-replace-section)[^3]. You can find [the official documentation](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html) on Cargo patch on the official Rust book. Cargo patch allows you to temporarily[^4] use an alternative version of a crate dependency (we will call this the _patched dependency_). This will be usually be a crates.io dependency specified in your `Cargo.toml` (the manifest file):
```toml
# Your Cargo.toml
# ...

[dependencies]
nix = "0.17"
libc = "0.2"
# ...
```
Dependencies are automatically downloaded from crates.io and compiled when your program is compiled. We will also look at examples when not using crates.io. The patched dependency is most commonly a local or Github version, but other locations are supported as well.

### Patching a Dependency With a Local Version
For the simplest case, patching a dependency is as easy as adding a `[patch]` section in your `Cargo.toml` file. Here we will be patching the great `Nix` crate:
```toml
# Your Cargo.toml
# ...

[dependencies]
nix = "0.17"
libc = "0.2"

[patch.crates-io]
nix = { path = "/path/to/local/version/nix" }
```
Even in this simplest case there are a few things to unpack. I had never seen the `.crate-io` or `{ path = .. }` syntax before. I just accepted the syntax as-is. This syntax comes from the TOML file format. See here for a quick [overview](https://learnxinyminutes.com/docs/toml/).

Depending on your specific needs, your local version of nix must be greater than or equal to the version specified in the `[dependencies]` section. Notice it is not enough to simply `git clone` the Nix source code from [github.com/nix-rust/nix](https://github.com/nix-rust/nix) as this will pull the latest version on master. Some projects provide releases or tag commits corresponding to crates.io versions, but in general there is no simple way to find which commit belongs to a specific version. ~~Even the [cargo clone](https://github.com/JanLikar/cargo-clone/) subcommand [seems to have problems](https://github.com/JanLikar/cargo-clone/issues/17) if you want an exact version~~ (Update: This seem to have been fixed since). Cargo fetches the source code of all dependencies when building a package, so my current solution is to copy this downloaded version of the crate. My IDE tells me the source code for Nix is stored in `~/.cargo/registry/src/github.com-1ecc6299db9ec823/nix-0.17.0/`[^5]. I am then free to modify this copy and `patch` by specifying its location on my filesystem.

Patching with a local version is not very portable. If your project is hosted on GitHub, anyone wanting to build your code will have to have a copy of the patched dependency located in the exact same filesytem path. A better approach is to patch with a [dependency located on Github](/blog/cargo-patch#patching-a-dependency-with-a-github-version).

#### Cargo Update

#### 🦀 Sharp Edge: Picking the Correct Package Version 
A dependency entry in the `Cargo.toml` file (e.g. `nix = "0.17"`) is generally not enough to tell what version of a crate Cargo is using. Cargo has some choice when picking the version. See [Dependency Resolution](https://doc.rust-lang.org/cargo/reference/resolver.html) for more information. Instead, consult the `Cargo.lock` file to see the exact version the resolver picked for you code:
```toml
# Cargo.lock file
...
[[package]]
name = "nix"
version = "0.17.0"
...
```

### Patching a Dependency With a Github Version
We can instead push a modified dependency to Github and have `Cargo patch` use this version. On `cargo build`, cargo will automatically clone the source code from the master branch of our GitHub repository:
```toml
# Your Cargo.toml
# ...

[dependencies]
nix = "0.17"
libc = "0.2"

[patch.crates-io]
nix = { git = "https://github.com/gatowololo/nix" }
```

This tells cargo to use the version of nix under my GitHub username. This presents its own set of problems. If we commit to `gatowololo/nix`, what will happen next time someone does `cargo build`? This is once again handled by the `Cargo.lock` file. The first time you `cargo build` with a patch manifest section cargo will write relevant information to the projects `Cargo.lock` file:
```toml
[[patch.unused]]
name = "nix"
version = "0.17.0"
source = "git+https://github.com/gatoWololo/nix#3b8180c430fe838e4fd71b83e5f92db6386e5c57"
```
Here we can see Cargo specifies the commit hash `3b8180c` which was used for patching as well as the URL for my version of Nix. If you are ever unsure if Cargo is using your patch, you can check here.

#### Using Different Branches
By default `Cargo` pulls our code from the default branch. We can also specify the branch to use:
```toml
[patch.crates-io]
nix = { git = "https://github.com/gatowololo/nix", branch="my-dev" }
```

#### Specifying the Commit Hash
You can also specify a specific commit using:
```toml
[patch.crates-io]
nix = { git = "https://github.com/gatowololo/nix", rev="3b8180c" }
```

#### 🦀 Sharp Edge: `Cargo.lock` and Version Control
If your project [does not check in `Cargo.lock` into version control](https://doc.rust-lang.org/cargo/faq.html#why-do-binaries-have-cargolock-in-version-control-but-not-libraries), `cargo build` will use the latest version on `master`. This irreproducible behavior is probably **not** what you want!

### Patching a GitHub Dependencies
Not all cargo crates live on crates.io. What happens when a dependency is itself specified as a Github dependency? Like:
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

We must specify the URL in the `[patch."URL"]` header. This patch replaces `servo/media` with my own version `gatowololo/servo-media` on Github.

#### 🦀 Sharp Edge: Patching with a Different Branch
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

Within the same GitHub repository, we want to test a change located on the `servo_development` branch. However, we get the following error when running `cargo build`:
```
error: failed to resolve patches for `https://github.com/gatowololo/rr_channel`

Caused by:
  patch for `rr_channel` in `https://github.com/gatowololo/rr_channel` points to the same source, but patches must point to different sources
```
This is a known [issue](https://github.com/rust-lang/cargo/issues/5478). The current workaround is to add an extra '/' to the URL of the patch so Cargo thinks the patch points to a different source, but resolves to the same path:
```toml
# Cargo.toml
# ...
[patch."https://github.com/gatowololo/rr_channel"]
# Notice extra '/' on URL.
rr_channel = { git = "https:///github.com//gatoWololo/rr_channel", branch = "servo_development"}
```
This works. but creates a new foot gun... See [Sharp Edge: Git Dependency with Extra `/`](/blog/cargo-patch#crab-sharp-edge-git-dependency-with-extra) below.

#### 🦀 Sharp Edge: Virtual Workspaces
Cargo enables crates to be broken down into multiple crates using [Cargo Workspaces](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html). Patching crates with workspaces is slightly different.

In this example both `servo-media-gstreamer` and `servo-media-dummy` are members of the `media` workspace. Patching when the patched dependency is on Github is straightforward enough:
```toml
[patch."https://github.com/servo/media"]
servo-media = {git = "https://github.com/gatoWololo/servo-media"}
servo-media-gstreamer = { git = "https://github.com/gatoWololo/servo-media"}
servo-media-dummy = { git = "https://github.com/gatoWololo/servo-media"}
```

However, the behavior for patching with a local copy is different! When patching a sub-package (e.g. `servo-media-gstreamer`), you cannot specify "path" as the root of the workspace (i.e `path/to/media/`). Instead, we must specify the path all the way down to the location of the sub-package:
```toml
[patch."https://github.com/servo/media"]
servo-media = {path = "../media/servo-media"}
servo-media-gstreamer = {path = "../media/backends/gstreamer"}
servo-media-dummy = { path = "../media/backends/dummy"}
```
Notice the difference in path for `servo-media-gstreamer` between the Github example vs our local example: `servo-media-gstreamer = { git = "https://github.com/gatoWololo/servo-media"}` vs `servo-media-gstreamer = {path = "../media/backends/gstreamer"}`. The local version requires the additional `backends/gstreamer` part of the path.


#### 🦀 Sharp Edge: Git Dependency with Extra `/`
Using the extra `/` trick shown [above](/blog/cargo-patch#crab-sharp-edge-patching-with-a-different-branch) is useful, but can lead to other sorts of errors. If your code is all under one workspace, consisting of multiple packages, each package has its own manifest file:
```toml
# our_package/.../crate_a/Cargo.toml
# ...
[dependencies]
rr_channel = { git = "https://github.com/gatoWololo/rr_channel"}
```

```toml
# our_package/.../crate_b/Cargo.toml
# ...
[dependencies]
rr_channel = { git = "https:///github.com//gatoWololo/rr_channel"}
```
Attempting to patch the workspace's manifest file caused an error because two incompatible versions of `rr_channel` were being used. If you look carefully, the Github URL specified for `our_package/.../crate_b/Cargo.toml` accidentally has an extra `/`! So `rr_channel` was only getting patched in `crate_a`! Using the `cargo tree` command proved invaluable for figuring out this was happening.

### Conclusion
Patching is a powerful, versatile, and useful mechanism, but beware of the many sharp edges. I believe Cargo is extremely well designed, but dependency management is a complicated problem and Cargo must support many edge cases. This leads to complicated and sometimes surprising results as the user.

#### Acknowledgements
Thanks to Kelly Shiptoski for revising this blog!

#### Footnotes
[^1]: One common feedback about the Rust ecosystem is the lack of intermediate and advanced documentation and blogs. I have several half-finished blogs because they were ambitious, so this blog will assume familiarity with `Cargo`, the `Cargo.toml` file, and roughly how Rust deals with dependencies. In the future I may come back and write an in-depth `Cargo.toml` and `Cargo.lock` blog.

[^2]: [crates.io](https://crates.io/) is Rust's offical crate repository, and all crate dependencies are all downloaded from here by default on `cargo build`.

[^3]: While Cargo replace is not technically depricated, the [docs](https://doc.rust-lang.org/edition-guide/rust-2018/cargo-and-crates-io/replacing-dependencies-with-patch.html) mention "while we don't intend to deprecate or remove [replace], you should prefer [patch] in all circumstances". So you should always use Cargo patch.

[^4]: There is nothing stopping you from leaving the `[patch]` permanently, but it does not seem to be good practice.

[^5]: The structure of the `~/.cargo/` directory is unclear to me. So I omit atempting to explain this path.