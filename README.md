**该实验实现的是“fork的写时复制（Copy-On-Write）”。实验链接：（https://pdos.csail.mit.edu/6.S081/2020/labs/cow.html ）**  
**如果fork时将父进程的全部内存都拷贝到子进程的内存通常不是一个理想的行为。因为，首先父进程的内存可能很大，这样拷贝可能很花时间；
其次，这样的拷贝可能是不必要的，因为子进程后续的行为（比如exec）会把原始的内存全部放弃掉。  
所以，在fork创建子进程时先共享父进程的内存，只是把共享的内存的“可写(writable)”这一性质暂时去掉；在进程出现“写”操作的行为时才真正去分配物理页面、执行自己的“写”操作。这是一个理想的行为。**  

******  
**无论是上一个实验的“延迟内存分配（Lazy Page allocation）”还是这个实验的“写时复制fork（Copy-On-Write Fork）”，本质上都是利用“页表项的标志位”和“trap”的机制实现的。实验要求上的这句话“There is a saying in computer systems that any systems problem can be solved with a level of indirection.”现在也许更有体会了。**
******  
