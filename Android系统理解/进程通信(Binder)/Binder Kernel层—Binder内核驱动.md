Binder系列：
[Binder Kernel层—Binder内核驱动](https://www.jianshu.com/p/fdf12cd7c28d)
[Binder Native层—服务管理(ServiceManager进程)](https://www.jianshu.com/p/fb514e3eed3a)
[Binder Native层—注册/查询服务](https://www.jianshu.com/p/a7e5bbb0ab72)
[Binder Framework层—注册和查询服务](https://www.jianshu.com/p/60be38017422)
[Binder 应用层-AIDL原理](https://www.jianshu.com/p/7ca78da80f95)

相关源码文件
> /drivers/staging/android/binder.c

在前面的文章中，无论是服务注册(addService)，还是服务管理ServiceManager进程中都涉及到与Binder内核驱动交互的三个方法：
  ```
// 1. 打开驱动设备
open("/dev/binder", O_RDWR)
// 2. 进行内存映射
mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0)
// 3. 与Binder驱动的通信
ioctl(bs->fd, BINDER_WRITE_READ, &bwr)

  ```
注释1，open方法对应binder内核驱动Kernel层的`binder_open()`,作用为打开驱动设备，并添加一个binder_proc结构体。

注释2，mmap方法对应binder内核驱动Kernel层的`binder_mmap()`,作用为创建一个内核虚拟地址和物理地址一帧区域，物理地址对内核虚拟地址和进程虚拟地址分别进行映射。

注释3，ioctl方法对应binder内核驱动Kernel层的`binder_ioctl()`，作用是通过ioctl命令与Binder内核驱动建立操作通信。

![binder驱动](https://upload-images.jianshu.io/upload_images/22650779-7fca8a9509281b9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**binder_open()、binder_mmap()、binder_ioctl()**这三个方法都在`binder.c`类中，在binder.c类中，在驱动启动时会先执行`binder_init()`方法
  ```
static int __init binder_init(void)
{
	int ret;
    ...
    //创建名为binder的工作队列
	binder_deferred_workqueue = create_singlethread_workqueue("binder");

    // 注册misc设备
	ret = misc_register(&binder_miscdev);
	...
	return ret;
}

static struct miscdevice binder_miscdev = {
	.minor = MISC_DYNAMIC_MINOR, //次设备号 动态分配
	.name = "binder", //设备名字
	.fops = &binder_fops // 设备的文件操作结构，这是file_operations结构
};

// 对应的一些操作
static const struct file_operations binder_fops = {
	.owner = THIS_MODULE,
	.poll = binder_poll,
	.unlocked_ioctl = binder_ioctl,
	.compat_ioctl = binder_ioctl,
	.mmap = binder_mmap,
	.open = binder_open,
	.flush = binder_flush,
	.release = binder_release,
};

  ```

## 1. binder_open 创建binder_proc
  ```
static int binder_open(struct inode *nodp, struct file *filp)
{
   // binder进程结构体	
   struct binder_proc *proc;

	binder_debug(BINDER_DEBUG_OPEN_CLOSE, "binder_open: %d:%d\n",
		     current->group_leader->pid, current->pid);
    // 在内核Kernel空间开辟一块连续的内存空间给proc结构体，并且大小不超过128k,初始化为0
	proc = kzalloc(sizeof(*proc), GFP_KERNEL);
	if (proc == NULL)
		return -ENOMEM;
    //获取当前线程的 task_struct，并赋值给proc->tsk
	get_task_struct(current);
	proc->tsk = current;
    // 初始化todo列表
	INIT_LIST_HEAD(&proc->todo);
    // 初始化等待队列
	init_waitqueue_head(&proc->wait);
	proc->default_priority = task_nice(current);

	binder_lock(__func__);
    // binder_proc加入到binder_procs驱动列表中
	hlist_add_head(&proc->proc_node, &binder_procs);
	// binder_proc保存到filp->private_data
	filp->private_data = proc;
    ...
	binder_unlock(__func__);
    ...
	return 0;
}

struct binder_proc {
	struct hlist_node proc_node; // 进程节点
	struct rb_root threads; // 线程处理的红黑树
	struct rb_root nodes; // 内部Binder对象的红黑树
	struct rb_root refs_by_desc; //外部对应的Binder对象的红黑树，以handle作为key
	struct rb_root refs_by_node;  //外部对应的Binder对象的红黑树，以地址作为key
	int pid; // 进程id
	struct vm_area_struct *vma; // 进程虚拟地址空间
	struct mm_struct *vma_vm_mm; // 虚拟地址管理
	struct task_struct *tsk; //当前线程task_struct
	struct files_struct *files;
	void *buffer; // 指向内核虚拟空间的地址
	ptrdiff_t user_buffer_offset;  // 用户虚拟地址空间与内核虚拟地址空间的偏移量

	struct list_head buffers;
	struct rb_root free_buffers;
	struct rb_root allocated_buffers;
	size_t free_async_space;

	struct page **pages; // 物理页的指针数组
	size_t buffer_size; // 映射虚拟内存的大小
	uint32_t buffer_free;
	struct list_head todo;
	wait_queue_head_t wait;
	struct binder_stats stats;
	struct list_head delivered_death;
	int max_threads;
	int requested_threads;
	int requested_threads_started;
	int ready_threads;
	long default_priority;
	struct dentry *debugfs_entry;
};

  ```
binder_open函数主要是创建了`binder_proc`进程结构体，并对binder_proc的一些值进行初始化，并将其加入到binder_procs链表中。

## 2.binder_mmap：创建内核虚拟空间和物理页，并进行内存映射
  ```
// binder_mmap(文件描述符，用户虚拟内存空间)
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	int ret;
    // 虚拟内核地址
	struct vm_struct *area;
    // binder_proc
	struct binder_proc *proc = filp->private_data;
	const char *failure_string;
	struct binder_buffer *buffer;

	if (proc->tsk != current)
		return -EINVAL;
     // 1. 普通进程：1M-8K, ServiceManager进程：128k
	if ((vma->vm_end - vma->vm_start) > SZ_4M)
		vma->vm_end = vma->vm_start + SZ_4M;

	...
    // 内核虚拟空间的首地址
   //2. 采用IOREMAP方式，分配一个连续的内核虚拟空间，与进程虚拟空间大小一致
	area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
	if (area == NULL) {
		ret = -ENOMEM;
		failure_string = "get_vm_area";
		goto err_get_vm_area_failed;
	}
    // proc->buffer指向内存虚拟空间首地址
	proc->buffer = area->addr;
    // 3. 用户虚拟地址和内核虚拟地址的偏移量
	proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
	mutex_unlock(&binder_mmap_lock);
    //  按页去开辟内存
	proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
    // 4. 计算大小
	proc->buffer_size = vma->vm_end - vma->vm_start;
    //5. 开辟和映射物理页，是一次拷贝的关键
	if (binder_update_page_range(proc, 1, proc->buffer, proc->buffer + PAGE_SIZE, vma)) {
		
	}
	buffer = proc->buffer;
	proc->files = get_files_struct(current);
	proc->vma = vma;
	proc->vma_vm_mm = vma->vm_mm;
	return ret;
}

// 开辟物理页，并对进程虚拟空间和内核虚拟空间进行映射
static int binder_update_page_range(struct binder_proc *proc, int allocate,
				    void *start, void *end,
				    struct vm_area_struct *vma)
{
	void *page_addr;
	unsigned long user_page_addr;
	struct vm_struct tmp_area;
	struct page **page;
	struct mm_struct *mm;  // 内存结构体

	if (vma)
		mm = NULL; //binder_mmap过程vma不为空，其他情况都为空
	else
		mm = get_task_mm(proc->tsk); //获取mm结构体

	if (mm) {
		down_write(&mm->mmap_sem);  //获取mm_struct的写信号量
		vma = proc->vma;
		if (vma && mm != proc->vma_vm_mm) {
			pr_err("%d: vma mm and task mm mismatch\n",
				proc->pid);
			vma = NULL;
		}
	}

    //此处allocate为1，代表分配过程。如果为0则代表释放过程
	if (allocate == 0)
		goto free_range;

	if (vma == NULL) {
		pr_err("%d: binder_alloc_buf failed to map pages in userspace, no vma\n",
			proc->pid);
		goto err_no_vma;
	}
    // 只执行一次，开辟一个物理页
	for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
		int ret;

		page = &proc->pages[(page_addr - proc->buffer) / PAGE_SIZE];

		BUG_ON(*page);
		//6.分配一个page的物理内存
		*page = alloc_page(GFP_KERNEL | __GFP_HIGHMEM | __GFP_ZERO);
		if (*page == NULL) {
			pr_err("%d: binder_alloc_buf failed for page at %p\n",
				proc->pid, page_addr);
			goto err_alloc_page_failed;
		}
		tmp_area.addr = page_addr;
		tmp_area.size = PAGE_SIZE + PAGE_SIZE /* guard page? */;
		//7. 物理空间映射到虚拟内核空间
		ret = map_vm_area(&tmp_area, PAGE_KERNEL, page);
		if (ret) {
			pr_err("%d: binder_alloc_buf failed to map page at %p in kernel\n",
			       proc->pid, page_addr);
			goto err_map_kernel_failed;
		}
        //  用偏移量计算用户虚拟空间的首地址
		user_page_addr =
			(uintptr_t)page_addr + proc->user_buffer_offset;
		//8. 物理空间映射到虚拟进程空间
		ret = vm_insert_page(vma, user_page_addr, page[0]);
		if (ret) {
			pr_err("%d: binder_alloc_buf failed to map page at %lx in userspace\n",
			       proc->pid, user_page_addr);
			goto err_vm_insert_page_failed;
		}
		/* vm_insert_page does not seem to increment the refcount */
	}
	if (mm) {
		up_write(&mm->mmap_sem);//释放内存的写信号量
		mmput(mm);//减少mm->mm_users计数
	}
	return 0;

free_range:
//释放内存的流程
...
}

  ```
注释1，vm_area_struct代表的是进程的虚拟空间，vm_struct代表的是内核的虚拟空间。把虚拟空间的大小限制在4M之内，不过一般进程的大小设置为：1M-8K，ServiceManager进程设置为128k。
注释2: 分配一个内核虚拟空间，大小和进程虚拟空间一样大。
注释3： 计算出进程虚拟空间和内核虚拟空间的偏移量，这样可以根据内核虚拟空间计算出进程虚拟空间。
注释4:计算出虚拟空间的大小
注释5:调用binder_update_page_range()去开辟一个物理页，并对进程虚拟空间和内核虚拟空间进行映射。
注释6: alloc_page()方法是去物理内存开辟一个物理页。
注释7: map_vm_area()将物理空间映射到虚拟内核空间。
注释8: vm_insert_page()将物理空间映射到虚拟进程空间

## 3. binder_ioctl：通过ioctl命令与Binder驱动操作交互
  ```
binder_ioctl(文件描述符，ioctl命令，数据类型)

static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
    //从filp中拿到binder_proc
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread; // binder线程
	unsigned int size = _IOC_SIZE(cmd);
    // 数据类型
	void __user *ubuf = (void __user *)arg;

	trace_binder_ioctl(cmd, arg);
     //进入休眠状态，直到中断唤醒
	ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	if (ret)
		goto err_unlocked;

	binder_lock(__func__);
    //1. 获取binder_thread
	thread = binder_get_thread(proc);
	if (thread == NULL) {
		ret = -ENOMEM;
		goto err;
	}

	switch (cmd) {
	case BINDER_WRITE_READ: // 2. 进行binder的读写操作
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
	case BINDER_SET_MAX_THREADS: //3. 设置binder最大支持的线程数
		if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
			ret = -EINVAL;
			goto err;
		}
		break;
	case BINDER_SET_CONTEXT_MGR: //4. 成为binder的上下文管理者，也就是ServiceManager成为守护进程
		ret = binder_ioctl_set_ctx_mgr(filp);
		if (ret)
			goto err;
		break;
	case BINDER_THREAD_EXIT://5. 当binder线程退出，释放binder线程
		binder_debug(BINDER_DEBUG_THREADS, "%d:%d exit\n",
			     proc->pid, thread->pid);
		binder_free_thread(proc, thread);
		thread = NULL;
		break;
	case BINDER_VERSION: {//6. 获取binder的版本号
        // 数据结构体 binder_version
		struct binder_version __user *ver = ubuf;
	     // 将BINDER_CURRENT_PROTOCOL_VERSION版本号拷贝给ver->protocol_version
		if (put_user(BINDER_CURRENT_PROTOCOL_VERSION,
			     &ver->protocol_version)) {
			ret = -EINVAL;
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
}

static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
	struct binder_thread *thread = NULL;
	struct rb_node *parent = NULL;
	struct rb_node **p = &proc->threads.rb_node;

	while (*p) {   //根据当前进程的pid，从binder_proc中查找相应的binder_thread
		parent = *p;
        // 通过 rb_node 去找到 binder_thread
		thread = rb_entry(parent, struct binder_thread, rb_node);

		if (current->pid < thread->pid)
			p = &(*p)->rb_left;
		else if (current->pid > thread->pid)
			p = &(*p)->rb_right;
		else
			break;
	}
	if (*p == NULL) {
        // 第一次 pid 创建对象
		thread = kzalloc(sizeof(*thread), GFP_KERNEL);  //新建binder_thread结构体
		if (thread == NULL)
			return NULL;
		binder_stats_created(BINDER_STAT_THREAD);
		thread->proc = proc;
		thread->pid = current->pid; //保存当前进程(线程)的pid
        // 初始化等待队列和工作队列
		init_waitqueue_head(&thread->wait);
		INIT_LIST_HEAD(&thread->todo);
        // 插入到当前进程线程列表的红黑树
		rb_link_node(&thread->rb_node, parent, p);
        // 调整红黑树的颜色
		rb_insert_color(&thread->rb_node, &proc->threads);
		thread->looper |= BINDER_LOOPER_STATE_NEED_RETURN;
		thread->return_error = BR_OK;
		thread->return_error2 = BR_OK;
	}
	return thread;
}
  ```
通过`binder_ioctl(文件描述符，ioctl命令，数据类型)`方法，binder驱动就可以成为一个数据处理中心的中转站，其他进程可以通过`ioctl命令`告诉Binder驱动自己要操作什么。
注释1，通过binder_get_thread()方法根据pid获取到binder线程。
注释2，`BINDER_WRITE_READ`的ioctl命令,是最常用的命令，可以与binder读写数据，其中数据的结构体为：binder_write_read,后面将对其进行分析。
注释3. `BINDER_SET_MAX_THREADS`设置binder最大支持的线程数。
注释4. `BINDER_SET_CONTEXT_MGR`,注册成为binder机制上下文管理者，ServiceManager进程就会调用此成为管理者。此管理者只有一个
注释5. `BINDER_THREAD_EXIT `当binder线程退出，释放binder线程
注释6. `BINDER_VERSION `获取binder的版本号

### 3.1 BINDER_SET_CONTEXT_MGR 成为Binder机制上下文管理者
这个ioctl命令在ServiceManager进程中会用调用设置其成为Binder机制上下文管理者。
  ```
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
  ...
  switch (cmd) {
   ...
  case BINDER_SET_CONTEXT_MGR: //4. 成为binder的上下文管理者，也就是ServiceManager成为守护进程
      ret = binder_ioctl_set_ctx_mgr(filp);
      if (ret)
          goto err;
      break;
   ...
  }
}

  ```
调用了`binder_ioctl_set_ctx_mgr(filp)`方法去执行设置binder机制上下文管理者操作。
  ```
static int binder_ioctl_set_ctx_mgr(struct file *filp)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	kuid_t curr_euid = current_euid();

	 //1. 静态 binder_context_mgr_node 是否有设置过，只能有一个进程成为管理者
	if (binder_context_mgr_node != NULL) {
		pr_err("BINDER_SET_CONTEXT_MGR already set\n");
		ret = -EBUSY;
		goto out;
	}
    // 2.创建一个单独 binder_node
	der_context_mgr_node = binder_new_node(proc, 0, 0);

	return ret;
}

// 返回一个binder_node
static struct binder_node *binder_new_node(struct binder_proc *proc,
					   binder_uintptr_t ptr,
					   binder_uintptr_t cookie)
{
	 // 3.proc->nodes.rb_node 代表的是内部（本）进程的 Binder 对象的红黑树
	struct rb_node **p = &proc->nodes.rb_node;
	struct rb_node *parent = NULL;
	struct binder_node *node;

	while (*p) {
		parent = *p;
		// 4. 根据 parent 的偏移量获取 node
		node = rb_entry(parent, struct binder_node, rb_node);

		if (ptr < node->ptr)
			p = &(*p)->rb_left;
		else if (ptr > node->ptr)
			p = &(*p)->rb_right;
		else
			return NULL;
	}

	 // 刚开始肯定是 null ，创建一个 binder_node
	node = kzalloc(sizeof(*node), GFP_KERNEL);
	if (node == NULL)
		return NULL;
	binder_stats_created(BINDER_STAT_NODE);
	// 把创建的 binder_node 添加到红黑树 
	rb_link_node(&node->rb_node, parent, p);
	// 调整红黑树的颜色
	rb_insert_color(&node->rb_node, &proc->nodes);
    // 设置一些参数  
	node->debug_id = ++binder_last_id;
	node->proc = proc;
	node->ptr = ptr;
	node->cookie = cookie;
	node->work.type = BINDER_WORK_NODE;
	return node;
}

  ```
注释1，静态变量`binder_context_mgr_node`记录着上下文管理机制的binder_node，如果不为`NULL`则说明已经有了上下文管理者，不能再被设置，只有一个进程能成为binder上下文管理者。
注释2，如果`der_context_mgr_node`为Null,则执行binder_new_node()方法返回一个`binder_node`赋值给der_context_mgr_node变量。
`binder_new_node()`方法中注释3，proc->nodes.rb_node为binder_node中的rb_node，通过rb_node可以找到binder_node。注释4是寻找binder_node。

### 3.2 BINDER_WRITE_READ: 数据读写
  ```
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
  ...
  switch (cmd) {
  case BINDER_WRITE_READ: // 进行binder的读写操作
      // 1. 这里的arg结构体为binder_write_read的地址
      ret = binder_ioctl_write_read(filp, cmd, arg, thread);
      if (ret)
          goto err;
      break;
  ...
   }
}

  ```

注释1中调用`binder_ioctl_write_read()`方法去处理binder的读写操作，这也涉及到binder和进程的通信交互。其中参数cmd为BINDER_WRITE_READ，参数arg为传进来的binder_write_read数据结构体。

  ```
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	 // 从 filp 中获取 binder_proc
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	// arg 是上层传下来的 binder_write_read 的结构体对象地址
	void __user *ubuf = (void __user *)arg;
     // binder 驱动的binder_write_read结构体变量
	struct binder_write_read bwr;

	if (size != sizeof(struct binder_write_read)) {
		ret = -EINVAL;
		goto out;
	}
	//1. 将用户空间的 binder_write_read 拷贝到内核空间的 bwr
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}

	//当写缓存中有数据，则执行binder_thread_write写操作
	if (bwr.write_size > 0) {
        // 2. 调用binder_thread_write函数
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		trace_binder_write_done(ret);
		//当写失败，再将bwr数据写回用户空间，并返回
		if (ret < 0) {
			bwr.read_consumed = 0;
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	//当读缓存中有数据，则执行binder_thread_read读操作
	if (bwr.read_size > 0) {
        // 3. 调用binder_thread_read函数
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
		trace_binder_read_done(ret);
		if (!list_empty(&proc->todo))
			wake_up_interruptible(&proc->wait); //唤醒等待状态的线程
		 //当读失败，再将bwr数据写回用户空间，并返回
		if (ret < 0) {
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	 // 4. 将内核空间的 bwr 数据拷贝到用户空间的 binder_write_read
	if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
out:
	return ret;
}
  ```
binder_ioctl_write_read()方法主要做了四件事：
注释1，把外面进程传递进来的binder_write_read结构体数据复制到binder驱动创建的binder_write_read结构体变量中。
注释2，如果`bwr.write_size>0`，说明有写数据，则调用`binder_thread_write()`方法处理数据的写操作
注释3，如果`bwr.read_size > 0`说明有读数据，则调用`binder_thread_read()`方法处理数据的写操作。
注释4，通过binder内核的读写操作处理后，把处理的binder_write_read结构体变量，返回给外面进程处理。

#### 3.1 binder_thread_read
  ```
binder_thread_read（）{
    wait_for_proc_work = thread->transaction_stack == NULL &&
            list_empty(&thread->todo);
    //根据wait_for_proc_work来决定wait在当前线程还是进程的等待队列
    if (wait_for_proc_work) {
        ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
        ...
    } else {
        ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
        ...
    }
    
    while (1) {
        //当&thread->todo和&proc->todo都为空时，goto到retry标志处，否则往下执行：
        struct binder_transaction_data tr;
        struct binder_transaction *t = NULL;
        switch (w->type) {
          case BINDER_WORK_TRANSACTION: ...
          case BINDER_WORK_TRANSACTION_COMPLETE: ...
          case BINDER_WORK_NODE: ...
          case BINDER_WORK_DEAD_BINDER: ...
          case BINDER_WORK_DEAD_BINDER_AND_CLEAR: ...
          case BINDER_WORK_CLEAR_DEATH_NOTIFICATION: ...
        }
        ...
    }
done:
    *consumed = ptr - buffer;
    //当满足请求线程加已准备线程数等于0，已启动线程数小于最大线程数(15)，
    //且looper状态为已注册或已进入时创建新的线程。
    if (proc->requested_threads + proc->ready_threads == 0 &&
        proc->requested_threads_started < proc->max_threads &&
        (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
         BINDER_LOOPER_STATE_ENTERED))) {
        proc->requested_threads++;
        // 生成BR_SPAWN_LOOPER命令，用于创建新的线程
        put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer)；
    }
    return 0;
}
  ```

#### 3.2 binder_thread_write
  ```
binder_thread_write(){
    while (ptr < end && thread->return_error == BR_OK) {
        get_user(cmd, (uint32_t __user *)ptr)；//获取IPC数据中的Binder协议(BC码)
        switch (cmd) {
            case BC_INCREFS: ...
            case BC_ACQUIRE: ...
            case BC_RELEASE: ...
            case BC_DECREFS: ...
            case BC_INCREFS_DONE: ...
            case BC_ACQUIRE_DONE: ...
            case BC_FREE_BUFFER: ... break;
            
            case BC_TRANSACTION:
            case BC_REPLY: {
                struct binder_transaction_data tr;
                copy_from_user(&tr, ptr, sizeof(tr))； //拷贝用户空间tr到内核
                // 执行binder_transaction()函数
                binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
                break;

            case BC_REGISTER_LOOPER: ...
            case BC_ENTER_LOOPER: ...
            case BC_EXIT_LOOPER: ...
            case BC_REQUEST_DEATH_NOTIFICATION: ...
            case BC_CLEAR_DEATH_NOTIFICATION:  ...
            case BC_DEAD_BINDER_DONE: ...
            }
        }
    }
}
  ```
**binder_transaction**

  ```
static void binder_transaction(struct binder_proc *proc,
               struct binder_thread *thread,
               struct binder_transaction_data *tr, int reply){
    //根据各种判定，获取以下信息：
    struct binder_thread *target_thread； //目标线程
    struct binder_proc *target_proc；    //目标进程
    struct binder_node *target_node；    //目标binder节点
    struct list_head *target_list；      //目标TODO队列
    wait_queue_head_t *target_wait；     //目标等待队列
    ...

    // 如何寻找目标进程
	if (reply) {
		in_reply_to = thread->transaction_stack;
		target_thread = in_reply_to->from;
		target_proc = target_thread->proc;
	} else {
		if (tr->target.handle) {
			struct binder_ref *ref;
			ref = binder_get_ref(proc, tr->target.handle);
			target_node = ref->node;
		} else {
			target_node = binder_context_mgr_node;
		}


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

    //分配两个结构体内存
    struct binder_transaction *t = kzalloc(sizeof(*t), GFP_KERNEL);
    struct binder_work *tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    // 当前发送数据进程
	tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
    //从target_proc分配一块buffer
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,


    for (; offp < off_end; offp++) {
        switch (fp->type) {
        case BINDER_TYPE_BINDER: ...
        case BINDER_TYPE_WEAK_BINDER: ...
        case BINDER_TYPE_HANDLE: ...
        case BINDER_TYPE_WEAK_HANDLE: ...
        case BINDER_TYPE_FD: ...
        }
    }
    //向接收数据的目标进程的target_list添加BINDER_WORK_TRANSACTION事务
    t->work.type = BINDER_WORK_TRANSACTION;
    list_add_tail(&t->work.entry, target_list);
    //向发送数据的当前线程的todo队列添加BINDER_WORK_TRANSACTION_COMPLETE事务
    tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
    list_add_tail(&tcomplete->entry, &thread->todo);
    if (target_wait)
        wake_up_interruptible(target_wait);
    return;
}
  ```

![](https://upload-images.jianshu.io/upload_images/22650779-4b6aa24294f94b04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**binder_write_read的数据结构**
  ```
struct binder_write_read {
	binder_size_t		write_size;	/* bytes to write */
	binder_size_t		write_consumed;	/* bytes consumed by driver */
	binder_uintptr_t	write_buffer;
	binder_size_t		read_size;	/* bytes to read */
	binder_size_t		read_consumed;	/* bytes consumed by driver */
	binder_uintptr_t	read_buffer;
};
  ```

**binder_transaction_data的数据结构**
  ```
struct binder_transaction_data {
    union {
        size_t  handle;     /* 目标的handle */
        void    *ptr;       /* target descriptor of return transaction */
    } target;
    void        *cookie;    /* target object cookie */
    unsigned int    code;   /* ADD_SERVICE_TRANSACTION等 */
 
    /* General information about the transaction. */
    unsigned int    flags;
    pid_t       sender_pid;
    uid_t       sender_euid;
    size_t      data_size;  /* number of bytes of data */
    size_t      offsets_size;   /* number of bytes of offsets */
 
    /* If this transaction is inline, the data immediately
     * follows here; otherwise, it ends with a pointer to
     * the data buffer.
     */
    union {
        struct {
            /* 数据*/
            const void  *buffer;
            /* offsets from buffer to flat_binder_object structs */
            const void  *offsets;
        } ptr;
        uint8_t buf[8];
    } data;
};

  ```
