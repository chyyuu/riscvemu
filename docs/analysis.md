### 编译的文件



iomem.c: 读写寄存器/内存

machine.c: VM utilities  读各种配置等

riscv_cpu.c: 各种csr,tlb_read/write/code, exceptin, intr, phy_mem_read/write_u8/32/64, target_read/write_u8/16/32/64/128
riscvemu.c: RISCV emulator
riscv_machine.c: RISCV machine

cutils.c: dbuf_init/putc/putstr/write/free, mallocz, pstr[cpy/cat], strstart

fs_disk.c:  Filesystem on disk
fs_net.c: Networked Filesystem using HTTP
fs.c:Filesystem utilities  fs_dup/walk_path/end
fs_utils.c: Misc FS utilitie
fs_wget.c: HTTP file download 

block_net.c: nouse

pci.c: Simple PCI bus driver

json.c:Pseudo JSON parser

sdl.c  simplefb.c  softfp.c  virtio.c



### 虚拟机参数
```
typedef struct {
    char *cfg_filename;
    uint64_t ram_size;
    BOOL rtc_real_time;
    BOOL rtc_local_time;
    char *display_device; /* NULL means no display */
    int width, height; /* graphic width & height */
    CharacterDevice *console;
    VMDriveEntry tab_drive[MAX_DRIVE_DEVICE];
    int drive_count;
    VMFSEntry tab_fs[MAX_FS_DEVICE];
    int fs_count;
    VMEthEntry tab_eth[MAX_ETH_DEVICE];
    int eth_count;

    char *cmdline; /* bios or kernel command line */
    BOOL accel_enable; /* enable acceleration (KVM) */
    char *input_device; /* NULL means no input */
    
    /* kernel, bios and other auxiliary files */
    VMFileEntry files[VM_FILE_COUNT];
} VirtMachineParams;
```

### 虚拟机

```
typedef struct RISCVMachine {
    VirtMachine common;
    PhysMemoryMap *mem_map;
    RISCVCPUState *cpu_state;
    uint64_t ram_size;
    /* RTC */
    BOOL rtc_real_time;
    uint64_t rtc_start_time;
    uint64_t timecmp;
    /* PLIC */
    uint32_t plic_pending_irq, plic_served_irq;
    IRQSignal plic_irq[32]; /* IRQ 0 is not used */
    /* HTIF */
    uint64_t htif_tohost, htif_fromhost;

    VIRTIODevice *keyboard_dev;
    VIRTIODevice *mouse_dev;

    int virtio_count;
} RISCVMachine;
```



### 虚拟机状态

```
struct RISCVCPUState {
    target_ulong pc;
    target_ulong reg[32];

#ifdef USE_GLOBAL_VARIABLES
    /* faster to use global variables with emscripten */
    uint8_t *__code_ptr, *__code_end;
    target_ulong __code_to_pc_addend;
#endif
    
#if FLEN > 0
    fp_uint fp_reg[32];
    uint32_t fflags;
    uint8_t frm;
#endif
    
    uint8_t cur_xlen;  /* current XLEN value, <= MAX_XLEN */
    uint8_t priv; /* see PRV_x */
    uint8_t fs; /* MSTATUS_FS value */
    uint8_t mxl; /* MXL field in MISA register */
    
    uint64_t insn_counter;
    BOOL power_down_flag;
    int pending_exception; /* used during MMU exception handling */
    target_ulong pending_tval;
    
    /* CSRs */
    target_ulong mstatus;
    target_ulong mtvec;
    target_ulong mscratch;
    target_ulong mepc;
    target_ulong mcause;
    target_ulong mtval;
    target_ulong mhartid; /* ro */
    uint32_t misa;
    uint32_t mie;
    uint32_t mip;
    uint32_t medeleg;
    uint32_t mideleg;
    uint32_t mcounteren;
    
    target_ulong stvec;
    target_ulong sscratch;
    target_ulong sepc;
    target_ulong scause;
    target_ulong stval;
#if MAX_XLEN == 32
    uint32_t satp;
#else
    uint64_t satp; /* currently 64 bit physical addresses max */
#endif
    uint32_t scounteren;

    target_ulong load_res; /* for atomic LR/SC */

    PhysMemoryMap *mem_map;

    TLBEntry tlb_read[TLB_SIZE];
    TLBEntry tlb_write[TLB_SIZE];
    TLBEntry tlb_code[TLB_SIZE];
};
```

### 中断


### 物理内存布局

### 物理内存结构
struct PhysMemoryMap {
    int n_phys_mem_range;
    PhysMemoryRange phys_mem_range[PHYS_MEM_RANGE_MAX];
    PhysMemoryRange *(*register_ram)(PhysMemoryMap *s, uint64_t addr,
                                     uint64_t size, int devram_flags);
    void (*free_ram)(PhysMemoryMap *s, PhysMemoryRange *pr);
    const uint32_t *(*get_dirty_bits)(PhysMemoryMap *s, PhysMemoryRange *pr);
    void (*set_ram_addr)(PhysMemoryMap *s, PhysMemoryRange *pr, uint64_t addr,
                         BOOL enabled);
    void *opaque;
    void (*flush_tlb_write_range)(void *opaque, uint8_t *ram_addr,
                                  size_t ram_size);
};

### 物理内存范围
typedef struct {
    PhysMemoryMap *map;
    uint64_t addr;
    uint64_t org_size; /* original size */
    uint64_t size; /* =org_size or 0 if the mapping is disabled */
    BOOL is_ram;
    /* the following is used for RAM access */
    int devram_flags;
    uint8_t *phys_mem;
    int dirty_bits_size; /* in bytes */
    uint32_t *dirty_bits; /* NULL if not used */
    uint32_t *dirty_bits_tab[2];
    int dirty_bits_index; /* 0-1 */
    /* the following is used for I/O access */
    void *opaque;
    DeviceReadFunc *read_func;
    DeviceWriteFunc *write_func;
    int devio_flags;
} PhysMemoryRange;


```
#define LOW_RAM_SIZE   0x00010000 /* 64KB */
#define RAM_BASE_ADDR  0x80000000
#define CLINT_BASE_ADDR 0x02000000
#define CLINT_SIZE      0x000c0000
#define HTIF_BASE_ADDR 0x40008000
#define IDE_BASE_ADDR  0x40009000
#define VIRTIO_BASE_ADDR 0x40010000
#define VIRTIO_SIZE      0x1000
#define VIRTIO_IRQ       1
#define PLIC_BASE_ADDR 0x40100000
#define PLIC_SIZE      0x00400000
#define FRAMEBUFFER_BASE_ADDR 0x41000000

  tohost = 0x40008000;
  fromhost = 0x40008008;

```


###　机器初始化
riscvemu.c::main-->riscv_machine.c: :virt_machine_init-->iscvemu.c::virt_machine_run--> virt_machine_interp-->riscv_cpu_interp-->riscv_cpu_interp32(即 riscvemu_template.h 中的glue(riscv_cpu_interp, XLEN))



riscv_machine.c: :virt_machine_init

 --> init  RISCVMachine s->mem_map

  -->riscv_cpu.c:: riscv_cpu_init

​       --> s->pc = 0x1000  //起始PC值　???　

​       -->riscv_cpu.c:: tlb_init

--> iomem.h::cpu_register_ram (0x80000000, ram_size (128MB in config)), (0x0, 64KB) 两个内存块

--> setup real-timer

--> iomem.c:: cpu_register_device  CLINT, PLIC, HTIF

-->iomem.c:: irq_init

--> init vbus( for virtio console/net/block/display/input/filesystem )

-->copy_kernel //拷贝bbl.bin 到0x80000000处，并添加了几条指令



添加的指令如下：

```
riscv_machine.c::copy_kernel()
...    
/* jump_addr = 0x80000000 */
    
    q = (uint32_t *)(ram_ptr + 0x1000);
    q[0] = 0x297 + 0x80000000 - 0x1000; /* auipc t0, jump_addr */
    q[1] = 0x597; /* auipc a1, dtb */
    q[2] = 0x58593 + ((fdt_addr - 4) << 20); /* addi a1, a1, dtb */
    q[3] = 0xf1402573; /* csrr a0, mhartid */
    q[4] = 0x00028067; /* jalr zero, t0, jump_addr */
    
```




### write char
app写两次，一次写到 40008000, 一次写到 40008004地址，
htif_write会被调用,第二次会调用 htif_handle_cmd完成实际的输出    

htif_handle_cmd--> console_write OR  console_read
### device tree


### FDT info
在riscv_machine.c::riscv_build_fdt 函数中


### 指令解释
riscvemu_template.h，被riscv_cpu.c　include了。
