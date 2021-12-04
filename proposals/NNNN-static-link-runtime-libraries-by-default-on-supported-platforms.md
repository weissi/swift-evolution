# Statically link Swift runtime libraries by default on supported platforms

* Proposal: [SE-NNNN](NNNN-static-link-runtime-libraries-by-default-on-supported-platforms.md)
* Authors: [tomerd](https://github.com/tomerd)
* Review Manager: TBD
* Status: **Awaiting review**

* Implementation: https://github.com/apple/swift-package-manager/pull/3905
* Initial discussion: [Forum Thread](https://forums.swift.org/t/pre-pitch-statically-linking-the-swift-runtime-libraries-by-default-on-linux)

## Introduction

Swift 5.3.1 introduced [static linking on Linux](https://forums.swift.org/t/static-linking-on-linux-in-swift-5-3-1/). With this feature, users can set the `--static-swift-stdlib` flag when invoking  SwiftPM commands (or the long form `-Xswiftc -static-stdlib`) in order to statically link the Swift runtime libraries into the program.

On some platforms, such as Linux, this is often the preferred way to link programs, since the program is easier to deploy to the target server or otherwise share. This proposal explores making it the default behavior on such platforms.

## Motivation

Darwin based platform ship with the Swift runtime libraries in the shared cache. This allows building smaller Swift programs by dynamically linking the Swift runtime libraries. The shared cache keeps the cost of loading these libraries low.

Other platforms, such as Linux, do not ship with the Swift runtime libraries. As such a deployment of Swift program (e.g. a web service built with Swift) on such platform required one of 3 non-convenient options:

1. Package the application with a "bag of SOs" (the runtime libraries) along side the program.
2. Statically link with the runtime libraries using the `--static-swift-stdlib` flag described above.
3. Use a "runtime" docker image that contain the _correct version_ of the the runtime libraries.

Out of the three options, the most convenient is #2 given that #1 requires manual intervention and/or additional wrapper scripts that use `ldd` to deduce the correct list of runtime libraries and #3 is convenient but version sensitive. #2 also has cold start performance advantage  because there is less dynamic library loading. However #2 comes at a cost of bigger binaries.  

Stepping outside the Swift ecosystem, deployment of statically linked programs is often the preferred way on server centric platforms such as Linux, as it dramatically simplifies deployment of server workloads. For reference, Go and Rust both chose to statically link programs by default for this reason.

## Proposed solution

We propose to make statically linking of the Swift runtime libraries the default behavior on platforms that support it, with an opt-out way to disable the default behavior.

Note this does not mean the resulting program is fully statically linked - only the Swift runtime libraries (stdlib, foundation, dispatch, etc) would be statically linked into the program, while external dependencies will continue to be dynamically linked and would remain the a concern left for the user when deploying the program. Such external dependencies include:
1. Glibc: On Linux, Swift relies on Glibc to interact with the system and its not possible to fully statically link programs based on Glibc. In practice this is not usually not a problem since most/all Linux systems ship with a compatible Glibc.
2. At this time, non-Darwin version of Foundation (aka libCoreFoundation) has two modules that rely on system dependencies (`FoundationXML` on `libxml2` and `FoundationNetwork` on `libcurl`) which cannot be statically linked at this time and require to be installed on the target system.
3. Any system dependencies the program itself brings (e.g. `libsqlite`, `zlib`) would not be statically linked and requires to be installed on the target system.

## Detailed design

The changes proposed are focused on the behavior of SwiftPM. We propose to change the default for statically vs. dynamically linking of the Swift runtime libraries as follows:

### Default behavior

* Darwin based platforms support static linking but the Swift runtime libraries are shipped with the operating system and statically linking them is a discouraged anti-pattern. As such the default on Darwin will remain dynamically linking the Swift runtime libraries.

* Linux and WASI support static linking and would benefit from it for the reasons highlighted above. We propose to change the default on these platforms to statically link the Swift runtime libraries.

* Windows may benefit from statically linking for the reasons highlighted above but it is not technically supported at this time. As such the default on Windows will remain dynamically linking the Swift runtime libraries.

* Support for static linking on other platform than then ones mentioned above is undefined. As such the default on platforms not listed above will remain dynamically linking the Swift runtime libraries.

### Opt-in vs. Opt-out

SwiftPM's `--static-swift-stdlib` CLI flag is designed as an opt-in way to achieve static linking of the Swift runtime libraries.

We propose to deprecate `--static-swift-stdlib` and introduce a new flag `--disable-static-swift-runtime` which is designed as an opt-out from the default behavior described above.

Users that want to force static linking (as with `--static-swift-stdlib`) can use the long form `-Xswiftc -static-stdlib`.

### Example

Consider the following simple program:

```bash
root@4c64d6ba11f0:/tmp# cd test/
root@4c64d6ba11f0:/tmp/test# swift package init --type executable
Creating executable package: test
Creating Package.swift
Creating README.md
Creating .gitignore
Creating Sources/
Creating Sources/test/main.swift
Creating Tests/
Creating Tests/testTests/
Creating Tests/testTests/testTests.swift

root@4c64d6ba11f0:/tmp/test# cat Sources/test/main.swift
print("Hello, world!")
```
Building the program with default dynamic linking yields the following:

```bash
root@4c64d6ba11f0:/tmp/test# swift build -c release
[3/3] Build complete!

root@4c64d6ba11f0:/tmp/test# ll --block-size=K .build/release/test
-rwxr-xr-x 1 root root 17K Dec  3 22:48 .build/release/test*

root@4c64d6ba11f0:/tmp/test# ldd .build/release/test
	linux-vdso.so.1 (0x00007ffc82be4000)
	libswift_Concurrency.so => /usr/lib/swift/linux/libswift_Concurrency.so (0x00007f0f5cfb5000)
	libswiftCore.so => /usr/lib/swift/linux/libswiftCore.so (0x00007f0f5ca55000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f0f5c85e000)
	libdispatch.so => /usr/lib/swift/linux/libdispatch.so (0x00007f0f5c7fd000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f0f5c7da000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f0f5c7d4000)
	libswiftGlibc.so => /usr/lib/swift/linux/libswiftGlibc.so (0x00007f0f5c7be000)
	libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f0f5c5dc000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f0f5c48d000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f0f5c472000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f0f5d013000)
	libicui18nswift.so.65 => /usr/lib/swift/linux/libicui18nswift.so.65 (0x00007f0f5c158000)
	libicuucswift.so.65 => /usr/lib/swift/linux/libicuucswift.so.65 (0x00007f0f5bf55000)
	libicudataswift.so.65 => /usr/lib/swift/linux/libicudataswift.so.65 (0x00007f0f5a4a2000)
	librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f0f5a497000)
	libBlocksRuntime.so => /usr/lib/swift/linux/libBlocksRuntime.so (0x00007f0f5a492000)
```

Building the program with static linking of the Swift runtime libraries yields the following:

```bash
root@4c64d6ba11f0:/tmp/test# swift build -c release --static-swift-stdlib
[3/3] Build complete!

root@4c64d6ba11f0:/tmp/test# ll --block-size=K .build/release/test
-rwxr-xr-x 1 root root 35360K Dec  3 22:50 .build/release/test*

root@4c64d6ba11f0:/tmp/test# ldd .build/release/test
	linux-vdso.so.1 (0x00007fffdaafa000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fdd521c5000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fdd521a2000)
	libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007fdd51fc0000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fdd51e71000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007fdd51e56000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fdd51c64000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fdd54211000)
```

These snippets demonstrates the following:
1. Statically linking of the Swift runtime libraries increases the binary size from 17K to ~35M.
2. Statically linking of the Swift runtime libraries reduces the dependencies reported by `ldd` to core Linux libraries.

This jump in binary size may be alarming at first sight, but since the program is not usable without the Swift runtime libraries, the actual size of the deployable unit is similar.

```bash
root@4c64d6ba11f0:/tmp/test# mkdir deps
root@4c64d6ba11f0:/tmp/test# ldd ".build/release/test" | grep swift | awk '{print $3}' | xargs cp -Lv -t ./deps
'/usr/lib/swift/linux/libswift_Concurrency.so' -> './deps/libswift_Concurrency.so'
'/usr/lib/swift/linux/libswiftCore.so' -> './deps/libswiftCore.so'
'/usr/lib/swift/linux/libdispatch.so' -> './deps/libdispatch.so'
'/usr/lib/swift/linux/libswiftGlibc.so' -> './deps/libswiftGlibc.so'
'/usr/lib/swift/linux/libicui18nswift.so.65' -> './deps/libicui18nswift.so.65'
'/usr/lib/swift/linux/libicuucswift.so.65' -> './deps/libicuucswift.so.65'
'/usr/lib/swift/linux/libicudataswift.so.65' -> './deps/libicudataswift.so.65'
'/usr/lib/swift/linux/libBlocksRuntime.so' -> './deps/libBlocksRuntime.so'
root@4c64d6ba11f0:/tmp/test# ll --block-size=K deps/
total 42480K
drwxr-xr-x 2 root root     4K Dec  3 22:59 ./
drwxr-xr-x 6 root root     4K Dec  3 22:58 ../
-rw-r--r-- 1 root root    17K Dec  3 22:59 libBlocksRuntime.so
-rw-r--r-- 1 root root   432K Dec  3 22:59 libdispatch.so
-rwxr-xr-x 1 root root 27330K Dec  3 22:59 libicudataswift.so.65*
-rwxr-xr-x 1 root root  4030K Dec  3 22:59 libicui18nswift.so.65*
-rwxr-xr-x 1 root root  2403K Dec  3 22:59 libicuucswift.so.65*
-rwxr-xr-x 1 root root   520K Dec  3 22:59 libswift_Concurrency.so*
-rwxr-xr-x 1 root root  7622K Dec  3 22:59 libswiftCore.so*
-rwxr-xr-x 1 root root   106K Dec  3 22:59 libswiftGlibc.so*
```

## Impact on existing packages

The new behavior will take effect with a new version of SwiftPM, and packages build with that version will be linked accordingly.

* Deployment of applications using "bag of SOs" technique (#1 above) will continue to work as before (though would be potentially redundant).
* Deployment of applications using explicit static linking (#2 above) will continue to work and emit a warning that its redundant.
* Deployment of applications using docker "runtime" images (#3 above) will continue to work as before (though would be redundant).

## Alternatives considered and future directions

The most obvious question this proposal brings is why not fully statically link the program instead of statically link only the runtime libraries.
Go is a good example for creating fully statically linked programs, contributing to its success in the server ecosystem at large.
Swift offers a flag for this linking mode: `-Xswiftc -static-executable`, but in reality Swift's ability to create fully statically linked programs is constrained.
First, Swift's dependency on Glibc (on Linux) makes it impossible given the nature of Glibc. Go chosen to implement its own version of libc which decouples it from the system well.
Further, Swift's interoperability with C and system libraries make it difficult (impossible?) to create fully statically linked programs as the build system cannot _reliably_ unwind the dependency tree without information from the underlying dependency management system (e.g. yum, apt) which is often lacking.
As Swift's ability to create fully statically linked programs improves, we should consider changing the default from `-Xswiftc -static-stdlib` to `-Xswiftc -static-executable`

A more immediate future direction which would improve programs that need to use of FoundationXML and FoundationNetworking is to replace the system dependencies of these modules with native implementation. This is outside the scope of this proposal which focuses on SwiftPM's behavior.

Another alternative is to do nothing. In practice, this proposal does not add new features, it only changes default behavior which is already achievable with the right knowledge of build flags. That said, we believe that changing the default will make using Swift on non-Darwin platforms easier saving time and costs to Swift users on such platforms.

The spelling of the new flag `--disable-static-swift-runtime` is open to alternative ideas, e.g. `--disable-static-swift-runtime-libraries`.
