Binder是android跨进程通信的基础。网上有很多比较详细的解读。我学习的时候主要参考了下面几篇博客

1. https://events.linuxfoundation.org/images/stories/slides/abs2013_gargentas.pdf
2. http://wangkuiwu.github.io/2014/09/01/Binder-Introduce/
3. http://gityuan.com/2015/11/01/binder-driver/

## 重要的数据结构
### Binder上下文
Bidner上下文保存了Service和binder驱动通信进程的上下文信息，它所对应的数据结构就是binder_proc。每当Service进程打开binder驱动时，binder_proc即在内核态的binder驱动中被创建。

### Binder实体
Binder实体是各种Service在内核态中存在的形式，它所对应的数据结构就是binder_node。每个binder_node中存储了Service在用户态的引用，
当我们通过在binder驱动中通过索引找到了binder_node后，就能与在用户态存在的Service进行通信。

### Binder引用
Binder引用是内核态中对于binder_node的引用，对应的数据结构是binder_ref。它与binder_node是多对一的关系。通过binder_ref，我们能找到binder_node，
进而完成跨进程通信。

### binder_write_read， flat_binder_object，binder_transaction_data
它们都是binder通信过程中用到的存储信息用的结构体
* flat_binder_object Srevice在Parcel中存储的结构
* binder_transaction_data 是对flat_binder_object的进一步包装，由IPCThreadState根据Parcel中的data生成
* binder_write_read 用户态Service与内核态binder驱动通信使用的数据结构，是ioctl函数的参数之一

## ServiceManager的启动

1. ServiceManager是一个守护进程，随着android系统的启动而启动
2. 通过binder_open()，与binder驱动取得联系，完成内存映射，创建bind_proc。
3. 通过binder_become_context_manager()，利用binder_ioctl向binder驱动发送BINDER_SET_CONTEXT_MGR。创建binder_node，binder_thread。全局变量binder_context_mgr_node此时指向ServiceManager的binder_node。
4. 通过binder_loop()，进入循环，阻塞读取，等待新的事物被加入到binder_thread的list中。

## ServiceManager的获取

1. 通过ProcessState::self()，创建ProcessState，与binder驱动取得联系，完成内存映射，创建bind_proc。
2. 通过ProcessState::getStrongProxyForHandle()，创建了和此线程对应的IPCThreadState（IPCThreadState是Thread Local的变量），返回了handle为0的BpBinder对象
3. 通过interface_cast<IServiceManager>的宏函数转换，返回了new BpServiceManager(new BpBinder(0))。事实上，当一个Service调用defaultServiceManager()获取ServiceManager时，得到的并不是ServiceManager本身，而是一个BpServiceManager对象。
