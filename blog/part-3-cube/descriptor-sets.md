---
title: Descriptor Sets
---

---

## Contents

- [Overview](#overview)
- [Descriptor Sets](#descriptor-sets)
- [Integration](#integration)

---

## Overview

The final piece of functionality needed to support texture sampling is the _descriptor set_.

* A descriptor set is comprised of the _resources_ used by the pipeline during rendering (samplers, uniform buffers, etc).

* Sets are allocated from a _descriptor set pool_ (similar to commands).

* A _descriptor set layout_ specifies the resource bindings for a descriptor set.

The pipeline layout will also be refactored to configure the descriptor sets used by the pipeline.

> Descriptor sets are one of the more complex aspects of a Vulkan application (or at least they proved to be for the author).  We have attempted to come up with a design that reflects the flexibility that the Vulkan designers were clearly aiming for while hopefully providing a relatively developer-friendly API.

---

## Descriptor Sets

### Layout

The first-cut class outline is as follows:

```java
public class DescriptorSet implements NativeObject {
    public static class Layout extends AbstractVulkanObject {
    }

    public static class Pool extends AbstractVulkanObject {
    }
}
```

The layout is tackled first:

```java
public static class Layout extends AbstractVulkanObject {
    private final Collection<Binding> bindings;

    @Override
    protected Destructor<Layout> destructor(VulkanLibrary lib) {
        return lib::vkDestroyDescriptorSetLayout;
    }
}
```

A layout is comprised of a number of indexed _resource bindings_ defined as follows:

```java
public record Binding(int binding, VkDescriptorType type, int count, Set<VkShaderStage> stages)
```

Where _count_ is the size of the resource (which can be an array) and _stages_ specifies where the resource is used in the pipeline.

A convenience builder is added to the binding and the following method populates the corresponding Vulkan structure:

```java
private void populate(VkDescriptorSetLayoutBinding info) {
    info.binding = binding;
    info.descriptorType = type;
    info.descriptorCount = count;
    info.stageFlags = new BitMask<>(stages);
}
```

Layouts are created by a factory method:

```java
public static Layout create(LogicalDevice dev, List<Binding> bindings) {
    // Init layout descriptor
    var info = new VkDescriptorSetLayoutCreateInfo();
    info.bindingCount = bindings.size();
    info.pBindings = StructureCollector.pointer(bindings, new VkDescriptorSetLayoutBinding(), Binding::populate);

    // Allocate layout
    VulkanLibrary lib = dev.library();
    PointerByReference handle = dev.factory().pointer();
    check(lib.vkCreateDescriptorSetLayout(dev, info, null, handle));

    // Create layout
    return new Layout(handle.getValue(), dev, bindings);
}
```

### Pool

Next up is the descriptor set pool:

```java
public static class Pool extends AbstractVulkanObject {
    @Override
    protected Destructor<Pool> destructor(VulkanLibrary lib) {
        return lib::vkDestroyDescriptorPool;
    }
}
```

Descriptor sets are requested from the pool as follows:

```java
public Collection<DescriptorSet> allocate(List<Layout> layouts) {
    // Build allocation descriptor
    int count = layouts.size();
    var info = new VkDescriptorSetAllocateInfo();
    info.descriptorPool = this.handle();
    info.descriptorSetCount = count;
    info.pSetLayouts = NativeObject.toArray(layouts);

    // Allocate descriptors sets
    DeviceContext dev = this.device();
    VulkanLibrary lib = dev.library();
    Pointer[] handles = new Pointer[count];
    check(lib.vkAllocateDescriptorSets(dev, info, handles));
    ...
}
```

The returned handles are transformed to the domain object:

```java
List<DescriptorSet> allocated = IntStream
    .range(0, count)
    .mapToObj(n -> new DescriptorSet(handles[n], layouts.get(n)))
    .toList();
```

Descriptor sets can also be programatically released back to the pool:

```java
public void free(Collection<DescriptorSet> sets) {
    DeviceContext dev = this.device();
    check(dev.library().vkFreeDescriptorSets(dev, this, sets.size(), NativeObject.array(sets)));
}
```

Finally the pool can be reset to recycle all descriptor sets back to the pool:

```java
public void reset() {
    final DeviceContext dev = this.device();
    check(dev.library().vkResetDescriptorPool(dev, this, 0));
}
```

Note that allocated descriptor sets are automatically released when the pool is destroyed.

### Builder

A builder is used to construct and configure a descriptor set pool:

```java
public static class Builder {
    private final Map<VkDescriptorType, Integer> pool = new HashMap<>();
    private final Set<VkDescriptorPoolCreateFlag> flags = new HashSet<>();
    private Integer max;
}
```

The _pool_ member is a table specifying the available number of each type of resource:

```java
public Builder add(VkDescriptorType type, int count) {
    pool.put(type, count);
    return this;
}
```

The _max_ member configures the maximum number of descriptors that can be allocated from the pool, allowing the implementation to potentially optimise and pre-allocate the pool.

In the `build` method the maximum possible number of descriptor sets is derived from the table:

```java
int limit = pool
    .values()
    .stream()
    .mapToInt(Integer::intValue)
    .max()
    .orElseThrow(() -> new IllegalArgumentException(...));
```

And the table is logically verified against this limit, or initialised to the table size if unspecified:

```java
if(max == null) {
    max = limit;
}
else
if(limit > max) {
    throw new IllegalArgumentException(...);
}
```

Next the Vulkan descriptor for the pool is configured:

```java
var info = new VkDescriptorPoolCreateInfo();
info.flags = new BitMask<>(flags);
info.poolSizeCount = entries.size();
info.pPoolSizes = StructureCollector.pointer(pool.entrySet(), new VkDescriptorPoolSize(), Builder::populate);
info.maxSets = max;
```

Which uses the following helper to populate each entry in the table:

```java
private static void populate(Entry<VkDescriptorType, Integer> entry, VkDescriptorPoolSize size) {
    size.type = entry.getKey();
    size.descriptorCount = entry.getValue();
}
```

Finally the pool is instantiated:

```java
VulkanLibrary lib = dev.library();
PointerByReference handle = dev.factory().pointer();
check(lib.vkCreateDescriptorPool(dev, info, null, handle));
return new Pool(handle.getValue(), dev, max);
```

### Resources

A descriptor set is essentially a mutable map of _resources_ indexed by the bindings in the layout.

A resource is defined as follows:

```java
public interface DescriptorResource {
    /**
     * @return Descriptor type
     */
    VkDescriptorType type();

    /**
     * Builds the Vulkan descriptor for this resource.
     * @return Vulkan descriptor
     */
    VulkanStructure build();
}
```

A factory can now be implemented to create a resource for a sampler:

```java
public DescriptorResource resource(View texture) {
    return new Resource() {
        @Override
        public VkDescriptorType type() {
            return VkDescriptorType.COMBINED_IMAGE_SAMPLER;
        }

        @Override
        public VkDescriptorImageInfo build() {
            var info = new VkDescriptorImageInfo();
            info.imageLayout = VkImageLayout.SHADER_READ_ONLY_OPTIMAL;
            info.sampler = Sampler.this.handle();
            info.imageView = view.handle();
            return info;
        }
    };
}
```

Finally the descriptor set domain object can be implemented:

```java
public class DescriptorSet implements NativeObject {
    private final Handle handle;
    private final Layout layout;
    private final Map<Binding, Entry> entries = new HashMap<>();
}
```

An `Entry` is a reference to the resource for a given binding:

```java
public class Entry {
    private final Binding binding;
    private DescriptorResource res;
    private boolean dirty = true;
}
```

Which is retrieved from the descriptor set by resource binding:

```java
public Entry entry(Binding binding) {
    Entry entry = entries.get(binding);
    if(entry == null) throw new IllegalArgumentException();
    return entry;
}
```

And populated via the following setter:

```java
public void set(DescriptorResource res) {
    if(binding.type() != res.type()) throw new IllegalArgumentException();
    this.res = notNull(res);
    dirty = true;
}
```

### Updates

Applying the updated resources to the descriptor sets is slightly complicated:

1. Each modified resource requires a separate `VkWriteDescriptorSet` structure.

2. This structure is dependant on the modified entry, the parent descriptor set _and_ the binding.

3. Updates are applied as a batch operation.

First the modified entries are enumerated from a collection of descriptor sets:

```java
public static int update(DeviceContext dev, Collection<DescriptorSet> descriptors) {
    // Enumerate pending updates
    List<Entry> updates = descriptors
        .stream()
        .flatMap(e -> e.entries.values().stream())
        .filter(e -> e.dirty)
        .toList();

    // Ignore if nothing to update
    if(updates.isEmpty()) {
        return 0;
    }
    
    ...

    return writes.length;
}
```

A write descriptor is generated for each modified entry and the API is invoked to update the resource bindings:

```java
VkWriteDescriptorSet[] writes = StructureCollector.array(updates, new VkWriteDescriptorSet(), Entry::populate);
dev.library().vkUpdateDescriptorSets(dev, writes.length, writes, 0, null);
```

The `populate` method of the modified resources first initialises the write descriptor for each entry:

```java
private void populate(VkWriteDescriptorSet write) {
    if(res == null) throw new IllegalStateException();

    write.sType = VkStructureType.WRITE_DESCRIPTOR_SET;
    write.dstBinding = binding.index();
    write.descriptorType = binding.type();
    write.dstSet = DescriptorSet.this.handle();
    write.descriptorCount = 1;
    write.dstArrayElement = 0;
}
```

And then delegates to the resource to populate the relevant field for that type of descriptor:

```java
switch(res.build()) {
    case VkDescriptorImageInfo image -> write.pImageInfo = image;
    case VkDescriptorBufferInfo buffer -> write.pBufferInfo = buffer;
    default -> throw new UnsupportedOperationException();
}
```

Finally the modified entries are marked as updated:

```java
for(Entry e : updates) {
    e.dirty = false;
}
```

A new library is created for the API methods in this chapter:

```java
interface VulkanLibraryDescriptorSet {
    int  vkCreateDescriptorSetLayout(DeviceContext device, VkDescriptorSetLayoutCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference pSetLayout);
    void vkDestroyDescriptorSetLayout(DeviceContext device, Layout descriptorSetLayout, Pointer pAllocator);

    int  vkCreateDescriptorPool(DeviceContext device, VkDescriptorPoolCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference pDescriptorPool);
    void vkDestroyDescriptorPool(DeviceContext device, Pool descriptorPool, Pointer pAllocator);

    int  vkAllocateDescriptorSets(DeviceContext device, VkDescriptorSetAllocateInfo pAllocateInfo, Pointer[] pDescriptorSets);
    int  vkResetDescriptorPool(DeviceContext device, Pool descriptorPool, int flags);
    int  vkFreeDescriptorSets(DeviceContext device, Pool descriptorPool, int descriptorSetCount, Pointer pDescriptorSets);
    void vkUpdateDescriptorSets(DeviceContext device, int descriptorWriteCount, VkWriteDescriptorSet[] pDescriptorWrites, int descriptorCopyCount, VkCopyDescriptorSet[] pDescriptorCopies);

    void vkCmdBindDescriptorSets(Buffer commandBuffer, VkPipelineBindPoint pipelineBindPoint, PipelineLayout layout, int firstSet, int descriptorSetCount, Pointer pDescriptorSets, int dynamicOffsetCount, int[] pDynamicOffsets);
}
```

---

## Integration

### Pipeline Layout

The previous demo implemented a bare-bones pipeline layout which is now fleshed out to support descriptor sets.

A new member is added to the builder for the pipeline layout:

```java
public static class Builder {
    private final List<DescriptorSet.Layout> sets = new ArrayList<>();

    public Builder add(DescriptorSet.Layout layout) {
        sets.add(layout);
        return this;
    }
}
```

Which is populated in the build method:

```java
info.setLayoutCount = sets.size();
info.pSetLayouts = NativeObject.array(sets);
```

Finally a new command factory is implemented to bind a group of descriptor sets to the pipeline:

```java
public static Command bind(Pipeline.Layout layout, Collection<DescriptorSet> sets) {
    return (api, cmd) -> api.vkCmdBindDescriptorSets(
        cmd,
        VkPipelineBindPoint.GRAPHICS,
        layout,
        0,
        sets.size(),
        NativeObject.array(sets),
        0,
        null
    );
}
```

### Descriptor Sets

To apply the texture in the demo we first implement a new configuration class to instantiate a descriptor set layout for the texture sampler:

```java
@Configuration
public class DescriptorConfiguration {
    @Autowired private LogicalDevice dev;

    private final Binding binding = new Binding.Builder()
        .type(VkDescriptorType.COMBINED_IMAGE_SAMPLER)
        .stage(VkShaderStage.FRAGMENT)
        .build();

    @Bean
    public Layout layout() {
        return Layout.create(dev, List.of(binding));
    }
}
```

Next a descriptor set pool is created that can allocate a sampler resource:

```java
@Bean
public Pool pool() {
    return new Pool.Builder()
        .add(VkDescriptorType.COMBINED_IMAGE_SAMPLER, 1)
        .max(1)
        .build(dev);
}
```

Finally the descriptor set is allocated and populated with the sampler resource:

```java
@Bean
public DescriptorSet descriptor(Pool pool, Layout layout, Sampler sampler, View texture) {
    DescriptorSet descriptor = pool.allocate(layout);
    Resource res = sampler.resource(texture);
    descriptor.set(binding, res);
    return descriptor;
}
```

Note that for the moment we are employing a single frame buffer, descriptor set and command buffer, since the demo is only generating a single frame.

The descriptor set layout is registered with the pipeline:

```java
class PipelineConfiguration {
    @Bean
    PipelineLayout pipelineLayout(DescriptorSet.Layout layout) {
        return new PipelineLayout.Builder()
            .add(layout)
            .build(dev);
    }
}
```

And finally the descriptor set is bound to the pipeline in the render sequence:

```java
.begin()
    .add(frame.begin())
        .add(pipeline.bind())
        .add(vbo.bind())
        .add(descriptor.bind(pipeline.layout()))
        .add(draw)
    .add(FrameBuffer.END)
.end();
```

### Conclusion

To use the texture sampler we make the following changes to the fragment shader:

1. Add a layout declaration for a `uniform sampler2D` with the binding index specified in the descriptor set layout.

2. Invoke the built-in `texture` function to sample the texture with the coordinate passed from the vertex shader.

The fragment shader now looks like this:

```glsl
#version 450 core

layout(binding=0) uniform sampler2D texSampler;

layout(location=0) in vec2 texCoord;

layout(location=0) out vec4 outColor;

void main(void) {
    outColor = texture(texSampler, texCoord);
}
```

If all goes well we should finally see the textured quad:

![Textured Quad](textured-quad.png)

We have jumped through a number of hoops in the last couple of chapters and therefore there is plenty to go wrong.  The validation layer will provide excellent diagnostics if the application attempts any invalid operations (and often seems to be able to work around them).  However it is very easy to specify a 'valid' pipeline that results in a black rectangle.

Here are some of the problems encountered by the author:

* The image loader is very crude and somewhat brittle - the Java image _should_ be a `TYPE_4BYTE_BGR` corresponding to the `R8G8B8A8_UNORM` Vulkan format.  It is definitely worth using the debugger to step through image loading and the complex barrier transition logic.

* Ensure the image data is being copied to the vertex buffer and not the other way around (yes we really did this).

* Check that the descriptor set layout is bound to the pipeline and added to the render sequence.

---

## Summary

In this chapter we:

* Implemented descriptor sets to support a texture sampler resource.

* Applied texture sampling to the demo application.

