- ## vidmm_init

()

1、查询system mem信息，填入mm_mgr

![image-20220304173026088](/home/sqa/.config/Typora/typora-user-images/image-20220304173026088.png)

2、定义vidmm相关的chip function

![image-20220304173110357](/home/sqa/.config/Typora/typora-user-images/image-20220304173110357.png)

3、虚拟地址空间分为三段（即三个zone），第一段位于local并为所有GPU进程共享，第二段位于pcie并为所有GPU进程共享，第三段为每个GPU进程私有的空间

![image-20220307164438364](/home/sqa/.config/Typora/typora-user-images/image-20220307164438364.png)

​	初始化3段虚拟地址空间的相关信息，包括虚拟地址的起始位置、虚拟空间的大小，并写到mm_mgr中的vm_info

![image-20220307164907873](/home/sqa/.config/Typora/typora-user-images/image-20220307164907873.png)

4、segment主要分为四个段，local visiable、local unvisiable、pcie snoop、pcie unsnoop

![image-20220307170638420](/home/sqa/.config/Typora/typora-user-images/image-20220307170638420.png)

​	初始化各个segment的信息，写到mm_mgr中的vidmm_segment_t  *segment

![image-20220307170517757](/home/sqa/.config/Typora/typora-user-images/image-20220307170517757.png)

5、创建给内核保留的GPU虚拟地址空间reserved_vma，写到adapter的reserved_vma字段，这个vma应该是这个adapter所有，不属于任何一个GPU进程，所有GPU进程共享。初始化vma中的各个zone

![image-20220308162222278](/home/sqa/.config/Typora/typora-user-images/image-20220308162222278.png)

​	初始化这个共享vma的page table，计算各级页表的条目数，以及所占的字节数

![image-20220308162905911](/home/sqa/.config/Typora/typora-user-images/image-20220308162905911.png)

​	从share_pt_segment_id这个segment中分配相应大小的物理内存空间

![image-20220308163044664](/home/sqa/.config/Typora/typora-user-images/image-20220308163044664.png)

​	为这段物理内存映射CPU虚拟地址空间

![image-20220308163228829](/home/sqa/.config/Typora/typora-user-images/image-20220308163228829.png)

​	计算各级页表的GPU物理地址和CPU虚拟地址，写到mm_mgr的vm_info中

![image-20220308163607676](/home/sqa/.config/Typora/typora-user-images/image-20220308163607676.png)

​	由于共享vma已经映射CPU虚拟地址，CPU可以访问这段内存，利用CPU填写L2和L3两级页表

![image-20220308163938171](/home/sqa/.config/Typora/typora-user-images/image-20220308163938171.png)

​	L1级页表就是GPU 上的一些寄存器，将地址写到相应寄存器中

![image-20220308164100613](/home/sqa/.config/Typora/typora-user-images/image-20220308164100613.png)

