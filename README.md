# Linux Test Project (core) Framework

The LTP (core) framework aims to bring all the [LTP Project] cool features and
to move them into a framework that can be used by any Linux project that is
using C as main programming language.

Here you can see a short example:

```c
#include "tst_test.h"

void setup(void) {
    // your setup code goes here

    tst_res(TINFO, "example setup");
}

void cleanup(void) {
    // your cleanup code goes here

    tst_res(TINFO, "example cleanup");
}

void run(void) {
    // your test code goes here

    tst_res(TPASS, "example test passed");
}

static struct tst_test test = {
    .test_all = run,
    .setup = setup,
    .cleanup = cleanup,
};
```

Now we just need to compile and run:

    gcc -lltp example.c -o example

    ./example
    ../lib/tst_test.c:1690: TINFO: LTP version: 20230929-148-g121b0e2ce
    ../lib/tst_test.c:1576: TINFO: Timeout per run is 0h 00m 30s
    example.c:6: TINFO: example setup
    example.c:18: TPASS: example test passed
    example.c:12: TINFO: example cleanup

    Summary:
    passed   1
    failed   0
    broken   0
    skipped  0
    warnings 0

# How to build/install/test the framework

LTP framework is using [Meson](https://mesonbuild.com/) as the main build
system. Following commands show how to use it in order to build/install/test
the LTP framework:

```sh
# prepare build directory
meson setup builddir && cd builddir

# compile library
meson compile

# compile static library
meson configure -Ddefault_library=static
meson compile

# install library
meson install

# running library tests
meson configure -Dbuild-tests=true
meson test
```

# Deprecated libraries

- libipc
- libmsgctl
- libnewipc
- sigwait
- libswap
- vdso_helpers

# Autotools to Meson conversion

When moving to meson build system, a few modifications have been made, some
variables have been dropped and some autotools function have been ignored.
And this is something you should know before compiling (for example) testcases
created with the previous LTP library.

The following paragraph provides this information.

## Unused LTP_CHECK_*

- `LTP_CHECK_ACL_SUPPORT`: used by tests
- `LTP_CHECK_CC_WARN_OLDSTYLE`: not needed
- `LTP_CHECK_CRYPTO`: used by tests
- `LTP_CHECK_KERNEL_DEVEL`: compile kernel, we don't need it
- `LTP_CHECK_LIBMNL`: used by tests
- `LTP_CHECK_LINUX_PTRACE`: only used by ptrace tests
- `LTP_CHECK_NOMMU_LINUX`: no idea how to use it
- `LTP_CHECK_SELINUX`: used by tests
- `LTP_CHECK_SYSCALL_FCNTL`: used by tests
- `LTP_CHECK_SYSCALL_NUMA`: used by tests
- `LTP_CHECK_SYSCALL_SIGNALFD`: used by tests
- `LTP_DETECT_HOST_CPU`: meson has cross-compilation mechanism

## Undefined flags

- `HAVE_LIBACL`
- `HAVE_LIBCRYPTO`
- `HAVE_LIBMNL`

## Flags to merge

- `HAVE_LIBCAP` / `HAVE_SYS_CAPABILITY_H`
- `HAVE_LIBAIO` / `HAVE_LIBAIO_H`

# TODO

- more documentation on tst_test features
- more documentation on cross-compiling
- CI configuration

[LTP Project]: https://github.com/linux-test-project/ltp
