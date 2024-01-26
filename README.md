**该实验实现的是“fork的写时复制（Copy-On-Write）”。实验链接：（https://pdos.csail.mit.edu/6.S081/2020/labs/cow.html ）**  
**如果fork时将父进程的全部内存都拷贝到子进程的内存通常不是一个理想的行为。因为，首先父进程的内存可能很大，这样拷贝可能很花时间；
其次，这样的拷贝可能是不必要的，因为子进程后续的行为（比如exec）会把原始的内存全部放弃掉。  
所以，在fork创建子进程时先共享父进程的内存，只是把共享的内存的“可写(writable)”这一性质暂时去掉；在进程出现“写”操作的行为时才真正去分配物理页面、执行自己的“写”操作。这是一个理想的行为。**  

******  
**无论是上一个实验的“延迟内存分配（Lazy Page allocation）”还是这个实验的“写时复制fork（Copy-On-Write Fork）”，本质上都是利用“页表项的标志位”和“trap”的机制实现的。实验要求上的这句话“There is a saying in computer systems that any systems problem can be solved with a level of indirection.”现在也许更有体会了。**
******  

方案流程：  
1. 如下图所示，这时的“fork”就不再是给子进程拷贝全部的父进程的内存空间了（原来的fork是这样的），现在的操作是：子进程创建新页表、并且子进程相应的PTE指向父进程的实际物理页面。  
**需要注意的是，此时的物理页面的reference_count需要 ‘+1’ ，并且父子进程页表项的‘Cow bit’和‘PTE_W’分别置位**  
（1）“reference_count”是一个整数数组。因为我们需要对每一个物理页面，统计它被多少个**用户页表** 共享。如果仅被1个用户页表拥有，那么该物理页面实际上是“可写”的；如果被多个用户页表共享，说明该页面不是“可写”的。  
（2）至于页表项的‘Cow bit’和‘PTE_W’，表示了该页面是否被共享、是否可写。（实际上，我认为‘Cow bit’本质上来说不需要，因为如果访问‘Cow bit’，本质上访问了三级页表的最后一级页表的页表项，而访问到最后一级页表的页表项了，实际上就拿到了想要的页面对应的物理地址了，直接访问“reference_count”数组即可）  
![](https://github.com/2351889401/Copy-On-Write-Fork/blob/main/images/copy.png)  

2. 当进程尝试向共享页面（COW page）执行“写”操作时，由于页表项的“PTE_W”已经被置位，所以此时会产生page fault。发生page fault时，通过“kalloc()”分配新的物理页面、拷贝旧物理页面的内存到新内存、修改当前页表的PTE的指向、并修改当前PTE的权限位（尤其是要注意COW bit和PTE_W的相应置位）
