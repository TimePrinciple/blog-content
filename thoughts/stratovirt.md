## StratoVirt 讲稿

[Slides](https://docs.google.com/presentation/d/1DX7MBCBqmqeskQrHWp_QlUL-Fry176KbpJb3SHcnoEg/edit?usp=sharing)

大家好，我是来自OERV的何若轻，今天我想分享的是StratoVirt的源码研究。

### Slide 1

首先是这一段代码，这段代码作为`stratovirt`的入口，在调用run方法之后立刻开始对程序读入的命令行参数进行解析，并对解析期间产生的任何错误直接panic，比如文件路径问题或者参数缺失问题，会直接导致程序错误退出。也许是因为`stratovirt`项目立项时间较早的原因，在设计上并未使用例如`clap`这样的crate来提升命令行的友好性和交互性，而是选择了自己实现参数解析相关的功能。虽然这样做给出的错误提示足以帮助有一定基础的程序员或从业者定位问题。

在参数解析完成之后，便会返回`vm_config`,config记录了将要启动的虚拟机类型，内核镜像，rootfs等文件的位置，以及其他的配置信息。在这之后`run()`调用`real_main()`，进入创建虚拟机的阶段。

### Slide 2

进入`read_main()`之后，`stratovirt`开始启动虚拟机前的初始化工作。`stratovirt`首先会初始化`TempCleaner`，该模块用于管理用于既定虚拟机实例运行时所需要的暂时的文件，比如使用qmp协议的socket，如果开启了daemonize选项，则还会有对应的pidfile。`QmpChannel`的初始化用于支持OCI标准。`EventLoop`是可选的，如果在启动时设置了`io_threads`这个选项，则会由`stratovirt`预先启动对应数量的线程来处理来自各个设备的事件，否则所有的事件都由主线程处理。之后主动设置signal hook，以便程序被kill的时候仍可以执行一些清理工作。

### Slide 3

在进入StandardVM分支之后，随后根据`vm_config`的`vm_type`字段来创建对应的（MicroVM 或者 StdVM），设置对应的EventLoop Manager，识别该vm的架构信息等。因为创建VM是一项很重要的操作，再加上我的工作是在riscv64的平台上适配stratovirt，所以我会围绕StdMachine也就是stradard VM纵深地分析。

### Slide 4

`stratovirt`通过crate`kvm_ioctls`来建立与kvm的连接，以及之后的通信，管理。这个crate本身是对KVM API的封装，其易用性和可读性经由该封装已经得到了很好的提升，`Kvm::new()`这个调用会尝试打开`/dev/kvm`，随后`stratovirt`可以通过其返回的fd来管理与kvm的连接。也就是在这个位置，`new()`返回值在match块中被处理，如果在宿主机没有kvm模块的话，下面的Err分支便会提示无法打开`/dev/kvm`.如果打开成功，便会进入下一步，尝试创建VM。

在建立对kvm的连接之后，`stratovirt`通过`create_vm`方法调用告知kvm，使其创建一个vm。这一步其实是`ioctl`对先前返回的`kvm_fd`调用`KVM_CREATE_VM`实现的。该调用会返回一个`vm_fd`,使得`stratovirt` 可以通过该fd来控制，配置之前创建的vm。这里说的配置是指对创建的vm实例的“硬件”加减，比如cpu，内存，设备等，都是通过对这个`vm_fd`进行对应的`ioctl`来实现的。

### Slide 5

在kvm收到来自`stratovirt`关于创建vm的ioctl之后，一个对应的空vm会被建立。这样的vm是没有cpu或者其他的设备的。`stratovirt`通过`HypervisorOp`这个trait为vm创建vcpu以及其他的设备。

对于kvm的vm来说，没有vcpu附着在其上面的话，vm就无法运行。也就是说，`stratovirt`在创建了vm之后，就需要依据先前解析出的`vm_config`来创建vcpu。`stratovirt`通过`create_vcpu`调用来对vm创建vcpu，这一步仍然是对`ioctl`的封装，是对`vm_fd`的`KVM_CREATE_VCPU`调用。该调用需要一个额外的`cpu_id`参数，该参数用于告知vm创建的vcpu及其id，但`stratovirt`对特定vcpu的控制是通过`create_vcpu`所返回的`vcpu_fd`来实现的，比如访问，设置cpu寄存器等等操作。

到这里为止，`stratovirt`为启动虚拟机做的准备工作就完成了。尽管已经为虚拟机创建了所需的vcpu，但其并不会直接开始执行。

### Slide 6

下面几幅图是我根据自己的理解画的，在`StratoVirt`视角下的CPU，其中的箭头表示的并不是数据的流向，而是数据结构之间的引用关系。

每一个CPU结构在`stratovirt`的主进程中拥有大致如下字段

### Slide 7

接下来的内容是虚拟机的启动流程。要启动虚拟机，需要先启动它的每一个vcpu。在这个方法内，主线程首先设置了一个barrier为CPU核心数 + 1，用于掌控CPU开始执行的时点。每一个CPU在创建完成且就绪后，其对应的线程会在这个barrier的位置开始等待。等到每一个CPU线程都就绪之后，且主线程在spawn每一个CPU线程完成自己的工作后，主线程在该barrier等待。知道在barrier等待的线程数达到之后，虚拟机才会开始执行。

### Slide 8

`StratoVirt`开始检查每一个CPU的状态，如果其已经处于运行态则代表当前状态的`StratoVirt`处于不可恢复的错误状态，会直接退出。如果CPU状态正常，主进程会继续行进到`spawn()`的位置，将该cpu的thread_worker和thread_barrier的所有权交给新创建的线程，让该线程作为该CPU的运行载体。

### Slide 9

这段代码已经不再由主线程执行，而是由每个CPU各自的线程来执行。对`CPUThreadWorker`调用init()方法会初始化当前线程的thread_local变量，避免在并发环境下归属于A CPU的进程拿到了归属于B CPU的`CPUThreadWorker`，从而导致`Stratovirt`执行混乱从而错误退出。在该线程初始化完成之后，会在barrier前等待其他的CPU以及主线程。

当barrier前等待的线程数达到要求，也就是主线程和其他的CPU都以及就绪，CPU则通过`kvm_vcpu_exec`开始执行。

### Slide 10

接下来这个位置是`StratoVirt`与kvm内核模块连接的地方，`StratoVirt`会通过在特定vcpu_fd的`ioctl`调用，即`KVM_RUN`使得虚拟机真正地开始执行，此时`StratoVirt`的这个CPU特有线程被阻塞，等待这个`run()`方法返回。该线程会根据`run（）`退出时的返回值来处理各种情况，然后让对应CPU继续执行或者退出。

那么到这里，`StratoVirt`启动虚拟机的大体流程就完成了。