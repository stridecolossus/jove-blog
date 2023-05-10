
## Screenshot Capture

### Capture Task

Capturing a screenshot is comprised of the following steps:

1. Retrieve the latest rendered swapchain image.

2. Create a target screenshot image.

3. Copy the swapchain image to the screenshot.

4. Application specific code to output the resultant image.

This process is implemented in the following new component:

```java
public class CaptureTask {
    private final Allocator allocator;
    private final Pool pool;

    public Image capture(Swapchain swapchain) {
        ...
        return screenshot;
    }
}
```

Where `capture` creates the screenshot image and executes the capture task on the transfer queue:

```java
// Retrieve latest rendered swapchain image
Image image = swapchain.latest().image();

// Create destination screenshot image
DeviceContext dev = swapchain.device();
DefaultImage screenshot = screenshot(dev, image.descriptor());

// Init copy command
Command copy = ImageCopyCommand.of(image, screenshot);

// Submit screenshot task
pool
    .allocate()
    .begin(VkCommandBufferUsage.ONE_TIME_SUBMIT)
        ...
    .end()
    .submit();
```

The target screenshot image is created with the same dimensions as the swapchain:

```java
private DefaultImage screenshot(DeviceContext dev, Descriptor target) {
    // Create descriptor
    Descriptor descriptor = new Descriptor.Builder()
        .type(VkImageType.TWO_D)
        .aspect(VkImageAspect.COLOR)
        .extents(target.extents().size())
        .format(VkFormat.R8G8B8A8_UNORM)
        .build();
    
    // Init image memory properties
    var props = new MemoryProperties.Builder<VkImageUsageFlag>()
        .usage(VkImageUsageFlag.TRANSFER_DST)
        .required(VkMemoryProperty.HOST_VISIBLE)
        .required(VkMemoryProperty.HOST_COHERENT)
        .build();
    
    // Create screenshot image
    return new DefaultImage.Builder()
        .descriptor(descriptor)
        .properties(props)
        .tiling(VkImageTiling.LINEAR)
        .build(dev, allocator);
}
```

Notes:

* The screenshot image has `LINEAR` tiling.

* The `latest` rendered image is updated by the swapchain in the `acquire` method.

* This approach assumes that the swapchain images are configured as `VkImageUsageFlag.TRANSFER_SRC` on creation.

### Image Copying

Copying from the swapchain to the screenshot image is a new command with a companion builder:

```java
public class ImageCopyCommand implements Command {
    private final Image src, dest;
    private final VkImageLayout srcLayout, destLayout;
    private final VkImageCopy[] regions;

    @Override
    public void execute(VulkanLibrary lib, Buffer buffer) {
        lib.vkCmdCopyImage(buffer, src, srcLayout, dest, destLayout, regions.length, regions);
    }
}
```

Where the copy regions are configured by the following transient type:

```java
public record CopyRegion(SubResource src, Extents srcOffset, SubResource dest, Extents destOffset, Extents extents)
```

A convenience factory method is provided for the general case of copying the whole image:

```java
public static ImageCopyCommand of(Image src, Image dest) {
    Descriptor descriptor = src.descriptor();
    CopyRegion region = new CopyRegion(descriptor, Extents.ZERO, descriptor, Extents.ZERO, descriptor.extents());
    return new ImageCopyCommand.Builder(src, dest).region(region).build();
}
```

> The copy step could have been implemented as a blit operation which would automatically convert the image without requiring the application to perform channel swizzles.

### Barrier Transitions

The screenshot task wraps the copy operations with barriers to transitions the images during the process:

```java
.add(destination(screenshot))
.add(source(image))
.add(copy)
.add(prepare(screenshot))
.add(restore(image))
```

First, the screenshot image is transitioned to a destination for the copy operation:

```java
private static Barrier destination(Image screenshot) {
    return new Barrier.Builder()
        .source(VkPipelineStage.TRANSFER)
        .destination(VkPipelineStage.TRANSFER)
        .image(screenshot)
            .newLayout(VkImageLayout.TRANSFER_DST_OPTIMAL)
            .destination(VkAccess.TRANSFER_WRITE)
            .build()
        .build();
}
```

Next the swapchain image is transitioned to a copy source:

```java
private static Barrier source(Image image) {
    return new Barrier.Builder()
        .source(VkPipelineStage.TRANSFER)
        .destination(VkPipelineStage.TRANSFER)
        .image(image)
            .oldLayout(VkImageLayout.PRESENT_SRC_KHR)
            .newLayout(VkImageLayout.TRANSFER_SRC_OPTIMAL)
            .source(VkAccess.MEMORY_READ)
            .destination(VkAccess.TRANSFER_READ)
            .build()
        .build();
}
```

After the copy step the screenshot is transitioned to the `GENERAL` layout allowing the image memory to be accessed:

```java
private static Barrier prepare(Image screenshot) {
    return new Barrier.Builder()
        .source(VkPipelineStage.TRANSFER)
        .destination(VkPipelineStage.TRANSFER)
        .image(screenshot)
            .oldLayout(VkImageLayout.TRANSFER_DST_OPTIMAL)
            .newLayout(VkImageLayout.GENERAL)
            .source(VkAccess.TRANSFER_WRITE)
            .destination(VkAccess.MEMORY_READ)
            .build()
        .build();
}
```

And finally the swapchain image is restored to its initial layout:

```java
private static Barrier restore(Image image) {
    return new Barrier.Builder()
        .source(VkPipelineStage.TRANSFER)
        .destination(VkPipelineStage.TRANSFER)
        .image(image)
            .oldLayout(VkImageLayout.TRANSFER_SRC_OPTIMAL)
            .newLayout(VkImageLayout.PRESENT_SRC_KHR)
            .source(VkAccess.TRANSFER_READ)
            .destination(VkAccess.MEMORY_READ)
            .build()
        .build();
}
```

### Integration

TODO

```java
            final ByteBuffer bb = screenshot.memory().map().buffer();
            final byte[] bytes = BufferHelper.array(bb);

            final Dimensions size = screenshot.descriptor().extents().size();
            final DataBufferByte data = new DataBufferByte(bytes, bytes.length);
            final WritableRaster raster = Raster.createInterleavedRaster(data, size.width(), size.height(), size.width() * 4, 4, new int[]{2, 1, 0}, null);

            final ColorModel cm = new ComponentColorModel(ColorSpace.getInstance(ColorSpace.CS_sRGB), new int[]{8, 8, 8}, false, false, Transparency.OPAQUE, DataBuffer.TYPE_BYTE);
            final var img = new BufferedImage(cm, raster, false, null);

            System.out.println(img);

            final boolean done = ImageIO.write(img, "jpg", new File("output.jpg"));
            System.out.println("done="+done);
```
