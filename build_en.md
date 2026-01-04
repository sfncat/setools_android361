# Building Setools based on AOSP code

AOSP has enabled new SELinux policy rules in the latest Android 16-QPR2-release (2025-12-05) (sdk\_full=36.1), causing the current latest Setools to be unable to handle binary policy files in Pixel Android 16 versions after 2025-12-05. It is necessary to recompile based on the latest AOSP 16 code to handle them.

## 1\. Compile libselinux and libsepol in AOSP 16

Assuming the root directory of AOSP is ~/aosp\_16 (code baseline android16-qpr2-release)

Set variable

```
export AOSP_BASE=~/aosp_16
```

Compile

```
cd $AOSP_BASE
source build/envsetup.sh
lunch aosp_arm64-bp4a-userdebug
m libselinux libsepol
```

Output

x64

```
so:
$AOSP_BASE/out/host/linux-x86/lib64
a:
$AOSP_BASE/out/host/linux-x86/obj/STATIC_LIBRARIES/libselinux_intermediates/libselinux.a
$AOSP_BASE/out/host/linux-x86/obj/STATIC_LIBRARIES/libsepol_intermediates/libsepol.a
```

arm64:

```
so:
$AOSP_BASE/out/target/product/generic_arm64/system/lib64
a:
$AOSP_BASE/target/product/generic_arm64/obj/STATIC_LIBRARIES/libselinux_intermediates/libselinux.a
$AOSP_BASE/target/product/generic_arm64/obj/STATIC_LIBRARIES/libsepol_intermediates/libsepol.a
```

## 2\. Compile x64 version of Setools under Linux

### 2.0 Preparation

```
sudo apt update
sudo apt install -y \
    build-essential \
    cmake \
    pkg-config \
    flex \
    bison \
    python3-dev \
    python3-setuptools \
    python3-cython \
    python3-networkx \
    swig
sudo apt install patchelf
```

### 2.1 Preparation for compiling Setools

Using the latest setools code, compile the static library .a and dynamic library .so for linux\_x64 built with aosp

#### Download the code

```
  git clone https://github.com/SELinuxProject/setools.git
  cd setools
  export SETOOLS_BASE=$PWD
   
  
  # Compile using python3.13 or python3.12
  uv venv .venv -p 3.13
  source .venv/bin/activate
  uv pip install setuptools cython
```

#### Copy the aosp files

```
cd $SETOOLS_BASE
# Copy static library files x64
mkdir static_libs
cd static_libs
cp $AOSP_BASE/out/host/linux-x86/obj/STATIC_LIBRARIES/libselinux_intermediates/libselinux.a .
cp $AOSP_BASE/out/host/linux-x86/obj/STATIC_LIBRARIES/libsepol_intermediates/libsepol.a .
cd ..
# Copy dynamic library files x64
mkdir libs
cd libs
cp $AOSP_BASE/out/host/linux-x86/lib64/*.so .
cd ..
# Copy static library files arm64
mkdir static_libs
cd static_libs
$AOSP_BASE/out/host/linux-x86/obj/STATIC_LIBRARIES/libselinux_intermediates/libselinux.a .
$AOSP_BASE/out/host/linux-x86/obj/STATIC_LIBRARIES/libsepol_intermediates/libsepol.a .
cd ..
# Copy dynamic library files arm64
mkdir libs_arm64
cd libs_arm64
cp $AOSP_BASE/out/target/product/generic_arm64/system/lib64/*.so .
cd ..
# Copy aosp header files
mkdir aosp_include
cd aosp_include
cp -r $AOSP_BASE/external/selinux/libsepol/include/sepol .
cp -r $AOSP_BASE/external/selinux/libselinux/include/selinux .
```

#### Final directory tree

```
setools/
├── Original files               
├── libs/                   # x64 so compiled from aosp
├── libs_arm64/             # arm64 so compiled from aosp
├── static_libs/             # x64 a compiled from aosp
├── static_libs_arm64/             # arm64 a compiled from aosp
├── aosp_include/             # aosp libselinux/libsepol header files
```

### 2.2. Compile setools

```
# Set variables
cd $SETOOLS_BASE
export AOSP_STATIC_LIB=$PWD/static_libs
export AOSP_INCLUDE=$PWD/aosp_include
export AOSP_LIB64=$PWD/libs

# If recompiling, clear previous compilation results first
python3 setup.py clean -all

# Due to missing symbol definitions in aosp, three symbols need to be supplemented during compilation
# Statically inject libselinux.a and libsepol.a into setools' so to resolve conflicts with system so
CFLAGS="-I$AOSP_INCLUDE -DANDROID" \
LDFLAGS="-Wl,--allow-multiple-definition \
         -Wl,--defsym=selinux_current_policy_path=0 \
         -Wl,--defsym=selinux_binary_policy_path=0 \
         -Wl,--defsym=selinux_policy_root=0 \
         -Wl,--whole-archive $AOSP_STATIC_LIB/libselinux.a $AOSP_STATIC_LIB/libsepol.a -Wl,--no-whole-archive \
         -L$AOSP_LIB64 -lc++ -lpcre2-8" \
python3 setup.py build_ext --inplace

# Must be executed after each compilation to tell setools to prioritize searching for so in setools/libs
patchelf --set-rpath '$ORIGIN/../libs' setools/policyrep.cpython-313-x86_64-linux-gnu.so

```

### 2.3 Verification

Using ldd, it can be seen that there are no libselinux and libsepol, libc++ uses libs

```
ldd setools/policyrep.cpython-313-x86_64-linux-gnu.so 
	linux-vdso.so.1 (0x00007ea9966ee000)
	libc++.so => /home/kali/setools/setools/../libs/libc++.so (0x00007ea9960ab000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ea995e00000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ea9966f0000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007ea9966ce000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007ea9966c9000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007ea9965e0000)
	librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007ea9960a6000)

```

### 2.4 Porting

To port to other Linux systems for operation, follow the directory structure below while using the same Python version

```
setools_dist/
├── Startup scripts such as sesearch        # Startup scripts
├── libs/                   # Dependency library folder
└── setools/                # Python package directory
    ├── __init__.py
    ├── policyrep.cpython-313-x86_64-linux-gnu.so  # Compiled so
    └── ... (other .py files)
```

### 2.5 Usage

sesearch

```
./sesearch -A -T --role_allow --role_transition --range_transition _sys_fs_selinux_policy >p1.txt
```

seinfo

```
./seinfo --all _sys_fs_selinux_policy >p_info.txt
```

## 3\. Compile arm64 version in termux

Currently, running the python version of setools directly on Android is relatively complex, so compiling setools in termux on Android and then integrating the compiled setools into an apk for execution.

### 3.1 termux preparation

Currently, termux uses python3.12 by default.

```
pkg update && pkg upgrade
pkg install termux-services python clang make binutils ndk-sysroot ldd networkx setuptools cython 
# Get phone storage permissions
termux-setup-storage
```

### 3.2 push setools

Push the previously compiled setools directory to Termux on Linux

```
cd $SETOOLS_BASE
cd ..
adb push setools /sdcard/Download #/sdcard/Download和termux中的/data/data/com.termux/files/home/storage/downloads目录对应 
```

### 3.3 termux set variables

```
# Set variables in the setools root directory, path points to the compiled arm64 directory
cd storage/downloads/setools
AOSP_STATIC_LIB=$PWD/static_libs_arm64
AOSP_INCLUDE=$PWD/aosp_include
AOSP_LIB64=$PWD/libs_arm64
```

### 3.4 compile

```
# If compilation is successful and needs to be recompiled, clear the previous compilation first
python3 setup.py clean -all

# Due to missing symbol definitions in aosp, three symbols need to be supplemented during compilation
# There are differences in ubsan definitions between termux and android, which need to be supplemented
# Statically inject libselinux.a and libsepol.a into setools' so to resolve conflicts with system so
CFLAGS="-I$AOSP_INCLUDE -DANDROID -D_GNU_SOURCE -fno-sanitize=all -O2" \
LDFLAGS="-Wl,--allow-multiple-definition \
         -Wl,--defsym=selinux_current_policy_path=0 \
         -Wl,--defsym=selinux_binary_policy_path=0 \
         -Wl,--defsym=selinux_policy_root=0 \
-Wl,--defsym=__ubsan_handle_add_overflow_minimal_abort=0 \
-Wl,--defsym=__ubsan_handle_sub_overflow_minimal_abort=0 \
-Wl,--defsym=__ubsan_handle_mul_overflow_minimal_abort=0 \
         -Wl,--whole-archive $AOSP_STATIC_LIB/libselinux.a $AOSP_STATIC_LIB/libsepol.a -Wl,--no-whole-archive \
         -L$AOSP_LIB64 -lc++ -lpcre2-8 -Wl,--no-undefined" \
python3 setup.py build_ext --inplace
```

### 3.5 Validation

```
ldd setools/policyrep.cpython-312.so 
	libandroid-support.so => /data/data/com.termux/files/usr/lib/libandroid-support.so
	libc++.so => /system/lib64/libc++.so
	libpython3.12.so.1.0 => /data/data/com.termux/files/usr/lib/libpython3.12.so.1.0
	libc.so => /system/lib64/libc.so
	ld-android.so => /system/lib64/ld-android.so
	libdl.so => /system/lib64/libdl.so
	libm.so => /system/lib64/libm.so
```

### 3.6 Integration Preparation

*   Copy the compiled .so files to the setools directory on Linux

```
cd $SETOOLS_BASE/setools
adb pull /sdcard/Download/setools/setools/policyrep.cpython-312.so _policyrep.so
# 1. Confirm current dependencies
patchelf --print-needed _policyrep.so
libandroid-support.so
libc++.so
libpython3.12.so.1.0
libc.so

# 2. Replace libpython3.12.so.1.0 with libpython3.12.so
patchelf --replace-needed libpython3.12.so.1.0 libpython3.12.so _policyrep.so

```

*   Copy the relevant .so files from Termux

Copy in Termux

```
cp $PREFIX/lib/libpython3.12.so.1.0 /sdcard/Download/
cp $PREFIX/lib/libandroid-support.so /sdcard/Download
```

Copy to libs\_arm64

```
cd $SETOOLS_BASE/libs_arm64
adb pull /sdcard/Download/libpython3.12.so.1.0 .
adb pull /sdcard/Download/libandroid-support.so .
cp libpython3.12.so.1.0 libpython3.12.so.1.0.so
```

Last file in libs\_arm64

```
libandroid-support.so  libc++.so  libpcre2.so  libpython3.12.so.1.0  libpython3.12.so.1.0.so  libselinux.so  libsepol.so
```

*   Integration directory based on Chaquopy

```
app/src/main/jniLibs/arm64-v8a/
├── libandroid-support.so
├── libpcre2.so
├── libpython3.12.so.1.0 
├── libpython3.12.so.1.0.so
├── libselinux.so
├── libsepol.so
└── libc++.so    

app/src/main/python/
├── Self-developed python scripts 
├── sesearch.py ... # setools command scripts
└── setools/
    ├── _policyrep.so          
    All files and subdirectories under the original setools/setools

```

## Appendix

### AOSP codebase

[https://source.android.com/docs/setup/download](https://source.android.com/docs/setup/download)

```
branch :android16-qpr2-release
tag: android-16.0.0_r4
BuildId:BP4A.251205.006
```

### setools

[https://github.com/SELinuxProject/setools](https://github.com/SELinuxProject/setools)

### Using SSH to access Termux

Termux operation

```
pkg update && pkg upgrade
pkg install termux-services openssh
# Set login password
passwd
# Start SSH service (default port is 8022)
sshd
# Start ssh by default
sv-enable sshd
# View IP
ifconfig
# Assuming IP is 192.168.200.102
# View user
whoami
# Assuming username is u0_a228
```

Linux operation

```
ssh u0_a228@192.168.200.102 -p 8022
```