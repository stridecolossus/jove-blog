---
title: Vertex Buffers
---

---

## Contents

- [Overview](#overview)
- [Vertex Data](#vertex-data)
- [Vertex Configuration](#vertex-configuration)
- [Integration](#integration)

---

## Overview

In this chapter the hard-coded triangle data in the shader is replaced with a _vertex buffer_ (also known as a _vertex buffer object_ or VBO).

The vertex data will then be transferred to a _device local_ buffer (i.e. memory on the GPU) for optimal performance.

This process involves the following steps:

1. Build the vertex data.

2. Convert it to an interleaved NIO buffer.

3. Allocate a _staging_ buffer (visible to the host).

4. Copy the interleaved data to this staging buffer.

5. Allocate a _device local_ buffer (visible only to the hardware).

6. Copy the staged data to this buffer.

7. Release the staging buffer.

The following components will be introduced:

* New domain classes to specify the vertex data.

* The vertex buffer object itself.

* A command to copy between buffers.

* The _vertex input_ pipeline stage to configure the structure of the vertex data in the shader.

---

## Vertex Data

### Vertices

To construct the vertex data programatically several new types are introduced.

The first is a definition for an arbitrary data object that can be written to an NIO buffer:

```java
public interface Bufferable {
    /**
     * Writes this object to the given buffer.
     * @param buffer Buffer
     */
    void buffer(ByteBuffer buffer);
}
```

A _vertex_ is generally comprised of some or all of the following:
- position
- normal
- texture coordinate
- colour

Normals and texture coordinates are not required for the triangle demo so these are glossed over in this chapter.

Positions and vectors are both floating-point tuples with common functionality, here a small hierarchy is introduced with the following base-class:

```java
public class Tuple implements Bufferable {
    public final float x, y, z;

    protected Tuple(float x, float y, float z) {
        this.x = x;
        this.y = y;
        this.z = z;
    }

    @Override
    public void buffer(ByteBuffer buffer) {
        buffer
            .putFloat(x)
            .putFloat(y)
            .putFloat(z);
    }
}
```

A vertex position is a point in 3D space:

```java
public final class Point extends Tuple {
    /**
     * Origin point.
     */
    public static final Point ORIGIN = new Point(0, 0, 0);

    public Point(float x, float y, float z) {
        super(x, y, z);
    }
}
```

And colours are defined as an RGBA record:

```java
public record Colour(float red, float green, float blue, float alpha) implements Bufferable {
    public static final Colour WHITE = new Colour(1, 1, 1, 1);
    public static final Colour BLACK = new Colour(0, 0, 0, 1);

    @Override
    public void buffer(ByteBuffer buffer) {
        buffer
            .putFloat(red)
            .putFloat(green)
            .putFloat(blue)
            .putFloat(alpha);
    }
}
```

The vertex implementation itself is a base-class that is assumed to always contain the vertex position:

```java
public class Vertex implements Bufferable {
    private final Point pos;

    @Override
    public void buffer(ByteBuffer bb) {
        pos.buffer(bb);
    }
}
```

And this is specialised for a vertex that also contains a colour:

```java
public class ColourVertex extends Vertex {
    private final Colour col;

    @Override
    public void buffer(ByteBuffer bb) {
        super.buffer(bb);
        col.buffer(bb);
    }
}
```

Note that these implementations assume the vertex data is _interleaved_ (the most common approach).

### Vertex Buffer

The outline class for the vertex buffer is as follows:

```java
public class VulkanBuffer extends AbstractVulkanObject {
    private final Set<VkBufferUsage> usage;
    private final DeviceMemory mem;
    private final long len;

    @Override
    protected Destructor<VulkanBuffer> destructor(VulkanLibrary lib) {
        return lib::vkDestroyBuffer;
    }
}
```

The buffer is bound to a pipeline using the following command factory:

```java
public Command bind(int binding) {
    require(VkBufferUsage.VERTEX_BUFFER);
    VulkanBuffer[] array = new VulkanBuffer[]{this};
    return (lib, buffer) -> lib.vkCmdBindVertexBuffers(buffer, binding, 1, array, new long[]{0});
}
```

The local `require` helper checks that the buffer supports a given operation, in this case that it is being used as a vertex buffer:

```java
public void require(VkBufferUsage... flags) {
    Collection<VkBufferUsage> required = Arrays.asList(flags);
    if(!usage.containsAll(required)) throw new IllegalStateException();
}
```

Copying the vertex data from staging to a device-local buffer is also a command:

```java
public Command copy(VulkanBuffer dest) {
    // Validate
    if(len > dest.len) throw new IllegalStateException();
    require(VkBufferUsage.TRANSFER_SRC);
    dest.require(VkBufferUsage.TRANSFER_DST);

    // Build copy descriptor
    var region = new VkBufferCopy();
    region.size = len;

    // Create copy command
    return (lib, buffer) -> lib.vkCmdCopyBuffer(buffer, this, dest, 1, new VkBufferCopy[]{region});
}
```

Finally the memory is released when the buffer is destroyed:

```java
@Override
protected void release() {
    if(!mem.isDestroyed()) {
        mem.destroy();
    }
}
```

### Buffer Creation

Creating a Vulkan buffer is comprised of the following steps:

1. Instantiate the buffer.

2. Retrieve the _memory requirements_ for the new buffer.

3. Allocate device memory given these requirements.

4. Bind the allocated memory to the buffer.

Instantiating the buffer follows the usual pattern of populating a descriptor and invoking the API:

```java
public static VulkanBuffer create(LogicalDevice dev, Allocator allocator, long len, MemoryProperties<VkBufferUsage> props) {
    // Build buffer descriptor
    var info = new VkBufferCreateInfo();
    info.usage = new BitMask<>(props.usage());
    info.sharingMode = props.mode();
    info.size = oneOrMore(len);

    // Allocate buffer
    VulkanLibrary lib = dev.library();
    PointerByReference handle = dev.factory().pointer();
    check(lib.vkCreateBuffer(dev, info, null, handle));
    
    ...
}
```

Next the memory requirements are retrieved for the new buffer:

```java
var reqs = new VkMemoryRequirements();
lib.vkGetBufferMemoryRequirements(dev, handle.getValue(), reqs);
```

Which are passed to the allocation service along with the configured memory properties:

```java
DeviceMemory mem = allocator.allocate(reqs, props);
```

The allocated memory is then bound to the buffer:

```java
check(lib.vkBindBufferMemory(dev, handle.getValue(), mem, 0L));
```

And finally the domain object is created:

```java
return new VulkanBuffer(handle.getValue(), dev, props.usage(), mem, len);
```

The API for vertex buffers consists of the following methods:

```java
interface Library {
    int  vkCreateBuffer(LogicalDevice device, VkBufferCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference pBuffer);
    void vkDestroyBuffer(DeviceContext device, VulkanBuffer buffer, Pointer pAllocator);

    void vkGetBufferMemoryRequirements(LogicalDevice device, Pointer buffer, VkMemoryRequirements pMemoryRequirements);
    int  vkBindBufferMemory(LogicalDevice device, Pointer buffer, Handle memory, long memoryOffset);

    void vkCmdBindVertexBuffers(Command.Buffer commandBuffer, int firstBinding, int bindingCount, VulkanBuffer[] pBuffers, long[] pOffsets);
    void vkCmdBindIndexBuffer(Command.Buffer commandBuffer, VulkanBuffer buffer, long offset, VkIndexType indexType);
    void vkCmdCopyBuffer(Command.Buffer commandBuffer, VulkanBuffer srcBuffer, VulkanBuffer dstBuffer, int regionCount, VkBufferCopy[] pRegions);
}
```

---

## Vertex Configuration

### Overview

To use the vertex buffer in the shader the structure of the data must be configured in the pipeline, which consists of two pieces of information:

1. A _binding_ that specifies the vertex data to be passed to the shader.

2. A number of _vertex attributes_ that define the structure of that data, i.e. the layout of each vertex component.

The new pipeline stage is implemented with nested builders for the bindings and attributes:

```java
public class VertexInputStageBuilder extends AbstractPipelineBuilder<VkPipelineVertexInputStateCreateInfo> {
    private final Map<Integer, BindingBuilder> bindings = new HashMap<>();
    private final List<AttributeBuilder> attributes = new ArrayList<>();

    public BindingBuilder binding() {
        return new BindingBuilder();
    }
}
```

The descriptor for the vertex configuration is generated as follows:

```java
VkPipelineVertexInputStateCreateInfo get() {
    // Create descriptor
    var info = new VkPipelineVertexInputStateCreateInfo();

    // Add binding descriptions
    info.vertexBindingDescriptionCount = bindings.size();
    info.pVertexBindingDescriptions = StructureCollector.pointer(bindings.values(), new VkVertexInputBindingDescription(), BindingBuilder::populate);

    // Add attributes
    info.vertexAttributeDescriptionCount = attributes.size();
    info.pVertexAttributeDescriptions = StructureCollector.pointer(attributes, new VkVertexInputAttributeDescription(), AttributeBuilder::populate);

    return info;
}
```

Note that the nested builders are added to the parent rather than using any transient or intermediate data objects.

### Bindings

The nested builder for the bindings is relatively simple:

```java
public class BindingBuilder {
    private int index = bindings.size();
    private int stride;
    private VkVertexInputRate rate = VkVertexInputRate.VERTEX;
}
```

Note that the binding _index_ is initialised to the next available slot but can also be explicitly overridden.

The _stride_ is the number of bytes per vertex.

The `populate` method fills an instance of the corresponding Vulkan descriptor for the binding:

```java
void populate(VkVertexInputBindingDescription desc) {
    desc.binding = binding;
    desc.stride = stride;
    desc.inputRate = rate;
}
```

The `build` method validates the data and returns to the parent builder:

```java
public VertexInputStageBuilder build() {
    // Validate binding description
    if(bindings.containsKey(index)) throw new IllegalArgumentException();
    if(locations.isEmpty()) throw new IllegalArgumentException();

    // Add binding
    bindings.put(index, this);

    return VertexInputStageBuilder.this;
}
```

### Attributes

Vertex attributes are configured via the following factory method:

```java
public class BindingBuilder {
    public AttributeBuilder attribute() {
        return new AttributeBuilder(this);
    }
}
```

The nested builder for the vertex attributes follows a similar pattern to the bindings:

```java
public class AttributeBuilder {
    private final BindingBuilder binding;
    private int loc;
    private VkFormat format;
    private int offset;

    private AttributeBuilder(BindingBuilder binding) {
        this.binding = binding;
        this.loc = binding.locations.size();
    }

    private void populate(VkVertexInputAttributeDescription attr) {
        attr.binding = binding.index;
        attr.location = loc;
        attr.format = format;
        attr.offset = offset;
    }
}
```

Where:

* _loc_ corresponds to the _layout_ directives in the shader (see below).

* _offset_ specifies the starting byte of each attribute.

Again the attribute location is initialised to the next available slot which is tracked in the parent builder:

```java
public class BindingBuilder {
    private final Set<Integer> locations = new HashSet<>();
}
```

The build method validates the attribute and returns to the parent builder:

```java
public BindingBuilder build() {
    // Validate attribute
    if(offset >= binding.stride) throw new IllegalArgumentException();
    if(format == null) throw new IllegalArgumentException();

    // Check location
    if(binding.locations.contains(loc)) throw new IllegalArgumentException();
    binding.locations.add(loc);

    // Add attribute
    attributes.add(this);

    return binding;
}
```

---

## Integration 

### Vertex Data

All this new functionality is brought together to programatically create the triangle vertices and copy the vertex data to the hardware.

A new configuration class is added for the vertex buffer and triangle data:

```java
@Configuration
public class VertexBufferConfiguration {
    private static final Vertex[] vertices = ...

    @Bean
    public static VulkanBuffer vbo(LogicalDevice dev, Allocator allocator, Pool pool) {
        ...
    }
}
```

The triangle vertices are specified as a simple array (copied from the original shader):

```java
Vertex[] vertices = {
    new ColourVertex(new Point(0, -0.5f, 0), new Colour(1, 0, 0, 1)),
    new ColourVertex(new Point(0.5f, 0.5f, 0), new Colour(0, 0, 1, 1)),
    new ColourVertex(new Point(-0.5f, 0.5f, 0), new Colour(0, 1, 0, 1)),
};
```

For the moment the vertices are simply wrapped into a compound bufferable object:

```java
var triangle = new ByteSizedBufferable() {
    public int length() {
        return vertices.length * (3 + 4) * Float.BYTES;
    };
    
    public void buffer(ByteBuffer bb) {
        for(Vertex v : vertices) {
            v.buffer(bb);
        }
    }
};
```

Which uses a new abstraction for a bufferable object with a byte length:

```java
public interface ByteSizedBufferable extends Bufferable {
    /**
     * @return Length of this bufferable (bytes)
     */
    int length();
}
```

This will be replaced with a more specialised _mesh_ implementation in a future chapter.

Next a helper is added to create a staging buffer:

```java
public static VulkanBuffer staging(LogicalDevice dev, Allocator allocator, ByteSizedBufferable data) {
    var props = new MemoryProperties.Builder<VkBufferUsageFlag>()
        .usage(VkBufferUsageFlag.TRANSFER_SRC)
        .required(VkMemoryProperty.HOST_VISIBLE)
        .build();

    VulkanBuffer buffer = create(dev, allocator, data.length(), props);

    ...
}
```

The memory properties define a buffer that is used as the source of a copy operation and is visible to the host (i.e. the application).

The `data` is then written to the staging buffer:

```java
ByteBuffer bb = buffer.memory().map().buffer();
data.buffer(bb);
```

In the `vbo` bean method the new helper is used to create the staging buffer and copy the interleaved triangle data:

```java
VulkanBuffer staging = VulkanBuffer.staging(dev, allocator, triangle);
```

Next the destination buffer on the GPU is instantiated:

```java
var props = new MemoryProperties.Builder<VkBufferUsageFlag>()
    .usage(VkBufferUsageFlag.TRANSFER_DST)
    .usage(VkBufferUsageFlag.VERTEX_BUFFER)
    .required(VkMemoryProperty.DEVICE_LOCAL)
    .build();

VulkanBuffer vbo = VulkanBuffer.create(dev, allocator, staging.length(), props);
```

The triangle data can now be copied from the staging buffer:

```java
staging.copy(vbo).submit(pool);
```

The `submit` method delegates to the following helper in the `Work` class:

```java
public static void submit(Command cmd, Pool pool) {
    // Record command to a one-time buffer
    Buffer buffer = pool
        .primary()
        .begin(VkCommandBufferUsage.ONE_TIME_SUBMIT)
        .add(cmd)
        .end();

    try {
        // Submit command
        Work work = Work.of(buffer);
        work.submit();

        // Wait for completion
        // TODO
        pool.waitIdle();
    }
    finally {
        buffer.free();
    }
}
```

Note that this approach currently blocks the work queue, this will be replaced later with proper synchronisation.

Finally the staging buffer can be released:

```java
staging.destroy();
```

### Configuration

In the demo application the VBO is bound to the render sequence:

```java
public Buffer sequence(...) {
    Command draw = ...
    return graphics
        .allocate()
        .begin()
            .add(frame.begin())
                .add(pipeline.bind())
                .add(vbo.bind(0))
                .add(draw)
            .add(FrameBuffer.END)
        .end();
}
```

And the structure of the vertex data is configured using the new pipeline stage builder:

```java
public Pipeline pipeline(...) {
    return new Pipeline.Builder()
        ...
        .input()
            .binding()
                .index(0)
                .stride((3 + 4) * Float.BYTES)
                .build()
            .attribute()            
                // Position            
                .binding(0)
                .location(0)
                .format(VkFormat.R32G32B32_SFLOAT)
                .offset(0)
                .build()
            .attribute()
                // Colour
                .binding(0)
                .location(1)
                .format(VkFormat.R32G32B32A32_SFLOAT)
                .offset(3 * Float.BYTES)
                .build()
            .build()
```

There is a lot of mucking about and hard-coded data here that will be addressed in the next chapter.

The final change to the demo is to remove the hard coded vertex data in the shader and configure the incoming VBO which involves:

- The addition of two `layout` directives to specify the incoming data from the vertex buffer, corresponding to the locations specified in the vertex attributes.

- Output the position of each vertex.

- Pass through the colour of each vertex onto the next stage.

The resultant vertex shader is:

```glsl
#version 450

layout(location = 0) in vec3 inPosition;
layout(location = 1) in vec4 inColour;

layout(location = 0) out vec4 fragColour;

void main() {
    gl_Position = vec4(inPosition, 1.0);
    fragColor = inColour;
}
```

Remember that the modified vertex shader will need to be recompiled (the fragment shader is unchanged).

If all goes well we should still see the same triangle.

A good test is to change the vertex positions and/or colours to make sure the new functionality and shader are actually being used.

---

## Summary

In this chapter we implemented:

- New domain objects and supporting framework code to define vertex data.

- The vertex buffer class to transfer the data from the application to the hardware.

- The vertex input pipeline stage builder.

