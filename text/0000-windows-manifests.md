- Feature Name: `windows-manifests`
- Start Date: 2020-08-18
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

One paragraph explanation of the feature.

An **Application manifest** in Windows is an XML document describing metadata that may also affect the runtime behaviour of an application. They may be packaged with a PE (Portable Executable) binary (i.e. .exe or .dll) as an emdedded resource or alternatively exist in the same folder with a **.manifest** extension (e.g. `foo.exe` and `foo.exe.manifest`). Application manifests for DLLs may sometimes be referred to as an *Assemby Manifest* but that should not be confused with another such named manifest specifically for CLI (Common Language Infrastructure) _assemblies_.

This RFC proposes adding a new rustc flag for including one or more manifests to embed into the final binary. As well as corresponding metadata in Cargo along with the logic to collect manifests across dependencies.

# Motivation
[motivation]: #motivation

As mentioned above, application manifests allow specifying metadata and modifying certain runtime behaviour. Those include:

(From https://github.com/rust-lang/rfcs/issues/721 & more)

* Allows easily viewable version numbers and descriptive text on an .exe or .dll
* Can specify which version of a system (or any other) .dll your application should use, thus avoiding DLL hell - notable alternative use case for this is to ensure version 6 or above of Microsoft Common Controls which will enable XP styles of window controls in 32-bit applications; this particular one probably ought to be automatically included for apps requiring COMCTL32.dll since without you'll get some ugly ancient looking GUIs.
* Ability to specify UAC privilege level required for your application and avoid Windows determining this heuristically (temporarily solved for test runs using an environment variable in rust-lang/rust#21496)
* Allows use of code signing for distributed binaries (also needs a certificate). This is a must for writing Windows drivers nowadays.
* Enable use of certain features/modes such as (super) high-res scrolling, DPI awareness, printer driver isolation.
* Allows specifying a compatibility mode to run under (i.e. Windows 7, 8, Vista etc)
* As of Windows 10 1903 release, allows forcing the application to use UTF-8 as the process code page
* As of Windows 10 2004 release, allows overriding the default heap implmentation to use the new Segment Heap
* and more @ [Microsoft's documentation](https://docs.microsoft.com/en-ca/windows/win32/sbscs/application-manifests).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

As a Rust developer building an `.exe` or `.dll` you may want to include a manifest with your binary for any of the aforementioned reasons. It would simply be a matter of updating your `Cargo.toml`:

```diff
[package]
 name = "myapp"
 version = "0.1.0"
 authors = ["Luqman Aden <me@luqman.ca>"]
 edition = "2018"
 
+[package.metadata.windows] # TODO: Should this be top-level instead?
+manifest = "myapp.manifest"
```

Alternatively, if using rustc directly it would be a command line flag that can specify multiple manifests:

> rustc main.rs -C windows-manifest=first.manifest,second.manifest


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Cargo

When building a dylib or bin crate, Cargo would need to collect together the manifest specified by the crate along with any specified by a dependencies's Cargo.toml. Cargo would then just pass the path to all the manifests to rustc via the rustc command line flag.

## Rustc

### MSVC

If we're using the MSVC linker, we can simply make use of its existing capabilities by specifying the [`/MANIFEST:EMBED`](https://docs.microsoft.com/en-us/cpp/build/reference/manifest-create-side-by-side-assembly-manifest) and [`/MANIFESTINPUT:*filename*`](https://docs.microsoft.com/en-us/cpp/build/reference/manifestinput-specify-manifest-input).

Furthermore, one can pass multiple `/MANIFESTINPUT` flags and link.exe will merge them together for you.

### LLD (lld-link)

The Link flavor of LLD tries to be compatible with MSVC's link.exe and thus also implements the same options above.

### GCC, LD, & LLD (ld.lld)

The remaining linker flavours are usually used for the `*-windows-gnu` target. Unfortunately they do not provide similar functionality as the `MANIFESTINPUT` flag. Today, gcc from mingw may attempt to link a `default-manifest.o` that a user can override by providing their own. But how does this object created? First a resource (`.rc`) file must be created:

```c
#define RT_MANIFEST 24
1 RT_MANIFEST "myapp.exe.manifest"
```

We can pass this to the `windres` tool shipped with mingw:

> windres myapp.rc default-manifest.o

The resulting object can be passed along to the linker to include in the final binary. Unfortunately that doesn't handle the multiple manifests case. Fortunately since LLD has to implement the manifest mergining functionaliy for lld-link, that same code is available in LLVM as the [WindowsManifestMerger](https://llvm.org/doxygen/WindowsManifestMerger_8cpp_source.html) class.

Thus, for this case we could call out to LLVM to merge together all the manifests, before using `windres` as described above.


# Drawbacks
[drawbacks]: #drawbacks

Probably the most common scenario means an application including just a single manifest which can be done via one of multiple existing crates that deal with the more general case of embeddeding a resource:

* https://crates.io/crates/windres
* https://crates.io/crates/embed-resource

This gets us 90% of the way.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

Windows developers familiar with the normal Visual Studio flow will expect to be able to specify an application manifest and more so, multiple if necessary. And to be able to do this out-of-the-box. 

Alternatives include:

* Using one of the aforementioned cargo crates in a build.rs
* Distributing a manifest alongside your exe/dll

# Prior art
[prior-art]: #prior-art

Build system wise, Visual Studio (exposes similar options)[https://docs.microsoft.com/en-us/cpp/build/reference/manifest-tool-property-pages?view=vs-2019]. Not to mention the aforementioned support directly in [lld-]link.exe

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should Rust include a default manifest? If so, how does one override it?
- Also support the more general case resources?

# Future possibilities
[future-possibilities]: #future-possibilities

Ship a default manifest for:

- Enabling long path support
- Enabling the new Segment Heap by default in Windows 10 2004