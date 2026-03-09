# DMA Engine

## DMA Architecture Overview

DMA (Direct Memory Access) Engine cho phép transfer data giữa memory và peripherals mà không cần CPU can thiệp, tăng hiệu suất đáng kể.

```
┌──────────────────────────────────────────┐
│   DMA Client (SPI, UART, MMC, etc.)     │  <- dma_request_chan(), dmaengine_prep_*
├──────────────────────────────────────────┤
│   DMA Engine Core                        │  <- Transfer management
├──────────────────────────────────────────┤
│   DMA Controller Driver (EDMA, etc.)    │  <- Hardware-specific implementation
└──────────────────────────────────────────┘
```

### Key Concepts
- **DMA Channel**: Kênh truyền DMA
- **DMA Transfer**: Quá trình copy data
- **DMA Direction**: MEM_TO_MEM, MEM_TO_DEV, DEV_TO_MEM, DEV_TO_DEV
- **Scatter-Gather**: Transfer nhiều memory regions trong một operation
- **Cyclic Transfer**: Transfer liên tục (dùng cho audio/video)

## Essential Headers

```c
#include <linux/dmaengine.h>
#include <linux/dma-mapping.h>
```

## DMA Consumer API

### Basic DMA Setup

```c
#include <linux/dmaengine.h>

struct mydev_data {
    struct dma_chan *dma_tx;
    struct dma_chan *dma_rx;
    dma_addr_t dma_addr;
    void *buffer;
    size_t buf_size;
};

static int mydev_probe(struct platform_device *pdev)
{
    struct mydev_data *data;
    int ret;
    
    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    // Request DMA channels
    data->dma_tx = dma_request_chan(&pdev->dev, "tx");
    if (IS_ERR(data->dma_tx)) {
        ret = PTR_ERR(data->dma_tx);
        if (ret != -EPROBE_DEFER)
            dev_err(&pdev->dev, "Failed to get TX DMA channel\n");
        return ret;
    }
    
    data->dma_rx = dma_request_chan(&pdev->dev, "rx");
    if (IS_ERR(data->dma_rx)) {
        ret = PTR_ERR(data->dma_rx);
        dma_release_channel(data->dma_tx);
        return ret;
    }
    
    // Allocate DMA buffer
    data->buf_size = PAGE_SIZE;
    data->buffer = dmam_alloc_coherent(&pdev->dev, data->buf_size,
                                        &data->dma_addr, GFP_KERNEL);
    if (!data->buffer) {
        ret = -ENOMEM;
        goto err_free_channels;
    }
    
    return 0;

err_free_channels:
    dma_release_channel(data->dma_rx);
    dma_release_channel(data->dma_tx);
    return ret;
}

static int mydev_remove(struct platform_device *pdev)
{
    struct mydev_data *data = platform_get_drvdata(pdev);
    
    // Release DMA channels
    dma_release_channel(data->dma_rx);
    dma_release_channel(data->dma_tx);
    
    return 0;
}
```

### Simple DMA Transfer (Memory to Device)

```c
static void mydev_dma_callback(void *param)
{
    struct completion *done = param;
    
    pr_info("DMA transfer complete\n");
    complete(done);
}

static int mydev_dma_transfer(struct mydev_data *data,
                               void *buffer, size_t len)
{
    struct dma_async_tx_descriptor *desc;
    struct completion done;
    dma_cookie_t cookie;
    enum dma_status status;
    
    init_completion(&done);
    
    // Prepare DMA transfer
    desc = dmaengine_prep_slave_single(data->dma_tx,
                                        data->dma_addr,
                                        len,
                                        DMA_MEM_TO_DEV,
                                        DMA_PREP_INTERRUPT);
    if (!desc) {
        dev_err(data->dev, "Failed to prepare DMA\n");
        return -EINVAL;
    }
    
    // Set callback
    desc->callback = mydev_dma_callback;
    desc->callback_param = &done;
    
    // Submit transfer
    cookie = dmaengine_submit(desc);
    if (dma_submit_error(cookie)) {
        dev_err(data->dev, "Failed to submit DMA\n");
        return -EINVAL;
    }
    
    // Start DMA engine
    dma_async_issue_pending(data->dma_tx);
    
    // Wait for completion
    wait_for_completion(&done);
    
    // Check status
    status = dma_async_is_tx_complete(data->dma_tx, cookie, NULL, NULL);
    if (status != DMA_COMPLETE) {
        dev_err(data->dev, "DMA transfer failed\n");
        return -EIO;
    }
    
    return 0;
}
```

### Scatter-Gather DMA Transfer

```c
static int mydev_sg_transfer(struct mydev_data *data,
                              struct scatterlist *sgl,
                              unsigned int sg_len)
{
    struct dma_async_tx_descriptor *desc;
    dma_cookie_t cookie;
    
    // Prepare scatter-gather transfer
    desc = dmaengine_prep_slave_sg(data->dma_tx,
                                    sgl, sg_len,
                                    DMA_MEM_TO_DEV,
                                    DMA_PREP_INTERRUPT);
    if (!desc)
        return -EINVAL;
    
    desc->callback = mydev_dma_callback;
    desc->callback_param = &data->done;
    
    cookie = dmaengine_submit(desc);
    if (dma_submit_error(cookie))
        return -EINVAL;
    
    dma_async_issue_pending(data->dma_tx);
    
    return 0;
}
```

### Cyclic DMA (for Audio/Video)

```c
static int mydev_cyclic_start(struct mydev_data *data)
{
    struct dma_async_tx_descriptor *desc;
    dma_cookie_t cookie;
    
    // Prepare cyclic transfer
    desc = dmaengine_prep_dma_cyclic(data->dma_tx,
                                      data->dma_addr,
                                      data->buf_size,
                                      data->period_size,
                                      DMA_MEM_TO_DEV,
                                      DMA_PREP_INTERRUPT);
    if (!desc)
        return -EINVAL;
    
    desc->callback = mydev_period_callback;
    desc->callback_param = data;
    
    cookie = dmaengine_submit(desc);
    if (dma_submit_error(cookie))
        return -EINVAL;
    
    dma_async_issue_pending(data->dma_tx);
    
    return 0;
}

static void mydev_period_callback(void *param)
{
    struct mydev_data *data = param;
    
    // Called at each period completion
    // Refill buffer for continuous playback
}
```

### DMA Configuration

```c
static int mydev_configure_dma(struct mydev_data *data)
{
    struct dma_slave_config config;
    int ret;
    
    memset(&config, 0, sizeof(config));
    
    // TX configuration
    config.direction = DMA_MEM_TO_DEV;
    config.dst_addr = data->fifo_addr;  // Hardware FIFO address
    config.dst_addr_width = DMA_SLAVE_BUSWIDTH_4_BYTES;
    config.dst_maxburst = 8;  // Burst size
    
    ret = dmaengine_slave_config(data->dma_tx, &config);
    if (ret) {
        dev_err(data->dev, "Failed to configure TX DMA\n");
        return ret;
    }
    
    // RX configuration
    config.direction = DMA_DEV_TO_MEM;
    config.src_addr = data->fifo_addr;
    config.src_addr_width = DMA_SLAVE_BUSWIDTH_4_BYTES;
    config.src_maxburst = 8;
    
    ret = dmaengine_slave_config(data->dma_rx, &config);
    if (ret) {
        dev_err(data->dev, "Failed to configure RX DMA\n");
        return ret;
    }
    
    return 0;
}
```

## Device Tree Bindings

```dts
&spi0 {
    dmas = <&edma 16 0>, <&edma 17 0>;
    dma-names = "tx", "rx";
};

&uart0 {
    dmas = <&edma 26 0>, <&edma 27 0>;
    dma-names = "tx", "rx";
};

&mmc1 {
    dmas = <&edma 24 0>, <&edma 25 0>;
    dma-names = "tx", "rx";
};
```

## DMA Controller Driver API

### DMA Controller Operations

```c
#include <linux/dmaengine.h>

struct my_dma_chan {
    struct dma_chan chan;
    void __iomem *base;
    int id;
    struct dma_slave_config config;
};

static struct dma_async_tx_descriptor *
my_prep_slave_sg(struct dma_chan *chan, struct scatterlist *sgl,
                 unsigned int sg_len, enum dma_transfer_direction dir,
                 unsigned long flags, void *context)
{
    struct my_dma_chan *mchan = container_of(chan, struct my_dma_chan, chan);
    struct dma_async_tx_descriptor *desc;
    struct scatterlist *sg;
    int i;
    
    desc = kzalloc(sizeof(*desc), GFP_NOWAIT);
    if (!desc)
        return NULL;
    
    // Setup transfer descriptor
    for_each_sg(sgl, sg, sg_len, i) {
        // Program DMA controller registers
        // for each scatter-gather element
    }
    
    return desc;
}

static int my_dma_terminate_all(struct dma_chan *chan)
{
    struct my_dma_chan *mchan = container_of(chan, struct my_dma_chan, chan);
    
    // Stop DMA channel
    // Clear pending transfers
    
    return 0;
}

static enum dma_status my_tx_status(struct dma_chan *chan,
                                     dma_cookie_t cookie,
                                     struct dma_tx_state *txstate)
{
    // Return transfer status
    return DMA_COMPLETE;
}

static void my_issue_pending(struct dma_chan *chan)
{
    struct my_dma_chan *mchan = container_of(chan, struct my_dma_chan, chan);
    
    // Start pending DMA transfers
}
```

## Debugging

```bash
# View DMA channels
ls /sys/class/dma/

# View DMA statistics
cat /sys/kernel/debug/dmaengine/summary

# Kernel messages
dmesg | grep -i dma
```

## Best Practices

✅ **DO:**
- Use coherent memory for DMA buffers
- Configure DMA before starting transfers
- Always check return values
- Handle DMA errors properly
- Use appropriate transfer direction

❌ **DON'T:**
- Use stack buffers for DMA
- Forget to synchronize cache
- Mix coherent and streaming DMA
- Forget to terminate DMA on error
- Access buffer during transfer

## Common Patterns

### DMA with Timeout

```c
int transfer_with_timeout(struct mydev_data *data)
{
    unsigned long timeout = msecs_to_jiffies(5000);
    
    // Start transfer
    dma_async_issue_pending(data->dma_tx);
    
    // Wait with timeout
    if (!wait_for_completion_timeout(&data->done, timeout)) {
        dmaengine_terminate_all(data->dma_tx);
        return -ETIMEDOUT;
    }
    
    return 0;
}
```
