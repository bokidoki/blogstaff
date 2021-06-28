---
title: Linux知识点拾遗
tags:
categories: Linux
date:
img:
---

### 配置Linux环境变量

- 在/etc/profile文件中增加环境变量，该变量将会对Linux下所有用户有效，并且是永久的。  
  例如

  ```vim
  gedit /etc/profile
  export KOTLIN_HOME=/PATH/TO/KOTLIN/HOME
  export PATH=$PATH:$KOTLIN_HOME/bin
  ``` 

- 在用户目录下的.bash_profile文件中增加环境变量【只对当前用户生效】，并且是永久的。
  例如

  ```vim
  gedit ~/.bash_profile
  export YOUR PATH
  ```

  上述两种方法修改后需运行下面指令才能生效

  ```vim
  source /etc/profile
  ```

- 直接在shell中输入，这种方法只对当前shell生效
  
  ```vim
  export 变量名=变量值
  ```

linux指令 [零教程]("http://www.lingjiaocheng.com/d/unix_system_calls_mknod-html/94/4869")  
> mknod // 创建一个特殊的文件货或普通文件

```c
int mknod(const char *pathname, mode_t mode, dev_t dev)
```

> epoll_create // 打开一个epoll文件描述符

```c
int epoll_create(int size)
```

> memset <string.h>

```c
void *memset(void *dest, int ch, size_t size)
```

> dlopen

dlopen加载以null-terminated string命名的dynamic shared object(shared libraray)，返回一个不可见的handle用于加载对象。handle被用于其它dlopen的api，如dlsym,dladdr,dlinfo,dlclose。

```c
#include <dlfcn.h>

/**
* flags
* RTLD_LAZY
* RTLD_NOW
* RTLD_GLOBAL
* RTLD_LOCAL
* RTLD_NODELETE
* RTLD_NOLOAD
* RTLD_DEEPBIND
*/
void *dlopen(const char *filename, int flags);
```

> dlsym

函数dlsym（）获取dlopen（3）返回的动态加载的共享对象的“句柄”以及以空字符结尾的符号名称，并返回将该符号加载到内存中的地址。

```c
void *dlsym(void *handle, const char *symbol);
```

### 参考

> [Linux变量总结 -Manfred_Zone](https://www.jianshu.com/p/ac2bc0a3d74)  
> [Kotlin环境搭建 - ZJ](https://laobadao.github.io/2017/06/15/kotlin-command-line/index.html)
  