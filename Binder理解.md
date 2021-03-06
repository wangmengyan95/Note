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

## 向ServiceManager添加Service

1. 以MediaPlayerService为例，MediaPlayerService通过调用defaultServiceManager()->addService(String16("media.player"), new MediaPlayerService())向ServiceManager添加服务。
2. 在addService()中，利用Parcel存储需要向ServiceManager发送的信息，包括InterfaceDescriptor，Name，StrongBinder等等。比较重要的就是StrongBinder这个信息，从源代码看data.writeStrongBinder(service)，这个就是MediaPlayerService本身。
3. 通过Parcel::flatten_binder()，我们对MediaPlayerService对象进行封装，生成flat_binder_object结构体obj。obj.cookie的值为MediaPlayerService对象本身，obj.binder的值为MediaPlayerService对象的弱引用，obj.type的值为BINDER_TYPE_BINDER。
4. 将相应的信息填入Parcel后，通过BpBinder::transact()，进而使用IPCThreadState::transact()，将Parcel中的data进一步包装成binder_transaction_data结构体tr。tr.target.handle的值为BpBinder中mHandle的值，即0，tr.code的值为ADD_SERVICE_TRANSACTION，tr.data.ptr.buffer的值为Parcel中data的起始地址。除了binder_transaction_data，最终传输的数据还包含了值为BC_TRANSACTION的cmd码。
5. 通过IPCThreadState::talkWithDriver()，我们将binder_transaction_data结构体进一步包装成binder_write_read结构体。bwr.write_buffer指向了binder_transaction_data中的数据地址。
6. 通过binder_ioctl()，进一步binder_thread_write()，binder驱动将数据从用户态拷贝到了内核态，并且转换为了binder_transaction_data结构体tr（此时binder驱动只是开始从用户态读取数据到内核态，并不是一次性将所有数据全部转移到内核态）。
7. 通过值为BC_TRANSACTION的cmd码，binder_transaction()被调用。首先，因为tr的handle值为0，我们可以知道消息的目标是ServiceManager，它的信息保存在了全局变量binder_context_mgr_node中。接下来，我们通过binder_transaction_data可以得到之前打包进去的flat_binder_object对象。因为MediaPlayerService第一次与binder驱动交互，binder驱动会生成与MediaPlayerService相对应的binder_node。此后，我们生成为binder_node在target_proc中的ref(即MediaPlayerService binder实体在ServiceManager进程中的Binder引用ref)，并且将flat_binder_object结构体中的handle修改为了ref->desc的值，type由BINDER_TYPE_BINDER变为了BINDER_TYPE_HANDLE。为了与ServiceManager通信，binder_transaction()中新建了一个结构体binder_transaction t。t中存储了target_proc，target_thread和从用户态拷贝到内核态的数据等等信息。当对flat_binder_object修改完毕后，我们将t加入了target_thread的todo列表中，并且唤醒了target_thread。
8. 当ServiceManager在内核态被从binder_thread_read()中的阻塞读中唤醒后，会根据binder_transaction t生成对应的binder_transaction_data tr，需要注意的是，在这个过程中，binder驱动将t中数据的地址转换为了用户态的地址tr.data.ptr.buffer = (void *)t->buffer->data + proc->user_buffer_offset。
9. ServiceManager在用户态从binder_loop()中继续执行进入binder_parse()。首先ServiceManager会将binder驱动中读取的数据转换为binder_txn结构体，binder_txn是binder_transaction_data在ServiceManager对应的结构体。接着，ServiceManager会用svcmgr_handler处理得到的binder_txn结构体。在进行了有效性检测后ServiceManager会尝试获取MediaPlayerService的binder索引，即在7中的ref->desc。此后，在do_add_service()中，会根据添加Service的name，ref等等信息生成svcinfo并且将svcinfo添加到svclist链表中。

## 从ServiceManager获取Service

1. 通过defaultServiceManager()获得IServiceManager的引用sm，然后通过sm->getService(String16("media.player"))获取MediaPlayerService的服务。
2. 通过BpServiceManager::checkService()，将服务名字写入Parcel data中。
3. 通过IPCThreadState::writeTransactionData()，将Parcel data转换为binder_transaction_data tr，并写入mOut buffer中。
4. 通过IPCThreadState::talkWithDriver()，调用ioctl()，与binder driver通信。
5. 通过binder驱动的binder_transaction()，将事务添加至ServiceManager的todo list中，并唤醒ServiceManager的进程。
6. 通过IPCThreadState::binder_thread_read()，将数据由内核空间拷贝至用户空间。
7. 通过ServiceManager::binder_parse()，将数据交给回调函数svcmgr_handler处理。
8. 通过do_find_service，根据服务名字"media.player"，取得MediaPlayerService binder实体在ServiceManager进程中的Binder引用ref。
9. 通过binder_send_reply(), binder_write(), 将binder_write_read结构体写入binder内核驱动。
10. 通过binder驱动的binder_transaction()，根据MediaPlayerService binder实体在ServiceManager进程中的Binder引用ref，获得MediaPlayerService的引用binder_ref结构体，通过binder_ref结构体获得MediaPlayerService的binder实体ref->node，最后为MediaPlayerService的binder实体在MediaPlayer的binder上下文中生成引用new_ref，并将这个new_ref->desc返回给MediaPlayer。
11. 通过Parcel::readStrongBinder()，Parcel::unflatten_binder(), 从binder驱动返回的数据中读出了flat->handle的值，生成了BpBinder(handle)对象。
12. 通过nterface_cast<IMediaPlayerService>(binder)，返回BpServiceManager(binder)对象。
