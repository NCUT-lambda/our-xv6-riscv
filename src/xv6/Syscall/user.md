### 2. User mode

**对 ecall 瞬间的状态做快照 (<u>trampoline.S</u>)**

- 填充 struct trapframe (*<u>proc.h</u>*) <= `sd regs` (page position definition: *<u>memlayout.h</u>*)

- 利用 `$sscratch` (S-mode scratch reg) 保存所有 register

- 切换到 *kernel stack* (切换进程对应的“内核线程”)

- 切换到 *kernel address space*

    - 修改 `$satp` 指向 (`csrw satp, t1`)
    - [`csrw`](https://five-embeddev.com/riscv-isa-manual/latest/csr.html)
    - [`sfence.vma`](https://five-embeddev.com/riscv-isa-manual/latest/supervisor.html)

- 跳转 (`jr t0`)到 *usertrap* 进入c代码!

    > usertrap: determine trap **cause**, process it, and return; it changes **stvec** so that kernel <= **kernelvec** rather than uservec; it saves **sepc** (saved user **pc**)

**RISC-V user-level ecall 指令 (<u>trap.c: usertrap</u>)**

- 打开中断 `intr_on();`

- 设置 `$sstatus` 为 `S-mode`

- 更改 `$stvec` 指向 `kernelvec` (`w_stvec((uint64)kernelvec);`)

- 复制 `$pc` 到 `$sepc` ; `$sepc += 4`

- 设置 `$scause` 为 trap 的原因 (*ecall, 8*)

- `$pc` 跳转到 `$stvec` (let `$pc = $stvec`) 并执行

    > **ps.** ***ecall*** **不能** switch page table.
    >
    > **Q.** pc->virtual address, 当 switch page table 时为什么程序没有crash或产生其他垃圾?