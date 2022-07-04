- ## vidmm_init

(这个函数的作用是GPU内存管理的初始化，具体实现方式主要是定义adapter_t中的vidmm_mgr_t)

![image-20220301171827717](/home/sqa/.config/Typora/typora-user-images/image-20220301171827717.png)

1、查询系统可以提供的pcie mem的大小，填到vidmm_mgr_t中

![image-20220301172248857](/home/sqa/.config/Typora/typora-user-images/image-20220301172248857.png)

2、定义vidmm相关的chip func

![image-20220301172352409](/home/sqa/.config/Typora/typora-user-images/image-20220301172352409.png)

3、从 chip层查询各个segment的信息，存到vidmm_chip_segment_info_t  *segment_desc中

![image-20220301191307360](/home/sqa/.config/Typora/typora-user-images/image-20220301191307360.png)

​	将chip层得到的segment信息写入到vidmm_mgr_t中的vidmm_segment_t字段

![image-20220301191731861](/home/sqa/.config/Typora/typora-user-images/image-20220301191731861.png)

​	每个segment是一段GPU 虚拟地址空间，建堆对这段空间进行管理

![image-20220301192154506](/home/sqa/.config/Typora/typora-user-images/image-20220301192154506.png)

4、初始化paging segment，paging segment这个段是中转用途，位于PCIE显存，用于将local显存通过PCIE显存中转最后放到system memory中。paging segment和上面的segment有所不同，不是一个并列关系，而是从上述pcie显存的snoop或者unsnoop段中划分一段内存空间，专门作为paging segment。

![image-20220301192859838](/home/sqa/.config/Typora/typora-user-images/image-20220301192859838.png)

5、从system mem中分配一页（即4）内存来初始化mm_mgr_t中的dummy page，dummy page主要是gart table初始化时用到。gart table的作用是将GPU的pcie mem部分的虚拟内存空间映射到system mem的物理地址空间，在gart table初始化时，将虚拟内存空间中的所有页都映射为dummy page对应的物理页，来预防访问出错。

![image-20220302184006979](/home/sqa/.config/Typora/typora-user-images/image-20220302184006979.png)

6、初始化gart table。从chip层查询gart table的大小

![image-20220303151523969](/home/sqa/.config/Typora/typora-user-images/image-20220303151523969.png)

​	根据gart table大小调用vidmm_allocate_segment_memory函数从local mem中分配内存，

![image-20220303151655703](/home/sqa/.config/Typora/typora-user-images/image-20220303151655703.png)

​	这块内存位于local mem，将其map到CPU进程的虚拟内存空间。这是因为后续填写gart table的动作都是CPU完成的，不映射的话CPU无法访问gart table所在的local mem。

![image-20220303152102887](/home/sqa/.config/Typora/typora-user-images/image-20220303152102887.png)

​	将3级页表的地址填到2级页表中

![image-20220303155333940](/home/sqa/.config/Typora/typora-user-images/image-20220303155333940.png)

7、设置一些GPU寄存器的值，包括将2级gart table的地址写到寄存器中，这样GPU才能访问gart table。这样一级页表即为寄存器MMIO_MMU_START_ADDRESS开始的一段区域，二级页表位于local mem可以通过一级页表访问，三级页表通过二级页表来访问。

![image-20220303155729090](/home/sqa/.config/Typora/typora-user-images/image-20220303155729090.png)











------

- ## vidmm_allocate_segment_memory

(这个函数的作用是从segment中分配虚拟内存，如果是local mem，则物理内存可以直接对应，如果是pcie mem，则分配system mem作为物理内存与之对应，并映射gart table记录对应关系)

1、数据结构vidmm_segment_t定义了每个segment，每个segment都是一段连续的虚拟内存空间，vidmm_segment_t中有一个heap成员来管理内存的分配和释放。从heap中分配一个list_node来表示分配出的虚拟内存区域。

![image-20220303140234102](/home/sqa/.config/Typora/typora-user-images/image-20220303140234102.png)

2、如果是pcie mem，需要分配system mem给它作为对应的物理内存

![image-20220303141102019](/home/sqa/.config/Typora/typora-user-images/image-20220303141102019.png)

​	将物理内存页和虚拟内存页的对应关系写到gart table中

![image-20220303150322148](/home/sqa/.config/Typora/typora-user-images/image-20220303150322148.png)











------

- ## vidmm_create_allocation

（这个函数的作用就是创建allocation）

1、创建vidmm_allocation_t结构，将allocation的描述信息写到vidmm_allocation_t中

![image-20220303162913242](/home/sqa/.config/Typora/typora-user-images/image-20220303162913242.png)

2、分配一段虚拟内存空间给vidmm_allocation_t

![image-20220303164734770](/home/sqa/.config/Typora/typora-user-images/image-20220303164734770.png)

​	如果没有分配成功，调用vidmmi_segment_unresident_allocations函数执行paging out操作

![image-20220303192150955](/home/sqa/.config/Typora/typora-user-images/image-20220303192150955.png)

3、如果分配的虚拟内存所在的segment属于pcie mem，则分配相同大小的system mem作为物理内存

![image-20220304101139801](/home/sqa/.config/Typora/typora-user-images/image-20220304101139801.png)

​	如果物理内存分配失败，则调用vidmmi_do_swap_out函数来执行swap out操作

![image-20220304101417265](/home/sqa/.config/Typora/typora-user-images/image-20220304101417265.png)

4、将虚拟内存和分配的system mem进行映射

![image-20220304102533898](/home/sqa/.config/Typora/typora-user-images/image-20220304102533898.png)

5、给allocation绑定同步对象fence

![image-20220304110512079](/home/sqa/.config/Typora/typora-user-images/image-20220304110512079.png)

6、根据allocation的flag（这个flag是由user mode设置的），将创建的allocation挂到segment的unpagable list或者pagable list下面

![image-20220304140746351](/home/sqa/.config/Typora/typora-user-images/image-20220304140746351.png)











------

- ## vidmmi_segment_unresident_allocations

(这个函数的作用是执行paging out操作)









------

- ## vidmmi_do_swap_out

（这个函数的作用是执行swap out操作。swap out是指当paging segment对应的system mem不足时，将这一部分system mem转移到硬盘的swap分区）

1、选择待swap out的allocation，并将这些allocation从对应的paging segment的pagable list中去除

![image-20220304144443545](/home/sqa/.config/Typora/typora-user-images/image-20220304144443545.png)

2、执行swap out操作，如果成功，则将这些allocation占用的system mem释放，如果失败，则将allocation重新放回paging segment

![image-20220304145017611](/home/sqa/.config/Typora/typora-user-images/image-20220304145017611.png)

3、检查是否还需要swap out更多的allocation，如果需要，则再次执行swap out，如果paging segment的allocation不够，则先调用函数vidmmi_segment_unresident_allocations执行paging out操作

![image-20220304154122238](/home/sqa/.config/Typora/typora-user-images/image-20220304154122238.png)

