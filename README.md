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
（1）“reference_count”是一个整数数组。因为我们需要对每一个物理页面，统计它被多少个**用户页表** 共享。  
如果仅被1个用户页表拥有，那么该物理页面实际上是“可写”的；如果被多个用户页表共享，说明该页面不是“可写”的。  
此外，如果仅被1个用户页表拥有，那么该物理页面实际上是“可释放”的；如果被多个用户页表共享，说明该页面是“不可直接释放”的。  
（2）至于页表项的‘Cow bit’和‘PTE_W’，表示了该页面是否被共享、是否可写。（实际上，我认为‘Cow bit’本质上来说不需要，因为如果访问‘Cow bit’，本质上访问了三级页表的最后一级页表的页表项，而访问到最后一级页表的页表项了，实际上就拿到了想要的页面对应的物理地址了，直接访问“reference_count”数组即可）  
![](https://github.com/2351889401/Copy-On-Write-Fork/blob/main/images/copy.png)  

修改后的“uvmcopy”函数实现如下（原来的fork函数主要调用“uvmcopy”为子进程复制父进程的内存空间）  
**注意下面的代码中实际上自定义了一个与“reference_count”数组相关的自旋锁，因为“reference_count”数组在内核中是共享的，不确定多CPU执行时会不会因为操作到同一个reference_count数组的元素而出错。**
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

关于“reference_lock”和“reference_count”数组的定义，实现在“kalloc.c”中，并且在“kalloc()”和“kfree()”中分别对“reference_count”进行了“置1”和“置0”的操作。  
（1）定义  
```
int reference_count[(128*1024*1024)>>12]; //因为查看内核源码内存的最高地址的宏定义：PHYSTOP 实际上是 KERNBASE + 128*1024*1024，
//KERNBASE是真实内存的起始地址，所以实际的内存空间是128*1024*1024，我们以每一页为单位进行访问，所以数组元素的个数为（128*1024*1024）>>12
struct spinlock reference_lock;
```  
（2）kinit()函数初始化“reference_lock”  
```
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
  initlock(&reference_lock, "reference_lock");
}
```
（3）kfree()函数中“reference_count”置0（**这里的置0特意放在了释放物理页面时候的“锁操作”中，因为会担心如果放在“锁操作”的外部（比如说下面语句的最后），此时物理页面已经释放，如果“kalloc”（看下面的代码(4)）将刚刚回收的那一页给分配出去，对reference_count进行置1，之后当前内核线程中再执行reference_count置0的操作，此时这个数据就发生了错误。  总而言之，内核中的共享数据“reference_count”结合多线程（多CPU）的场景可能很多，这里可能考虑的并不完善。但是有一些考虑是好事。**）
```
  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;

  reference_count[((uint64)r - KERNBASE) >> 12] = 0;

  release(&kmem.lock);
```
（4）kalloc()函数中“reference_count”置1（**这里没有放在“锁操作”里面，因为个人认为在正确释放物理页面的情况下，分配时，如果拿到的那一页一定是在内核中独一无二的数据，因此无需加锁**）
```
if(r)
{
  memset((char*)r, 5, PGSIZE); // fill with junk
  reference_count[((uint64)r - KERNBASE) >> 12] = 1;
}
```

2. 当进程尝试向共享页面（COW page）执行“写”操作时，由于页表项的“PTE_W”已经被置位，所以此时会产生page fault。发生page fault时，通过“kalloc()”分配新的物理页面、拷贝旧物理页面的内存到新内存、修改当前页表的PTE的指向、并修改当前PTE的权限位（尤其是要注意COW bit和PTE_W的相应置位）
在“usertrap()”函数中，代码如下：
```
if(r_scause() == 15) {//因为这个实验中是有内存不能write 所以fault的原因应当是store相关的指令 此时该寄存器结果为15
  uint64 va_fault = r_stval(); //page fault的地址在什么地方 在stval()寄存器中
  uint64 down = PGROUNDDOWN(va_fault); //对该地址取基址
  pte_t* pte = walk(p->pagetable, down, 0); //这里没有考虑 pte==0 或者 *pte==0 的情况，假设所有的store指令希望写入的都是真实的、在页表中存在的地址
  //这里是简单起见，在copyout中则考虑了上面的情况；但是这里即使不考虑结果仍然是正确的。
  uint64 pa = PTE2PA(*pte);
  acquire(&reference_lock);
  if(reference_count[(pa-KERNBASE) >> 12] == 1) {
    //这个页面发生了fault 但实际上它的reference_count为1 不应该发生fault的 所以发生fault的原因是这个页面以前是COW页面 
    //但是在其伙伴进程分配新页面、复制、修改COW bit、PTE_W的时候 没有改到当前进程的页表项
    //这就导致当前进程也会fault 
    //而实际上 当前进程不应该fault 所以过来只需要修改一下COW bit和PTE_W即可
    *pte &= ~(1<<8);//把COW bit置为0 表示现在这个页面不被共享
    *pte |= PTE_W;
  }
  else {
    reference_count[(pa-KERNBASE) >> 12] -= 1;//我们现在要分配新的物理页面 所以原来的页面的reference_count需要-1
    //原来页面的PTE_W和COW bit实际上是取决于具体的页表的 我们在这里没有办法修改其他进程的页表
    char* mem = kalloc();
    if(mem == 0) p->killed = 1;//如果没有足够的内存分配 杀死当前进程
    else {
      memset(mem, 0, PGSIZE);
      memmove(mem, (char*)pa, PGSIZE);
      uint flags = PTE_FLAGS(*pte);
      flags |= PTE_W;
      flags &= ~(1<<8);
      mappages(p->pagetable, down, PGSIZE, (uint64)mem, flags);
    }
  }
  release(&reference_lock);
}
else {
  printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
  printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
  p->killed = 1;
}
```

3. 关于“reference_count”的修改，应该遵循以下的原则：
（1） 当进行“fork”时，物理页面的“reference_count”进行“+1”  
（2） 当进程从页表中将物理页面逐出时，“reference_count”进行“-1”  
（3） 当真正希望释放物理页面时，如果“reference_count”为1时真正释放；如果“reference_count”大于2说明被共享，此时不需要真正的释放

5. 对“copyout”进行修改，因为它的操作是将内核数据读取到用户空间，可能遇到写入的
