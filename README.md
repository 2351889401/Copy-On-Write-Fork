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

修改后的“uvmcopy”函数实现如下（原来的fork函数主要调用“uvmcopy”为子进程复制父进程的内存空间）  
**注意下面的代码中实际上自定义了一个与“reference_count”数组相关的自旋锁，因为“reference_count”数组在内核中是共享的，不确定多CPU执行时会不会因为操作到同一个reference_count而出错**
```
// Given a parent process's page table, copy
// its memory into a child's page table.
// Copies both the page table and the
// physical memory.
// returns 0 on success, -1 on failure.
// frees any allocated pages on failure.
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  // char *mem;

  pte_t *new_pte;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    

    *pte &= (~PTE_W); //父进程对该页面失去了写的权限
    *pte |= (1<<8); //父进程的cow bit置位
    flags = PTE_FLAGS(*pte);//新的flags
    acquire(&reference_lock);
    reference_count[(pa - KERNBASE) >> 12] += 1;
    release(&reference_lock);

    //子进程此时不再申请新页面了
    new_pte = walk(new, i, 1);
    *new_pte = PA2PTE(pa);
    *new_pte |= flags;

    // if((mem = kalloc()) == 0)
    //   goto err;
    // memmove(mem, (char*)pa, PGSIZE);
    // if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
    //   kfree(mem);
    //   goto err;
    // }
  }
  return 0;

//  err:
//   uvmunmap(new, 0, i / PGSIZE, 1);
//   return -1;
}
```


2. 当进程尝试向共享页面（COW page）执行“写”操作时，由于页表项的“PTE_W”已经被置位，所以此时会产生page fault。发生page fault时，通过“kalloc()”分配新的物理页面、拷贝旧物理页面的内存到新内存、修改当前页表的PTE的指向、并修改当前PTE的权限位（尤其是要注意COW bit和PTE_W的相应置位）
