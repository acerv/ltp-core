# Linux Test Project (core)

LTP is a lightweight and versatile testing framework based on [LTP library] and
designed specifically for C applications on the Linux platform. With an emphasis
on simplicity and ease of use, LTP provides developers with a powerful toolset
for creating and executing test cases, both for Kernel and regular C
applications.

Here you can see a short example:

```c
#include <tst_test.h>

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

	tst_res(TPASS, "Doing hardly anything is easy");
}

static struct tst_test test = {
	.test_all = run,
	.setup = setup,
	.cleanup = cleanup,
};
```

Now we just need to compile and run:

```sh
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
```

Each test has some default options which are linked into the test. They can be
seen with `./example -h` command.

# How to build/install/test the framework

LTP framework is using [Meson](https://mesonbuild.com/) as the main build
system. Following commands show how to use it in order to build/install/test
the LTP framework:

```sh
# prepare build directory
meson setup builddir && cd builddir

# compile library
meson compile

# compile shared library
meson configure -Ddefault_library=shared
meson compile

# install library
meson install

# running library tests
meson configure -Dbuild_tests=true
meson test
```

# Features

Following paragraph shows available features in the LTP core framework.

## Show messages

Inside the framework we have many ways to send messages without stopping the
test. All of them can be sent using `tst_res` function with the following flags:

- `TINFO`: information message
- `TWARN`: warning message
- `TPASS`: PASS message
- `TFAIL`: FAIL message

When `TERRNO` is used in combination with one of the flags, framework will
append message with `errno` at the end.

All tests have to send **at least** a `TPASS` or `TFAIL` message before the end
of test execution. An error will be shown as remainder otherwise.

```c
static void run(void)
{
	tst_res(TINFO, "This is an information message");
	tst_res(TWARN, "This is a warning message");
	tst_res(TPASS, "This is a pass message");
	tst_res(TFAIL, "This is a fail message");

	// TERRNO can be used by all flags above
	tst_res(TINFO | TERRNO, "This is a message with errno at the end");
}
```

## Raise a breaking error

Any time we need to stop the test because something broke during the execution,
we can use `tst_brk` function to send a broken error via `TBROK` flag.
The test will immediately stop.

When `TERRNO` is used in combination with `TBROK`, the framework will print
`errno` that caused the failure.

```c
static void run(void)
{
	// if something is broken raise an error and stop the test

	tst_brk(TBROK, "Something is broken. Stop the test.");
	tst_brk(TBROK | TERRNO, "Something is broken. Print errno");
}
```

## Raise a configuration error

Each time we want to end the test because it's not satisfying our needs, we can
use `tst_brk` function to send a configuration error via `TCONF` flag.
Test will immediately stop.

```c
static void setup(void)
{
	// my requirements are not satisfied, so stop the test with message

	tst_brk(TCONF, "Configuration problem. Stop the test.");
}

static struct tst_test test = {
	.setup = setup,
	.test_all = run,
};
```

## Test results using TST_* macros

The framework provides a few macros that can be used to automatically check if
a specific function has passed or failed. These macros take a function call as
the first parameter and a printf-like format string and parameters as well.
These test macros then expand to a code that runs the call, checks the return
value and `errno` and reports the test result.

```c
static void run(void)
{
	...
	TST_EXP_PASS(stat(fname, &statbuf), "stat(%s, ...)", fname);

	if (!TST_PASS)
		return;
	...
}
```

The `TST_EXP_PASS()` can be used for calls that return -1 on failure and 0 on
success. It will check for the return value and reports failure if the return
value is not equal to 0. The call also sets the `TST_PASS` variable to 1 if
the call succeeeded.

As seen above, this and similar macros take optional variadic arguments. These
begin with a format string and then appropriate values to be formatted.

```c
static void run(void)
{
	...
	TST_EXP_FD(open(fname, O_RDONLY), "open(%s, O_RDONLY)", fname);

	SAFE_CLOSE(TST_RET);
	...
}
```

The `TST_EXP_FD()` is the same as `TST_EXP_PASS()` the only difference is that
the return value is expected to be a file descriptor so the call passes if
positive integer is returned.

```c
static void run(void)
{
	...
	TST_EXP_FAIL(stat(fname, &statbuf), ENOENT, "stat(%s, ...)", fname);
	...
}
```

The `TST_EXP_FAIL()` is similar to `TST_EXP_PASS()` but it fails the test if
the call haven't failed with -1 and `errno` wasn't set to the expected one
passed as the second argument.

```c
static void run(void)
{
	...
	TST_EXP_FAIL2(msgget(key, flags), EINVAL, "msgget(%i, %i)", key, flags);
	...
}
```

The `TST_EXP_FAIL2()` is the same as `TST_EXP_FAIL()` except the return value is
expected to be non-negative integer if call passes. 

`TST_EXP_FAIL_SILENT()` and `TST_EXP_FAIL2_SILENT()` variants are less verbose
and do not print `TPASS` messages when SCALL fails as expected.

```c
TEST(socket(AF_INET, SOCK_RAW, 1));
if (TST_RET > -1) {
	tst_res(TFAIL, "Created raw socket");
	SAFE_CLOSE(TST_RET);
} else if (TST_ERR != EPERM) {
	tst_res(TFAIL | TTERRNO,
		"Failed to create socket for wrong reason");
} else {
	tst_res(TPASS | TTERRNO, "Didn't create raw socket");
}
```

The `TST_*` macro sets `TST_RET` to its argument's return value and `TST_ERR` to
`errno`+ The `TTERNO` flag can be used to print the error number's symbolic
value.

No library function or macro, except those in `tst_test_macros.h`, will
write to these variables. So their values will not be changed unexpectedly.

```c
TST_EXP_POSITIVE(wait(&status));

if (!TST_PASS)
	return;
```

If the return value of `wait` is positive or zero, this macro will print a pass
result and set `TST_PASS` appropriately. If the return value is negative, then
it will print fail.  There are many similar macros to those shown here, please
see `tst_test_macros.h`.

```c
TST_EXP_EQ_LI(val1, val2);
TST_EXP_EQ_UI(val1, val2);
TST_EXP_EQ_SZ(val1, val2);
TST_EXP_EQ_SSZ(val1, val2);

/* Use as */
TST_EXP_EQ_LI(sig_caught, SIGCHLD);
```

Set of macros for different integer type comparsions. These macros print the
variable names as well as values in both pass and fail scenarios.

## Utilities functions

Some of the following utilities can be used to obtain information during tests
execution.

```c
const char *tst_strsig(int sig);
```

Return the given signal number's corresponding string.

```c
const char *tst_strerrno(int err);
```

Return the given errno number's corresponding string. Using this function to
translate `errno` values to strings is preferred. You should not use the
`strerror()` function in the testcases.

```c
const char *tst_strstatus(int status);
```

Returns string describing the status as returned by `wait()`. This function is
not thread safe.

```c
void tst_set_max_runtime(int max_runtime);
```

Allows for setting max_runtime per test iteration dynamically in the test 'setup()',
the timeout is specified in seconds. There are a few testcases whose runtime
can vary arbitrarily, these can disable timeouts by setting it to
TST_UNLIMITED_RUNTIME.

```c
void tst_flush(void);
```

Flush output streams, handling errors appropriately.

This function is rarely needed when you have to flush the output streams
before calling `fork()` or `clone()`. Note that the `SAFE_FORK()` and 
`SAFE_CLONE()` calls this function automatically. See 2.4 FILE buffers and
`fork()` for explanation why is this needed.

## Using LTP API without defining a test

Sometimes we need to define a simple binary, without linking to the `tst_test`
struct definition, but using the framework functionalities, such as macros,
broken errors, etc. This is common in such cases where we need to call an
external binary that has to perform specific operations and we don't want to
waste time redefining features implemented inside the framework. To do so,
`TST_NO_DEFAULT_MAIN` can be used.

```c
#define TST_NO_DEFAULT_MAIN
#include "tst_test.h"

int main(void)
{
	tst_res(TPASS, "Child passed!");
	return 0;
}
```

## Skip test using preprocessor

In some cases we need to skip the entire test if system does not satisfy
our needs. It can be the case of a test that requires to be run on x86. In this
case, `TST_TEST_TCONF` macro can be used.

```c
#include <tst_test.h>

#if defined(__i386__) || defined(__x86_64__)

static void run(void)
{
	// my test code
}

static struct tst_test test = {
	.test_all = run,
};

#else
TST_TEST_TCONF("Test supported only on x86");
#endif
```

## Customize test options

Test options can be customized assigning `.options` argument to the `tst_test`
struct definition.

```c
static struct tst_test test = {
	.test_all = run,
	.options = (struct tst_option[]) {
		{"v", &verbose, "Verbose output"},
		{"s", &size, "Size of the file to generate"},
		{},
	},
};
```

## Run test inside a temporary folder

When we need to create files or directories, we can use a temporary folder so
we won't generate data in the current directory.
This can be achieved setting `.needs_tmpdir` flag.
The framework will take care to create a temporary directory before test, to
move inside it and to delete it once test is completed.

```c
static struct tst_test test = {
	.test_all = run,
	.needs_tmpdir = 1,
};
```

## Root requirement

Some operations inside test might require root. To ensure that we are running
the test as root user, we can set `.needs_root`.
If test will be executed as normal user, the framework will send a configuration
error and stop the test.

```c
static struct tst_test test = {
	.test_all = run,
	.needs_root = 1,
};
```

## Maximum time

If we want that our test never exceed a specific execution time, we can set
`.max_runtime` flag in seconds. At the end of this time, test will be killed
by the framework raising a `TBROK` error.

```c
static struct tst_test test = {
	.test_all = run,
	// 1 hour max execution
	.max_runtime = 3600,
};
```

## Forked children

Some tests require to spawn one or multiple children using `SAFE_FORK()` macro.
The framework automatically handles spawned children inside the test, ensuring
that all of them will be completed before the end of the test. This feature can
be activated setting `.forks_child` flag.
When `SAFE_FORK()` is used, the framework will remind that the flag has to be
set.

```c
static struct tst_test test = {
	.test_all = run,
	.forks_child = 1,
};
```

## Checkpoint handling

When using `TST_CHECKPOINT_*` macros, `.needs_checkpoints` flag has to be set.
In this way, futex will be released just before ending of the test.
When `TST_CHECKPOINT_` macros are used, the framework will remind that the
flag has to be set.

```c
static struct tst_test test = {
	.test_all = run,
	.needs_checkpoints = 1,
};
```

## Using a device

If `.needs_device` flag is set, the `tst_device` structure is initialized with
path the test device and default filesystem to be used.

```c
static void run(void)
{
	tst_res(TINFO, "My device is %s", tst_device->dev);

	// my test code using device
}

static struct tst_test test = {
	.test_all = run,
	.needs_device = 1,
};
```

## Create and mount a device

If `.mount_device` is set, the device is mounted at `.mntpoint` which is used
to pass a directory name that will be created and used as mount destination.
You can pass additional flags and data to the mount command via `.mnt_flags`
and `.mnt_data` pointers. Note that `.mount_device` implies `.needs_device`
and `.format_device` so there is no need to set the later two. To pass
additional options to `mkfs.$fs` utility, `.dev_fs_type` and `dev_extra_opts`
can be used.
Device filesystem can be defined by `.dev_fs_type`.
If `.dev_min_size` is set, framework will check that device has that minimum
size in megabytes.

```c
static void run(void)
{
	tst_res(TINFO,
		"My device %s is formatted with %s filesystem",
		tst_device->dev,
		tst_device->fs_type);

	// my test code using tst_device
}

static struct tst_test test = {
	.test_all = run,
	.needs_device = 1,
	.mount_device = 1,
	.format_device = 1,
	.dev_min_size = 2, // reserve 2MB device
	.dev_fs_type = "xfs",
	.mntpoint = "mntpoint",
	.mnt_data = "usrquota",
	.mnt_flags = MS_STRICTATIME,
};
```

**IMPORTANT**: Close all file descriptors (that point to files in test temporary
directory, even the unlinked ones) either in the 'test()' function or in the
test 'cleanup()' otherwise the test may break temporary directory removal on NFS
(look for "NFS silly rename").

### Overlay filesystem

If `.needs_overlay` is set, mount point will use an overlay fs.

```c
static struct tst_test test = {
	.test_all = run,
	.needs_device = 1,
	.mount_device = 1,
	.format_device = 1,
	.mntpoint = "mntpoint",
	.needs_overlay = 1,
};
```

### Read-only filesystem

If `.needs_rofs` is set, read-only filesystem is mounted at `.mntpoint`. This
one is supposed to be used for `EROFS` tests.

```c
static struct tst_test test = {
	.test_all = run,
	.needs_device = 1,
	.mount_device = 1,
	.format_device = 1,
	.mntpoint = "mntpoint",
	.needs_rofs = 1,
};
```

### Mount hugepages

If `.needs_hugelbfs` is set, the hugetlbfs will be mounted at `.mntpoint`.

```c
static struct tst_test test = {
	.test_all = run,
	.needs_device = 1,
	.mount_device = 1,
	.format_device = 1,
	.mntpoint = "mntpoint",
	.needs_hugelbfs = 1,
};
```

## Execute for all filesystems

When the `.all_filesystems` flag is set the `.skip_filesystems` list is passed
to the function that detects supported filesystems any listed filesystem is
not included in the resulting list of supported filesystems.

If test needs to adjust expectations based on filesystem type it's also
possible to detect filesystem type at the runtime. This is preferably used
when only subset of the test is not applicable for a given filesystem.

```c
static struct tst_test test = {
	.test_all = run,
	.all_filesystems = 1,
	.skip_filesystems = (const char *const []) {
			"exfat",
			"tmpfs",
			"ramfs",
			"nfs",
			"vfat",
			NULL
	},
};
```

**NOTE** that ext2, ext3 or ext4 in `.skip_filesystems` on tests which does
**not** use `.all_filesystems` needs to be defined as 'ext2/ext3/ext4'. The
reason is that it is hard to detect used filesystem due to overlapping the
functionality.

OTOH tests which use `.skip_filesystems` **with** `.all_filesystems` can skip
only filesystems which are actually used in `.all_filesystems`: ext2, ext3,
ext4, xfs, btrfs, vfat, exfat, ntfs, tmpfs (defined in `fs_type_whitelist[]`).
It does not make sense to list other filesystems.

## Minimum supported kernel

To avoid using kernels which are too old for the test, we can use `.min_kver`
attribute in the `tst_test` struct definition. If minimum kernel is not
satisfied, a configuration error will be shown.

```c
static struct tst_test test = {
	.test_all = run,
	.min_kver = "4.11",
};
```

## Supported architectures

Test's supported architectures can be defined using the `.supported_archs` flag
inside `tst_test` struct definition. If current architecture is not supported,
a configuration error will be shown.

```c
static struct tst_test test = {
	.test_all = run,
	.supported_archs = (const char *const []) {
		"x86_64",
		"x86",
		NULL
	}
}
```

## Skip tests based on system state

Test can be skipped on various conditions:

- if `.skip_in_secureboot` is set, on enabled SecureBoot
- if `.skip_in_lockdown` is set, on lockdown
- if `.skip_in_compat` is set, in 32-bit compat mode

```c
static struct tst_test test = {
	.test_all = run,
	.skip_in_secureboot = 1,
	.skip_in_lockdown = 1,
	.skip_in_compat = 1,
}
```

## Require minimum numbers of CPU

Some tests require more than specific number of CPU. It can be defined with
`.min_cpus`.

```c
static struct tst_test test = {
	.test_all = run,
	.min_cpus = 2,
}
```

## Require minimum memory or swap size

Some tests require a minimum amount of memory or swap.
To make sure that test will run only on systems with more than minimal required
amount of RAM set `.min_mem_avail`. Similarily for tests that require certain
amount of free Swap use `.min_swap_avail`. Both flags are expressed in MB.

```c
static struct tst_test test = {
	.test_all = run,
	// check that minimum memory is 20 MB
	.min_mem_avail = 20,
	// check that minimum swap is 2 MB
	.min_swap_avail = 2,
}
```

## Reserving hugepages

Many of the LTP tests need to use hugepage in their testing, this allows the
test can reserve hugepages from system via `.hugepages = {xx, TST_REQUEST}`.

We achieved two policies for reserving hugepages:

- `TST_REQUEST`: It will try the best to reserve available huge pages and return
  the number of available hugepages in `tst_hugepages`, which may be 0 if
  hugepages are not supported at all.

- `TST_NEEDS`: This is an enforced requirement, LTP should strictly do hpages
  applying and guarantee the **HugePages_Free** no less than pages which makes
  that test can use these specified numbers correctly. Otherwise, test exits
  with `TCONF` if the attempt to reserve hugepages fails or reserves less than
  requested.

With success test stores the reserved hugepage number in `tst_hugepages`. For
system without hugetlb supporting, variable `tst_hugepages` will be set to 0.
If the hugepage number needs to be set to 0 on supported hugetlb system, please
use `.hugepages = {TST_NO_HUGEPAGES}`.

Also, we do cleanup and restore work for the hpages resetting automatically.

```c
static void run(void)
{
	...

	if (tst_hugepages == test.hugepages.number)
		TEST(do_hpage_test);
	else
		...
	...
}

struct tst_test test = {
	.test_all = run,
	.hugepages = {2, TST_REQUEST},
};
```

## Tainted kernels

In some cases we need to detect whether a testcase triggers a kernel warning,
bug or oops. The `.taint_check` flag can be used to detect them.

List of supported taint flags:

- `TST_TAINT_G`: a module with non-GPL license loaded
- `TST_TAINT_F`: a module was force-loaded
- `TST_TAINT_S`: SMP with Non-SMP kernel
- `TST_TAINT_R`: module force unloaded
- `TST_TAINT_M`: machine check error occurred
- `TST_TAINT_B`: page-release function found bad page
- `TST_TAINT_U`: user requested taint flag
- `TST_TAINT_D`: kernel died recently - OOPS or BUG
- `TST_TAINT_A`: ACPI table has been overwritten
- `TST_TAINT_W`: a warning has been issued by kernel
- `TST_TAINT_C`: driver from drivers/staging was loaded
- `TST_TAINT_I`: working around BIOS/Firmware bug
- `TST_TAINT_O`: out of tree module loaded
- `TST_TAINT_E`: unsigned module was loaded
- `TST_TAINT_L`: A soft lock-up has previously occurred
- `TST_TAINT_K`: kernel has been live-patched
- `TST_TAINT_X`: auxiliary taint, for distro's use
- `TST_TAINT_T`: kernel was built with the struct randomization plugin

```c
void run(void)
{
	...
	if (tst_taint_check() != 0)
		tst_res(TFAIL, "kernel has issues");
	else
		tst_res(TPASS, "kernel seems to be fine");
}

static struct tst_test test = {
	.test_all = run,
	.tint_check = TST_TAINT_W | TST_TAINT_D,
}
```

## Testing similar syscalls in one test

In some cases kernel has several very similar syscalls that do either the same
or very similar job. This is most noticeable on i386 where we commonly have
two or three syscall versions. That is because i386 was first platform that
Linux was developed on and because of that most mistakes in API happened there
as well. However this is not limited to i386 at all, it's quite common that
version two syscall has added missing flags parameters or so.

In such cases it does not make much sense to copy&paste the test code over and
over, rather than that the test library provides support for test variants.
The idea behind test variants is simple, we run the test several times each
time with different syscall variant.

The implementation consist of `test_variants` integer that, if set, denotes
number of test variants. The test is then forked and executed `test_variants`
times each time with different value in global `tst_variant` variable.

```c
static int do_foo(void)
{
	switch (tst_variant) {
	case 0:
		return foo();
	case 1:
		return syscall(__NR_foo);
	}

	return -1;
}

static void run(void)
{
	TEST(do_foo);
}

static void setup(void)
{
	switch (tst_variant) {
	case 0:
		tst_res(TINFO, "Testing foo variant 1");
	break;
	case 1:
		tst_res(TINFO, "Testing foo variant 2");
	break;
	}
}

struct tst_test test = {
	.setup = setup,
	.test_all = run,
	.test_variants = 2,
};
```

## Checking kernel for the driver support

Some tests may need specific kernel drivers, either compiled in, or built
as a module. If `.needs_drivers` points to a `NULL` terminated array of kernel
module names these are all checked and the test exits with `TCONF` on the
first missing driver.

The detection is based on reading `modules.dep` and `modules.builtin` files
generated by kmod. The check is skipped on Android.

```c
static struct tst_test test = {
	.test_all = run,
	.needs_drivers = (const char *const []) {
		"zram",
		NULL
	},
};
```

## Saving & restoring /proc|sys values

Framework can be instructed to save and restore value of specified (/proc|sys)
files. This is achieved by initialized tst_test struct field 'save_restore'.
It is a NULL-terminated array of struct `tst_path_val` where each
`tst_path_val.path` represents a file, whose value is saved at the beginning and
restored at the end of the test.

If non-NULL string is passed in `tst_path_val.val`, it is written to the
respective file at the beginning of the test. Only the first line of a specified
file is saved and restored.

By default, the test will end with `TCONF` if the file is read-only or does not
exist. If the optional write of new value fails, the test will end with `TBROK`.
This behavior can be changed using `tst_path_val.flags`:

- `TST_SR_TBROK_MISSING`: End test with `TBROK` if the file does not exist
- `TST_SR_TCONF_MISSING`: End test with `TCONF` if the file does not exist
- `TST_SR_SKIP_MISSING`: Continue without saving the file if it does not exist
- `TST_SR_TBROK_RO`: End test with `TBROK` if the file is read-only
- `TST_SR_TCONF_RO`: End test with `TCONF` if the file is read-only
- `TST_SR_SKIP_RO`: Continue without saving the file if it is read-only
- `TST_SR_IGNORE_ERR`: Ignore errors when writing new value into the file

Common flag combinations also have shortcuts:

- `TST_SR_TCONF`: Equivalent to `TST_SR_TCONF_MISSING | TST_SR_TCONF_RO`
- `TST_SR_TBROK`: Equivalent to `TST_SR_TBROK_MISSING | TST_SR_TBROK_RO`
- `TST_SR_SKIP`: Equivalent to `TST_SR_SKIP_MISSING | TST_SR_SKIP_RO`

`restore` is always strict and will `TWARN` if it encounters any error.

```c
static struct tst_test test = {
	...
	.setup = setup,
	.save_restore = (const struct tst_path_val[]) {
		{"/proc/sys/kernel/core_pattern", NULL, TST_SR_TCONF},
		{"/proc/sys/user/max_user_namespaces", NULL, TST_SR_SKIP},
		{"/sys/kernel/mm/ksm/run", "1", TST_SR_TBROK},
		{}
	},
};
```

## Checking for kernel configuration

Generally testcases should attempt to autodetect as much kernel features as
possible based on the currently running kernel. We do have `tst_check_driver()`
to check if functionality that could be compiled as kernel module is present
on the system, disabled syscalls can be detected by checking for `ENOSYS`
errno etc.

However, in rare cases core kernel features couldn't be detected based on the
kernel userspace API and we have to resort to parse the kernel `.config`.

For this cases the test should set the NULL-terminated `.needs_kconfigs`
array of boolean expressions with constraints on the **kconfig** variables. The
boolean expression consits of variables, two binary operations `&` and `|`,
negation `!` and correct sequence of parentesis `()`. Variables are expected
to be in a form of `CONFIG_FOO[=bar]`.

The test will continue to run if all expressions are evaluated to `True`.
Missing variable is mapped to `False` as well as variable with different than
specified value, e.g. `CONFIG_FOO=bar` will evaluate to `False` if the value
is anything else but `bar`. If config variable is specified as plain
`CONFIG_FOO`, it's evaluated to true it's set to any value (typically =y or =m).

```c
static struct tst_test test = {
	.test_all = run,
	.needs_kconfigs = (const char *[]) {
		"CONFIG_VETH",
		"CONFIG_USER_NS=y",
		"CONFIG_NET_NS=y",
		"CONFIG_NET_SCH_HTB",
		"CONFIG_NET_CLS_TCINDEX",
		NULL
	},
};
```

## Guarded buffers

The test library supports guarded buffers, which are buffers allocated so
that:

- The end of the buffer is followed by a `PROT_NONE` page
- The remainder of the page before the buffer is filled with random canary data

Which means that the any access after the buffer will yield a `Segmentation
fault` or `EFAULT` depending on if the access happened in userspace or the kernel
respectively. The canary before the buffer will also catch any write access
outside of the buffer.

The purpose of the feature is to catch off-by-one bugs which happens when
buffers and structures are passed to syscalls. New tests should allocate
guarded buffers for all data passed to the tested syscall which are passed by
a pointer.

```c
static struct timex *buf;

static void run(void)
{
	// normally use buf as it is
}

static struct tst_test test = {
	.test_all = run,
	.bufs = (struct tst_buffers []) {
		{&buf, .size = sizeof(*buf)},
		{},
	},
};
```

Guarded buffers can be allocated on runtime in a test `setup()` by a
`tst_alloc()` or by `tst_strdup()` as well as by filling up the `.bufs` array in
the `tst_test` struct.

So far the `tst_test` struct supports allocating either a plain buffer by
setting up the size or `struct iovec`, which is allocated recursively including
the individual buffers as described by an `-1` terminated array of buffer
sizes.

## Adding and removing capabilities

Some tests may require the presence or absence of particular
capabilities. Using the API provided by `tst_capability.h` the test author can
try to ensure that some capabilities are either present or absent during the
test.

For example; below we try to create a raw socket, which requires
`CAP_NET_ADMIN`. During setup we should be able to do it, then during run it
should be impossible. The LTP capability library will check before setup that
we have this capability, then after setup it will drop it.

```c
#include "tst_test.h"
#include "tst_capability.h"
#include "tst_safe_net.h"
#include "lapi/socket.h"

static void run(void)
{
	TEST(socket(AF_INET, SOCK_RAW, 1));
	if (TST_RET > -1) {
		tst_res(TFAIL, "Created raw socket");
	} else if (TST_ERR != EPERM) {
		tst_res(TFAIL | TTERRNO,
			"Failed to create socket for wrong reason");
	} else {
		tst_res(TPASS | TTERRNO, "Didn't create raw socket");
	}
}

static void setup(void)
{
	TEST(socket(AF_INET, SOCK_RAW, 1));
	if (TST_RET < 0)
		tst_brk(TCONF | TTERRNO, "We don't have CAP_NET_RAW to begin with");

	SAFE_CLOSE(TST_RET);
}

static struct tst_test test = {
	.setup = setup,
	.test_all = run,
	.caps = (struct tst_cap []) {
		TST_CAP(TST_CAP_REQ, CAP_NET_RAW),
		TST_CAP(TST_CAP_DROP, CAP_NET_RAW),
		{}
	},
};
```

## Checking for required commands

Required commands can be checked with `.needs_cmds`, which points to a `NULL`
terminated array of strings such as:

```c
.needs_cmds = (const char *const []) {
	"useradd",
	"userdel",
	NULL
},
```

Also can check required command version whether is satisfied by using
`needs_cmds` such as:

```c
.needs_cmds = (const char *const []) {
	"mkfs.ext4 >= 1.43.0",
	NULL
},
```

## Test tags

Test tags are name-value pairs that can hold any test metadata.

We have additional support for CVE entries, git commit in mainline kernel,
stable kernel or glibc git repository.  If a test is a regression test it
should include these tags.  They are printed when test fails.

CVE, mainline and stable kernel git commits in a regression test for a
kernel bug:

```c
struct tst_test test = {
	.test_all = run,
	.tags = (const struct tst_tag[]) {
		{"linux-git", "9392a27d88b9"},
		{"linux-git", "ff002b30181d"},
		{"known-fail", "ustat() is known to fail with EINVAL on Btrfs"},
		{"linux-stable-git", "c4a23c852e80"},
		{"CVE", "2020-29373"},
		{}
	}
};
```

Glibc and musl git commits in a regression test for `glibc` and `musl` bugs:

```c
struct tst_test test = {
	.test_all = run,
	.tags = (const struct tst_tag[]) {
		{"glibc-git", "574500a108be"},
		{"musl-git", "fa4a8abd06a4"},
		{}
	}
};
```

# From LTP to LTP core

The following paragraph will show what's changed between LTP and LTP core.

## Deprecated libraries

- libipc
- libmsgctl
- libnewipc
- sigwait
- libswap
- vdso_helpers

## Autotools to Meson conversion

When moving to meson build system, a few modifications have been made, some
variables have been dropped and some autotools function have been ignored.
And this is something you should know before compiling (for example) testcases
created with the previous LTP library.

### Unused LTP_CHECK_*

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

### Undefined flags

- `HAVE_LIBACL`
- `HAVE_LIBCRYPTO`
- `HAVE_LIBMNL`

### Flags to merge

- `HAVE_LIBCAP` / `HAVE_SYS_CAPABILITY_H`
- `HAVE_LIBAIO` / `HAVE_LIBAIO_H`

## TODO

- remove old API
- more documentation on tst_test features
- more documentation on cross-compiling
- CI configuration

[LTP library]: https://github.com/linux-test-project/ltp
