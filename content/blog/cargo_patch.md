+++
date = 2021-02-22
title = "Using Cargo [patch] to Override Dependencies"
description = "Using Cargo Patch"
+++

_Recommended Rust Skill_: **Intermediate** [^1]

### Summary
Here we cover Cargo's useful mechanism for temporarily and easily overriding dependencies: `[patch]`. I found the official documentation quite terse. Making it difficult to use in situations more complicated than simple examples. So here, we motivate and show many scenarios. Finally, we point out various edge-cases and footguns beyond what the Cargo book covers. 

### Motivation
Cargo does a great job at managing Rust dependencies and most of the time _it just works_. However, there will come a time when we might need to modify the code for a dependency. The Cargo Book's section on [Overriding Dependencies](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html) does states several use cases. For example, our application relies on the `nix` crate (a crate for idiomatic Rust bindings around libc) but the esoteric libc function we need does not exist as of the latest `crates.io` version[^2]. `nix`'s source code is hosted on Github and let's assume the latest commit on master includes bindings for the function we need. How can we use this alternative (`alt`) version for our project until the changes make it to the next version on crates.io?

### Cargo `[patch]`
`[patch]`[^3] allows us to temporarily[^4] specify an alternate (`alt`) version of a crate dependency. This will be usually be a `crates.io` dependency specified in our package's [manifest file](https://doc.rust-lang.org/cargo/reference/manifest.html), `Cargo.toml`. For example:
```toml
# Our Cargo.toml
# ...

[dependencies]
nix = "0.17"
libc = "0.2"
# ...
```
Cargo defaults to downloading dependencies from crates.io when a package is built (later we cover examples of patching non-crates.io dependencies).

Why have `[patch]` at all? Why not manually override the dependencies by changing the `nix = "0.17"` line directly? Patching is **transitive** and Cargo ensures dependencies are transitively patched if they depend on the patched dependency. Larger projects may be separated into separate crates using [workspaces](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html), each containing their own manifest file. `[patch]` allows us to change a **single location**. Namely, the top level manifest file. `[patch]` subsumes the older [`[replace]`](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html#the-replace-section) [^5] mechanism which should no longer be used.
### Patching with a Path to Local `alt` Dependency 
In the most common case, patching is as simple as adding a `[patch]` section to our `Cargo.toml`. Here we will be patching `nix`:
```toml
# Cargo.toml
# ...

[dependencies]
nix = "0.17"
libc = "0.2"

[patch.crates-io]
nix = { path = "/path/to/local/version/nix" }

```
There are a few things to unpack here. I had never seen the `.crate-io` or `{ path = .. }` syntax before. This syntax is specified in the TOML format. While not necessary, see here for a quick [overview](https://learnxinyminutes.com/docs/toml/) of TOML. We may also use a relative path like `nix = { path = "../alt/nix/" }` for patching. Our path should point to the top level directory containing the `Cargo.toml` file of `alt` dependency.

In general, using filesystem paths is not very portable. This will only work with our set up, which others must manually replicate if they want to use our patch. A better approach is to use `alt` versions hosted online, e.g. Github.

### Cargo Update
Above, we were purposely ambiguous about where the `alt` version came from. For the patch to take effect, our `alt` version of `nix` must match or be higher than the version on the manifest. Otherwise Cargo will helpfully tell us the patch will not be used. Using a real example:
```
Patch `mio v0.6.22 (https://github.com/servo/mio.git?branch=servo-mio-0.6.22#f640fd66)` was not used in the crate graph.
Check that the patched package version and available features are compatible
with the dependency requirements. If the patch has a different version from
what is locked in the Cargo.lock file, run `cargo update` to use the new
version. This may also occur with an optional dependency that is not enabled.

```
Unfortunately, an in depth look at the `Cargo.lock` file and the Cargo update command is beyond the scope of this blog. See [here](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) for the official documentation. A `cargo update` is not always necessary for the patch to work. When exactly it is needed remains unclear to me. Furthermore, it is not always possible `cargo update` depending on the semver constraints of our dependencies... but thankfully is not common in practice.

#### 🦀 Sharp Edge: Picking the Correct Package Version

It is often insufficient to `git clone` the `nix` source code from [github.com/nix-rust/nix](https://github.com/nix-rust/nix) as this will simply clone the latest version of the code. If this latest version is significantly newer (e.g. a new major version), it may be incompatible without major changes to our code.

Some projects provide releases or tagged commits corresponding to crates.io versions. However, **in general there is no easy way to find which commit belongs to a specific version**. Event the [cargo clone](https://github.com/JanLikar/cargo-clone/) subcommand [seems to have problems](https://github.com/JanLikar/cargo-clone/issues/17) if we want an exact version.

Cargo fetches the source code of all dependencies when building a package, so my current solution is to copy the locally downloaded version of the crate. My IDE tells me the source code for `nix` is stored in `~/.cargo/registry/src/github.com-1ecc6299db9ec823/nix-0.17.0/`[^6]. We can copy this directory to modify the code for our patching.

To exacerbate the difficulty, a dependency entry in the manifest (e.g. `nix = "0.17"`) is generally not enough to tell what version of a package Cargo is actually using. Cargo has some choice when picking version. See [Dependency Resolution](https://doc.rust-lang.org/cargo/reference/resolver.html) for more information. Instead, consult the `Cargo.lock` file see the exact version the resolver picked for our code:
```toml
# Cargo.lock file
# ...
[[package]]
name = "nix"
version = "0.17.0" # May be different from the version listed on the manifest.
# ...
```

### Patching Dependency With a Github `alt` Dependency
  We can push modified dependency to Github and have `cargo patch` use this version. This way, Cargo will automatically download our patch when building the project. This is convenient for others who may be using our patch. 
```toml
# Our Cargo.toml
# ...
[dependencies]
nix = "0.17"
libc = "0.2"

[patch.crates-io]
nix = { git = "https://github.com/gatowololo/nix" }
```
This tells cargo to use my personal version on the `nix`, which lives under my GitHub username `gatowololo/nix`. Cargo will automatically use the default branch.

This presents its own set of problems, if we commit to`gatowololo/nix`'s what will happen next time someone does `cargo build`? This is once again handled by the `Cargo.lock` file. The first time we `cargo build` with a `[patch]` manifest section, Cargo will write relevant information to the project's `Cargo.lock` file:
```toml
[[package]]
name = "nix"
version = "0.17.0"
source = "git+https://github.com/gatoWololo/nix#3b8180c430fe838e4fd71b83e5f92db6386e5c57"
```
Here we can see Cargo specifies the commit hash `3b8180c` which was used for patching as well as the URL for my version of `nix`. If we are ever unsure whether Cargo is using our patch, we can check the `Cargo.lock` file for entries like:
```
[[patch.unused]]
name = "mio"
version = "0.6.22"
source = "git+https://github.com/servo/mio.git?branch=servo-mio-0.6.22#f640fd662db03026ca2a1b21ac1e161ec6f8150b"

```
Notice the `[[patch.unused]]` section.

#### Using a Different Branch 
By default `Cargo` pulls our code from the default branch, we can specify what branch to use as well:
```toml
[patch.crates-io]
nix = { git = "https://github.com/gatowololo/nix", branch="my-dev" }
```
This may not always work. See [below example](@/blog/cargo_patch.md#crab-sharp-edge-patching-with-different-branch).
#### Specifying Commit Hash
We can also specify the commit to use as well!
```toml
[patch.crates-io]
nix = { git = "https://github.com/gatowololo/nix", rev="3b8180c" }
```

#### 🦀 Sharp Edge: `Cargo.lock` and Version Control
If our project [does not check `Cargo.lock` into version control](https://doc.rust-lang.org/cargo/faq.html#why-do-binaries-have-cargolock-in-version-control-but-not-libraries), `cargo build` will use the latest version on `master`. This irreproducible behavior is probably **not** what you want!

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
We specify the URL as `[patch."URL"]`. This patch replaces `servo/media` with my own version `gatowololo/servo-media` on Github.

#### 🦀 Sharp Edge: Patching with Different Branch
What happens if we want to patch a git dependency using the same repository but a different branch? Like so:
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
This is a known [issue](https://github.com/rust-lang/cargo/issues/5478). The current workaround is to add extra `/` to the URL of the patch so Cargo thinks the patch points to a different source but actually resolves to the same URL:
```toml
# Cargo.toml
# ...
[patch."https://github.com/gatowololo/rr_channel"]
# Notice extra '/' on URL.
rr_channel = { git = "https:///github.com//gatoWololo/rr_channel", branch = "servo_development"}
```
This works but creates a new foot gun, see [Sharp Edge: Git Dependency with Extra `/`](@/blog/cargo_patch.md#crab-sharp-edge-git-dependency-with-extra) below.
#### 🦀 Sharp Edge: Workspaces
Patching crates with workspaces works slightly differently, and is not obvious. Patching a Git dependency is straightforward enough. Notice `servo-media-gstreamer` and `servo-media-dummy` are other crates in `servo media` package:
```toml
[patch."https://github.com/servo/media"]
servo-media = {git = "https://github.com/gatoWololo/servo-media"}
servo-media-gstreamer = { git = "https://github.com/gatoWololo/servo-media"}
servo-media-dummy = { git = "https://github.com/gatoWololo/servo-media"}
```
Notice all patches point to the same `https://github.com/gatoWololo/servo-media` URL. The behavior for patching `servo-media` with a local `alt` dependency is different though. When patching other crates in the package we must specify the path not to the root of the package, but down to the individual crate roots:
```toml
[patch."https://github.com/servo/media"]
servo-media = {path = "../media/servo-media"}
servo-media-gstreamer = {path = "../media/backends/gstreamer"}
servo-media-dummy = { path = "../media/backends/dummy"}
```

#### 🦀 Sharp Edge: Git Dependency with Extra `/`
Using the extra `/` trick shown [above](@/blog/cargo_patch.md#crab-sharp-edge-patching-with-different-branch) is useful but can lead to a different error. If our code is made of different crates each containing their own manifest:
```toml
# crate_a/Cargo.toml
# ...
[dependencies]
rr_channel = { git = "https://github.com/gatoWololo/rr_channel"}
```
And:
```toml
# crate_b/Cargo.toml
# ...
[dependencies]
rr_channel = { git = "https:///github.com//gatoWololo/rr_channel"}
```
Attempting to patch in the root manifest file led to two incompatible versions of `rr_channel` being used. Why was this happening? Notice the git URL for `crate_b/Cargo.toml` accidentally has extra `/`! So `rr_channel` was only getting patched in the first instance! Using the `cargo tree` command proved invaluable for finding the offending crate's manifest.

### Conclusion
Patching is a powerful and very useful mechanism. Beware of the many sharp edges. I believe Cargo is extremely well designed, but dependency management is complicated as Cargo must handle many use cases.

### Footnotes
[^1]: One common feedback about the Rust ecosystem is the lack of intermediate and advanced documentation and blogs. I have several half-finished blogs because they were ambitious, so this blog will assume familiarity with `Cargo`, `Cargo.toml`, and `Cargo.lock`. And roughly, how Rust deals with dependencies. In the future I may come back and do a `Cargo.toml` and `Cargo.lock` in depth blog.

[^2]: [crates.io](https://crates.io/) is Rust's official crate repository. By default all crate dependencies are downloaded from here on `cargo build`.

[^3]: [Here](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html) is the official Cargo Book documentation on `[patch]`.

[^4]: While Cargo replace is not technically deprecated. The [docs](https://doc.rust-lang.org/edition-guide/rust-2018/cargo-and-crates-io/replacing-dependencies-with-patch.html) state "while we don't intend to deprecate or remove [replace], you should prefer [patch] in all circumstances". So we should always use Cargo patch.

[^5]: There is nothing stopping us from leaving the `[patch]` permanently, but it does not seem to be good practice.

[^6]: The structure of the `~/.cargo/` directory is unclear to me. So I omit attempting to explain finding this path.

### Comments
Please leave comments below. You can also leave comments directly in the relevant [Github Issue](https://github.com/gatoWololo/PersonalWebsite/issues) and it will show up below.