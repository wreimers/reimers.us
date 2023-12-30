---
title: "Building LLVM and Clang on macOS 10.14.6 with Xcode 11.1"
date: 2019-11-03T11:00:00-08:00
categories: ["Development", "LLVM"]
tags: ["clang", "llvm"]
description: "Wherein I become excited about LLVM and set about trying to build it."
disqus: false
draft: false
---

I recently attended the 2019 LLVM Developer Meeting:

`Sorry, X (née Twitter) deleted my post.`

After the first hour, spent listening to the keynote called "Generating
Optimized Code with GlobalISel" my brain hurt, but... it was that good kind of
pain, like the one you get after a particularly hard workout. By the end of the
day, after having seen several more talks, I was psyched up!

Naturally, when I went home and immediately tried to build LLVM on my macBook
Pro running macOS 10.14.6 Mojave, and with Xcode 11.1 installed.

## Before We Begin

Make sure you've launched Xcode at least once and it's completed the First
Launch Experience, which includes accepting the license agreement. You can also
accept using the command line:

```
$ sudo xcodebuild -license accept
```

## System Headers

I'm runing Xcode 11.1, which comes with the macOS 10.15 Catalina SDK. According
to [the Xcode 10.0 release notes](https://developer.apple.com/documentation/xcode_release_notes/xcode_10_release_notes),
Apple has helpfully provided a package to install the system headers into
/usr/include. You'll need these for the LLVM unit tests.

```
$ open /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg
``` 

This command will open the package installer UI for you so you can install the
headers:

![Package Installer UI](/img/macos-1014-headers-installer-ui.png)

## Install Requirements

Homebrew has some flaws, but I just want dependency management to get out of my
way and let me do work, and it's really excellent at that. 

I updated Homebrew and upgraded all of my installed software. What can I say? I
like to live on the edge.

```
$ brew update
[...]
$ brew upgrade
==> Upgrading 9 outdated packages:
[...]
$
```

This is all I had to do to get the tools I needed to build LLVM:

```
$ brew install cmake
==> Downloading https://homebrew.bintray.com/bottles/cmake-3.15.4.mojave.bottle.tar.gz
Already downloaded: /Users/liamr/Library/Caches/Homebrew/downloads/9013e9f40f45899dbcfa580fa058dfd988d4f12a2d900b9d8b79c4ea99cba1de--cmake-3.15.4.mojave.bottle.tar.gz
==> Pouring cmake-3.15.4.mojave.bottle.tar.gz
==> Caveats
Emacs Lisp files have been installed to:
  /usr/local/share/emacs/site-lisp/cmake
==> Summary
🍺  /usr/local/Cellar/cmake/3.15.4: 5,800 files, 53.4MB
$ brew install ninja
[...]
$ brew install ccache
[...]
$
```

## Clone, Configure, Build & Test

Next, it was time to clone the repository.

```
$ git clone git@github.com:llvm/llvm-project.git
Cloning into 'llvm-project'...
remote: Enumerating objects: 1141, done.
remote: Counting objects: 100% (1141/1141), done.
remote: Compressing objects: 100% (155/155), done.
remote: Total 3577397 (delta 998), reused 1011 (delta 981), pack-reused 3576256
Receiving objects: 100% (3577397/3577397), 1.14 GiB | 5.98 MiB/s, done.
Resolving deltas: 100% (2862094/2862094), done.
Checking out files: 100% (86709/86709), done.
$
```

Whoo-baby, that's a monorepo behemoth. Next, I configured a build.

```
$ cd llvm-project
$ mkdir build
$ cd build/
$ cmake -G Ninja ../llvm/ \
    -DLLVM_ENABLE_PROJECTS="clang" \
    -DCMAKE_BUILD_TYPE="Release" \
    -DLLVM_ENABLE_ASSERTIONS=On \
    -DLLVM_CCACHE_BUILD=On
[...]
-- Build files have been written to: /Users/liamr/Code/llvm-project/build
$
```

And now, we build! (And build and run all the tests.)

```
$ ninja check-all
[184/4275] Generating VCSRevision.h
-- Found Git: /usr/bin/git (found version "2.21.0 (Apple Git-122)")
[2412/4275] Generating VCSVersion.inc
-- Found Git: /usr/bin/git (found version "2.21.0 (Apple Git-122)")
[3470/4275] cd /Users/liamr/Code/llvm-project/clang/bindings/python && /usr/local/Cellar/cmake/3.15.4/...E env CLANG_LIBRARY_PATH=/Users/liamr/Code/llvm-project/build/lib /usr/bin/python -m unittest discover
.................................................s................................................s.s.......s...s.........s...
----------------------------------------------------------------------
Ran 126 tests in 0.693s

OK (skipped=6)
[4274/4275] Running all regression tests
llvm-lit: /Users/liamr/Code/llvm-project/llvm/utils/lit/lit/llvm/config.py:342: note: using clang: /Users/liamr/Code/llvm-project/build/bin/clang
llvm-lit: /Users/liamr/Code/llvm-project/llvm/utils/lit/lit/util.py:408: note: using SDKROOT: u'/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.15.sdk'
llvm-lit: /Users/liamr/Code/llvm-project/build/utils/lit/tests/lit.cfg:69: warning: Setting a timeout per test not supported. Requires the Python psutil module but it could not be found. Try installing it via pip or via your operating system's package manager. Some tests will be skipped and the --timeout command line argument will not work.
Testing Time: 680.16s
  Expected Passes    : 48896
  Expected Failures  : 172
  Unsupported Tests  : 1191

1 warning(s) in tests.
$
```

## Test Compilation

About an hour later, my laptop's fans have had a great workout, and I've built something! But... does it work?

```
$ mkdir outputs
$ mkdir inputs
$ vi inputs/helloworld.c
[...]
$ bat inputs/helloworld.c
───────┬───────────────────────────────────────────────────────────────────────
       │ File: inputs/helloworld.c
───────┼───────────────────────────────────────────────────────────────────────
   1   │ #include <stdio.h>
   2   │
   3   │ int main() {
   4   │    printf("Hello, World!\n");
   5   │    return 0;
   6   │ }
───────┴───────────────────────────────────────────────────────────────────────
$ bin/clang -Wall inputs/helloworld.c -o outputs/helloworld
$ outputs/helloworld
Hello, World!
$ bin/clang -S -emit-llvm -O0 inputs/helloworld.c -o outputs/helloworld.ll
$ bat outputs/helloworld.ll
───────┬───────────────────────────────────────────────────────────────────────
       │ File: outputs/helloworld.ll
───────┼───────────────────────────────────────────────────────────────────────
   1   │ ; ModuleID = 'inputs/helloworld.c'
   2   │ source_filename = "inputs/helloworld.c"
   3   │ target datalayout = "e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
   4   │ target triple = "x86_64-apple-macosx10.14.6"
   5   │
   6   │ @.str = private unnamed_addr constant [15 x i8] c"Hello, World!\0A\00", align 1
   7   │
   8   │ ; Function Attrs: noinline nounwind optnone ssp uwtable
   9   │ define i32 @main() #0 {
  10   │ entry:
  11   │   %retval = alloca i32, align 4
  12   │   store i32 0, i32* %retval, align 4
  13   │   %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([15 x i8], [15 x i8]* @.str, i64 0, i64 0))
  14   │   ret i32 0
  15   │ }
  16   │
  17   │ declare i32 @printf(i8*, ...) #1
  18   │
  19   │ attributes #0 = { noinline nounwind optnone ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="fals
       │ e" "min-legal-vector-width"="0" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-prot
       │ ector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+cx8,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
  20   │ attributes #1 = { "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "no-infs-fp-math"="false" "no-nans-
       │ fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+cx8,+fxsr,+mmx,+sahf,
       │ +sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
  21   │
  22   │ !llvm.module.flags = !{!0, !1, !2}
  23   │ !llvm.ident = !{!3}
  24   │
  25   │ !0 = !{i32 2, !"SDK Version", [2 x i32] [i32 10, i32 15]}
  26   │ !1 = !{i32 1, !"wchar_size", i32 4}
  27   │ !2 = !{i32 7, !"PIC Level", i32 2}
  28   │ !3 = !{!"clang version 10.0.0 (git@github.com:llvm/llvm-project.git 23fdd513a3ba7594ccdab1ea30608f4cab7faee4)"}
───────┴───────────────────────────────────────────────────────────────────────
$
```

## Success

IT LIVES!

YMMV, but this worked for me. Let me know on Twitter if you run into anything
amiss!
