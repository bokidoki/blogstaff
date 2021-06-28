---
title: Binder_Driver
categories: Android
top: 0
date: 2020-05-29 22:22:22
tags:
thumbnail:
---

> 从Binder驱动分析用户空间->内核空间的ioctl逻辑

## ioctl函数

```cpp
// [目录 /linux/drivers/android/binder.c]
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;
    /*pr_info("binder_ioctl: %d:%d %x %lx\n",
            proc->pid, current->pid, cmd, arg);*/
    binder_selftest_alloc(&proc->alloc);
    trace_binder_ioctl(cmd, arg);
    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret)
        goto err_unlocked;
    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    }
    switch (cmd) {
    case BINDER_WRITE_READ:
        // 接收到此信号 binder开始做读写操作
        ret = binder_ioctl_write_read(filp, cmd, arg, thread); // 注释1
        if (ret)
            goto err;
        break;
    case BINDER_SET_MAX_THREADS: {
        int max_threads;
        if (copy_from_user(&max_threads, ubuf,
                   sizeof(max_threads))) {
            ret = -EINVAL;
            goto err;
        }
        binder_inner_proc_lock(proc);
        proc->max_threads = max_threads;
        binder_inner_proc_unlock(proc);
        break;
    }
    case BINDER_SET_CONTEXT_MGR_EXT: {
        struct flat_binder_object fbo;
        if (copy_from_user(&fbo, ubuf, sizeof(fbo))) {
            ret = -EINVAL;
            goto err;
        }
        ret = binder_ioctl_set_ctx_mgr(filp, &fbo);
        if (ret)
            goto err;
        break;
    }
    case BINDER_SET_CONTEXT_MGR:
        ret = binder_ioctl_set_ctx_mgr(filp, NULL);
        if (ret)
            goto err;
        break;
    case BINDER_THREAD_EXIT:
        binder_debug(BINDER_DEBUG_THREADS, "%d:%d exit\n",
                 proc->pid, thread->pid);
        binder_thread_release(proc, thread);
        thread = NULL;
        break;
    case BINDER_VERSION: {
        struct binder_version __user *ver = ubuf;
        if (size != sizeof(struct binder_version)) {
            ret = -EINVAL;
            goto err;
        }
        if (put_user(BINDER_CURRENT_PROTOCOL_VERSION,
                 &ver->protocol_version)) {
            ret = -EINVAL;
            goto err;
        }
        break;
    }
    case BINDER_GET_NODE_INFO_FOR_REF: {
        struct binder_node_info_for_ref info;
        if (copy_from_user(&info, ubuf, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }
        ret = binder_ioctl_get_node_info_for_ref(proc, &info);
        if (ret < 0)
            goto err;
        if (copy_to_user(ubuf, &info, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }
        break;
    }
    case BINDER_GET_NODE_DEBUG_INFO: {
        struct binder_node_debug_info info;
        if (copy_from_user(&info, ubuf, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }
        ret = binder_ioctl_get_node_debug_info(proc, &info);
        if (ret < 0)
            goto err;
        if (copy_to_user(ubuf, &info, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }
        break;
    }
    default:
        ret = -EINVAL;
        goto err;
    }
    ret = 0;
err:
    if (thread)
        thread->looper_need_return = false;
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret && ret != -ERESTARTSYS)
        pr_info("%d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
    trace_binder_ioctl_done(ret);
    return ret;
}

// 源码地址[https://code.woboq.org/linux/linux/drivers/android/binder.c.html#binder_ioctl_write_read]
static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;
    if (size != sizeof(struct binder_write_read)) {
        ret = -EINVAL;
        goto out;
    }
    // unsigned long copy_from_user(void *to, const void __user *from, unsigned long n) 负责将数据从用户空间传递到内核空间
    // 将用户空间数据ubuf拷贝至&bwr
    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
    binder_debug(BINDER_DEBUG_READ_WRITE,
             "%d:%d write %lld at %016llx, read %lld at %016llx\n",
             proc->pid, thread->pid,
             (u64)bwr.write_size, (u64)bwr.write_buffer,
             (u64)bwr.read_size, (u64)bwr.read_buffer);
    // bwr.write_size对应mOut的数据，大于0表示有数据从用户空间进入内核空间
    if (bwr.write_size > 0) {
        // 注释2
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        trace_binder_write_done(ret);
        if (ret < 0) {
            bwr.read_consumed = 0;
            // unsigned long copy_to_user(void __user *to, const void *from, unsigned long n) 负责将数据从内核空间传递到用户空间
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    if (bwr.read_size > 0) {
        // 注释3
        ret = binder_thread_read(proc, thread, bwr.read_buffer,
                     bwr.read_size,
                     &bwr.read_consumed,
                     filp->f_flags & O_NONBLOCK);
        trace_binder_read_done(ret);
        binder_inner_proc_lock(proc);
        if (!binder_worklist_empty_ilocked(&proc->todo))
            binder_wakeup_proc_ilocked(proc);
        binder_inner_proc_unlock(proc);
        if (ret < 0) {
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    binder_debug(BINDER_DEBUG_READ_WRITE,
             "%d:%d wrote %lld of %lld, read return %lld of %lld\n",
             proc->pid, thread->pid,
             (u64)bwr.write_consumed, (u64)bwr.write_size,
             (u64)bwr.read_consumed, (u64)bwr.read_size);
    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}

// 接注释2 [源码目录 https://code.woboq.org/linux/linux/drivers/android/binder.c.html#binder_thread_write]
// binder_thread_write(proc, thread, bwr.write_buffer, bwr.write_size, &bwr.write_consumed);
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
        // 省略...
        // 注释3
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;
            // 将参数从用户空间拷贝至内核空间（一次拷贝）
            if (copy_from_user(&tr, ptr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);
            // 调用binder_transaction函数，有一个struct也是binder_transaction
            binder_transaction(proc, thread, &tr,
                       cmd == BC_REPLY, 0);
            break;
        }
        // 省略...
    return 0;
}

// 接注释3
// binder_transaction(proc, thread, &tr, cmd == BC_REPLY, 0);
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply,
                   binder_size_t extra_buffers_size)
{
    // 省略...
    if (reply) {
        // cmd == BC_TRANSACTION 所以reply = 0
    } else {
        if (tr->target.handle) {
            // tr->target.handle = 0
        } else {
            mutex_lock(&context->context_mgr_node_lock);
            target_node = context->binder_context_mgr_node;
            if (target_node)
                target_node = binder_get_node_refs_for_txn(
                        target_node, &target_proc,
                        &return_error);
            else
                return_error = BR_DEAD_REPLY;
            mutex_unlock(&context->context_mgr_node_lock);
            if (target_node && target_proc == proc) {
                binder_user_error("%d:%d got transaction to context manager from process owning it\n",
                          proc->pid, thread->pid);
                return_error = BR_FAILED_REPLY;
                return_error_param = -EINVAL;
                return_error_line = __LINE__;
                goto err_invalid_target_handle;
            }
        }
        if (!target_node) {
            /*
             * return_error is set above
             */
            return_error_param = -EINVAL;
            return_error_line = __LINE__;
            goto err_dead_binder;
        }
        e->to_node = target_node->debug_id;
        if (security_binder_transaction(proc->tsk,
                        target_proc->tsk) < 0) {
            return_error = BR_FAILED_REPLY;
            return_error_param = -EPERM;
            return_error_line = __LINE__;
            goto err_invalid_target_handle;
        }
        binder_inner_proc_lock(proc);
        w = list_first_entry_or_null(&thread->todo,
                         struct binder_work, entry);
        if (!(tr->flags & TF_ONE_WAY) && w &&
            w->type == BINDER_WORK_TRANSACTION) {
            /*
             * Do not allow new outgoing transaction from a
             * thread that has a transaction at the head of
             * its todo list. Only need to check the head
             * because binder_select_thread_ilocked picks a
             * thread from proc->waiting_threads to enqueue
             * the transaction, and nothing is queued to the
             * todo list while the thread is on waiting_threads.
             */
            binder_user_error("%d:%d new transaction not allowed when there is a transaction on thread todo\n",
                      proc->pid, thread->pid);
            binder_inner_proc_unlock(proc);
            return_error = BR_FAILED_REPLY;
            return_error_param = -EPROTO;
            return_error_line = __LINE__;
            goto err_bad_todo_list;
        }
        if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
            struct binder_transaction *tmp;
            tmp = thread->transaction_stack;
            if (tmp->to_thread != thread) {
                spin_lock(&tmp->lock);
                binder_user_error("%d:%d got new transaction with bad transaction stack, transaction %d has target %d:%d\n",
                    proc->pid, thread->pid, tmp->debug_id,
                    tmp->to_proc ? tmp->to_proc->pid : 0,
                    tmp->to_thread ?
                    tmp->to_thread->pid : 0);
                spin_unlock(&tmp->lock);
                binder_inner_proc_unlock(proc);
                return_error = BR_FAILED_REPLY;
                return_error_param = -EPROTO;
                return_error_line = __LINE__;
                goto err_bad_call_stack;
            }
            while (tmp) {
                struct binder_thread *from;
                spin_lock(&tmp->lock);
                from = tmp->from;
                if (from && from->proc == target_proc) {
                    atomic_inc(&from->tmp_ref);
                    target_thread = from;
                    spin_unlock(&tmp->lock);
                    break;
                }
                spin_unlock(&tmp->lock);
                tmp = tmp->from_parent;
            }
        }
        binder_inner_proc_unlock(proc);
    }
    if (target_thread)
        e->to_thread = target_thread->pid;
    e->to_proc = target_proc->pid;
    /* TODO: reuse incoming transaction for reply */
    t = kzalloc(sizeof(*t), GFP_KERNEL);
    if (t == NULL) {
        return_error = BR_FAILED_REPLY;
        return_error_param = -ENOMEM;
        return_error_line = __LINE__;
        goto err_alloc_t_failed;
    }
    INIT_LIST_HEAD(&t->fd_fixups);
    binder_stats_created(BINDER_STAT_TRANSACTION);
    spin_lock_init(&t->lock);
    tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    if (tcomplete == NULL) {
        return_error = BR_FAILED_REPLY;
        return_error_param = -ENOMEM;
        return_error_line = __LINE__;
        goto err_alloc_tcomplete_failed;
    }
    binder_stats_created(BINDER_STAT_TRANSACTION_COMPLETE);
    t->debug_id = t_debug_id;
    if (reply)
        binder_debug(BINDER_DEBUG_TRANSACTION,
                 "%d:%d BC_REPLY %d -> %d:%d, data %016llx-%016llx size %lld-%lld-%lld\n",
                 proc->pid, thread->pid, t->debug_id,
                 target_proc->pid, target_thread->pid,
                 (u64)tr->data.ptr.buffer,
                 (u64)tr->data.ptr.offsets,
                 (u64)tr->data_size, (u64)tr->offsets_size,
                 (u64)extra_buffers_size);
    else
        binder_debug(BINDER_DEBUG_TRANSACTION,
                 "%d:%d BC_TRANSACTION %d -> %d - node %d, data %016llx-%016llx size %lld-%lld-%lld\n",
                 proc->pid, thread->pid, t->debug_id,
                 target_proc->pid, target_node->debug_id,
                 (u64)tr->data.ptr.buffer,
                 (u64)tr->data.ptr.offsets,
                 (u64)tr->data_size, (u64)tr->offsets_size,
                 (u64)extra_buffers_size);
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else
        t->from = NULL;
    t->sender_euid = task_euid(proc->tsk);
    t->to_proc = target_proc;
    t->to_thread = target_thread;
    t->code = tr->code;
    t->flags = tr->flags;
    t->priority = task_nice(current);
    if (target_node && target_node->txn_security_ctx) {
        u32 secid;
        security_task_getsecid(proc->tsk, &secid);
        ret = security_secid_to_secctx(secid, &secctx, &secctx_sz);
        if (ret) {
            return_error = BR_FAILED_REPLY;
            return_error_param = ret;
            return_error_line = __LINE__;
            goto err_get_secctx_failed;
        }
        extra_buffers_size += ALIGN(secctx_sz, sizeof(u64));
    }
    trace_binder_transaction(reply, t, target_node);
    t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size,
        tr->offsets_size, extra_buffers_size,
        !reply && (t->flags & TF_ONE_WAY));
    if (IS_ERR(t->buffer)) {
        /*
         * -ESRCH indicates VMA cleared. The target is dying.
         */
        return_error_param = PTR_ERR(t->buffer);
        return_error = return_error_param == -ESRCH ?
            BR_DEAD_REPLY : BR_FAILED_REPLY;
        return_error_line = __LINE__;
        t->buffer = NULL;
        goto err_binder_alloc_buf_failed;
    }
    if (secctx) {
        size_t buf_offset = ALIGN(tr->data_size, sizeof(void *)) +
                    ALIGN(tr->offsets_size, sizeof(void *)) +
                    ALIGN(extra_buffers_size, sizeof(void *)) -
                    ALIGN(secctx_sz, sizeof(u64));
        t->security_ctx = (uintptr_t)t->buffer->user_data + buf_offset;
        binder_alloc_copy_to_buffer(&target_proc->alloc,
                        t->buffer, buf_offset,
                        secctx, secctx_sz);
        security_release_secctx(secctx, secctx_sz);
        secctx = NULL;
    }
    t->buffer->debug_id = t->debug_id;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;
    trace_binder_transaction_alloc_buf(t->buffer);
    if (binder_alloc_copy_user_to_buffer(
                &target_proc->alloc,
                t->buffer, 0,
                (const void __user *)
                    (uintptr_t)tr->data.ptr.buffer,
                tr->data_size)) {
        binder_user_error("%d:%d got transaction with invalid data ptr\n",
                proc->pid, thread->pid);
        return_error = BR_FAILED_REPLY;
        return_error_param = -EFAULT;
        return_error_line = __LINE__;
        goto err_copy_data_failed;
    }
    if (binder_alloc_copy_user_to_buffer(
                &target_proc->alloc,
                t->buffer,
                ALIGN(tr->data_size, sizeof(void *)),
                (const void __user *)
                    (uintptr_t)tr->data.ptr.offsets,
                tr->offsets_size)) {
        binder_user_error("%d:%d got transaction with invalid offsets ptr\n",
                proc->pid, thread->pid);
        return_error = BR_FAILED_REPLY;
        return_error_param = -EFAULT;
        return_error_line = __LINE__;
        goto err_copy_data_failed;
    }
    if (!IS_ALIGNED(tr->offsets_size, sizeof(binder_size_t))) {
        binder_user_error("%d:%d got transaction with invalid offsets size, %lld\n",
                proc->pid, thread->pid, (u64)tr->offsets_size);
        return_error = BR_FAILED_REPLY;
        return_error_param = -EINVAL;
        return_error_line = __LINE__;
        goto err_bad_offset;
    }
    if (!IS_ALIGNED(extra_buffers_size, sizeof(u64))) {
        binder_user_error("%d:%d got transaction with unaligned buffers size, %lld\n",
                  proc->pid, thread->pid,
                  (u64)extra_buffers_size);
        return_error = BR_FAILED_REPLY;
        return_error_param = -EINVAL;
        return_error_line = __LINE__;
        goto err_bad_offset;
    }
    off_start_offset = ALIGN(tr->data_size, sizeof(void *));
    buffer_offset = off_start_offset;
    off_end_offset = off_start_offset + tr->offsets_size;
    sg_buf_offset = ALIGN(off_end_offset, sizeof(void *));
    sg_buf_end_offset = sg_buf_offset + extra_buffers_size;
    off_min = 0;
    for (buffer_offset = off_start_offset; buffer_offset < off_end_offset;
         buffer_offset += sizeof(binder_size_t)) {
        struct binder_object_header *hdr;
        size_t object_size;
        struct binder_object object;
        binder_size_t object_offset;
        binder_alloc_copy_from_buffer(&target_proc->alloc,
                          &object_offset,
                          t->buffer,
                          buffer_offset,
                          sizeof(object_offset));
        object_size = binder_get_object(target_proc, t->buffer,
                        object_offset, &object);
        if (object_size == 0 || object_offset < off_min) {
            binder_user_error("%d:%d got transaction with invalid offset (%lld, min %lld max %lld) or object.\n",
                      proc->pid, thread->pid,
                      (u64)object_offset,
                      (u64)off_min,
                      (u64)t->buffer->data_size);
            return_error = BR_FAILED_REPLY;
            return_error_param = -EINVAL;
            return_error_line = __LINE__;
            goto err_bad_offset;
        }
        hdr = &object.hdr;
        off_min = object_offset + object_size;
        switch (hdr->type) {
        case BINDER_TYPE_BINDER:
        case BINDER_TYPE_WEAK_BINDER: {
            struct flat_binder_object *fp;
            fp = to_flat_binder_object(hdr);
            ret = binder_translate_binder(fp, t, thread);
            if (ret < 0) {
                return_error = BR_FAILED_REPLY;
                return_error_param = ret;
                return_error_line = __LINE__;
                goto err_translate_failed;
            }
            binder_alloc_copy_to_buffer(&target_proc->alloc,
                            t->buffer, object_offset,
                            fp, sizeof(*fp));
        } break;
        case BINDER_TYPE_HANDLE:
        case BINDER_TYPE_WEAK_HANDLE: {
            struct flat_binder_object *fp;
            fp = to_flat_binder_object(hdr);
            ret = binder_translate_handle(fp, t, thread);
            if (ret < 0) {
                return_error = BR_FAILED_REPLY;
                return_error_param = ret;
                return_error_line = __LINE__;
                goto err_translate_failed;
            }
            binder_alloc_copy_to_buffer(&target_proc->alloc,
                            t->buffer, object_offset,
                            fp, sizeof(*fp));
        } break;
        case BINDER_TYPE_FD: {
            struct binder_fd_object *fp = to_binder_fd_object(hdr);
            binder_size_t fd_offset = object_offset +
                (uintptr_t)&fp->fd - (uintptr_t)fp;
            int ret = binder_translate_fd(fp->fd, fd_offset, t,
                              thread, in_reply_to);
            if (ret < 0) {
                return_error = BR_FAILED_REPLY;
                return_error_param = ret;
                return_error_line = __LINE__;
                goto err_translate_failed;
            }
            fp->pad_binder = 0;
            binder_alloc_copy_to_buffer(&target_proc->alloc,
                            t->buffer, object_offset,
                            fp, sizeof(*fp));
        } break;
        case BINDER_TYPE_FDA: {
            struct binder_object ptr_object;
            binder_size_t parent_offset;
            struct binder_fd_array_object *fda =
                to_binder_fd_array_object(hdr);
            size_t num_valid = (buffer_offset - off_start_offset) *
                        sizeof(binder_size_t);
            struct binder_buffer_object *parent =
                binder_validate_ptr(target_proc, t->buffer,
                            &ptr_object, fda->parent,
                            off_start_offset,
                            &parent_offset,
                            num_valid);
            if (!parent) {
                binder_user_error("%d:%d got transaction with invalid parent offset or type\n",
                          proc->pid, thread->pid);
                return_error = BR_FAILED_REPLY;
                return_error_param = -EINVAL;
                return_error_line = __LINE__;
                goto err_bad_parent;
            }
            if (!binder_validate_fixup(target_proc, t->buffer,
                           off_start_offset,
                           parent_offset,
                           fda->parent_offset,
                           last_fixup_obj_off,
                           last_fixup_min_off)) {
                binder_user_error("%d:%d got transaction with out-of-order buffer fixup\n",
                          proc->pid, thread->pid);
                return_error = BR_FAILED_REPLY;
                return_error_param = -EINVAL;
                return_error_line = __LINE__;
                goto err_bad_parent;
            }
            ret = binder_translate_fd_array(fda, parent, t, thread,
                            in_reply_to);
            if (ret < 0) {
                return_error = BR_FAILED_REPLY;
                return_error_param = ret;
                return_error_line = __LINE__;
                goto err_translate_failed;
            }
            last_fixup_obj_off = parent_offset;
            last_fixup_min_off =
                fda->parent_offset + sizeof(u32) * fda->num_fds;
        } break;
        case BINDER_TYPE_PTR: {
            struct binder_buffer_object *bp =
                to_binder_buffer_object(hdr);
            size_t buf_left = sg_buf_end_offset - sg_buf_offset;
            size_t num_valid;
            if (bp->length > buf_left) {
                binder_user_error("%d:%d got transaction with too large buffer\n",
                          proc->pid, thread->pid);
                return_error = BR_FAILED_REPLY;
                return_error_param = -EINVAL;
                return_error_line = __LINE__;
                goto err_bad_offset;
            }
            if (binder_alloc_copy_user_to_buffer(
                        &target_proc->alloc,
                        t->buffer,
                        sg_buf_offset,
                        (const void __user *)
                            (uintptr_t)bp->buffer,
                        bp->length)) {
                binder_user_error("%d:%d got transaction with invalid offsets ptr\n",
                          proc->pid, thread->pid);
                return_error_param = -EFAULT;
                return_error = BR_FAILED_REPLY;
                return_error_line = __LINE__;
                goto err_copy_data_failed;
            }
            /* Fixup buffer pointer to target proc address space */
            bp->buffer = (uintptr_t)
                t->buffer->user_data + sg_buf_offset;
            sg_buf_offset += ALIGN(bp->length, sizeof(u64));
            num_valid = (buffer_offset - off_start_offset) *
                    sizeof(binder_size_t);
            ret = binder_fixup_parent(t, thread, bp,
                          off_start_offset,
                          num_valid,
                          last_fixup_obj_off,
                          last_fixup_min_off);
            if (ret < 0) {
                return_error = BR_FAILED_REPLY;
                return_error_param = ret;
                return_error_line = __LINE__;
                goto err_translate_failed;
            }
            binder_alloc_copy_to_buffer(&target_proc->alloc,
                            t->buffer, object_offset,
                            bp, sizeof(*bp));
            last_fixup_obj_off = object_offset;
            last_fixup_min_off = 0;
        } break;
        default:
            binder_user_error("%d:%d got transaction with invalid object type, %x\n",
                proc->pid, thread->pid, hdr->type);
            return_error = BR_FAILED_REPLY;
            return_error_param = -EINVAL;
            return_error_line = __LINE__;
            goto err_bad_object_type;
        }
    }
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    t->work.type = BINDER_WORK_TRANSACTION;
    if (reply) {
        binder_enqueue_thread_work(thread, tcomplete);
        binder_inner_proc_lock(target_proc);
        if (target_thread->is_dead) {
            binder_inner_proc_unlock(target_proc);
            goto err_dead_proc_or_thread;
        }
        BUG_ON(t->buffer->async_transaction != 0);
        binder_pop_transaction_ilocked(target_thread, in_reply_to);
        binder_enqueue_thread_work_ilocked(target_thread, &t->work);
        binder_inner_proc_unlock(target_proc);
        wake_up_interruptible_sync(&target_thread->wait);
        binder_free_transaction(in_reply_to);
    } else if (!(t->flags & TF_ONE_WAY)) {
        BUG_ON(t->buffer->async_transaction != 0);
        binder_inner_proc_lock(proc);
        /*
         * Defer the TRANSACTION_COMPLETE, so we don't return to
         * userspace immediately; this allows the target process to
         * immediately start processing this transaction, reducing
         * latency. We will then return the TRANSACTION_COMPLETE when
         * the target replies (or there is an error).
         */
        binder_enqueue_deferred_thread_work_ilocked(thread, tcomplete);
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        thread->transaction_stack = t;
        binder_inner_proc_unlock(proc);
        if (!binder_proc_transaction(t, target_proc, target_thread)) {
            binder_inner_proc_lock(proc);
            binder_pop_transaction_ilocked(thread, t);
            binder_inner_proc_unlock(proc);
            goto err_dead_proc_or_thread;
        }
    } else {
        BUG_ON(target_node == NULL);
        BUG_ON(t->buffer->async_transaction != 1);
        binder_enqueue_thread_work(thread, tcomplete);
        if (!binder_proc_transaction(t, target_proc, NULL))
            goto err_dead_proc_or_thread;
    }
    if (target_thread)
        binder_thread_dec_tmpref(target_thread);
    binder_proc_dec_tmpref(target_proc);
    if (target_node)
        binder_dec_node_tmpref(target_node);
    /*
     * write barrier to synchronize with initialization
     * of log entry
     */
    smp_wmb();
    WRITE_ONCE(e->debug_id_done, t_debug_id);
    return;
err_dead_proc_or_thread:
    return_error = BR_DEAD_REPLY;
    return_error_line = __LINE__;
    binder_dequeue_work(proc, tcomplete);
err_translate_failed:
err_bad_object_type:
err_bad_offset:
err_bad_parent:
err_copy_data_failed:
    binder_free_txn_fixups(t);
    trace_binder_transaction_failed_buffer_release(t->buffer);
    binder_transaction_buffer_release(target_proc, t->buffer,
                      buffer_offset, true);
    if (target_node)
        binder_dec_node_tmpref(target_node);
    target_node = NULL;
    t->buffer->transaction = NULL;
    binder_alloc_free_buf(&target_proc->alloc, t->buffer);
err_binder_alloc_buf_failed:
    if (secctx)
        security_release_secctx(secctx, secctx_sz);
err_get_secctx_failed:
    kfree(tcomplete);
    binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
err_alloc_tcomplete_failed:
    kfree(t);
    binder_stats_deleted(BINDER_STAT_TRANSACTION);
err_alloc_t_failed:
err_bad_todo_list:
err_bad_call_stack:
err_empty_call_stack:
err_dead_binder:
err_invalid_target_handle:
    if (target_thread)
        binder_thread_dec_tmpref(target_thread);
    if (target_proc)
        binder_proc_dec_tmpref(target_proc);
    if (target_node) {
        binder_dec_node(target_node, 1, 0);
        binder_dec_node_tmpref(target_node);
    }
    binder_debug(BINDER_DEBUG_FAILED_TRANSACTION,
             "%d:%d transaction failed %d/%d, size %lld-%lld line %d\n",
             proc->pid, thread->pid, return_error, return_error_param,
             (u64)tr->data_size, (u64)tr->offsets_size,
             return_error_line);
    {
        struct binder_transaction_log_entry *fe;
        e->return_error = return_error;
        e->return_error_param = return_error_param;
        e->return_error_line = return_error_line;
        fe = binder_transaction_log_add(&binder_transaction_log_failed);
        *fe = *e;
        /*
         * write barrier to synchronize with initialization
         * of log entry
         */
        smp_wmb();
        WRITE_ONCE(e->debug_id_done, t_debug_id);
        WRITE_ONCE(fe->debug_id_done, t_debug_id);
    }
    BUG_ON(thread->return_error.cmd != BR_OK);
    if (in_reply_to) {
        thread->return_error.cmd = BR_TRANSACTION_COMPLETE;
        binder_enqueue_thread_work(thread, &thread->return_error.work);
        binder_send_failed_reply(in_reply_to, return_error);
    } else {
        thread->return_error.cmd = return_error;
        binder_enqueue_thread_work(thread, &thread->return_error.work);
    }
}

```

### 涉及的结构体

```cpp
// file结构体
struct file {
    union {
        struct llist_node    fu_llist;
        struct rcu_head     fu_rcuhead;
    } f_u;
    struct path        f_path;
    struct inode        *f_inode;    /* cached value */
    const struct file_operations    *f_op;
    /*
     * Protects f_ep_links, f_flags.
     * Must not be taken from IRQ context.
     */
    spinlock_t        f_lock;
    enum rw_hint        f_write_hint;
    atomic_long_t        f_count;
    unsigned int         f_flags;
    fmode_t            f_mode;
    struct mutex        f_pos_lock;
    loff_t            f_pos;
    struct fown_struct    f_owner;
    const struct cred    *f_cred;
    struct file_ra_state    f_ra;
    u64            f_version;
#ifdef CONFIG_SECURITY
    void            *f_security;
#endif
    /* needed for tty driver, and maybe others */
    // binder_proc结构体
    void            *private_data;
#ifdef CONFIG_EPOLL
    /* Used by fs/eventpoll.c to link all the hooks to this file */
    struct list_head    f_ep_links;
    struct list_head    f_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
    struct address_space    *f_mapping;
    errseq_t        f_wb_err;
} __randomize_layout
  __attribute__((aligned(4)));    /* lest something weird decides that 2 is OK */

/*
 * On 64-bit platforms where user code may run in 32-bits the driver must
 * translate the buffer (and local binder) addresses appropriately.
 */
 // binder的传输对象
struct binder_write_read {
    binder_size_t        write_size;    /* bytes to write */
    binder_size_t        write_consumed;    /* bytes consumed by driver */
    binder_uintptr_t    write_buffer;
    binder_size_t        read_size;    /* bytes to read */
    binder_size_t        read_consumed;    /* bytes consumed by driver */
    binder_uintptr_t    read_buffer;
};

// 记录binder进程的相关信息
struct binder_proc {
    // 当前线程的相关binder进程？
    struct hlist_node proc_node;
    // 进程中的binder_threads，红黑树结构
    struct rb_root threads;
    // 与当前进程相关的binder节点以node->ptr排序
    struct rb_root nodes;
    // rbtree of refs ordered by ref->desc
    struct rb_root refs_by_desc;
    // rbtree of refs ordered by ref->node
    struct rb_root refs_by_node;
    // 等待进程工作的线程
    struct list_head waiting_threads;
    int pid;
    // task_struct for group_leader of process
    struct task_struct *tsk;
    // element for binder_deferred_list
    struct hlist_node deferred_work_node;
    // bitmap of deferred work to perform
    int deferred_work;
    bool is_dead;
    // 当前线程将要做的工作
    struct list_head todo;
    struct binder_stats stats;
    struct list_head delivered_death;
    int max_threads;
    int requested_threads;
    int requested_threads_started;
    int tmp_ref;
    long default_priority;
    struct dentry *debugfs_entry;
    struct binder_alloc alloc;
    struct binder_context *context;
    spinlock_t inner_lock;
    spinlock_t outer_lock;
};

struct hlist_node {
    // 一个双向链表结构
    struct hlist_node *next, **pprev;
};

// 红黑树结构
struct rb_node {
    unsigned long  __rb_parent_color;
    struct rb_node *rb_right;
    struct rb_node *rb_left;
    // __attribute__ ((packed)) 的作用就是告诉编译器取消结构在编译过程中的优化对齐,按照实际占用字节数进行对齐，是GCC特有的语法。[参考 https://blog.csdn.net/hpu11/article/details/53326609]
} __attribute__((aligned(sizeof(long))));

struct rb_root {
    struct rb_node *rb_node;
};

```
