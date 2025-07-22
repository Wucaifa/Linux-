# Linux内核架构

## 内存管理

### 写时复制COW

![image-20250717135658013](C:\Users\w50050217\AppData\Roaming\Typora\typora-user-images\image-20250717135658013.png)

当进程执行fork()时，会把页表中的权限设置为RD-ONLY（此时两个task_struct还是映射的同一个物理地址），当有一个进程去写该页时，由于该页只读，所以会触发Page Fault，进而申请新的内存，Linux再将页表中的虚拟地址映射到新的物理地址


## 网络协议栈
### 源码目录

