# 基于aosp代码编译Setools

aosp在最新的android16-qpr2-release(2025-12-05)(sdk_full=36.1)中启用了新的selinux policy规则,导致当前最新的setools也无法处理pixel android16 2025-12-05以后版本中的二进制policy文件,需要基于最新的aosp16代码重新编译才能处理.

## 1. 在aosp16中编译libselinux和libsepol

假设aosp的根目录为~/aosp_16(代码基线android16-qpr2-release)

设置变量

```
export AOSP_BASE=~/aosp_16
```

编译

```
cd $AOSP_BASE
source build/envsetup.sh
lunch aosp_arm64-bp4a-userdebug
m libselinux libsepol
```

输出

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

## 2. 在linunx下编译x64版本Setools

### 2.0 准备

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

### 2.1 编译setools准备

使用最新的setools代码,用aosp编译出来的linux_x64的静态库.a和动态库so进行编译

#### 下载代码

```
  git clone https://github.com/SELinuxProject/setools.git
  cd setools
  export SETOOLS_BASE=$PWD
   
  
  # 使用python3.13或python3.12进行编译
  uv venv .venv -p 3.13
  source .venv/bin/activate
  uv pip install setuptools cython
```

#### 复制aosp文件

```
cd $SETOOLS_BASE
#复制静态库文件x64
mkdir static_libs
cd static_libs
cp $AOSP_BASE/out/host/linux-x86/obj/STATIC_LIBRARIES/libselinux_intermediates/libselinux.a .
cp $AOSP_BASE/out/host/linux-x86/obj/STATIC_LIBRARIES/libsepol_intermediates/libsepol.a .
cd ..
# 复制动态库文件x64
mkdir libs
cd libs
cp $AOSP_BASE/out/host/linux-x86/lib64/*.so .
cd ..
# 复制静态库文件arm64
mkdir static_libs
cd static_libs
$AOSP_BASE/out/host/linux-x86/obj/STATIC_LIBRARIES/libselinux_intermediates/libselinux.a .
$AOSP_BASE/out/host/linux-x86/obj/STATIC_LIBRARIES/libsepol_intermediates/libsepol.a .
cd ..
# 复制动态库文件arm64
mkdir libs_arm64
cd libs_arm64
cp $AOSP_BASE/out/target/product/generic_arm64/system/lib64/*.so .
cd ..
#复制aosp头文件
mkdir aosp_include
cd aosp_include
cp -r $AOSP_BASE/external/selinux/libsepol/include/sepol .
cp -r $AOSP_BASE/external/selinux/libselinux/include/selinux .
```

#### 最终目录树

```
setools/
├── 原有文件               
├── libs/                   # aosp编译出来的x64 so
├── libs_arm64/             # aosp编译出来的arm64 so
├── static_libs/             # aosp编译出来的x64 a
├── static_libs_arm64/             # aosp编译出来的arm64 a
├── aosp_include/             # aosp libselinux/libsepol的头文件
```



### 2.2. 编译setools

```
# 设置变量
cd $SETOOLS_BASE
export AOSP_STATIC_LIB=$PWD/static_libs
export AOSP_INCLUDE=$PWD/aosp_include
export AOSP_LIB64=$PWD/libs

#如果需要再次编译,需要先清除前面编译结果
python3 setup.py clean -all

# 由于aosp的缺少部分符号定义,需要在编译时补齐三个符号
# 将libselinux.a和libsepol.a静态注入setools的so中,解决和系统的so冲突问题
CFLAGS="-I$AOSP_INCLUDE -DANDROID" \
LDFLAGS="-Wl,--allow-multiple-definition \
         -Wl,--defsym=selinux_current_policy_path=0 \
         -Wl,--defsym=selinux_binary_policy_path=0 \
         -Wl,--defsym=selinux_policy_root=0 \
         -Wl,--whole-archive $AOSP_STATIC_LIB/libselinux.a $AOSP_STATIC_LIB/libsepol.a -Wl,--no-whole-archive \
         -L$AOSP_LIB64 -lc++ -lpcre2-8" \
python3 setup.py build_ext --inplace

# 每次编译后必须执行,告诉setools优先从setools/libs里找so
patchelf --set-rpath '$ORIGIN/../libs' setools/policyrep.cpython-313-x86_64-linux-gnu.so
```

### 2.3 校验

通过ldd可以看到,没有libselinux和libsepol,libc++使用libs里面的

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

### 2.4 运行移植

要移植到其它linux下运行,需要按以下目录移植,同时使用相同的python版本

```
setools_dist/
├── sesearch等启动脚本        # 启动脚本
├── libs/                   # 依赖库文件夹
└── setools/                # Python 包目录
    ├── __init__.py
    ├── policyrep.cpython-313-x86_64-linux-gnu.so  # 编译出来的so
    └── ... (其他 .py 文件)
```

### 2.5 使用

sesearch

```
./sesearch -A -T --role_allow --role_transition --range_transition _sys_fs_selinux_policy >p1.txt
```

seinfo

```
./seinfo --all _sys_fs_selinux_policy >p_info.txt
```

## 3. 在termux中编译arm64版本

当前直接在android上运行python版本的setools比较复杂，因此在android上的termux中编译setools，然后把编译出来的setools集成到apk中运行。

### 3.1 termux准备

当前termux默认使用python3.12

```
pkg update && pkg upgrade
pkg install termux-services python clang make binutils ndk-sysroot ldd networkx setuptools cython 
# 获取手机存储权限
termux-setup-storage
```

### 3.2 push setools

在linux中把前面编译的setools目录push到termux下

```
cd $SETOOLS_BASE
cd ..
adb push setools /sdcard/Download #/sdcard/Download和termux中的/data/data/com.termux/files/home/storage/downloads目录对应 
```

### 3.3 termux设置变量

```
# 在setools根目录下设置变量，路径指向编译出来的arm64所在目录
cd storage/downloads/setools
AOSP_STATIC_LIB=$PWD/static_libs_arm64
AOSP_INCLUDE=$PWD/aosp_include
AOSP_LIB64=$PWD/libs_arm64
```



### 3.4 编译

```
#如果编译成功后需要再次编译,需要先清除编译
python3 setup.py clean -all

# 由于aosp的缺少部分符号定义,需要在编译时补齐三个符号
# termux和anndroid在ubsan的定义存在差异，需要补齐
# 将libselinux.a和libsepol.a静态注入setools的so中,解决和系统的so冲突问题
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

### 3.5 校验

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

### 3.6 集成准备

- 将编译好的so复制到linux的setools下

```
cd $SETOOLS_BASE/setools
adb pull /sdcard/Download/setools/setools/policyrep.cpython-312.so _policyrep.so
# 1. 确认当前的依赖项
patchelf --print-needed _policyrep.so
libandroid-support.so
libc++.so
libpython3.12.so.1.0
libc.so

# 2. 将 libpython3.12.so.1.0 替换为 libpython3.12.so
patchelf --replace-needed libpython3.12.so.1.0 libpython3.12.so _policyrep.so
```

- 从termux中复制相关so

在termux中复制

```
cp $PREFIX/lib/libpython3.12.so.1.0 /sdcard/Download/
cp $PREFIX/lib/libandroid-support.so /sdcard/Download
```

复制到libs_arm64

```
cd $SETOOLS_BASE/libs_arm64
adb pull /sdcard/Download/libpython3.12.so.1.0 .
adb pull /sdcard/Download/libandroid-support.so .
cp libpython3.12.so.1.0 libpython3.12.so.1.0.so
```

libs_arm64最后文件

```
libandroid-support.so  libc++.so  libpcre2.so  libpython3.12.so.1.0  libpython3.12.so.1.0.so  libselinux.so  libsepol.so
```

- 基于Chaquopy的集成目录

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
├── 自己开发的python脚本 
├── sesearch.py ... # setools的命令脚本
└── setools/
    ├── _policyrep.so          
    原setools/setools下所有文件和子目录

```



## 附录

### aosp代码基线

https://source.android.com/docs/setup/download

```
branch :android16-qpr2-release
tag: android-16.0.0_r4
BuildId:BP4A.251205.006
```

### setools

https://github.com/SELinuxProject/setools

### 使用ssh访问termux

termux操作

```
pkg update && pkg upgrade
pkg install termux-services openssh
# 设置登录密码
passwd
# 启动 SSH 服务（默认端口是 8022）
sshd
# 默认启动ssh
sv-enable sshd
# 查看ip
ifconfig
# 假设ip为192.168.200.102
# 查看用户
whoami
# 假设用户名为u0_a228
```

liunx下操作

```
ssh u0_a228@192.168.200.102 -p 8022
```



