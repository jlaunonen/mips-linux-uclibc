# mips-linux-uclibc cross-toolchain on Gentoo

This document aims to guide the reader to creating their own mips-linux-uclibc toolchain using tools provided by Gentoo linux.
The versions and configurations used here are not totally vanilla and thus need more hands-on involvement than
just running a long magic command.

Resulting (stage 3) toolchain will contain following versions of various packages (excluding Gentoo revisions):

- `gcc-6.4.0` C-compiler (different/more recent version may probably be used freely)
- `binutils-2.29.1` (different/more recent version may probably be used freely)
- `uclibc-0.9.33.2` (must mostly match TARGET uclibc version)
- `linux-headers-2.6.10` (must mostly match TARGET kernel version)

**Caveat:**
Originally the TARGET would have needed `uclibc-0.9.30` but as that does not work with recent `binutils` and
sufficiently old version (i.e. `binutils-0.20`) is not easily found, `0.9.33.2` is used here.
It may work with `0.9.30` libc on TARGET, but ymmv.

This documentation is licensed under [CC-BY-SA-4.0](LICENSE).
The `files/1-makefile.patch` is licensed under [GNU General Public License 2](LICENSE-patch).

## Requirements

- mips1 TARGET CPU. (mips1..mips4 and mips32..mips32r5 probably work.)
- Gentoo linux HOST with root shell. `sudo` works too if that is installed and preferred.
  All commands here are to be run as root unless otherwise noted.
- `sys-devel/crossdev`. Version `20171230` is used in this guide, more recent may work, ymmv.

## Assumptions

- `sys-devel/binutils` is at version `2.29.1-r1`, and `sys-libs/binutils-libs` has same ('ish) or more recent version.
- Portage contains `sys-libs/uclibc-0.9.33.2-r15`. The `0.9.33.9999` version can also be used.
- The HOST Gentoo has local portage overlay configured.
  If not, follow [Defining a custom repository](https://wiki.gentoo.org/wiki/Handbook:AMD64/Portage/CustomTree#Defining_a_custom_repository) in Gentoo handbook.



## Setting up the custom ebuilds
### linux-headers

The required `2.6.10` is not found on portage anymore, but `2.4.36` still is.
Here we use that as a base for the required `2.6.10` version.

```sh
# Prepare category, package and aux files directories.
cd /usr/local/portage
mkdir -p sys-kernel/linux-headers/files

# Copy ebuild from portage for base.
cd sys-kernel/linux-headers
cp -av /usr/portage/sys-kernel/linux-headers/linux-headers-2.4.36.ebuild ./linux-headers-2.6.10.ebuild
```

Next we need a patch file to fix the `linux-headers-2.6.10` source package.
The [1-makefile.patch](files/1-makefile.patch) (in raw form) must be copied in the previously created `files`-directory,
or more specifically, into `/usr/local/portage/sys-kernel/linux-headers/files` .
After that, the `2.6.10` ebuild must be modified of few variables so that the patch is actually applied,
and no other patches are tried, and that the package is actually supported on mips:

```
H_SUPPORTEDARCH="alpha amd64 arm m68k mips ppc sh sparc x86"

PATCHES_V=""

SRC_URI="${KERNEL_URI}"

UNIPATCH_LIST="${FILESDIR}/1-makefile.patch"
```

To finish this step, build Manifest of the ebuild, patch, and source package (which is downloaded automatically):

```sh
ebuild linux-headers-2.6.10.ebuild digest
```

**Checkup:** Ensure that running the `find`-command below results in list below (order may vary).
This may be done as regular user.

```
$ find /usr/local/portage/sys-kernel
/usr/local/portage/sys-kernel
/usr/local/portage/sys-kernel/linux-headers
/usr/local/portage/sys-kernel/linux-headers/linux-headers-2.6.10.ebuild
/usr/local/portage/sys-kernel/linux-headers/files
/usr/local/portage/sys-kernel/linux-headers/files/1-makefile.patch
/usr/local/portage/sys-kernel/linux-headers/Manifest
```



### uclibc

By the time of writing this, the `uclibc-0.9.*` ebuilds are not working correctly, due a [bug](https://bugs.gentoo.org/588554).
Fortunately, the ticket contains [a fix in comment #5](https://bugs.gentoo.org/588554#c5).
Download that attachment (named `uclibc.ebuild-001-mips.patch`) to somewhere, e.g. `/tmp`

```sh
# Prepare category and package directories.
cd /usr/local/portage
mkdir -p sys-libs/uclibc

# Copy ebuild from portage for base.
cd sys-libs/uclibc
cp -av /usr/portage/sys-libs/uclibc/uclibc-0.9.33.2-r15.ebuild .

# Apply the patch.
patch -p1 --no-backup-if-mismatch < /tmp/uclibc.ebuild-001-mips.patch

# Create The Manifest.
ebuild uclibc-0.9.33.2-r15.ebuild digest


# These commands are **optional**, only relevant if you want to use uclibc-0.9.33.9999
cp -av /usr/portage/sys-libs/uclibc/uclibc-0.9.33.9999.ebuild .
sed -e 's/0.9.33.2-r15/0.9.33.9999/;' /tmp/uclibc.ebuild-001-mips.patch > /tmp/uclibc.ebuild-001-mips-9999.patch
patch -p1 --no-backup-if-mismatch < /tmp/uclibc.ebuild-001-mips-9999.patch
ebuild uclibc-0.9.33.9999.ebuild digest
```

**Checkup:** As before, ensure stuff are in correct places.
The actual ebuilds may differ if optional steps were followed in above snippet.

```
$ find /usr/local/portage/sys-libs
/usr/local/portage/sys-libs/
/usr/local/portage/sys-libs/uclibc
/usr/local/portage/sys-libs/uclibc/uclibc-0.9.33.2-r15.ebuild
/usr/local/portage/sys-libs/uclibc/Manifest
```



## Actual building

Ebuilds prepared we can now use the *Gentoo Cross-toolchain generator* aka `crossdev` to build the toolchain.
For `gcc` we disable PIE (although this may not be required).
For `uclibc` we disable `nptl` as there seems to not be support for it in mips.
Please note of the amount of dashes, equal signs and hyphens.

```sh
# Create  mips-example-linux-uclibc  toolchain.
# You may replace the "example" vendor field to something meaningful.
# If you decided to use uclibc-0.9.33.9999, replace the
# version on --l line (still keeping the equal sign).
crossdev \
  -ol /usr/local/portage \
  -ok /usr/local/portage \
  -s3 \
  --genv 'USE=-pie' \
  --lenv 'USE=-nptl' \
  --b =2.29.1-r1 \
  --l =0.9.33.2-r15 \
  --k =2.6.10 \
  --g =6.4.0-r1 \
  --target mips-example-linux-uclibc

# You may add arguments  -P -v  to above command to see portage build output realtime.
# Otherwise, follow the logs in /var/log/portage/.
```



### Somewhat detailed description:

- `-ol` and `-ok` guide crossdev to read libc and kernel-/linux-headers ebuilds from given
  overlay (instead of Gentoo portage), as we just created thoose.
- `-s3` makes crossdev to build stage3 toolchain containing binutils (`-s0`), gcc (`-s1`), kernel headers (`-s2`) and libc (`-s3`).
- `--genv` and `--lenv` adds environment variables to gcc and libc build processes, respectively.
- `--b` declares binutils ebuild version to use.
- `--l` declares libc ebuild version to use.
- `--k` declares linux-headers ebuild version to use.
- `--g` declares gcc ebuild version to use.
- `--target mips-example-linux-uclibc` declares the toolchain name in form ARCH-VENDOR-OS-LIBC [crossdev --help].
- Optional `-P -v` adds `-v` to portage command line, giving verbose output from the build process.

The command creates a new category `cross-mips-example-linux-uclibc` in the portage for the cross-toolchain located
in the local overlay: `/usr/local/portage/cross-mips-example-linux-uclibc` .
The new sysroot resides in `/usr/mips-example-linux-uclibc` .
Files in the installed packages can be seen with `equery files` as usual, but the cross-category should be included
in the requested name, e.g. `equery files cross-mips-example-linux-uclibc/gcc`
(`equery` tool belongs to `app-portage/gentoolkit` package).



## Testing time!

Write into `hello.c`:

```c
#include <stdio.h>
int main(void) {
  printf("Hello World, MIPS!\n");
  return 0;
}
```

Then run the newly created compiler on the file:

```sh
mips-example-linux-uclibc-gcc -o hello hello.c
# or for static result:
mips-example-linux-uclibc-gcc -o hello -static hello.c
```

Then copy the resulting ELF binary to the MIPS machine and run it.



## Removing the toolchain (totally optional)

To remove the installed toolchain parts, the toolchain portage package category and configuration files related to
the toolchain, run following:

```sh
crossdev --clean mips-example-linux-uclibc-gcc
# Previous command leaves two binaries behind (on crossdev-20171230). Remove them manually:
rm /usr/bin/mips-example-linux-uclibc-gcov-{dump,tool}
```

This does not remove the self-created ebuilds in overlay, allowing to continue from [Actual building](#actual-building) later if needed.
