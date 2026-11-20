# librime UOS Rolling Builds

This repository is a small, independent build pipeline for distributing
UOS-compatible `librime.so` files. It is not a fork of librime and does not
change the upstream project. On each scheduled or manually requested run, the
workflow checks out `rime/librime` and its submodules, builds the selected
plugins, and publishes downloadable GitHub Actions artifacts.

The purpose is to make it easier to use librime on UOS systems where a
prebuilt library may require newer libc or runtime components than the target
system provides.

## Why glibc 2.28

The target UOS installation reports glibc 2.28. A shared library built on a
newer distribution can reference newer versioned glibc symbols and fail to
load on UOS. Building against a glibc 2.28 baseline avoids introducing
references to newer glibc symbol versions. The workflow uses the
`manylinux_2_28` x86_64 and aarch64 images and checks that `librime.so` does not
require a GLIBC symbol newer than 2.28.

This baseline is chosen for the target system, not because glibc 2.28 is the
newest or fastest option. It is generally suitable for systems with glibc
2.28 or later, but the result should still be tested on each supported UOS
edition and architecture.

## What “static” means here

A dynamically loaded `.so` cannot be made independent of the host's dynamic
loader and libc in a reliable, portable way. This build produces a shared
`librime.so` with the project dependencies linked statically where possible;
glibc remains a runtime dependency.

The workflow:

- builds `librime.so` for x86_64 and aarch64;
- builds dependencies as position-independent static libraries and links them
  into the shared library;
- statically links libstdc++ and libgcc to reduce dependence on the versions
  installed on UOS;
- builds the `librime-lua` and `librime-octagram` plugins;
- checks the GLIBC symbol-version ceiling, unresolved dynamic symbols, and a
  small program linked against `librime.so`.

The public librime API is C-based. Avoid passing C++ objects or exceptions
across the library boundary, especially because the shared object contains its
own statically linked C++ runtime.

## Schedule and artifacts

The workflow runs every Monday at 02:15 UTC (10:15 China Standard Time) and
can also be started with **Actions → Scheduled UOS-compatible librime build →
Run workflow**.

Each run uploads one artifact per architecture. Artifact names follow this
pattern:

```text
rime-uos-glibc228-x86_64-<run-number>
rime-uos-glibc228-aarch64-<run-number>
```

Artifacts are retained for 30 days. The archive contains `lib64/librime.so`
and its versioned symlinks, headers, static dependency archives, and bundled
data such as OpenCC dictionaries.

## Using an artifact

Choose the artifact matching the target machine's architecture, download and
extract it, then use `lib64/librime.so` and the headers from `include/`. The
shared library still requires a compatible glibc dynamic loader; check it on
the target system before deployment:

```sh
ldd -r lib64/librime.so
```

Install the shared library in a location known to the dynamic linker (or
configure the application's library path). Keep the bundled data available
where the application expects it; in particular, OpenCC needs its dictionary
files at runtime.

## Rolling-build note

The workflow tracks the `master` branch of librime and the current revisions
of its submodules and plugins. Consecutive scheduled artifacts may therefore
contain different upstream code. For a reproducible or production release,
pin librime to a tag or commit and pin plugin revisions before publishing.
