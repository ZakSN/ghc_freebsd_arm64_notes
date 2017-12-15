# Attempt At Cross Building GHC 8.0.2 for FreeBSD 12.0-CURRENT arm64

## Contents:
1. Introduction
	1. Current State
2. Overview
	1. GHC's Build System
	2. Plan of Attack
3. Set Up Build Environment
4. Build
5. Installation On Target
6. Conclusion
7. Additional Resources

## 1. Introduction:
The Glasgow Haskell Compiler (GHC) is a compiler for the functional programming language Haskell [1]. This document deals with version 8.0.2 which is the current version in the ports tree. Currently GHC only has support for amd64 and i386 architectures under FreeBSD. It would be advantageous to port GHC to arm64 since it is a build requirement for a number of other ports. While GHC does not have native code generation support for arm64 it does have support for GCC or LLVM backends [2] both of which do exist on the FreeBSD arm64 platform.

### 1.1. Current State
As of this writing I have not gotten GHC-8.0.2 to install successfully on FreeBSD 12.0 arm64. I have built a working amd64 -> arm64 GHC cross compiler (stage 1) and I used it to build a working native arm64 (stage 2) compiler (backed by GCC). Unfortunately I haven't been able to successfully install the stage 2 compiler on the target system.

## 2. Overview:
This section is intended to give a broad view of GHC's build system, and outline the plan to get a working GHC installation on an arm64 target. The reason that porting GHC is difficult is that GHC is largely written in Haskell and requires an earlier or equal version of GHC to build. This makes it difficult to port it to a new platform since there is no existing version of GHC on the new platform. This circular dependency is resolved by first building a cross compiler and then using the cross compiler to build a native compiler that targets the new platform. GHC's two stage build system can be configured to support this build procedure.

### 2.1. GHC's Build System
The GHC website provides plenty of documentation [3] about the specifics of the build system, as such this section will only cover the basics. Information about crossbuilding GHC can also be found on the GHC website [4]. The GHC documentation [3][4] defines three terms: *build*, *host*, and *target* which will be used in this document. 

- The *build* system is the computer that is used to build an instance of GHC. 
- The *host* system is the platform that an instance of GHC runs on. 
- The *target* system is the platform that an instance of GHC generates code for. 

The goal of this document is to produce an instance of GHC where *host* = *target* = FreeBSD/arm64, which could then be used to build the `lang/ghc` port on an FreeBSD/arm64 system.

GHC has a two stage build system [4]. For the common case where *build* = *host* = *target* the two stage system ensures that the final instance of GHC built does not have any dependence on the original instance of GHC that was used to build it. This separation of build stages means that the build system can be configured to produce an instance of GHC that runs on and targets a different platform than the instance of GHC used to do the building.

### 2.2. Plan of Attack
The goal is to create an instance of GHC where *host* = *target* = FreeBSD/arm64, given that an instance of GHC where *build* = *host* = *target* = FreeBSD/amd64 (or i386) already exists. To achieve this goal stage 1 must be a cross compiler, i.e. *build* = *host* = FreeBSD/amd64 and *target* = FreeBSD/arm64. Once a stage 1 cross compiler is built it can be used to build a stage 2 compiler where *build* = FreeBSD/amd64 and *host* = *target* = arm64. In theory a working stage 2 would be sufficient to build FreeBSD ports that depend on GHC, however it would be nice to use stage 2 to build the `lang/ghc` port on an arm64 system so that the packages built from the ports tree can remain internally consistent. As such this document will refer to building `lang/ghc` on arm64 as 'stage 3'. A working 'stage 3' build (where *build* = *host* = *target*) is the end goal. The remainder of this document will describe the procedure that the author followed in an attempt to achieve this goal.

## 3. Set Up Build Environment:
In order to do an unregistersied (section 4) crossbuild of GHC it is necessary to have a C cross compiler. Since clang/llvm is the default C compiler on FreeBSD and is a native cross compiler it made sense to try to use it. Unfortunately the GHC build system is very picky about versions of clang/llvm. GHC 8.0.2 needs version 3.7 of clang/llvm [5]. Since version 3.7 clang/llvm is no longer available in the ports tree the binary distribution must be downloaded from upstream [6]. Since it is necessary to manually install cross build tools on the *build* computer there is not really any advantage to using clang/llvm vs. gcc-arm64, but the author used clang 3.7 so that is what this document deals with. Once a crossbuild toolchain has been selected and installed it may be necessary to provide libraries and headers for FreeBSD arm64. Providing headers/libraries was definitely necessary when using clang 3.7 however if an appropriate version GCC can be installed from ports it may not be required. The easiest way to get the necessary files is to copy them from the target arm64 computer. The directories needed were:

```
/
├lib/
├usr/
├─local/
│  ├include/
│  └lib/
├─include/
└─lib/
```

GHC's build system assumes that the crossbuild tools are prefixed with the *target*'s target triplet. Since it is desirable to have 'stage 3' work with the arm64 ports infrastructure the target triple to use is `aarch64-portbld-freebsd`. Most tools can be specified during the configure stage of the actual build (section 4), but the C compiler is slightly more involved, since it needs to know about the arm64 libraries. The (inelegant) method used to specify the correct compiler was to place a shell script named `aarch64-portbld-freebsd-clang` in the $PATH. The script used looked something like:
```
#!/bin/sh
"${PATH_TO_CLANG_3.7}"/clang \
	--sysroot="${PATH_TO_ARM64_LIBS}" \
	--target=aarch64-portbld-freebsd \
	"${@}"
```
Note that the `--sysroot` switch assumes all libraries and binaries are in the same directory, so when the arm64 files are copied over make sure directory structure is not preserved. Alternatively one could specify `-I="${PATH_TO_ARM64_INCLUDES}"` and `-L=${PATH_TO_ARM64_LIBS}`, however this causes clang to complain about unused command line options during some stages of the build. 

Most of the tools needed to build GHC are relatively standard and probably already present if the system has been used to build software before, However if the Haskell platform is not already present on the build system some other ports will need to be installed, specifically: `devel/hs-happy` and `devel/hs-alex`.

## 4. Build:
GHC follows a configure/make build, However before starting the build there are some patches to apply. All of the patches from `lang/ghc/files` where applied. Two other patches are provided with this document. The patches applied were, where `"${TOP}"` is the top of the build tree:

- from the ghc port:
	- `patch-configure.ac` applied to `"${TOP}"/configure.ac`
	- `patch-ghc.mk` applied to `${TOP}/ghc.mk`
	- `patch-libraries_Cabal_Cabal_Distribution_Simple_GHC.hs` applied to `"${TOP}"/libraries/Cabal/Cabal/Distribution/Simple/GHC.hs`
	- `patch-libraries__Cabal__Cabal__Distribution__Simple__Program__Builtin.hs` applied to `/usr/home/zak/ghc_arm64/ghc-8.0.2/libraries/Cabal/Cabal/Distribution/Simple/Program/Builtin.hs`
- provided patches:
	- `dynflags_hs.diff` applied to `"${TOP}"/compiler/main/DynFlags.hs` 
	- `ppr_hs.diff` applied to `"${TOP}"/compiler/llvmGen/LlvmCodeGen/Ppr.hs`

since `configure.ac` was patched it is necessary to run `autoreconf` to rebuild the configure script. Some build options are configured by `"${TOP}"/mk/build.mk`, the `build.mk` used was:
```
HADDOCK_DOCS=NO
```
This turns off the documentation build and reduces build time and build dependencies. Additionally the the line
```
WITH_TERMINFO=NO
```
can be added to remove dependence on ncurses. Example build configurations are provided in `"${TOP}"/mk/build.mk.sample`

Next configure the build. The configure command used was:
```
$ ./configure \
--target=aarch64-unknown-freebsd12.0 \
--with-nm=nm \
--with-ar=ar \
--with-ranlib=ranlib \
--with-llc="${PATH_TO_LLVM_37_BIN_DIST}"/bin/llc \
--with-opt="${PATH_TO_LLVM_37_BIN_DIST}"/bin/opt \
--with-"${PATH_TO_LLVM_37_BIN_DIST}"/llvm-objdum \
--with-ld=aarch64-freebsd-ld \
--enable-unregisterised
```
the `--target=` switch specifies the stage 2 target. If it is different than the *build* architecture the build system infers that a cross build is required. the `--with-*=` switches specify the build tools to use. the `--enable-unregisterised` option causes GHC's frontend to output portable C that can be compiled by a standard C compiler such as GCC or clang. This will likely get automatically set during the build since GHC does not support FreeBSD/arm64 by default but it is fine to set it in the configure command as well. If the configure goes well it should generate a report detailing which tools the build will use and whether or not it is a cross build. Once configuration is done make can be run. GHC's build depends on gnumake and supports job parallelization:
```
$ gmake -j4 
```
The build takes a while (~1.5hrs). When it's finished the directory `"${TOP}"/inplace/bin` should contain the shell scripts `ghc-stage1` and `ghc-stage2`. `ghc-stage1` should run on the *build* machine and produce exectuables that run on the *target* and `ghc-stage2` should run on and target the *target* machine.

## 5. Installation On Target:
This is where things started to fall apart. Simply copying the build tree to the *target* machine and running `gmake install` does not work, since the `inplace/cabal` is built for the wrong architecture. The next option tried was to make a binary distribution on the *build* machine and copy it to the *target* for installation. To make a binary distribution run:
```
$ gmake binary-dist
```
from the *build* machine. This produces an archive named `ghc-8.0.2-aarch64-portbld-freebsd.tar.xz` in the top of the build tree. Copying the archive to the *target* and uncompressing it produces a tree similar to the build tree. After some fiddling I was able to get configure to run successfully in the bin-dist tree. From there running the `install` target worked at least partially. It got as far as installing GHC in the $PATH, but fails when trying to install (register?) Haskell's standard libraries. As far as I can tell the failure is related to cabal. This is pretty much the end of my knowledge of the Haskell platform.

Despite not being able to install GHC I was able to use `inplace/ghc-stage2` to compile some simple haskell programs on the *target*. stage 2 seems to still rely on tools named with the same target-triplet (`aarch64-portbld-freebsd`) which is a little odd, but easy to compensate for. Additionally since stage 2 is unregisterised it requires a backend to compile the C generated by the frontend. I imagine clang 3.7 would work, but GCC is already in the arm64 ports tree and therefore easier to install. As such GCC was used as the backend on the *target* machine.

## 6. Conclusion:
This attempt at installing GHC on FreeBSD/arm64 was ultimately unsuccessful. However it is definitely possible to build an instance of GHC that will compile Haskell source for arm64. As such someone with a better knowledge of the Haskell language/platform would probably be able to get this to work.

## 7. Additional Resources:
[1]. GHC homepage: <https://www.haskell.org/ghc/>

[2]. GHC backends. This link is for GHC-7.8.3, hopefully it is still relevant for GHC-8.0.2: <https://downloads.haskell.org/~ghc/7.8.3/docs/html/users_guide/code-generators.html>

[3]. GHC build system documentation: <https://ghc.haskell.org/trac/ghc/wiki/Building>

[4]. GHC crossbuild information: <https://ghc.haskell.org/trac/ghc/wiki/Building/CrossCompiling>

[5]. GHC 8.0.2 release information: <https://www.haskell.org/ghc/download_ghc_8_0_2.html>

[6]. clang/llvm 3.7 upstream binary distribution: <http://releases.llvm.org/download.html#3.7.1>

---
Author:

Zakary Nafziger (zsnafzig_at_edu.uwaterloo.ca)

Written 2017-12-15