# xDL

![](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat)
![](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat)
![](https://img.shields.io/badge/release-1.0.4-red.svg?style=flat)
![](https://img.shields.io/badge/Android-4.1%20--%2011-blue.svg?style=flat)
![](https://img.shields.io/badge/arch-armeabi--v7a%20%7C%20arm64--v8a%20%7C%20x86%20%7C%20x86__64-blue.svg?style=flat)

xDL is an enhanced implementation of the Android DL series functions.

[README 中文版](README.zh-CN.md)


## Features

* Enhanced `dlopen()` + `dlsym()` + `dladdr()`.
    * Bypass the restrictions of Android 7.0+ linker namespace.
    * Lookup dynamic link symbols in `.dynsym`.
    * Lookup debuging symbols in `.symtab` and "`.symtab` in `.gnu_debugdata`".
* Enhanced `dl_iterate_phdr()`.
    * Compatible with Android 4.x on ARM32.
    * Including linker / linker64 (for Android <= 8.x).
    * Return full pathname instead of basename (for Android 5.x).
    * Return app\_process32 / app\_process64 instead of package name.
* Support Android 4.1 - 11 (API level 16 - 30).
* Support armeabi-v7a, arm64-v8a, x86 and x86_64.
* MIT licensed.


## Artifacts Size

If xDL is compiled into an independent dynamic library:

| ABI         | Compressed (KB) | Uncompressed (KB) |
| :---------- | --------------: | ----------------: |
| armeabi-v7a | 6.6             | 13.9              |
| arm64-v8a   | 7.4             | 18.3              |
| x86         | 7.5             | 17.9              |
| x86_64      | 7.8             | 18.6              |


## Usage

### 1. Add dependency in build.gradle

xDL is published on [Maven Central](https://search.maven.org/), and uses [Prefab](https://google.github.io/prefab/) package format for [native dependencies](https://developer.android.com/studio/build/native-dependencies), which is supported by [Android Gradle Plugin 4.0+](https://developer.android.com/studio/releases/gradle-plugin?buildsystem=cmake#native-dependencies).

```Gradle
allprojects {
    repositories {
        mavenCentral()
    }
}
```

```Gradle
android {
    buildFeatures {
        prefab true
    }
}

dependencies {
    implementation 'io.hexhacking:xdl:1.0.4'
}
```

### 2. Add dependency in CMakeLists.txt or Android.mk

> CMakeLists.txt

```CMake
find_package(xdl REQUIRED CONFIG)

add_library(mylib SHARED mylib.c)
target_link_libraries(mylib xdl::xdl)
```

> Android.mk

```
include $(CLEAR_VARS)
LOCAL_MODULE           := mylib
LOCAL_SRC_FILES        := mylib.c
LOCAL_SHARED_LIBRARIES += xdl
include $(BUILD_SHARED_LIBRARY)

$(call import-module,prefab/xdl)
```

### 3. Specify one or more ABI(s) you need

```Gradle
android {
    defaultConfig {
        ndk {
            abiFilters 'armeabi-v7a', 'arm64-v8a', 'x86', 'x86_64'
        }
    }
}
```

### 4. Add packaging options

If you are using xDL in an SDK project, you may need to avoid packaging libxdl.so into your AAR, so as not to encounter duplicate libxdl.so file when packaging the app project.

```Gradle
android {
    packagingOptions {
        exclude '**/libxdl.so'
    }
}
```

On the other hand, if you are using xDL in an APP project, you may need to add some options to deal with conflicts caused by duplicate libxdl.so file.

```Gradle
android {
    packagingOptions {
        pickFirst '**/libxdl.so'
    }
}
```

There is a sample app in the [xdl-sample](xdl_sample) folder you can refer to.


## API

```C
#include "xdl.h"
```

### 1. `xdl_open()` and `xdl_close()`

```C
void *xdl_open(const char *filename);
void *xdl_close(void *handle);
```

They are very similar to `dlopen()` and `dlclose()`. But there are the following differences:

* `xdl_open()` can bypass the restrictions of Android 7.0+ linker namespace.
* If the library has been loaded into memory, `xdl_open()` will not `dlopen()` it again. (Because most of the system basic libraries will never be `dlclose()` after `dlopen()`)
* If `xdl_open()` really uses `dlopen()` to load the library, `xdl_close()` will return the handle from linker (the return value of `dlopen()`), and then you can decide whether and when to close it with `dlclose()`.

`filename` can be basename or full pathname. However, Android linker has used the namespace mechanism since 7.0. If you pass basename, you need to make sure that no duplicate ELF is loaded into the current process. `xdl_open()` will only return the first matching ELF. Please consider this fragment of `/proc/self/maps` on Android 10:

```
756fc2c000-756fc7c000 r--p 00000000 fd:03 2985  /system/lib64/vndk-sp-29/libc++.so
756fc7c000-756fcee000 --xp 00050000 fd:03 2985  /system/lib64/vndk-sp-29/libc++.so
756fcee000-756fcef000 rw-p 000c2000 fd:03 2985  /system/lib64/vndk-sp-29/libc++.so
756fcef000-756fcf7000 r--p 000c3000 fd:03 2985  /system/lib64/vndk-sp-29/libc++.so
7571fdd000-757202d000 r--p 00000000 07:38 20    /apex/com.android.conscrypt/lib64/libc++.so
757202d000-757209f000 --xp 00050000 07:38 20    /apex/com.android.conscrypt/lib64/libc++.so
757209f000-75720a0000 rw-p 000c2000 07:38 20    /apex/com.android.conscrypt/lib64/libc++.so
75720a0000-75720a8000 r--p 000c3000 07:38 20    /apex/com.android.conscrypt/lib64/libc++.so
760b9df000-760ba2f000 r--p 00000000 fd:03 2441  /system/lib64/libc++.so
760ba2f000-760baa1000 --xp 00050000 fd:03 2441  /system/lib64/libc++.so
760baa1000-760baa2000 rw-p 000c2000 fd:03 2441  /system/lib64/libc++.so
760baa2000-760baaa000 r--p 000c3000 fd:03 2441  /system/lib64/libc++.so
```

### 2. `xdl_sym()` and `xdl_dsym()`

```C
void *xdl_sym(void *handle, const char *symbol);
void *xdl_dsym(void *handle, const char *symbol);
```

They are very similar to `dlsym()`. They all takes a "handle" of an ELF returned by `xdl_open()` and the null-terminated symbol name, returning the address where that symbol is loaded into memory.

`xdl_sym()` lookup "dynamic link symbols" in `.dynsym` as `dlsym()` does.

`xdl_dsym()` lookup "debuging symbols" in `.symtab` and "`.symtab` in `.gnu_debugdata`".

Notice:

* The symbol sets in `.dynsym` and `.symtab` do not contain each other. Some symbols only exist in `.dynsym`, and some only exist in `.symtab`. You may need to use tools such as readelf to determine which ELF section the symbol you are looking for is in.
* `xdl_dsym()` needs to load debuging symbols from disk file, and `xdl_sym()` only lookup dynamic link symbols from memory. So `xdl_dsym()` runs slower than `xdl_sym()`.
* The dynamic linker only uses symbols in `.dynsym`. The debugger actually uses the symbols in both `.dynsym` and `.symtab`.

### 3. `xdl_addr()`

```C
int xdl_addr(void *addr, Dl_info *info, void **cache);
void xdl_addr_clean(void **cache);
```

`xdl_addr()` is similar to `dladdr()`. But there are a few differences:

* `xdl_addr()` can lookup not only dynamic link symbols, but also debugging symbols. 
* `xdl_addr()` needs to pass an additional parameter (`cache`), which will cache the ELF handle opened during the execution of `xdl_addr()`. The purpose of caching is to make subsequent executions of `xdl_addr()` of the same ELF faster. When you do not need to execute `xdl_addr()`, please use `xdl_addr_clean()` to clear the cache. For example:

```C
void *cache = NULL;
Dl_info info;
xdl_addr(addr_1, &info, &cache);
xdl_addr(addr_2, &info, &cache);
xdl_addr(addr_3, &info, &cache);
xdl_addr_clean(&cache);
```

### 4. `xdl_iterate_phdr()`

```C
#define XDL_DEFAULT       0x00
#define XDL_WITH_LINKER   0x01
#define XDL_FULL_PATHNAME 0x02

int xdl_iterate_phdr(int (*callback)(struct dl_phdr_info *, size_t, void *), void *data, int flags);
```

`xdl_iterate_phdr()` is similar to `dl_iterate_phdr()`. But `xdl_iterate_phdr()` is compatible with android 4.x on ARM32.

`xdl_iterate_phdr()` has an additional "flags" parameter, one or more flags can be bitwise-or'd in it:

* `XDL_DEFAULT`: Default behavior.
* `XDL_WITH_LINKER`: Always including linker / linker64.
* `XDL_FULL_PATHNAME`: Always return full pathname instead of basename.

These flags are needed because these capabilities require additional execution time, and you don't always need them.


## Official Repositories

* https://github.com/hexhacking/xDL
* https://gitlab.com/hexhacking/xDL
* https://gitee.com/hexhacking/xDL


## Support

* Check the [xdl-sample](xdl_sample).
* Communicate on [GitHub issues](https://github.com/hexhacking/xDL/issues).
* Email: <a href="mailto:caikelun@gmail.com">caikelun@gmail.com</a>
* QQ group: 603635869


## Contributing

[xDL Contributing Guide](CONTRIBUTING.md).


## License

xDL is MIT licensed, as found in the [LICENSE](LICENSE) file.

xDL documentation is Creative Commons licensed, as found in the [LICENSE-docs](LICENSE-docs) file.


## History

[xCrash 2.x](https://github.com/hexhacking/xCrash/tree/4748d183c1395c54bfb760ec6c454966d52ab73f) contains a very rudimentary module [xc_dl](https://github.com/hexhacking/xCrash/blob/4748d183c1395c54bfb760ec6c454966d52ab73f/src/native/libxcrash/jni/xc_dl.c) for searching system library symbols, which has many problems in performance and compatibility. xCrash 2.x uses it to search a few symbols from libart, libc and libc++.

Later, some other projects began to use the xc_dl module alone, including in some performance-sensitive usage scenarios. At this time, we began to realize that we need to rewrite this module, and we need a better implementation.
