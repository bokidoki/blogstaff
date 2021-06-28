---
title: 从系统服务注册开始分析android的ipc机制
categories: Android
top: 0
date: 2020-05-30 18:37:56
tags:
thumbnail:
---

> 系统服务的注册也是ipc，服务进程初始化时，会将服务注册到ServiceManager中，通过分析MediaPlayerService的注册流程，梳理一下android中的Binder机制。

```note
//本章源码目录
\system\core\rootdir\init.rc
\frameworks\av\media\mediaserver\main_mediaserver.cpp
\frameworks\native\libs\binder\IServiceManager.cpp

\frameworks\native\include\binder\ProcessState.h
\frameworks\native\libs\binder\ProcessState.cpp

\frameworks\native\include\binder\IServiceManager.h
\frameworks\native\libs\binder\IServiceManager.cpp

\frameworks\native\include\binder\IPCThreadState.h
\frameworks\native\libs\binder\IPCThreadState.cpp

\frameworks\native\include\binder\IInterface.h
\frameworks\native\libs\binder\Binder.cpp
```

## mediaserver启动流程

```note
// /system/bin 是mediaserver编译后的目录？是在\frameworks\av\media\mediaserver\Android.mk文件中配置的吗？
service media /system/bin/mediaserver
    class main
    user media
    group audio camera inet net_bt net_bt_admin net_bw_acct drmrpc mediadrm
    ioprio rt 4
```

```cpp
// 目录\frameworks\av\media\mediaserver\main_mediaserver.cpp
// 是否已经启动了mediaserver进程？
int main(int argc __unused, char **argv __unused)
{
    signal(SIGPIPE, SIG_IGN);
    // 遇到SIGPIPE信号忽略它
    // 如果在管道的读进程已终止时写管道则产生此信号
    // 当类型为SOCK_STREAM的套接字已不再连接时，进程写该套接字也产生此信号。
    sp<ProcessState> proc(ProcessState::self()); // 注释①
    // defaultServiceManager() 初始化全局对象 gDefaultServiceManager
    // gDefaultServiceManager 实际上为BpServiceManager(new BpBinder(0))
    sp<IServiceManager> sm(defaultServiceManager()); // 注释②
    ALOGI("ServiceManager: %p", sm.get());
    InitializeIcuOrDie();
    MediaPlayerService::instantiate(); // 注释③
    ResourceManagerService::instantiate();
    registerExtensions();
    ProcessState::self()->startThreadPool(); // 注释④
    IPCThreadState::self()->joinThreadPool(); // 注释⑤
}
```

init进程通过init.rc配置文件启动mediaserver，执行main_mediaserver的main方法。

- 注释1 获取ProcessState实例
- 注释2 得到一个默认的ServiceManager，即IServiceManager
- 注释3 初始化MediaPlayerService
- 注释4 创建一个线程池
- 注释5 加入到线程池中

### ProcessState::self()

```cpp
// 目录 \frameworks\native\libs\binder\ProcessState.cpp
// self函数
sp<ProcessState> ProcessState::self()
{
    // \system\core\include\utils\Mutex.h
    // Autolock自动锁，构造时执行Mutex.lock，析构时执行Mutex.unlock
    // 析构函数是在方法执行完成后执行吗？
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    // gProcess是定义在\frameworks\native\include\private\binder\Static.h的全局变量
    // ProcessState为每个进程中只存在一个
    gProcess = new ProcessState; // 注释①
    return gProcess;
}

// 构造方法
ProcessState::ProcessState()
    // open_driver()打开binder_driver驱动返回Binder的文件描述符赋值给mDriverFD
    : mDriverFD(open_driver()) // 注释②
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
#if !defined(HAVE_WIN32_IPC)
        // mmap binder，提供一块虚拟地址空间接收事务
        // mmap函数见[unix环境高级编程 14.8存储映射I/O]
        // 调用binder_mmap[https://code.woboq.org/linux/linux/drivers/android/binder.c.html#binder_mmap]
        // 映射到缓冲区的大小 #define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0); // 注释④
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");
            close(mDriverFD);
            mDriverFD = -1;
        }
#else
        mDriverFD = -1;
#endif
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
}

static int open_driver()
{
    // dev为linux驱动目录 /dev/binder为binder驱动文件
    int fd = open("/dev/binder", O_RDWR);
    if (fd >= 0) {
        // fcntl(int fd, int cmd, .../* arg */);
        // fcntl(int fd, int cmd, struct flock *lock);
        // F_SETFD 设置文件描述符标志 FD_CLOEXEC(close on exec)
        // 在使用execl调用执行的程序，此描述符将在子进程中会被自动关闭，不能使用。但是在父进程仍然可以使用。
        // fcntl函数见[unix环境高级编程 3.14函数fcntl]
        // exec函数族见[unix环境高级编程 8.10函数exec]
        fcntl(fd, F_SETFD, FD_CLOEXEC);
        int vers = 0;
        // ioctl函数见[unix环境高级编程 3.15函数ioctl]
        // ioctl最终映射到binder_ioctl[参考 linux内核5.6.15 https://elixir.bootlin.com/linux/latest/source/drivers/android/binder.c]
        // binder_version protocol_version 匹配binder的版本
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        if (result == -1) {
            ALOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
            close(fd);
            fd = -1;
        }
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
            ALOGE("Binder driver protocol does not match user space protocol!");
            close(fd);
            fd = -1;
        }
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        // 设置binder的最大线程数
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads); // 注释③
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));
    }
    // 返回/dev/binder 的文件描述符
    return fd;
}

```

### defaultServiceManager

```cpp
// 目录 \frameworks\native\libs\binder\IServiceManager.cpp
sp<IServiceManager> defaultServiceManager()
{
    // gDefaultServiceManager 位于\frameworks\native\include\private\binder\Static.h 也是个全局变量
    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
    {
        AutoMutex _l(gDefaultServiceManagerLock);
        while (gDefaultServiceManager == NULL) {
            // 这个函数很重要
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL)); // 注释①
            if (gDefaultServiceManager == NULL)
                sleep(1);
        }
    }
    return gDefaultServiceManager;
}
```

- 注释1:分解注释1中的函数

```cpp
// 目录 \frameworks\native\libs\binder\ProcessState.cpp
// ProcessState::self() 上面已分析返回gProcess
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);
}

// handle = 0
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);
    // 根据索引查询 struct handle_entry { IBinder* binder; RefBase::weakref_type* refs; }
    // 这个函数作用是看Vector<handle_entry>mHandleToObject中下标为handle的handle_entry是否存在，不存在插入一个，如果存在就直接取值。
    // [Vector \system\core\libutils\VectorImpl.cpp]
    // 为什么要存入mHandleToObject？
    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        // We need to create a new BpBinder if there isn't currently one, OR we
        // are unable to acquire a weak reference on this current one.  See comment
        // in getWeakProxyForHandle() for more info about this.
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            // 上面传入的handle == 0
            if (handle == 0) {
                Parcel data;
                // IPCThreadState与当前的线程绑定，根据IPCThreadState.h的定义可以看出IPCThreadState主要负责与binder_driver进行交互的。
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0); //注释一
                if (status == DEAD_OBJECT)
                   return NULL;
            }

            b = new BpBinder(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            // This little bit of nastyness is to allow us to add a primary
            // reference to the remote proxy when this team doesn't have one
            // but another team is sending the handle to us.
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}
```

接注释一，handle = 0时，向Binder发送了PING_TRANSACTION

```cpp
// 目录 \frameworks\native\libs\binder\IPCThreadState.cpp
static pthread_mutex_t gTLSMutex = PTHREAD_MUTEX_INITIALIZER;
static bool gHaveTLS = false;
static pthread_key_t gTLS = 0;

IPCThreadState* IPCThreadState::self()
{
    // 第一次进来为false
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        // pthread_getspecific [unix环境高级编程 12.6线程的特定数据]
        // 返回值：线程的特定数据值；若没有值与该键关联，返回NULL。
        // 类似java的ThreadLocal?
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState;// 注释二
    }

    if (gShutdown) return NULL;

    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS) {
        // 在分配线程特定数据之前，需要创建与该数据关联的键。这个键将用于获取对线程特定数据的访问。
        // pthread_key_create [unix环境高级编程 12.6线程的特定数据 446]
        // int pthread_key_create(pthread_key_t *keyp, void (*destructor)(void *))
        // 可以关联一个析构函数，当线程退出时，数据地址已经被置为非空值，那么析构函数就会被调用，它以为的参数就是该数据地址。
        if (pthread_key_create(&gTLS, threadDestructor) != 0) {
            pthread_mutex_unlock(&gTLSMutex);
            return NULL;
        }
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}

// 接注释二
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mMyThreadId(gettid()),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0)
{
    // pthread_setspecific [unix环境高级编程 12.6线程的特定数据 448]
    pthread_setspecific(gTLS, this);
    clearCaller();
    // 初始化mIn用于存储Binder发送回来的数据
    mIn.setDataCapacity(256);
    // 初始化mOut用于存储发送给Binder的数据
    mOut.setDataCapacity(256);
}

// 注释一的传参 transact(0, IBinder::PING_TRANSACTION, data, NULL, 0)
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    // 省略...

    if (err == NO_ERROR) {
        LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
            (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
        // 将binder_transaction_data写入mOut，cmd为BC_TRANSACTION
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }

    if ((flags & TF_ONE_WAY) == 0) {
        #if 0
        if (code == 4) { // relayout
            ALOGI(">>>>>> CALLING transaction 4");
        } else {
            ALOGI(">>>>>> CALLING transaction %d", code);
        }
        #endif
        // 不管Parcel* reply是否为nullptr，都是调用waitForResponse
        if (reply) {
            err = waitForResponse(reply); // 注释三
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        #if 0
        if (code == 4) { // relayout
            ALOGI("<<<<<< RETURNING transaction 4");
        } else {
            ALOGI("<<<<<< RETURNING transaction %d", code);
        }
        #endif
    } else {
        err = waitForResponse(NULL, NULL);
    }

    return err;
}

// 接注释三
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        // 通过talkWithDriver与Binder驱动通信
        if ((err=talkWithDriver()) < NO_ERROR) break; // 注释四
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;

        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;
            goto finish;

        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;

        case BR_ACQUIRE_RESULT:
            {
                const int32_t result = mIn.readInt32();
                if (!acquireResult) continue;
                *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
            }
            goto finish;

        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));

                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(NULL,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t), this);
                    }
                } else {
                    freeBuffer(NULL,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t), this);
                    continue;
                }
            }
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}

// 接注释四
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD <= 0) {
        return -EBADF;
    }

    // struct binder_write_read [参考 https://code.woboq.org/linux/linux/include/uapi/linux/android/binder.h.html#binder_write_read]
    // binder_write_read 数据交换的载体，既保存mOut数据交给内核层驱动去处理BINDER_WRITE_READ指令，又能向mIn传输数据
    binder_write_read bwr;

    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    // write_size对应mOut的长度或者为0
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    // 省略 IF_LOG_COMMANDS

    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        // 省略 IF_LOG_COMMANDS
#if defined(HAVE_ANDROID_OS)
        // talkWithDriver最终还是调用ioctl与binder进行数据交换，执行binder_ioctl
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD <= 0) {
            err = -EBADF;
        }
        // 省略 IF_LOG_COMMANDS
    } while (err == -EINTR);

    // 省略IF_LOG_COMMANDS

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else
                mOut.setDataSize(0);
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        // 省略 IF_LOG_COMMANDS
        return NO_ERROR;
    }

    return err;
}
```

### 相关结构体

```cpp
struct binder_transaction_data {
    /* The first two are only used for bcTRANSACTION and brTRANSACTION,
     * identifying the target and contents of the transaction.
     */
    union {
        /* target descriptor of command transaction */
        __u32    handle;
        /* target descriptor of return transaction */
        binder_uintptr_t ptr;
    } target;
    binder_uintptr_t    cookie;    /* target object cookie */
    __u32        code;        /* transaction command */
    /* General information about the transaction. */
    __u32            flags;
    pid_t        sender_pid;
    uid_t        sender_euid;
    binder_size_t    data_size;    /* number of bytes of data */
    binder_size_t    offsets_size;    /* number of bytes of offsets */
    /* If this transaction is inline, the data immediately
     * follows here; otherwise, it ends with a pointer to
     * the data buffer.
     */
    union {
        struct {
            /* transaction data */
            binder_uintptr_t    buffer;
            /* offsets from buffer to flat_binder_object structs */
            binder_uintptr_t    offsets;
        } ptr;
        __u8    buf[8];
    } data;
};
```
