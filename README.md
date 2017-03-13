# Dapper-Kernel-Grsec

## Dapper Linux Grsecurity Hardened Kernel with Fedora Patches

### Introduction

This repository contains the current Linux kernel used by Dapper Linux.
The build process is heavily based on the Fedora Linux kernel build process, and should be familliar to those who build their own kernels. 

### Current supported versions by Dapper Linux:

| Dapper Linux | Linux Version | Grsecurity Patch        |
| ------------ | ------------- | ----------------------- |
| 25           | 4.9.14        | 3.1-4.9.14-201703121245 |


### Packaging and Building a Source RPM for COPR

This section should serve as a step by step guide as to how Dapper Linux readies kernels for building.
This is an updated version of a guide found on [thiébaud.fr](http://thiébaud.fr/fedora_grsec.html), which was extremly useful.

We need to install a RPM development toolchain:

```bash
$ sudo dnf group install c-development
$ sudo dnf install rpmdevtools yum-utils gcc-plugin-devel
$ rpmdev-setuptree
```

The rpm-setuptree will create a rpmbuild directory in your $HOME folder. If you'd rather use another path, you can put %_topdir %{getenv:HOME}/my_path in ~/.rpmmacros.

Next, we get the current kernel source RPM, install it to the rpmbuild dir and fetch the kernel's build dependancies

```bash
$ yumdownloader --source kernel
$ rpm -Uvh kernel-4.9.8-201.fc25.src.rpm
$ sudo dnf builddep kernel
$ sudo dnf install numactl-devel pesign
```

Now we fetch the test patch from [grsecurity](https://grsecurity.net/download.php) and place it in the SOURCES directory.

```bash
$ cd ~/rpmbuild/SOURCES
$ wget https://grsecurity.net/test/grsecurity-3.1-4.9.8-201702071801.patch
$ wget https://grsecurity.net/test/grsecurity-3.1-4.9.8-201702071801.patch.sig
```

Now we verify the signiture of the patch (you might have to [import](http://en.wikibooks.org/wiki/Grsecurity/Obtaining_grsecurity#Verifying_the_Downloads) the signing key first). Ensure the signature is good.

```bash
$ gpg --verify grsecurity-3.1-4.9.8-201702071801.patch.sig
gpg: assuming signed data in `grsecurity-3.1-4.9.8-201702071801.patch'
gpg: Signature made Wed 08 Feb 2017 12:05:31 NZDT using RSA key ID 2525FE49
gpg: Good signature from "Bradley Spengler (spender) <spender@grsecurity.net>"
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: DE94 52CE 46F4 2094 907F  108B 44D1 C0F8 2525 FE49
```

Now, add the grsecurity patch to the kernel.spec file. In the SPECS directory, edit kernel.spec and change

```spec
#define buildid .local
```

to:

```spec
%define buildid .grsec
```

Since Dapper Linux is only interested in supporting x86_64 at this point in time, remove the other architectures by adding to the nobuild arches flag:

Change

```spec
%define nobuildarches i386 s390
```

to:

```spec
# We only build kernel-headers on the following...
%define nobuildarches i386 s390 ppc64 ppc64p7 s390 s390x %{arm} aarch64 ppc64le
```

We also do not want particular packages to be built, since it saves a lot of time and effort since they will never be used. So we will be disabling the debug, pref, tools and debuginfo packages.

Change

```spec
# kernel-debug
%define with_debug     %{?_without_debug:     0} %{?!_without_debug:     1}
# kernel-headers
%define with_headers   %{?_without_headers:   0} %{?!_without_headers:   1}
%define with_cross_headers   %{?_without_cross_headers:   0} %{?!_without_cross_headers:   1}
# perf
%define with_perf      %{?_without_perf:      0} %{?!_without_perf:      1}
# tools
%define with_tools     %{?_without_tools:     0} %{?!_without_tools:     1}
# kernel-debuginfo
%define with_debuginfo %{?_without_debuginfo: 0} %{?!_without_debuginfo: 1}
```

to

```spec
# kernel-debug
%define with_debug     %{?_without_debug:     0} %{?!_without_debug:     0}
# kernel-headers
%define with_headers   %{?_without_headers:   0} %{?!_without_headers:   1}
%define with_cross_headers   %{?_without_cross_headers:   0} %{?!_without_cross_headers:   1}
# perf
%define with_perf      %{?_without_perf:      0} %{?!_without_perf:      0}
# tools
%define with_tools     %{?_without_tools:     0} %{?!_without_tools:     0}
# kernel-debuginfo
%define with_debuginfo %{?_without_debuginfo: 0} %{?!_without_debuginfo: 0}
```

And note that we do wish to build headers for cross compilation compatibility.

Now we need to add the patch. So before:

```spec
# END OF PATCH DEFINITIONS
```

add:

```spec
Patch26000: grsecurity-3.1-4.9.8-201702071801.patch
```

Then try and apply the patch. Note the -bp flag on rpmbuild will run the %prep section of the .spec file, which does the uncompressing and patching.

```bash
$ cd ~/rpmbuild/SPECS
$ rpmbuild -bp kernel.spec
[...]
error: patch failed: arch/x86/entry/vdso/Makefile:170
error: arch/x86/entry/vdso/Makefile: patch does not apply
error: patch failed: arch/x86/kernel/ioport.c:30
error: arch/x86/kernel/ioport.c: patch does not apply
error: patch failed: drivers/acpi/custom_method.c:29
error: drivers/acpi/custom_method.c: patch does not apply
error: patch failed: drivers/platform/x86/asus-wmi.c:1872
error: drivers/platform/x86/asus-wmi.c: patch does not apply
error: patch failed: init/Kconfig:1168
error: init/Kconfig: patch does not apply
Patch failed at 0087 
[...]
```

It is completly normal to fail at this stage. Most of these patches will fail because Fedora ship a patch that may already exist in grsecurity's patchset, causing a collision. Or a particular patch may have conflicitng changes with what is found in grsecurity. We can find the reasons behind failure by running a quick grep over the SOURCES diretory over the offending files.

```bash
$ cd ~/rpmbuild/SOURCES
$ grep -Rin ioport\.c .
x86-Lock-down-IO-port-access-when-module-security-is.patch:13: arch/x86/kernel/ioport.c | 5 +++--
x86-Lock-down-IO-port-access-when-module-security-is.patch:17:diff --git a/arch/x86/kernel/ioport.c b/arch/x86/kernel/ioport.c
x86-Lock-down-IO-port-access-when-module-security-is.patch:19:--- a/arch/x86/kernel/ioport.c
x86-Lock-down-IO-port-access-when-module-security-is.patch:20:+++ b/arch/x86/kernel/ioport.c
grsecurity-3.1-4.9.8-201702071801.patch:31446:diff --git a/arch/x86/kernel/ioport.c b/arch/x86/kernel/ioport.c
grsecurity-3.1-4.9.8-201702071801.patch:31448:--- a/arch/x86/kernel/ioport.c
grsecurity-3.1-4.9.8-201702071801.patch:31449:+++ b/arch/x86/kernel/ioport.c
```

We can see the x86-Lock-down-IO-port-access-when-module-security-is.patch is causing problems, so we can comment it out in the kernel.spec file. 

```spec
# Fails to patch with grsecurity
#Patch475: x86-Lock-down-IO-port-access-when-module-security-is.patch
```

You can continue to find all of the other collisions. Here is the list that Dapper Linux comments out (as of Linux 4.9.8)

```spec
# Fails to patch with grsecurity
#Patch475: x86-Lock-down-IO-port-access-when-module-security-is.patch
#Patch476: ACPI-Limit-access-to-custom_method.patch
#Patch477: asus-wmi-Restrict-debugfs-interface-when-module-load.patch
#Patch478: Restrict-dev-mem-and-dev-kmem-when-module-loading-is.patch
#Patch486: hibernate-Disable-in-a-signed-modules-environment.patch
#Patch494: disable-i8042-check-on-apple-mac.patch
```

Now when we run the patch command we just find:

```bash
$ rpmbuild -bp kernel.spec
[...]
error: patch failed: arch/x86/entry/vdso/Makefile:170
error: arch/x86/entry/vdso/Makefile: patch does not apply
error: patch failed: init/Kconfig:1168
error: init/Kconfig: patch does not apply
Patch failed at 0081 
[...]
```

Now, both Fedora and grsecurity both patch the vsdo Makefile, and the Kconfig with their own values. Lets fix the vsdo Makefile first. 

We need to find which patch file is in disagreement with the grsecurity patch, and then decide which patch we want to ship. So we will do a grep over the source files like so:

```bash
$ grep -Rin "a/arch/x86/entry/vdso/Makefile"
grsecurity-3.1-4.9.8-201702071801.patch:20838:diff --git a/arch/x86/entry/vdso/Makefile b/arch/x86/entry/vdso/Makefile
grsecurity-3.1-4.9.8-201702071801.patch:20840:--- a/arch/x86/entry/vdso/Makefile
kbuild-AFTER_LINK.patch:93:diff --git a/arch/x86/entry/vdso/Makefile b/arch/x86/entry/vdso/Makefile
kbuild-AFTER_LINK.patch:95:--- a/arch/x86/entry/vdso/Makefile

```

Since the grsecurity version is much more in depth than the simple changes to provide some compiler warning from Fedora, we will remove the vsdo patch from Fedora. Take note from grep, since it tells you the line you need to modify. I recommend using vim, and using the dd command to delete a line at a time.

You want to remove the following from the fedora patch.

```git
$ vim kbuild-AFTER_LINK.patch +93
diff --git a/arch/x86/entry/vdso/Makefile b/arch/x86/entry/vdso/Makefile
index d540966..eeb47b6 100644
--- a/arch/x86/entry/vdso/Makefile
+++ b/arch/x86/entry/vdso/Makefile
@@ -167,8 +167,9 @@ $(obj)/vdso32.so.dbg: FORCE \
 quiet_cmd_vdso = VDSO    $@
       cmd_vdso = $(CC) -nostdlib -o $@ \
                       $(VDSO_LDFLAGS) $(VDSO_LDFLAGS_$(filter %.lds,$(^F))) \
-                      -Wl,-T,$(filter %.lds,$^) $(filter %.o,$^) && \
-                sh $(srctree)/$(src)/checkundef.sh '$(NM)' '$@'
+                      -Wl,-T,$(filter %.lds,$^) $(filter %.o,$^) \
+               $(if $(AFTER_LINK),; $(AFTER_LINK)) && \
+               sh $(srctree)/$(src)/checkundef.sh '$(NM)' '$@'
 
 VDSO_LDFLAGS = -fPIC -shared $(call cc-ldoption, -Wl$(comma)--hash-style=both) \
        $(call cc-ldoption, -Wl$(comma)--build-id) -Wl,-Bsymbolic $(LTO_CFLAGS)
```

Next up is fixing the Kconfig file. It seems Fedora and grsecurity double up on CHECKPOINT_RESTORE

We find what Line it belongs to by searching for "CHECKPOINT_RESTORE"

```bash
$ grep -Rin "CHECKPOINT_RESTORE" grsecurity-3.1-4.9.8-201702071801.patch
146243: config CHECKPOINT_RESTORE
152162: 	if (IS_ENABLED(CONFIG_CHECKPOINT_RESTORE) &&
155876: 	depends on CHECKPOINT_RESTORE && HAVE_ARCH_SOFT_DIRTY && PROC_FS
```

We want to go to where it is defined, ie on line 139340:

```bash
$ vim grsecurity-3.1-4.9.8-201702071801.patch +146243
@@ -1158,6 +1163,7 @@ endif # CGROUPS
 config CHECKPOINT_RESTORE
    bool "Checkpoint/restore support" if EXPERT
    select PROC_CHILDREN
+   depends on !GRKERNSEC
    default n
    help
      Enables additional kernel features in a sake of checkpoint/restore.
```

Grsecurity also adds a "-grsec" suffix to the end of the kernel local build string. We already do this in the .spec file, so we can remove this to save us from further problems.

```bash
$ grep -Rin "localversion-grsec" grsecurity-3.1-4.9.8-201702071801.patch
155829:diff --git a/localversion-grsec b/localversion-grsec
155833:+++ b/localversion-grsec
```

So we need to jump to line 155829 and remove the following:

```git
$ vim grsecurity-3.1-4.9.8-201702071801.patch +155829
diff --git a/localversion-grsec b/localversion-grsec
new file mode 100644
index 0000000..7cd6065
--- /dev/null
+++ b/localversion-grsec
@@ -0,0 +1 @@
+-grsec
```

In recent Kernels (Linux 4.9.8+), there has been a rarther odd collision in the x86 section of the tree. Since Dapper Linux is amd64 only, this can be safely removed.

```git
$ grep -Rin "arch/x86/include/asm/asm-prototypes.h" grsecurity-3.1-4.9.13-201702270729.patch
grsecurity-3.1-4.9.13-201702270729.patch:22248:diff --git a/arch/x86/include/asm/asm-prototypes.h b/arch/x86/include/asm/asm-prototypes.h
grsecurity-3.1-4.9.13-201702270729.patch:22250:--- a/arch/x86/include/asm/asm-prototypes.h
grsecurity-3.1-4.9.13-201702270729.patch:22251:+++ b/arch/x86/include/asm/asm-prototypes.h
```

So we need to jump to line 22248 and remove:

```git
diff --git a/arch/x86/include/asm/asm-prototypes.h b/arch/x86/include/asm/asm-prototypes.h
index 44b8762..59e9d90 100644
--- a/arch/x86/include/asm/asm-prototypes.h
+++ b/arch/x86/include/asm/asm-prototypes.h
@@ -11,6 +11,8 @@
 #include <asm/special_insns.h>
 #include <asm/preempt.h>
 
+#include <asm/desc.h>
+
 #ifndef CONFIG_X86_CMPXCHG64
 extern void cmpxchg8b_emu(void);
 #endif
```

Now, when we go for a rebuild, we see the next very strange error:

```bash
$ rpmbuild -bp kernel.spec 
[...]
warning: squelched 110 whitespace errors
warning: 115 lines add whitespace errors.
fatal: empty ident name (for <>) not allowed
error: Bad exit status from /var/tmp/rpm-tmp.LkHtMV (%prep)
```

This is because the patch system is managed with git, and a new user.name and user.email are attached to every patch submitted to the branch we are modifying. Now, grsecurity does not ship with this information causing the filds in author-script to become empty:

```bash
$ cd rpmbuild/BUILD/kernel-4.9.fc25/linux-4.9.8-201.grsec.fc25.x86_64/.git/rebase-apply/
$ cat author-script
GIT_AUTHOR_NAME=''
GIT_AUTHOR_EMAIL=''
GIT_AUTHOR_DATE=''
```

We can fix this by adding the following to the top of the grsecurity patch:

```bash
$ vim grsecurity-3.1-4.9.8-201702071801.patch +1
From: "kernel-team@dapperlinux.com" <kernel-team@dapperlinux.com>
Date: Wed,  8 Feb 2017 00:00:00 +0000
Subject: [PATCH] Grsecurity Patchset
---
```

There is also a bug in the kernel.spec file where it will try and merge and then run newoptions over build configs that we aren't even going to build which prevents us from continuing. We can fix it by changing:

```spec
# now run oldconfig over all the config files
for i in *.config
```

to:

```spec
# now run oldconfig over all the config files
for i in %{all_arch_configs}
```

Now, grsecurity requires we add an extra dependancy for build, gcc-plugin-devel, since many security features are added at compile time. So add this just below the BuildRequires section, and just above the Sources section.

```spec
#Required for grsecurity
BuildRequires: gcc-plugin-devel
```

and now we can try and patch again and find the patches now work, but we have more errors:

```bash
$ rpmbuild -bp kernel.spec 
[...]
warning: squelched 110 whitespace errors
warning: 115 lines add whitespace errors.
+ chmod +x scripts/checkpatch.pl
[...]
+ grep -E '^CONFIG_'
+ '[' -s .newoptions ']'
+ cat .newoptions
CONFIG_GCC_PLUGINS
CONFIG_GRKERNSEC
error: Bad exit status from /var/tmp/rpm-tmp.ioWAuT (%prep)
```

This just means that there are options that are required to be configured in the kernel that haven't been configured yet. Namely, the grsecurity options haven't been configured yet. So, we will go to the build directory and run make menuconfig to select what grsecurity options we require. Grsecurity options live in Security -> Grsecurity.

```bash
$ cd ~/rpmbuild/BUILD/kernel-4.9.fc25/linux-4.9.8-201.grsec.fc25.x86_64/
$ make menuconfig
```

Once you have set your options, save them, and copy all custom options, which will include all CONFIG_GRKERNSEC and CONFIG_PAX options into SOURCES/config-local. You can find Dapper Linux's options [here](config-local)

```bash
$ grep "CONFIG_GRKERNSEC*" .config >> ~/rpmbuild/SOURCES/config-local
$ grep "CONFIG_PAX*" .config >> ~/rpmbuild/SOURCES/config-local
```

Grsecurity also adds in some new misc options, which we must also address

```bash
$ cat >> ~/rpmbuild/SOURCES/config-local << EOF
CONFIG_NETFILTER_XT_MATCH_GRADM=n
CONFIG_GCC_PLUGINS=y
# CONFIG_GCC_PLUGIN_CYC_COMPLEXITY is not set
# CONFIG_KMEMCHECK is not set
CONFIG_DEFAULT_MODIFY_LDT_SYSCALL=n
CONFIG_STACKTRACE=n
EOF
```

You can view your final config here:

```bash
$ most ~/rpmbuild/SOURCES/config-local
```

Hopefully that's all we need to do, so change back to the SPECS directory and do a test patch to see that everything goes smoothly

```bash
$ cd ~/rpmbuild/SPECS
$ rpmbuild -bp kernel.spec 
```

And finally, we can generate a source RPM for the copr or koji build system

```bash
$ rpmbuild -bs kernel.spec
Wrote: ~/rpmbuild/SRPMS/dapper-kernel-grsec-4.9.8-201.grsec.fc25.src.rpm
```

It's probably worth doing a test build before submitting it to a build server, so we can weed out any last minute compilation bugs. The following will build just the dapper-kernel-grsec, dapper-kernel-grsec-core, dapper-kernel-grsec-modules and dapper-kernel-grsec-modules-extra

```bash
$ rpmbuild -bb --without debug --without debuginfo --without extra --without perf --without tools kernel.spec
```

