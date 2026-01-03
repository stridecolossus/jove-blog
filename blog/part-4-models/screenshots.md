---
title: Screenshots
---

---

# Contents

- [Introduction](#introduction)
- [Capture Task](#capture-task)
- [Image Writer](#image-writer)
- [Integration](#integration)
- [Summary](#summary)

---

# Introduction

Capturing a screenshot is a surprisingly complex activity requiring the following:

* A new task that manages the process of capturing a screenshot.

* An event handler that 'interrupts' the render loop and invokes the capture task.

* Additional supporting code to write the resultant image to disk.

---

# Capture Task

## Overview

Generating a screenshot image requires the following steps:

1. Retrieve the latest colour attachment image from the swapchain.

2. Create a screenshot image with the same dimensions, format, etc.

3. Execute a command sequence to copy the colour attachment to the screenshot image.

To retrieve the latest swapchain image the existing framebuffer `index` is promoted to a class member and reset before each frame is acquired:

```java
public class Swapchain extends VulkanObject {
    ...
    private final IntegerReference index = new IntegerReference();

    public int acquire(VulkanSemaphore semaphore, Fence fence) throws Invalidated {
        index.set(null);
        int code = library.vkAcquireNextImageKHR(..., index);
        ...
    }
}
```

## Screenshot Image

A new component is introduced that encapsulates the screenshot process:

```java
public class CaptureTask {
    private final Allocator allocator;
    private final Command.Pool pool;
    private final LogicalDevice device;
}
```

First the latest colour attachment image is retrieved from the swapchain:

```java
public DefaultImage capture(Swapchain swapchain) {
    int index = swapchain.index();
    Image image = swapchain.attachments().get(index);
    Instance instance = new Instance(image);
    
    ...
    
    return instance.screenshot;        
}
```

The `Instance` is a local helper that first allocates the screenshot image:

```java
private class Instance {
    private final Image attachment;
    private final DefaultImage screenshot;

    private Instance(Image attachment) {
        this.attachment = attachment;
        this.screenshot = create(attachment.descriptor());
    }
}
```

The `create` method initialises a screenshot descriptor to match the swapchain attachment:

```java
var descriptor = new Image.Descriptor.Builder()
    .type(VkImageType.TYPE_2D)
    .aspect(VkImageAspectFlags.COLOR)
    .extents(target.extents().size())
    .format(target.format())
    .build();
```

The screenshot resides in host-visible memory:

```java
var properties = new MemoryProperties.Builder<VkImageUsageFlags>()
    .usage(TRANSFER_DST)
    .required(HOST_VISIBLE)
    .required(HOST_COHERENT)
    .build();
```

And finally the image is created:

```java
return new DefaultImage.Builder()
    .descriptor(descriptor)
    .properties(properties)
    .tiling(VkImageTiling.LINEAR)
    .build(allocator);
```

## Copy Command

The process of capturing the screenshot image is comprised of a series of commands:

1. Transition the screenshot to a destination image.

2. Transition the colour attachment to a source.

3. Copy the attachment to the screenshot.

4. Restore the colour attachment to its previous layout.

5. Transition the resultant screenshot to a general image that can be written to disk.

These steps are implemented as follows:

```java
var buffer = pool
    .allocate()
    .begin(VkCommandBufferUsageFlags.ONE_TIME_SUBMIT)
        .add(instance.destination())
        .add(instance.source())
        .add(instance.copy())
        .add(instance.restore())
        .add(instance.prepare())
    .end();
```

Transitioning the screenshot to a destination is performed by an image pipeline barrier (in a similar fashion to creating a texture):

```java
public Barrier destination() {
    return new Barrier.Builder()
        .source(TRANSFER)
        .destination(TRANSFER)
        .add(Set.of(), Set.of(TRANSFER_WRITE), new ImageBarrier(screenshot, UNDEFINED, TRANSFER_DST_OPTIMAL))
        .build(device);
}
```

And similarly for transitioning the colour attachment to a source image:

```java
public Barrier source() {
    return new Barrier.Builder()
        .source(TRANSFER)
        .destination(TRANSFER)
        .add(Set.of(MEMORY_READ), Set.of(TRANSFER_READ), new ImageBarrier(attachment, PRESENT_SRC_KHR, TRANSFER_SRC_OPTIMAL))
        .build(device);
}
```

The `ImageCopyCommand` is a new command that copies _between_ images:

```java
public Command copy() {
    return ImageCopyCommand.of(attachment, screenshot, device.library());
}
```

This new component is virtually identical to the `ImageTransferCommand` used previously to copy from a buffer to an image and is therefore not covered in detail here.

Restoring the colour attachment to its previous layout is essentially the inverse of the above transition:

```java
public Barrier restore() {
    return new Barrier.Builder()
        .source(TRANSFER)
        .destination(TRANSFER)
        .add(Set.of(TRANSFER_READ), Set.of(MEMORY_READ), new ImageBarrier(attachment, TRANSFER_SRC_OPTIMAL, PRESENT_SRC_KHR))
        .build(device);
}
```

And similarly to prepare the resultant screenshot:

```java
public Barrier prepare() {
    return new Barrier.Builder()
        .source(TRANSFER)
        .destination(TRANSFER)
        .add(Set.of(TRANSFER_WRITE), Set.of(MEMORY_READ), new ImageBarrier(screenshot, TRANSFER_DST_OPTIMAL, GENERAL))
        .build(device);
}
```

Finally the command sequence is submitted to the hardware to perform the capture:

```java
Work.submit(buffer);
```

---

# Image Writer

For the model demo the screenshot will be written to a fixed file location by a new utility class:

```java
public class NativeImageWriter {
    /**
     * Creates a Java image from the given image data.
     * @param bytes     Image data
     * @param size      Dimensions
     * @return Image
     */
    public static BufferedImage image(byte[] bytes, Dimensions size) {
    }
}
```

For the sake of simplicity the writer will be a quick-and-dirty implementation using AWT and the ImageIO helper, we are likely to want to replace this a more general solution in the future.

First the raw image bytes are wrapped as a data buffer:

```java
var data = new DataBufferByte(bytes, bytes.length);
```

From which an image raster is created:

```java
var raster = Raster.createInterleavedRaster(
    data,
    size.width(), size.height(),
    size.width() * 4,                   // Line stride
    4,                                  // Pixel stride
    new int[]{2, 1, 0},                 // Colour band offsets
    null                                // Top-left location
);
```

Note the offsets array which swizzles the BGR format of the Vulkan image to RGB.

The colour model and channel layout are as follows:

```java
var model = new ComponentColorModel(
    ColorSpace.getInstance(ColorSpace.CS_sRGB),
    new int[]{8, 8, 8},         // Bits per colour channel
    false, false,               // No alpha channel
    Transparency.OPAQUE,
    DataBuffer.TYPE_BYTE
);
```

And finally all these elements are composed into a Java image:

```java
return new BufferedImage(model, raster, false, null);
```

A second utility method writes the resultant image to disk:

```java
public static void write(BufferedImage image, String format, File file) {
    try {
        ImageIO.write(image, format, file);
    }
    catch(IOException e) {
        throw new RuntimeException(...);
    }
}
```

---

# Integration

First, the swapchain attachments must be configured as source images:

```java
@Configuration
class Presentation {
    @Bean
    static SwapchainManager swapchain(...) {
        ...
        var builder = new Swapchain.Builder();
        builder.usage(VkImageUsageFlags.TRANSFER_SRC);
        ...
    }
}
```

In the demo application, the screenshot capture process is wrapped up as a simple runnable:

```java
@Bean
static Runnable screenshot(SwapchainManager manager, Allocator allocator, Command.Pool graphics) {
    return () -> {
        ...
    }
}
```

Which first captures the screenshot:

```java
var task = new CaptureTask(allocator, graphics, graphics.device());
DefaultImage screenshot = task.capture(manager.swapchain());
```

Converts it to a Java image:

```java
MemorySegment memory = screenshot.memory().map();
byte[] bytes = memory.toArray(ValueLayout.JAVA_BYTE);
var size = screenshot.descriptor().extents().size();
var image = NativeImageWriter.image(bytes, size);
```

And writes the image to disk:

```java
NativeImageWriter.write(image, "png", new File("screenshot.png"));
```

Finally, the screenshot task is bound as an event handler:

TODO...

```java
    @Autowired
    void screenshot(Window window, RenderLoop loop, LogicalDevice device, Runnable screenshot) {
        final Consumer<ButtonEvent> handler = button -> {
            // TODO
            if(button.action() != ButtonAction.PRESS) {
                return;
            }

            // Pause render loop
            loop.stop();
            device.waitIdle();

            // Capture screenshot and resume rendering
            try {
                screenshot.run();
            }
            finally {
                loop.start();
            }
        };

        window.keyboard().bind(handler);
    }
```

Note that the action pauses the render loop and waits for the device to become idle before invoking the capture process.

This new functionality was used in the demo to capture the images seen in the previous chapters.

---

# Summary

This chapter added functionality to support capturing screenshots from a given swapchain.
