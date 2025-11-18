·## 简介
#### 混合关键性系统总体介绍

#### 内核模块

 mcs内核部分通过注册一个字符设备，用户态的`micad`进程通过对`/dev/mcs`这个文件使用`ioctl`,`mmap`等系统调用，触发内核模块中的回调函数实现保留内存的信息读取，启动`remote cpu`,注册/发送IPI中断，mmap共享内存区域到用户空间，为用户态的`MICA`提供的`rtos client`注册，生命周期管理等功能提供支持。

![](imgs/Pasted%20image%2020250616155643.png)

 对于内核模块部分，架构相关的代码就是`remote cpu`唤醒与`IPI`中断的部分。对于`MilkV-Duo`的`MCS`部分来说，这两个功能需要通过`MilkV-Duo`板卡特殊的驱动实现，这部分通过消息队列与`mailbox`与片上的小核通信，实现`cpu startup`，发送`IPI`中断。对于`qemu-riscv64`平台的MCS，与`Linux smp boot`过程中使用同样的方式用`sbi_ecall`调用`HSM`扩展`（HART State Management Extension）`部分功能去唤醒`remote cpu`。

![](imgs/Pasted%20image%2020251128143453.png)

 在`IPI`中断涉及到中断子系统的部分，`MilkV-Duo`的支持中同样需要使用与`remote cpu`唤醒时一样的消息队列的方式实现`IPI`类似的给`client OS`发送中断信号。`qemu-riscv64`平台实现使用`Linux`提供的`IPI`相关通用API来注册回调函数与发送IPI中断，这部分代码与`aarch64`部分兼容，只有`IPI`中断号稍有不同。

![](imgs/Pasted%20image%2020251128143955.png)
#### 用户态 MICA

 `MCS`用户态部分`MICA`通过在`OpenAMP`框架下通过对`kernel module`注册的字符设备进行`ioctl`，`mmap`等系统调用，触发内核模块中负责处理系统调用的回调函，实现了了对`rtos`的生命周期管理，系统资源初始化两个主要功能。

![](imgs/Pasted%20image%2020250630171456.png)
 `MICA`的操作逻辑`create`,`start`,`stop`映射了在`OpenAMP`设计框架下的`remote proc`状态转移流程，在用户态提供了一个符合规范的生命周期管理接口，与`kernel module`相互协同完成`MCS`混合部署的功能实现与逻辑框架。

| State Transition      | Transition Trigger                                                                                                                            |
| :-------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------- |
| Offline -> Configured | Configure the remote to make it able to load application;<br>`remoteproc_configure(&rproc, &config_data)`                                     |
| Configured -> Ready   | load firmware ;<br>`remoteproc_load(&rproc, &path, &image_store, &image_store_ops, &image_info)`                                              |
| Ready -> Running      | start the processor; <br>`remoteproc_start(&rproc)`                                                                                           |
| Ready -> Stopped      | stop the processor; <br>`remoteproc_stop(&rproc)`; <br>`remoteproc_shutdown(&rproc)`(Stopped is the intermediate state of shutdown operation) |
| Running -> Stopped    | stop the processor; <br>`remoteproc_stop(&rproc)`; <br>`remoteproc_shutdown(&rproc)`                                                          |
| Stopped -> Offline    | shutdown the processor; <br>`remoteproc_shutdown(&rproc)`                                                                                     |
