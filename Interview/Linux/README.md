

## fork()流程
对父进程程序段、数据段、堆段以及栈段创建拷贝。

如果简单地将父进程虚拟内存页拷贝到新的子进程，这样很容易造成浪费(如fork之后exec)。
采用两种技术避免这种浪费：
1. 内核将每一进程的代码段标记为只读，从而使进程无法修改自身代码。这样父子进程就可以共享同意代码段
2. 对于父进程数据段、堆段和栈段中的各页，内核采用写时复制技术来处理。
fork()之后，内核会捕获所有父进程或子进程针对这些页面的修改企图，并为将要修改的页面创建拷贝。
系统将新的页面拷贝分配给遭到内核捕获的进程，还或对子进程的相应页表项做适当调整

## vfork、clone与fork的区别
三个函数最终由同一个函数实现(kernel/fork.c中的do_fork())

vfork是为子进程立刻执行exec而专门设计的

区别：
+ 无需为子进程复制虚拟内存页或页表，子进程共享父进程的内存，直至成功执行了exec()或是调用_exit()退出
+ 在子进程调用exec()或_exit()之前，将暂停执行父进程
在不影响父进程的前提下，子进程能在vfork和exec之间做的操作屈指可数，包括对打开文件描述符进行操作

clone也能创建一个新进程，对创建进程期间的步骤控制更为精确

## 僵尸进程
在父进程执行wait()之前，其子进程就已经终止。内核通过将子进程转化为僵尸进程来处理这种情况。
这意味着将释放进程所把持的大部分资源，以便提供其他进程重新使用。
该进程所唯一保留的是内核进程表中的一条记录，其中包含了子进程ID终止状态、资源使用数据等信息

如果存在大量僵尸进程，将会填满内核进程表，从而阻碍新进程的创建。既然无法用信号杀死僵尸进程，
那么从系统中移除的唯一办法是杀掉它们的父进程，此时init进程将会接管和等待这些僵尸进程，从而从系统将它们清理掉。

## SIGCHLD信号
无论一个子进程于何时终止，系统都会向其父进程发送SIGCHILD信号，对该信号的默认处理是忽略。

子进程的终止属于异步事件，父进程应使用wait来防止僵尸进程的累积，或者采用以下两种办法：
+ 父进程调用不带WNOHANG标志的wait()，或waitpid()方法，此时如果尚无已经终止的子进程，那么调用将会阻塞
+ 父进程周期性地调用调用WHOHANG标志的waitpid()，执行针对已经终止子进程的非阻塞检查
这两种办法都使用便，可以采用针对SIGCHLD信号的处理程序，用来捕获终止子进程。

进程可以将对SIGCHLD信号的处理设置置为忽略(SIG_IGN)，这将立即丢弃终止子进程的状态。

## 四种硬件信号
这四种硬件异常， 一般是由程序自身引发的， 不是由其他进程发送的信号引发的， 并且这些异常都比较致命， 以至于进程无法
继续下去。 所以这些信号产生之后， 会立刻递送给进程。 默认情况下， 这四种信号都会使进程终止， 并且产生core dump文件以供调试。 对于这些信号， 进程既不能忽略， 也不能阻塞
### SIGBUS
总线错误，表示发生内存访问错误
+ 变量地址未对齐： 很多架构访问数据时有对齐的要求。 比如int型变量占用4个字节， 因此架构要求int变量的地址必须为4字节对齐， 否则就会触发SIGBUS信号
+ mmap映射文件： 使用mmap将文件映射入内存， 如果文件大小被其他进程截短， 那么在访问文件大小以外的内存时， 会触发SIGBUS信号

### SIGFPE
算术错误
+ 整数除以0

### SIGILL
进程尝试执行非法的机器语言指令
+ 一般是函数指针遭到破坏， 当执行函数指针指向
的函数时， 就会触发SIGILL信号。 另外也可能是由指令集的演进引起的。 比如， 很多在新的体系结构中编译出来的可执行程序， 在老的机器上可能会无法运行， 故而在老机器上运行时， 也可能产生SIGILL信号

### SIGSEGV
段错误，访问无效地址
+ 访问未初始化的指针或NULL指针指向的地址
+ 进程企图在用户态访问内核部分的地址
+ 进程尝试去修改只读的内存地址


## epoll & poll & select
select的缺点：
1. 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
2. 同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销很多时也很大
3. select支持的文件描述符数量太小，默认是1024

epoll避免了上述三个缺点：
1. 每次注册新的事件到epoll句柄中时(epoll_ctl中指定EPOLL_CTL_ADD)，会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程只会拷贝一次
2. 不像select或poll一样每次都把current(当前进程)轮流加入fd对应的设备等带队列中，而只在epoll_ctl时把current挂一遍并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd
3. epoll所支持的fd上限就是可以最大打开文件的数目

+ select能支持的文件描述符数是有限的，最大1024个，并且每次调用前都需要将其监听的读集、写集、错误集从用户态向内核态拷贝，返回后又拷贝回去，而且，select返回的时候是将所有的文件描述符返回，也就意味着一旦有个事件触发，只能通过遍历的方式才能找到具体是哪一个事件，效率比较低、开销也比较大，但是也有好处，就是他的超时的单位是微秒级别；

+ epoll能支持的文件描述符数很大，可以上万，他的高效由3个部分组成：红黑树、双向链表、回调函数，每次将监听事件拷贝到内核后就存放在红黑树中(epoll_ctl)，以EventPoll的结构体存在，如果有相应的事件发生，对应的回调函数就会触发，进而就会将该事件拷贝至双向链表中返回，epoll_wait只需观察双向链表中有没有数据，有的话就返回

+ 总的来说epoll适用于连接数较多，活跃数较少的场景、而select适用于连接数不多，但大多都活跃的场景。

## epoll的ET和LT
水平触发：如果文件描述符上可以非阻塞地执行IO系统调用，此时认为它已经就绪了
边缘触发：如果文件描述符自上次状态检查以来有了新的IO活动，此时需要触发通知

采用水平触发时，可以在任意时刻检查文件描述符的就绪状态，
而采用边缘触发时，只有当IO事件发生时我们才会收到通知，
所以在接收到一个IO事件通知后，要尽可能多地执行IO

## ET使用非阻塞socket
如果程序采用 **循环** 来对文件描述符执行尽可能多的IO，而文件描述符又被置为可阻塞的，
那么最终当没有更多的IO可执行时（最后一次操作），IO系统调用就会阻塞，所以每个被检查的文件描述符应该是非阻塞的

## 进程与线程的区别？
+ 进程是资源分配的最小单位，线程是程序执行的最小单位。

+ 进程有自己的独立地址空间，每启动一个进程，系统就会为它分配地址空间，建立数据表来维护代码段、堆栈段和数据段，这种操作非常昂贵。而线程是共享进程中的数据的，使用相同的地址空间，因此CPU切换一个线程的花费远比进程要小很多，同时创建一个线程的开销也比进程要小很多。

+ 线程之间的通信更方便，同一进程下的线程共享全局变量、静态变量等数据，而进程之间的通信需要以通信的方式（IPC)进行。不过如何处理好同步与互斥是编写多线程程序的难点。

+ 但是多进程程序更健壮，多线程程序只要有一个线程死掉，整个进程也死掉了，而一个进程死掉并不会对另外一个进程造成影响，因为进程有自己独立的地址空间。

## 协程
+ python第三方库gevent
    + 当一个greenlet遇到IO操作时，比如访问网络，就自动切换到其他的greenlet，等到IO操作完成，再在适当的时候切换回来继续执行。由于IO操作非常耗时，经常使程序处于等待状态，有了gevent为我们自动切换协程，就保证总有greenlet在运行，而不是等待IO
    + monkey_patch，将标准库中大部分的阻塞式调用替换成非阻塞的方式，包括socket、ssl、threading、select、httplib等
+ 腾讯开源的C++协程库：libco
    + 协程切换：glibc的ucontext
    + IO阻塞：同名函数+dlsym来hook socket族的阻塞IO，如read、write等，劫持系统调用后把这些IO注册到epoll的事件循环中，之后让出CPU，IO完成后唤醒这个协程
    + 如果一个协程没有发起IO，一直占用CPU资源？无解，协程不适合处理CPU密集计算

## 惊群效应
### accept惊群（Linux2.6已解决）
一个连接过来，只有一个进程可以accept成功
### epoll惊群
如果多个进程/线程阻塞在监听同一个监听socket fd的epoll_wait上，当有一个新的连接到来时，所有的进程都会被唤醒。


## 静态库和动态库
静态库:
+ 静态库对函数库的链接是放在编译时期完成的。
+ 程序在运行时与函数库再无瓜葛，移植方便。
+ 浪费空间和资源，因为所有相关的目标文件与牵涉到的函数库被链接合成一个可执行文件

动态库:
不同的应用程序如果调用相同的库，那么在内存里只需要有一份该共享库的实例，规避了空间浪费问题。动态库在程序运行是才被载入，也解决了静态库对程序的更新、部署和发布页会带来麻烦

+ 动态库把对一些库函数的链接载入推迟到程序运行的时期。
+ 可以实现进程之间的资源共享。（因此动态库也称为共享库）
+ 将一些程序升级变得简单。
+ 甚至可以真正做到链接载入完全由程序员在程序代码中控制（显示调用）。
Window与Linux执行文件格式不同，在创建动态库的时候有一些差异。
+ 在Windows系统下的执行文件格式是PE格式，动态库需要一个DllMain函数做出初始化的入口，通常在导出函数的声明时需要有_declspec(dllexport)关键字。
+ Linux下gcc编译的执行文件默认是ELF格式，不需要初始化入口，亦不需要函数做特别的声明，编写比较方便。


## 加载并运行a.out
1. 删除已存在的用户区域。删除当前进程虚拟地址的用户部分中的已存在的区域结构
2. 映射私有区域。为新程序的代码、数据、bss和栈区域创建新的区域结构。所有这些区域都是私有的、写时复制的。
代码和数据区域被映射为a.out文件中的.text和.data区。bss区域是请求二进制零的，映射到匿名文件，其大小包含在a.out中。
栈和堆区域也是为二进制清零的，初始长度为零。
3. 映射共享区域。如果a.out程序与共享对象链接，比如标准C库libc.so，
那么这些对象都是动态链接到这个程序的，然后再映射到用户虚拟地址空间中的共享区域内。
4. 设置程序计数器(PC)。设置当前进程上下文中的程序计数器指向代码区域的入口点。


## 创建一个daemon
1. 执行一个fork()，之后父进程退出，子进程继续执行。(结果是daemon成为了init进程的子进程)。之所以要做这一步是因为：
    1. 假设daemon是从命令行启动的，父进程的终止会被shell发现，shell在发现之后会显示出另一个shell提示符并让子进程继续在后台执行
    2. 子进程被确保不会成为一个进程组首进程
2. 子进程调用setsid()开启一个新会话并释放它与控制终端之间的所有联系
3. 如果daemon从来没有打开过终端设备，那么就无需担心daemon会重新请求一个控制终端了。
如果daemon后面可能湖打开一个终端设备，那么必须要采取措施来确保这个不会成为控制终端
4. 清除进程umask以确保daemon创建文件和目录时拥有所需的权限
5. 修改进程的当前工作目录，通常改为根目录
6. 关闭daemon从其父进程继承而来的所有打开的文件描述符(可选)
7. 关闭文件描述符0、1、2之后，daemon通常会打开/dev/null并使用dup2()使所有这些描述符指向这个设备。


## 为什么要分段和分页
在分段之前，程序运行需要从内存分配足够多的连续的内存，然后把程序装载进去，这样有是三个问题：
1. 地址空间不隔离
2. 程序运行时候的地址不确定
3. 内存使用率低下

这有时候 **虚拟内存** 的概念就出现了，用虚拟地址空间映射到物理地址空间，程序操作的是虚拟地址

分段能解决前两个问题，就是一种虚拟地址空间到物理地址空间的映射，但还是将整个程序放入虚拟地址，还存在第三个问题

分页的粒度更小，当需要某些代码和数据就以页为单位装载进内存，不需要则换出带磁盘，即按需分配

## Linux内存管理
![](https://img-blog.csdnimg.cn/20190425154028523.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlc3Ricm9va2xpdQ==,size_16,color_FFFFFF,t_70)
+ CPU访问的地址是虚拟地址，需要MMU进行虚拟地址到物理地址的转换
+ 页表是存储在主存中，如果是二级页表就需要访问两次主存，比较耗时
+ 为了加快速度，**TLB**(快表)缓存了上一次虚拟地址到物理地址的页表项

![](https://img-blog.csdnimg.cn/20190425202721468.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlc3Ricm9va2xpdQ==,size_16,color_FFFFFF,t_70)
+ malloc -> brk -> sys_brk
+ 在堆上分配内存也是虚拟内存 -> **vma** 
+ **缺页中断** -> 映射虚拟地址到物理地址
+ 物理页面分为 **匿名页面**(没有关联任何文件的页面，如malloc分配的) 和 page cache(与文件有关)
+ 匿名页面和page cache 需要页面分配器分配
+ **slab分配器** -> 管理固定大小的分配，不用麻烦页面分配器
+ **页面回收** -> 当系统内存低于某个水位的时候 **kswapd** (用于页面回收的内核线程)会被唤醒，去扫**LR**链表(匿名页面和page cache)
+ KSM:合并匿名页面，释放一些物理页面
+ Huge Page:大页面，减少TLB(块表)miss 的次数
+ 内存规整:处理内存碎片化

### 知识网络
1. 虚拟内存 -> MMU、页表、物理内存、物理页面、建立映射关系、按需分配、缺页中断、写时复制
2. MMU -> 如何建立页表映射，其中包括用户空间页表的建立和内核空间页表的建立，以及内核如何查询页表和修改页表
3. 物理内存、物理页面 -> struct pg_data_t、struct zone、struct page等，这三个数据结构描述了系统中物理内存的组织架构
4. 分配物理页面 -> 伙伴系统和页面分配器
5. 建立映射关系 -> 描述进程的虚拟内存用struct vm_area_struct，物理内存和虚拟内存通过建立页表的方式建立映射关系
6. malloc分配内存 -> 缺页中断
7. 映射关系以页为基础 -> 需要小于一个页面大小的内存 -> slab机制
8. 物理内存不足 -> 页面回收机制和反向映射机制
9. 碎片化 -> 内存规整机制

### 细节
#### 物理内存初始化
1. 在ARM Linux中，各种设备的相关属性描述都采用DTS方式来呈现，内存定义在*dts文件中，内核启动过程中需要解析这些DTS文件
2. 在内核使用内存前，需要初始化内核的页表，初始化页表主要在map_lowmem()中
3. 对页表的初始化完成后，内核就可以对内存进行管理，采用区块zone的方式管理
4. 在内核启动时，内核知道物理内存DDR的大小并且计算出高端内存的起始地址和内核空间的内存布局后，物理内存页面page就需要加入到伙伴系统中
5. 伙伴系统中，把所有所有空闲页面分组成11个内存块链表，每个内存块链表分别包括1、2、4、...、1024个连续页面

#### 分配物理页面
1. 伙伴系统分配内存，alloc_pages()，分配掩码确定从那些zone中分配内存
2. 释放内存页面，相邻的内存块空闲就合并放置到高一阶的空闲列表中

#### slab分配器
![](https://img-blog.csdnimg.cn/20190504220126269.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlc3Ricm9va2xpdQ==,size_16,color_FFFFFF,t_70)
1. slab系统由slab描述符、slab节点、本地对象缓冲池、共享对象缓冲池、3个slab链表、n个slab、以及众多slab缓存对象组成
2. 每个slab由一个或多个page连续页面组成
3. slab通过两种方式回收内存：使用kmem_cache_free释放一个对象，当发现本地和共享对象缓冲池中的开共享对象数目等于一个极限值时，系统会主动释放bacthcount个对象，当空闲对象大于一个极限值时，并且这个slab没有活跃对象，就销毁这个slab。第二种方法是注册一个定时器，定时去扫描所有的slab描述符，回收一部分空闲对象

#### vmalloc和kmalloc
+ kmalloc基于slab分配器，物理内存是连续的
+ vmalloc仅仅需要虚拟地址是连续的

#### VMA
+ 进程地址空间 -> struct vm_area_struct -> vma
+ struct mm_struct -> 描述进程内存管理 -> 包含 vm_area_struct的链表和红黑树
+ 每个vma都要链接到 mm_struct 中的链表和红黑树中，以起始地址递增的方式
+ vma 离散分布在3GB的用户内存空间中

#### 缺页中断
malloc和mmap值建立了进程地址空间，在用户空间可以看到虚拟内存，但没有建立虚拟内存和物理内存之间的映射关系，
当进程访问这些还没有建立映射关系的虚拟内存时，处理器自动触发一个缺页异常，Linux内核必须处理此异常

#### 反向映射RMAP
原因：
+ PTE页表保存着从虚拟内存页面映射到物理内存页面的记录，
page数据结构中的_mapcount成员记录有多少个用户PTE页表项映射了物理页面
+ 有的页面需要被迁移，有的页面长时间不使用需要被交换到磁盘
+ 在交换之前必须找出哪些进程使用这个页面，然后断开这些映射的PTE
+ Linux内核2.4中，为了确定某一个页面是否被某个进程映射，必须遍历每个进程的页表，效率低
+ Linux2.5后，提出了反向映射的概念

早期2.6的实现：
![](https://img-blog.csdnimg.cn/20190508232659739.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlc3Ricm9va2xpdQ==,size_16,color_FFFFFF,t_70)
缺陷：假如父进程有1000个子进程，每个子进程都有一个vma，这个vma有1000个匿名页面，当所有子进程的vma中的所有匿名页面都同时发生写时复制时，因为新分配的页面依然指向父进程的AVp数据结构，父进程的AVp队列中会有100万个匿名页面，扫面这个队列会很耗时间


## Linux进程管理
### 进程控制块 task_struct
+ 进程属性：
    + state：用来记录进程的状态
    + pid：进程唯一的标识符
    + flag：描述进程属性的一些标志位
    + exit_code 和 exit_signal：用来存放进程退出值和终止信号，这样父进程可以知道子进程的退出原因
    + pdeath_signal：父进程消亡时发出的信号
    + comm：存放可执行程序的名称
    + real_cred 和 cred：用来存放进程的一些认证信息

+ 调度管理：
    + prio：保存着进程动态优先级，是调度类考虑的优先级
    + static_prio：静态优先级。内核不存储nice值，取而代之的是static_prio
    + normal_prio：基于static_prio和调度策略计算出来的优先级
    + rt_prio：实时进程的优先级
    + sched_class：调度类
    + se：普通进程调度实体
    + rt：实时进程调度实体
    + dl：deadline进程调度实体
    + policy：进程的类型，比如普通进程还是实时进程
    + cpus_allowed：进程可以在那几个CPU上运行    
    
+ 进程间关系：
    + real_parent：指向当前进程的父进程的task_struct数据结构
    + children：指向当前进程的子进程的链表
    + sibling：指向当前进程的兄弟进程的链表
    + group_leader：进程组的组长    

+ 内存管理与文件管理：
    + mm：指向进程多管理的内存的一个总的抽象的数据结构mm_struct
    + fs：保存一个指向文件系统信息的指针
    + files：保存一个指向进程的文件描述符表的指针

总结：
+ 进程的运行状态
+ 程序计数器
+ CPU寄存器，保存上下文
+ 内存管理信息
+ 统计信息
+ 文件相关信息　

### 进程的创建 fork 0.11版
+ sys_fork
    + find_empty_process
        + 找到task数组(task_struct，全局，最大64)中空闲的pid
    + copy_process
        + get_free_page
            + 分配4k内存
            + 把task_struct放进去
        + 初始化task_struct成员
        + copy_mem
            + 拷贝父进程的虚拟地址空间


## ctrl-c...
+ ctrl-c: ( kill foreground process ) 发送 **SIGINT** 信号给前台进程组中的所有进程，强制终止程序的执行；
+ ctrl-z: ( suspend foreground process ) 发送 **SIGTSTP** 信号给前台进程组中的所有进程，常用于挂起一个进程，而并
            非结束进程，用户可以使用使用fg/bg操作恢复执行前台或后台的进程。fg命令在前台恢复执行被挂起的进
            程，此时可以使用ctrl-z再次挂起该进程，bg命令在后台恢复执行被挂起的进程，而此时将无法使用ctrl-z
            再次挂起该进程；
+ ctrl-d: ( Terminate input, or exit shell ) 一个特殊的二进制值，表示 EOF，作用相当于在终端中输入exit后回车；
+ ctrl-/   发送 **SIGQUIT** 信号给前台进程组中的所有进程，终止前台进程并生成 core 文件
+ ctrl-s   中断控制台输出
+ ctrl-q  恢复控制台输出
+ ctrl-l   清屏
其实，控制字符都是可以通过 **stty** 命令更改的，可在终端中输入命令"stty -a"查看终端配置

## Linux中进程的七种状态
1. R运行状态（runing）：并不意味着进程一定在运行中，也可以在运行队列里；
2. S睡眠状态（sleeping）：进程在等待事件完成；（可中断睡眠，可以被唤醒）
3. D磁盘睡眠状态（Disk sleep）:不可中断睡眠（不可中断睡眠，不可以被唤醒，通常在磁盘写入时发生）
4. T停止状态（stopped）：可以通过发送SIGSTOP信号给进程来停止进程，可以发送SIGCONT信号让进程继续运行
5. X死亡状态（dead）:该状态是返回状态，在任务列表中看不到；
6. Z僵尸状态（zombie）:子进程退出，父进程还在运行，但是父进程没有读到子进程的退出状态，子进程进入僵尸状态；
7. t追踪停止状态（trancing stop）