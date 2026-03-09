# Memory Management

## Memory Allocation Patterns

### Kernel Memory (kmalloc/kfree)
```c
// Small allocations (< PAGE_SIZE, typically 4KB)
void *ptr = kmalloc(size, GFP_KERNEL);     // Can sleep, most common
if (!ptr)
    return -ENOMEM;

// Zero-initialized allocation
void *ptr = kzalloc(size, GFP_KERNEL);     // Prefer this when you need zeroed memory

// Atomic allocation (in IRQ, spinlock, or atomic context)
void *ptr = kmalloc(size, GFP_ATOMIC);     // Cannot sleep!

// Always free memory
kfree(ptr);                                 // Safe to call with NULL
```

### GFP Flags (CRITICAL TO UNDERSTAND)
```c
GFP_KERNEL      // Normal allocation, can sleep, use in process context
GFP_ATOMIC      // Cannot sleep, use in interrupt/atomic context
GFP_NOWAIT      // Like ATOMIC but can fail more easily
GFP_NOIO        // Cannot do I/O operations (filesystem code)
GFP_NOFS        // Cannot call filesystem operations
GFP_DMA         // Allocate from DMA-capable memory
GFP_DMA32       // Allocate DMA memory below 4GB
GFP_HIGHUSER    // User space page allocation

// Common combinations
GFP_KERNEL | __GFP_ZERO     // Like kzalloc
GFP_KERNEL | __GFP_NOWARN   // Don't warn on allocation failure
```

### Large Memory Allocation (vmalloc/vfree)
```c
// For large allocations (> PAGE_SIZE)
// Creates virtually contiguous memory (physically may be scattered)
void *ptr = vmalloc(size);
if (!ptr)
    return -ENOMEM;

// Zero-initialized vmalloc
void *ptr = vzalloc(size);

// Free vmalloc memory
vfree(ptr);

// NOTE: vmalloc is slower than kmalloc
// Use vmalloc only for large allocations
```

### Page Allocation
```c
// Allocate 2^order pages (physically contiguous)
struct page *page = alloc_pages(GFP_KERNEL, order);
if (!page)
    return -ENOMEM;

// Get kernel virtual address
void *addr = page_address(page);

// Free pages
__free_pages(page, order);
// or
free_pages((unsigned long)addr, order);

// Examples:
// order=0: 1 page (4KB)
// order=1: 2 pages (8KB)
// order=2: 4 pages (16KB)
```

### DMA Memory Allocation
```c
// For DMA operations
dma_addr_t dma_handle;
void *cpu_addr = dma_alloc_coherent(&dev->dev, size, &dma_handle, GFP_KERNEL);
if (!cpu_addr)
    return -ENOMEM;

// Free DMA memory
dma_free_coherent(&dev->dev, size, cpu_addr, dma_handle);

// Streaming DMA mapping
dma_addr_t dma_addr = dma_map_single(&dev->dev, ptr, size, DMA_TO_DEVICE);
if (dma_mapping_error(&dev->dev, dma_addr))
    return -ENOMEM;

// Unmap after DMA complete
dma_unmap_single(&dev->dev, dma_addr, size, DMA_TO_DEVICE);
```

### Per-CPU Memory
```c
// Allocate per-CPU variable
static DEFINE_PER_CPU(int, my_counter);

// Access on current CPU
int val = this_cpu_read(my_counter);
this_cpu_write(my_counter, val);
this_cpu_inc(my_counter);
this_cpu_add(my_counter, delta);

// Access on specific CPU
int val = per_cpu(my_counter, cpu_id);
per_cpu(my_counter, cpu_id) = val;
```

### Memory Barriers
```c
// Compiler barrier (prevent compiler reordering)
barrier();

// Memory barriers (prevent CPU reordering)
mb();       // Full memory barrier
rmb();      // Read memory barrier
wmb();      // Write memory barrier
smp_mb();   // SMP memory barrier
smp_rmb();  // SMP read barrier
smp_wmb();  // SMP write barrier

// Example usage:
buffer_ready = true;
wmb();  // Ensure write to buffer_ready is visible
wake_up(&wait_queue);
```

## Memory Debugging

### Enable in .config
```
CONFIG_DEBUG_KMEMLEAK=y          // Detect memory leaks
CONFIG_SLUB_DEBUG=y              // SLUB allocator debugging
CONFIG_DEBUG_PAGEALLOC=y         // Page allocation debugging
CONFIG_KASAN=y                   // Kernel Address Sanitizer
CONFIG_DEBUG_OBJECTS=y           // Object lifecycle debugging
```

### Runtime Detection
```c
// Detect buffer overruns at runtime
BUILD_BUG_ON(sizeof(struct foo) > PAGE_SIZE);

// Check allocation size at compile time
#define MY_ALLOC_SIZE 1024
static_assert(MY_ALLOC_SIZE <= PAGE_SIZE, "too large");
```
