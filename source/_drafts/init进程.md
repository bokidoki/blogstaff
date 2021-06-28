---
title: init进程
date: 2020-04-13 14:18:17
tags: framework
categories: Android
img:
---
### 相关源码

```C++
/system/core/init/Init.cpp
/system/core/rootdir/init.rc
/system/core/init/init_parser.cpp
/system/core/init/builtins.cpp
/system/core/init/signal_handler.cpp
```

### init.cpp的起点

文件位置 /system/core/init/Init.cpp

```C++
int main(int argc, char** argv) {
    // 比较argv第一参数的basename与ueventd值
    // strcmp 两个数相等时为 0 等于0时为假
    // 因此argv[0]与ueventd相等时，执行ueventd_main
    if (!strcmp(basename(argv[0]), "ueventd")) {
        // uevent是kobject的一部分用于在Kobject改变时，例如增加、移除时通知用户空间，用户
        // 空间接收到这样的事件后会做出相应的处理。（kobject在内核中被用作设备驱动模型）
        // 在这里就会启动ueventd守护进程
        return ueventd_main(argc, argv);
    }
    // 比较参数是不是watchdogd
    if (!strcmp(basename(argv[0]), "watchdogd")) {
        // 启动watchdogd
        // watchdog负责定时向"dev/watchdog"执行写操作，监测系统是否运转正常
        return watchdogd_main(argc, argv);
    }

    // Clear the umask.
    // 执行umask函数，用于屏蔽权限
    // 表示后面所有创建的文件和目录都会被限制，但是这里为0，代表没有限制
    umask(0);

    // _PATH_DEFPATH添加到环境变量
    add_environment("PATH", _PATH_DEFPATH);

    // 如果argc == 1并且参数中不含有second-stage，则执行第一阶段
    bool is_first_stage = (argc == 1) || (strcmp(argv[1], "--second-stage") != 0);

    // Get the basic filesystem setup we need put together in the initramdisk
    // on / and then we'll let the rc file figure out the rest.
    if (is_first_stage) {
        // 第一阶段做了如下事情：
        // 挂载与linux相关的一些文件
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        mount("proc", "/proc", "proc", 0, NULL);
        mount("sysfs", "/sys", "sysfs", 0, NULL);
    }

    // We must have some place other than / to create the device nodes for
    // kmsg and null, otherwise we won't be able to remount / read-only
    // later on. Now that tmpfs is mounted on /dev, we can actually talk
    // to the outside world.
    // 创建/dev/__null__文件用于输出错误日志
    open_devnull_stdio();
    // 创建/dev/kmsg用于输出日志
    klog_init();
    // 设置日志的输出级别
    klog_set_level(KLOG_NOTICE_LEVEL);

    NOTICE("init%s started!\n", is_first_stage ? "" : " second stage");

    if (!is_first_stage) {
        // Indicate that booting is in progress to background fw loaders, etc.
        close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
        // 创建一块共享内存用于属性服务
        property_init();

        // If arguments are passed both on the command line and in DT,
        // properties set in DT always have priority over the command-line ones.
        // DT即device-tree，记录自己的硬件设备和系统运行参数
        process_kernel_dt(); // 处理DT属性
        process_kernel_cmdline(); // 处理命令行属性

        // Propogate the kernel variables to internal variables
        // used by init as well as the current required properties.
        export_kernel_boot_props(); // 处理其它属性
    }

    // Set up SELinux, including loading the SELinux policy if we're in the kernel domain.
    // SE(security) linux 控制所有进程的访问权限
    selinux_initialize(is_first_stage);

    // If we're in the kernel domain, re-exec init to transition to the init domain now
    // that the SELinux policy has been loaded.
    if (is_first_stage) {
        if (restorecon("/init") == -1) {
            ERROR("restorecon failed: %s\n", strerror(errno));
            security_failure();
        }
        char* path = argv[0];
        char* args[] = { path, const_cast<char*>("--second-stage"), nullptr };
        if (execv(path, args) == -1) {
            ERROR("execv(\"%s\") failed: %s\n", path, strerror(errno));
            security_failure();
        }
    }

    // These directories were necessarily created before initial policy load
    // and therefore need their security context restored to the proper value.
    // This must happen before /dev is populated by ueventd.
    INFO("Running restorecon...\n");
    restorecon("/dev");
    restorecon("/dev/socket");
    restorecon("/dev/__properties__");
    restorecon_recursive("/sys");

    // epoll中主要涉及3个函数 epoll_create()、epoll_ctl()、epoll_wait()
    // epoll_create成功时，epoll_fd非负
    // epoll用于linux内核为处理大批量文件描述符，是linux下多路复用接口select/poll的增强版本
    // 这里初始化epoll
    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        ERROR("epoll_create1 failed: %s\n", strerror(errno));
        exit(1);
    }
    //初始化signal handler
    signal_handler_init();

    property_load_boot_defaults();
    start_property_service();
    // 解析init.rc
    init_parse_config_file("/init.rc");
    // 将early-init加入队列尾部 对应init.rc on early-init
    action_for_each_trigger("early-init", action_add_queue_tail);

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    // 插入action监听冷插拔完成状态
    queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    // 从硬件PNG的设备文件/dev/hw_random中读取512字节并写到Linux RNG的设备文件/dev/urandom中
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    // keychord_init_action()函数：初始化组合键监听模块
    queue_builtin_action(keychord_init_action, "keychord_init");
    // console_init_action()函数：在屏幕个上显示Android字样的Logo
    queue_builtin_action(console_init_action, "console_init");

    // Trigger all the boot actions to get us started.
    // 执行init.rc on init
    action_for_each_trigger("init", action_add_queue_tail);

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

    // Don't mount filesystems or start core system services in charger mode.
    char bootmode[PROP_VALUE_MAX];
    //当处于充电模式，则charger加入执行队列；否则late-init加入队列。
    if (property_get("ro.bootmode", bootmode) > 0 && strcmp(bootmode, "charger") == 0) {
        action_for_each_trigger("charger", action_add_queue_tail);
    } else {
        action_for_each_trigger("late-init", action_add_queue_tail);
    }

    // Run all property triggers based on current state of the properties.
    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");

    while (true) {
        if (!waiting_for_exec) {
            // 执行命令
            execute_one_command();
            // 按需要重新执行命令
            restart_processes();
        }

        int timeout = -1;
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }

        if (!action_queue_empty() || cur_action) {
            timeout = 0;
        }

        bootchart_sample(&timeout);

        epoll_event ev;
        // epoll wait 等待后续处理大批量文件任务
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }

    return 0;
}
```

> signal_handler_init 位于 signal_handler.cpp

```C++
// 用于注册SIGCHLD信号
// 在linux中父进程是通过捕捉SIGCHLD信号来得知子进程运行结束情况，SIGCHLD信号会在子进程结束后发出
// init为守护进程，当子进程结束时获取子进程的结束码，通过结束码将子进程从程序表中移除
void signal_handler_init() {
    // Create a signalling mechanism for SIGCHLD.
    int s[2];
    // socketpair会返回一对文件描述符，一端写入，另一端就会被通知
    if (socketpair(AF_UNIX, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0, s) == -1) {
        ERROR("socketpair failed: %s\n", strerror(errno));
        exit(1);
    }

    signal_write_fd = s[0];
    signal_read_fd = s[1];

    // Write to signal_write_fd if we catch SIGCHLD.
    struct sigaction act;
    memset(&act, 0, sizeof(act));
    act.sa_handler = SIGCHLD_handler;
    // 表示只在子进程结束时处理，因为子进程暂停时也会发出SIGCHLD信号
    act.sa_flags = SA_NOCLDSTOP;
    // 建立信号绑定关系，收到信号时由act处理
    sigaction(SIGCHLD, &act, 0);

    reap_any_outstanding_children();
    // 注册监听当s[1]发生变化时，触发handle_signal
    register_epoll_handler(signal_read_fd, handle_signal);
}

static void handle_signal() {
    // Clear outstanding requests.
    char buf[32];
    read(signal_read_fd, buf, sizeof(buf));

    reap_any_outstanding_children();
}

static void reap_any_outstanding_children() {
    while (wait_for_one_process()) {
    }
}

static bool wait_for_one_process() {
    int status;
    // 等待子进程退出，如果没有退出返回为0，否则返回子进程pid
    pid_t pid = TEMP_FAILURE_RETRY(waitpid(-1, &status, WNOHANG));
    if (pid == 0) {
        return false;
    } else if (pid == -1) {
        ERROR("waitpid failed: %s\n", strerror(errno));
        return false;
    }

    // 通过pid查找相对的服务
    service* svc = service_find_by_pid(pid);

    std::string name;
    if (svc) {
        name = android::base::StringPrintf("Service '%s' (pid %d)", svc->name, pid);
    } else {
        name = android::base::StringPrintf("Untracked pid %d", pid);
    }

    NOTICE("%s %s\n", name.c_str(), DescribeStatus(status).c_str());

    if (!svc) {
        return true;
    }

    // TODO: all the code from here down should be a member function on service.
    // 子进程不是oneshot或者时restart，kill进程
    if (!(svc->flags & SVC_ONESHOT) || (svc->flags & SVC_RESTART)) {
        NOTICE("Service '%s' (pid %d) killing any children in process group\n", svc->name, pid);
        kill(-pid, SIGKILL);
    }

    // Remove any sockets we may have created.
    // 移除子进程相关的socket
    for (socketinfo* si = svc->sockets; si; si = si->next) {
        char tmp[128];
        snprintf(tmp, sizeof(tmp), ANDROID_SOCKET_DIR"/%s", si->name);
        unlink(tmp);
    }

    /// 当flag为exec时，释放相应的服务
    if (svc->flags & SVC_EXEC) {
        INFO("SVC_EXEC pid %d finished...\n", svc->pid);
        waiting_for_exec = false;
        list_remove(&svc->slist);
        free(svc->name);
        free(svc);
        return true;
    }

    svc->pid = 0;
    svc->flags &= (~SVC_RUNNING);

    // Oneshot processes go into the disabled state on exit,
    // except when manually restarted.
    if ((svc->flags & SVC_ONESHOT) && !(svc->flags & SVC_RESTART)) {
        svc->flags |= SVC_DISABLED;
    }

    // Disabled and reset processes do not get restarted automatically.
    if (svc->flags & (SVC_DISABLED | SVC_RESET))  {
        svc->NotifyStateChange("stopped");
        return true;
    }

    time_t now = gettime();
    if ((svc->flags & SVC_CRITICAL) && !(svc->flags & SVC_RESTART)) {
        if (svc->time_crashed + CRITICAL_CRASH_WINDOW >= now) {
            if (++svc->nr_crashed > CRITICAL_CRASH_THRESHOLD) {
                ERROR("critical process '%s' exited %d times in %d minutes; "
                      "rebooting into recovery mode\n", svc->name,
                      CRITICAL_CRASH_THRESHOLD, CRITICAL_CRASH_WINDOW / 60);
                android_reboot(ANDROID_RB_RESTART2, 0, "recovery");
                return true;
            }
        } else {
            svc->time_crashed = now;
            svc->nr_crashed = 1;
        }
    }

    svc->flags &= (~SVC_RESTART);
    svc->flags |= SVC_RESTARTING;

    // Execute all onrestart commands for this service.
    struct listnode* node;
    list_for_each(node, &svc->onrestart.commands) {
        command* cmd = node_to_item(node, struct command, clist);
        cmd->func(cmd->nargs, cmd->args);
    }
    svc->NotifyStateChange("restarting");
    return true;
}
```

> init_parse_config_file

```C++
int init_parse_config_file(const char* path) {
    INFO("Parsing %s...\n", path);
    Timer t;
    std::string data;
    // 将文件内容读取至字符串data中
    if (!read_file(path, &data)) {
        return -1;
    }

    data.push_back('\n'); // TODO: fix parse_config.
    parse_config(path, data);
    dump_parser_state();

    NOTICE("(Parsing %s took %.2fs.)\n", path, t.duration());
    return 0;
}

static void parse_config(const char *fn, const std::string& data)
{
    struct listnode import_list;
    struct listnode *node;
    char *args[INIT_PARSER_MAXARGS];

    int nargs = 0;
    // 存储文件信息
    parse_state state;
    state.filename = fn;
    state.line = 0;
    // strdup 拷贝字符串
    // NULL就是指的是0值，Null-terminated string就是以0值结尾的string，也就是以'\0'。
    // data.c_str() 返回一个指针，指向一个数组，这个数组包含一个序列的Null-terminated string，代表当前字符串的值，简单来说就是将字符串变为了character数组，指针指向数组的头部
    state.ptr = strdup(data.c_str());  // TODO: fix this code!
    state.nexttoken = 0;
    state.parse_line = parse_line_no_op;

    list_init(&import_list);
    state.priv = &import_list;

    for (;;) {
        // 解析文件
        // next_token逻辑
        // 1.值为0表示eof
        // 2.值为\n换行 newline
        // 3.\r 回车 指针++
        // 4. # newline or eof
        // 5.text
        switch (next_token(&state)) {
        // end of file
        case T_EOF:
            state.parse_line(&state, 0, 0);
            goto parser_done;
        case T_NEWLINE:
            state.line++;
            if (nargs) {
                // 对应init.rc文件中的相关指令，不同的指令，用不同的解析工具解析
                int kw = lookup_keyword(args[0]);
                // kw的类型
                // SECTION 包括 import on service
                // COMMAND
                // OPTION
                if (kw_is(kw, SECTION)) {
                    state.parse_line(&state, 0, 0);
                    // SECTION对应三种解析器 parse_import/parse_action/parse_service
                    parse_new_section(&state, kw, nargs, args);
                } else {
                    state.parse_line(&state, nargs, args);
                }
                // 将init.rc的操作收集起来
                // service_list/action_list/action_queue
                nargs = 0;
            }
            break;
        case T_TEXT:
            if (nargs < INIT_PARSER_MAXARGS) {
                // 将每个字符拼接起来
                args[nargs++] = state.text;
            }
            break;
        }
    }

parser_done:
    list_for_each(node, &import_list) {
         struct import *import = node_to_item(node, struct import, list);
         int ret;

         ret = init_parse_config_file(import->filename);
         // 批量导入init.rc中的依赖文件
         if (ret)
             ERROR("could not import file '%s' from '%s'\n",
                   import->filename, fn);
    }
}

static void parse_new_section(struct parse_state *state, int kw,
                       int nargs, char **args)
{
    printf("[ %s %s ]\n", args[0],
           nargs > 1 ? args[1] : "");
    switch(kw) {
    case K_service:
        state->context = parse_service(state, nargs, args);
        if (state->context) {
            state->parse_line = parse_line_service;
            return;
        }
        break;
    case K_on:
        state->context = parse_action(state, nargs, args);
        if (state->context) {
            state->parse_line = parse_line_action;
            return;
        }
        break;
    case K_import:
        parse_import(state, nargs, args);
        break;
    }
    state->parse_line = parse_line_no_op;
}

static void *parse_service(struct parse_state *state, int nargs, char **args)
{
    if (nargs < 3) {
        parse_error(state, "services must have a name and a program\n");
        return 0;
    }
    if (!valid_name(args[1])) {
        parse_error(state, "invalid service name '%s'\n", args[1]);
        return 0;
    }

    service* svc = (service*) service_find_by_name(args[1]);
    if (svc) {
        parse_error(state, "ignored duplicate definition of service '%s'\n", args[1]);
        return 0;
    }

    nargs -= 2;
    svc = (service*) calloc(1, sizeof(*svc) + sizeof(char*) * nargs);
    if (!svc) {
        parse_error(state, "out of memory\n");
        return 0;
    }
    svc->name = strdup(args[1]);
    svc->classname = "default";
    memcpy(svc->args, args + 2, sizeof(char*) * nargs);
    trigger* cur_trigger = (trigger*) calloc(1, sizeof(*cur_trigger));
    svc->args[nargs] = 0;
    svc->nargs = nargs;
    list_init(&svc->onrestart.triggers);
    cur_trigger->name = "onrestart";
    list_add_tail(&svc->onrestart.triggers, &cur_trigger->nlist);
    list_init(&svc->onrestart.commands);
    list_add_tail(&service_list, &svc->slist);
    return svc;
}
```

> init.rc

```C++
on early-init
# OOM_ADJ (Out of Memory Adjustment) 是android系统在内存不足情况下进行内存调整的重要参数
# Set init and its forked children's oom_adj.
write /proc/1/oom_score_adj -1000

# Set the security context of /adb_keys if present.
restorecon /adb_keys

start ueventd // 启动ueventd，前面已提及，创建设备节点，跟冷/热插拔有关
```

> excute_one_command

```C++
void execute_one_command() {
    Timer t;

    char cmd_str[256] = "";
    char name_str[256] = "";
    // cur_action = NULL cur_command = NULL init.cpp的局部变量
    if (!cur_action || !cur_command || is_last_command(cur_action, cur_command)) {
        cur_action = action_remove_queue_head();
        cur_command = NULL;
        if (!cur_action) {
            return;
        }

        build_triggers_string(name_str, sizeof(name_str), cur_action);

        INFO("processing action %p (%s)\n", cur_action, name_str);
        cur_command = get_first_command(cur_action);
    } else {
        cur_command = get_next_command(cur_action, cur_command);
    }

    if (!cur_command) {
        return;
    }
    // 服务的 func 跳转到 service_for_each_class
    int result = cur_command->func(cur_command->nargs, cur_command->args);
    if (klog_get_level() >= KLOG_INFO_LEVEL) {
        for (int i = 0; i < cur_command->nargs; i++) {
            strlcat(cmd_str, cur_command->args[i], sizeof(cmd_str));
            if (i < cur_command->nargs - 1) {
                strlcat(cmd_str, " ", sizeof(cmd_str));
            }
        }
        char source[256];
        if (cur_command->filename) {
            snprintf(source, sizeof(source), " (%s:%d)", cur_command->filename, cur_command->line);
        } else {
            *source = '\0';
        }
        INFO("Command '%s' action=%s%s returned %d took %.2fs\n",
             cmd_str, cur_action ? name_str : "", source, result, t.duration());
    }
}

// command结构体
struct command
{
        /* list of commands in an action */
    struct listnode clist;

    int (*func)(int nargs, char **args);

    int line;
    const char *filename;

    int nargs;
    char *args[1];
};

void service_for_each_class(const char *classname,
                            void (*func)(struct service *svc))
{
    struct listnode *node;
    struct service *svc;
    list_for_each(node, &service_list) {
        svc = node_to_item(node, struct service, slist);
        if (!strcmp(svc->classname, classname)) {
            func(svc);
        }
    }
}

// builtins
// KEYWORD(class_start, COMMAND, 1, do_class_start)
int do_class_start(int nargs, char **args)
{
        /* Starting a class does not start services
         * which are explicitly disabled.  They must
         * be started individually.
         */
         // 回调service_start_if_not_disabled方法
    service_for_each_class(args[1], service_start_if_not_disabled);
    return 0;
}

// KEYWORD(class_stop,  COMMAND, 1, do_class_stop)
int do_class_stop(int nargs, char **args)
{
    service_for_each_class(args[1], service_stop);
    return 0;
}

// KEYWORD(class_reset, COMMAND, 1, do_class_reset)
int do_class_reset(int nargs, char **args)
{
    service_for_each_class(args[1], service_reset);
    return 0;
}

static void service_start_if_not_disabled(struct service *svc)
{
    if (!(svc->flags & SVC_DISABLED)) {
        service_start(svc, NULL);
    } else {
        svc->flags |= SVC_DISABLED_START;
    }
}

// service启动逻辑
void service_start(struct service *svc, const char *dynamic_args)
{
    // Starting a service removes it from the disabled or reset state and
    // immediately takes it out of the restarting state if it was in there.
    svc->flags &= (~(SVC_DISABLED|SVC_RESTARTING|SVC_RESET|SVC_RESTART|SVC_DISABLED_START));
    svc->time_started = 0;

    // Running processes require no additional work --- if they're in the
    // process of exiting, we've ensured that they will immediately restart
    // on exit, unless they are ONESHOT.
    if (svc->flags & SVC_RUNNING) {
        return;
    }

    bool needs_console = (svc->flags & SVC_CONSOLE);
    if (needs_console && !have_console) {
        ERROR("service '%s' requires console\n", svc->name);
        svc->flags |= SVC_DISABLED;
        return;
    }

    struct stat s;
    if (stat(svc->args[0], &s) != 0) {
        ERROR("cannot find '%s', disabling '%s'\n", svc->args[0], svc->name);
        svc->flags |= SVC_DISABLED;
        return;
    }

    if ((!(svc->flags & SVC_ONESHOT)) && dynamic_args) {
        ERROR("service '%s' must be one-shot to use dynamic args, disabling\n",
               svc->args[0]);
        svc->flags |= SVC_DISABLED;
        return;
    }

    char* scon = NULL;
    // 安全检测
    if (is_selinux_enabled() > 0) {
        if (svc->seclabel) {
            scon = strdup(svc->seclabel);
            if (!scon) {
                ERROR("Out of memory while starting '%s'\n", svc->name);
                return;
            }
        } else {
            char *mycon = NULL, *fcon = NULL;

            INFO("computing context for service '%s'\n", svc->args[0]);
            int rc = getcon(&mycon);
            if (rc < 0) {
                ERROR("could not get context while starting '%s'\n", svc->name);
                return;
            }

            rc = getfilecon(svc->args[0], &fcon);
            if (rc < 0) {
                ERROR("could not get context while starting '%s'\n", svc->name);
                freecon(mycon);
                return;
            }

            rc = security_compute_create(mycon, fcon, string_to_security_class("process"), &scon);
            if (rc == 0 && !strcmp(scon, mycon)) {
                ERROR("Warning!  Service %s needs a SELinux domain defined; please fix!\n", svc->name);
            }
            freecon(mycon);
            freecon(fcon);
            if (rc < 0) {
                ERROR("could not get context while starting '%s'\n", svc->name);
                return;
            }
        }
    }

    NOTICE("Starting service '%s'...\n", svc->name);

    // 前面一系列校验
    // fork()
    // 一个进程调用fork函数后，系统会给新的进程分配资源，例如存储空间和代码空间。把原进程的值复制到新进程中，相当于克隆了一个自己。
    pid_t pid = fork();
    if (pid == 0) {
        struct socketinfo *si;
        struct svcenvinfo *ei;
        char tmp[32];
        int fd, sz;

        umask(077);
        if (properties_initialized()) {
            get_property_workspace(&fd, &sz);
            snprintf(tmp, sizeof(tmp), "%d,%d", dup(fd), sz);
            add_environment("ANDROID_PROPERTY_WORKSPACE", tmp);
        }

        for (ei = svc->envvars; ei; ei = ei->next)
            add_environment(ei->name, ei->value);

        for (si = svc->sockets; si; si = si->next) {
            int socket_type = (
                    !strcmp(si->type, "stream") ? SOCK_STREAM :
                        (!strcmp(si->type, "dgram") ? SOCK_DGRAM : SOCK_SEQPACKET));
            int s = create_socket(si->name, socket_type,
                                  si->perm, si->uid, si->gid, si->socketcon ?: scon);
            if (s >= 0) {
                publish_socket(si->name, s);
            }
        }

        freecon(scon);
        scon = NULL;

        if (svc->writepid_files_) {
            std::string pid_str = android::base::StringPrintf("%d", pid);
            for (auto& file : *svc->writepid_files_) {
                if (!android::base::WriteStringToFile(pid_str, file)) {
                    ERROR("couldn't write %s to %s: %s\n",
                          pid_str.c_str(), file.c_str(), strerror(errno));
                }
            }
        }

        if (svc->ioprio_class != IoSchedClass_NONE) {
            if (android_set_ioprio(getpid(), svc->ioprio_class, svc->ioprio_pri)) {
                ERROR("Failed to set pid %d ioprio = %d,%d: %s\n",
                      getpid(), svc->ioprio_class, svc->ioprio_pri, strerror(errno));
            }
        }

        if (needs_console) {
            setsid();
            open_console();
        } else {
            zap_stdio();
        }

        if (false) {
            for (size_t n = 0; svc->args[n]; n++) {
                INFO("args[%zu] = '%s'\n", n, svc->args[n]);
            }
            for (size_t n = 0; ENV[n]; n++) {
                INFO("env[%zu] = '%s'\n", n, ENV[n]);
            }
        }

        setpgid(0, getpid());

        // As requested, set our gid, supplemental gids, and uid.
        if (svc->gid) {
            if (setgid(svc->gid) != 0) {
                ERROR("setgid failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (svc->nr_supp_gids) {
            if (setgroups(svc->nr_supp_gids, svc->supp_gids) != 0) {
                ERROR("setgroups failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (svc->uid) {
            if (setuid(svc->uid) != 0) {
                ERROR("setuid failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (svc->seclabel) {
            if (is_selinux_enabled() > 0 && setexeccon(svc->seclabel) < 0) {
                ERROR("cannot setexeccon('%s'): %s\n", svc->seclabel, strerror(errno));
                _exit(127);
            }
        }

        if (!dynamic_args) {
            if (execve(svc->args[0], (char**) svc->args, (char**) ENV) < 0) {
                ERROR("cannot execve('%s'): %s\n", svc->args[0], strerror(errno));
            }
        } else {
            char *arg_ptrs[INIT_PARSER_MAXARGS+1];
            int arg_idx = svc->nargs;
            char *tmp = strdup(dynamic_args);
            char *next = tmp;
            char *bword;

            /* Copy the static arguments */
            memcpy(arg_ptrs, svc->args, (svc->nargs * sizeof(char *)));

            while((bword = strsep(&next, " "))) {
                arg_ptrs[arg_idx++] = bword;
                if (arg_idx == INIT_PARSER_MAXARGS)
                    break;
            }
            arg_ptrs[arg_idx] = NULL;
            //在父进程中fork一个子进程，在子进程中调用exec函数启动新的程序。exec函数一共有六个，其中execve为内核级系统调用，其他（execl，execle，execlp，execv，execvp）都是调用execve的库函数。
            execve(svc->args[0], (char**) arg_ptrs, (char**) ENV);
        }
        _exit(127);
    }

    freecon(scon);

    if (pid < 0) {
        ERROR("failed to start '%s'\n", svc->name);
        svc->pid = 0;
        return;
    }

    svc->time_started = gettime();
    svc->pid = pid;
    svc->flags |= SVC_RUNNING;

    if ((svc->flags & SVC_EXEC) != 0) {
        INFO("SVC_EXEC pid %d (uid %d gid %d+%zu context %s) started; waiting...\n",
             svc->pid, svc->uid, svc->gid, svc->nr_supp_gids,
             svc->seclabel ? : "default");
        waiting_for_exec = true;
    }

    svc->NotifyStateChange("running");
}
```

### 总结

init.cpp的功能

- 创建目录,挂载分区
- 解析启动脚本（init.rc）
- 启动脚本中的服务 (init.rc中的各种服务 zygote servicemanager等)
- 开启守护进程监听

### 参考资料

[uevent的解释]("https://www.jianshu.com/p/10653d83909d")  
[uevent的启动分析]("https://www.jianshu.com/p/8a34ba82ac1f")  
[umask函数分析]("https://blog.csdn.net/guaiguaihenguai/article/details/79934142")  
[Linux内核中的kobject和kset介绍]("https://blog.csdn.net/jasonchen_gbd/article/details/78013643")  
[SELinux]("https://source.android.com/security/selinux")
[关于linux epoll功能解释]("https://blog.csdn.net/lixungogogo/article/details/52226479")
[Android的OOM_ADJ]("https://www.jianshu.com/p/8897b7e47466")  
[linux系统编程之进程（五）：exec系列函数（execl,execlp,execle,execv,execvp)使用]("https://www.cnblogs.com/mickole/p/3187409.html")  
[(连载)Android 8.0 : 系统启动流程之init进程(二)](https://juejin.im/post/5a9c900ff265da2380591cee)
