# Locking & Concurrency

## Mutex (Sleeping Lock)

### When to Use
- Process context only (can sleep)
- Long critical sections
- May call functions that might sleep

### Pattern
```c
#include <linux/mutex.h>

struct my_device {
    struct mutex lock;
    int shared_data;
};

// Initialization
struct my_device *dev = kzalloc(sizeof(*dev), GFP_KERNEL);
mutex_init(&dev->lock);

// Lock and unlock
mutex_lock(&dev->lock);
dev->shared_data++;
mutex_unlock(&dev->lock);

// Try lock (non-blocking)
if (mutex_trylock(&dev->lock)) {
    dev->shared_data++;
    mutex_unlock(&dev->lock);
}

// Interruptible lock (can be interrupted by signals)
if (mutex_lock_interruptible(&dev->lock))
    return -EINTR;
dev->shared_data++;
mutex_unlock(&dev->lock);

// Cleanup
mutex_destroy(&dev->lock);
```

## Spinlock (Non-Sleeping Lock)

### When to Use
- Short critical sections
- Cannot sleep
- Interrupt context
- Atomic context

### Pattern
```c
#include <linux/spinlock.h>

struct my_device {
    spinlock_t lock;
    int shared_data;
};

// Initialization
struct my_device *dev = kzalloc(sizeof(*dev), GFP_KERNEL);
spin_lock_init(&dev->lock);

// Basic spinlock
spin_lock(&dev->lock);
dev->shared_data++;
spin_unlock(&dev->lock);

// Spinlock with IRQ disable (use in IRQ handlers)
unsigned long flags;
spin_lock_irqsave(&dev->lock, flags);
dev->shared_data++;
spin_unlock_irqrestore(&dev->lock, flags);

// Spinlock BH (bottom half) - disable softirq
spin_lock_bh(&dev->lock);
dev->shared_data++;
spin_unlock_bh(&dev->lock);

// Try lock
if (spin_trylock(&dev->lock)) {
    dev->shared_data++;
    spin_unlock(&dev->lock);
}
```

## RCU (Read-Copy-Update)

### When to Use
- Read-mostly data structures
- High-performance reads
- Writers are rare

### Pattern
```c
#include <linux/rcupdate.h>

struct my_data {
    int value;
    struct rcu_head rcu;
};

struct my_data __rcu *global_ptr;

// Reader (no locks needed!)
rcu_read_lock();
struct my_data *data = rcu_dereference(global_ptr);
if (data)
    printk("Value: %d\n", data->value);
rcu_read_unlock();

// Writer - allocate new, update pointer
struct my_data *new_data = kzalloc(sizeof(*new_data), GFP_KERNEL);
new_data->value = 42;

struct my_data *old_data = global_ptr;
rcu_assign_pointer(global_ptr, new_data);

// Free old data after grace period
synchronize_rcu();  // Wait for all readers to finish
kfree(old_data);

// Or use callback for async free
static void my_rcu_callback(struct rcu_head *rcu)
{
    struct my_data *data = container_of(rcu, struct my_data, rcu);
    kfree(data);
}
call_rcu(&old_data->rcu, my_rcu_callback);
```

## Semaphore

### Pattern
```c
#include <linux/semaphore.h>

struct semaphore sem;

// Initialize with count
sema_init(&sem, 1);  // Binary semaphore (like mutex)
sema_init(&sem, 5);  // Counting semaphore (max 5 concurrent)

// Acquire
down(&sem);
// ... critical section ...
up(&sem);

// Try acquire
if (down_trylock(&sem) == 0) {
    // ... critical section ...
    up(&sem);
}

// Interruptible
if (down_interruptible(&sem))
    return -EINTR;
// ... critical section ...
up(&sem);
```

## Atomic Operations

### Atomic Integers
```c
#include <linux/atomic.h>

atomic_t counter = ATOMIC_INIT(0);

// Read and write
int val = atomic_read(&counter);
atomic_set(&counter, 10);

// Arithmetic
atomic_inc(&counter);            // counter++
atomic_dec(&counter);            // counter--
atomic_add(5, &counter);         // counter += 5
atomic_sub(3, &counter);         // counter -= 3

// Test and modify
if (atomic_dec_and_test(&counter))   // --counter == 0
    printk("Counter reached zero\n");

if (atomic_inc_and_test(&counter))   // ++counter == 0
    printk("Counter is zero\n");

// Compare and swap
int old = 5, new = 10;
if (atomic_cmpxchg(&counter, old, new) == old)
    printk("Swapped %d to %d\n", old, new);
```

### Atomic Bit Operations
```c
unsigned long flags = 0;

// Set and clear bits
set_bit(0, &flags);              // Set bit 0
clear_bit(0, &flags);            // Clear bit 0
change_bit(0, &flags);           // Toggle bit 0

// Test and modify
if (test_and_set_bit(0, &flags))
    printk("Bit was already set\n");

if (test_and_clear_bit(0, &flags))
    printk("Bit was set, now cleared\n");

// Test only
if (test_bit(0, &flags))
    printk("Bit 0 is set\n");
```

## Completion

### Pattern
```c
#include <linux/completion.h>

struct completion done;

// Initialization
init_completion(&done);

// Wait for completion
wait_for_completion(&done);

// Wait with timeout
if (wait_for_completion_timeout(&done, msecs_to_jiffies(5000)) == 0)
    printk("Timeout!\n");

// Wait interruptible
if (wait_for_completion_interruptible(&done))
    return -EINTR;

// Signal completion
complete(&done);        // Wake one waiter
complete_all(&done);    // Wake all waiters
```

## Lock Selection Guide

| Context | Can Sleep? | Lock Type |
|---------|-----------|-----------|
| Process context | Yes | Mutex |
| Process context | No (short) | Spinlock |
| Interrupt handler | No | Spinlock with irqsave |
| Softirq/Tasklet | No | Spinlock with bh |
| Read-mostly data | Varies | RCU |
| Counting resource | Yes | Semaphore |

## Common Pitfalls

❌ **NEVER DO THESE:**
- Sleep while holding spinlock
- Take mutex in interrupt context
- Recursive locking (lock same lock twice)
- Wrong lock order (causes deadlock)
- Forget to unlock
- Use global data without locking
