---
title: Memory Allocation
---

---

## Contents

- [Overview](#overview)
- [Memory Allocation](#memory-allocation)
- [Memory Pool](#memory-pool)

---

## Overview

The following chapters will introduce vertex buffers and textures, both of which are dependant on _device memory_ allocated by Vulkan.  Device memory resides on the hardware and can optionally be visible to the host (i.e. the application) depending on the usage requirements.

A Vulkan implementation specifies a set of _memory types_ which can be selected by the application depending on the scenario.  The documentation suggests implementing a _fallback strategy_ when choosing a memory type: the application specifies _optimal_ and _minimal_ properties for the requested memory, with the algorithm falling back to the minimal properties if the optimal memory type is not available.

Additionally the maximum number of memory allocations supported by the hardware is limited.  A real-world application would allocate a memory _pool_ and then serve allocation requests as offsets into that pool, growing the available memory as required.  Initially we will simply allocate new memory for each request and then extend the implementation to use a memory pool.

References:
- [VkPhysicalDeviceMemoryProperties](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPhysicalDeviceMemoryProperties.html)
- [Custom Allocators (Blog)](http://kylehalladay.com/blog/tutorial/2017/12/13/Custom-Allocators-Vulkan.html)

---

## Memory Allocation

### Device Memory

We start with the following abstraction for device memory:

```java
public interface DeviceMemory extends NativeObject, TransientObject {
    /**
     * @return Size of this memory (bytes)
     */
    long size();

    /**
     * @return Mapped memory region
     */
    Optional<Region> region();

    /**
     * Maps a region of this device memory.
     * @param offset        Offset into this memory
     * @param size          Size of the region to map
     * @return Mapped memory region
     * @throws IllegalArgumentException if the {@code offset} and {@code size} exceeds the size of this memory
     * @throws IllegalStateException if a mapping already exists or this memory has been destroyed
     */
    Region map(long offset, long size);
}
```

A _region_ is a _mapped_ segment of this memory that can be accessed for read or write operations:

```java
interface Region {
    /**
     * @return Size of this region (bytes)
     */
    long size();

    /**
     * Provides a byte-buffer to access a sub-section of this memory region.
     * @param offset        Offset
     * @param size          Region size (bytes)
     * @return Byte-buffer
     * @throws IllegalArgumentException if the {@code offset} and {@code size} exceeds the size of this region
     */
    ByteBuffer buffer(long offset, long size);

    /**
     * Un-maps this mapped region.
     * @throws IllegalStateException if the mapping has also been released or the memory has been destroyed
     */
    void unmap();
}
```

Next a default implementation is created:

```java
public class DefaultDeviceMemory extends AbstractVulkanObject implements DeviceMemory {
    private final long size;

    @Override
    protected Destructor<DefaultDeviceMemory> destructor(VulkanLibrary lib) {
        return lib::vkFreeMemory;
    }
}
```

With an inner class for the mapped region of the memory:

```java
public class DefaultDeviceMemory ... {
    private volatile DefaultRegion region;

    private class DefaultRegion implements Region {
        private final Pointer ptr;
        private final long size;
        private final long offset;
    }

    @Override
    public Optional<Region> region() {
        return Optional.ofNullable(region);
    }

    @Override
    protected void release() {
        region = null;
    }
}
```

The `map` method creates a mapped region for the memory:

```java
public Region map(long offset, long size) {
    // Map memory
    DeviceContext dev = this.device();
    VulkanLibrary lib = dev.library();
    PointerByReference ref = dev.factory().pointer();
    check(lib.vkMapMemory(dev, this, offset, size, 0, ref));

    // Create mapped region
    region = new DefaultRegion(ref.getValue(), offset, size);

    return region;
}
```

Note that only one mapped region is permitted for a given memory instance.

An NIO buffer can then be retrieved from the mapped region to read or write the memory:

```java
public ByteBuffer buffer(long offset, long size) {
    return ptr.getByteBuffer(offset, size);
}
```

Finally a mapped region can be released:

```java
public void unmap() {
    DeviceContext dev = device();
    VulkanLibrary lib = dev.library();
    lib.vkUnmapMemory(dev, DefaultDeviceMemory.this);
    region = null;
}
```

### Memory Types

The memory types supported by the hardware are specified by the `VkPhysicalDeviceMemoryProperties` structure.

Each memory type is represented by a new domain class:

```java
public record MemoryType(int index, Heap heap, Set<VkMemoryProperty> properties) {
    public record Heap(long size, Set<VkMemoryHeapFlag> flags) {
    }
}
```

The memory types and heaps are enumerated from the structure via the following factory method:

```java
public static MemoryType[] enumerate(VkPhysicalDeviceMemoryProperties descriptor) {
    class Helper {
        ...
    }
    
    Helper helper = new Helper();
    return helper.types();
}
```

The local helper class wraps up the extraction process by first retrieving the array of memory heaps:

```java
class Helper {
    private final ReverseMapping<VkMemoryHeapFlag> mapper = IntEnum.reverse(VkMemoryHeapFlag.class);
    private final List<Heap> heaps;

    Helper() {
        heaps = Arrays
                .stream(descriptor.memoryHeaps)
                .map(this::heap)
                .toList();
    }

    private Heap heap(VkMemoryHeap heap) {
        Set<VkMemoryHeapFlag> flags = heap.flags.enumerate(mapper);
        return new Heap(heap.size, flags);
    }
}
```

Followed by the memory types:

```java
private final ReverseMapping<VkMemoryProperty> properties = IntEnum.reverse(VkMemoryProperty.class);

MemoryType[] types() {
    return IntStream
            .range(0, descriptor.memoryTypeCount)
            .mapToObj(this::type)
            .toArray(MemoryType[]::new);
}

private MemoryType type(int index) {
    VkMemoryType type = descriptor.memoryTypes[index];
    Heap heap = heaps.get(type.heapIndex);
    Set<VkMemoryProperty> props = type.propertyFlags.enumerate(properties);
    return new MemoryType(index, heap, props);
}
```

Finally the following new type specifies the requirements of a memory request:

```java
public record MemoryProperties<T>(
    Set<T> usage,
    VkSharingMode mode,
    Set<VkMemoryProperty> required,
    Set<VkMemoryProperty> optimal
)
```

Notes:

* This type is generic based on the relevant usage enumeration, e.g. `VkImageUsage` for device memory used by an image.

* The _optimal_ properties are assumed to be in __addition__ to the _required_ properties.

### Allocator

All these collaborating types are brought together into a new service that allocates memory for a given request:

```java
public class Allocator {
    private final DeviceContext dev;
    private final MemoryType[] types;

    public DeviceMemory allocate(VkMemoryRequirements reqs, MemoryProperties<?> props) throws AllocationException {
        MemoryType type = select(reqs, props);
        return allocate(type, reqs.size);
    }
}
```

The Vulkan documentation outlines the suggested approach for selecting the appropriate memory type for a given allocation request as follows:

1. Walk through the supported memory types.

2. Filter by _index_ using the _filter_ bit-mask from the `VkMemoryRequirements` of the object in question.

3. Filter by matching the supported memory properties against the _optimal_ properties of the request.

4. Or fall back to the _required_ properties.

This algorithm is implemented in the following local helper that maps the memory requirements filter to candidate memory types and applies a predicate:

```java
private MemoryType select(VkMemoryRequirements reqs, MemoryProperties<?> props) throws AllocationException {
    var matcher = new FallbackMatcher();
    return new BitField(reqs.memoryTypeBits)
        .stream()
        .mapToObj(n -> types[n])
        .filter(matcher)
        .findAny()
        .or(matcher::fallback)
        .orElseThrow(() -> new AllocationException());
}
```

The predicate tests each candidate against the memory properties and records the fallback as a side-effect:

```java
class FallbackMatcher implements Predicate<MemoryType> {
    private MemoryType fallback;

    @Override
    public boolean test(MemoryType type) {
        // Skip if does not satisfy the minimal requirements
        if(!type.matches(props.required())) {
            return false;
        }

        // Check for optimal match
        if(type.matches(props.optimal())) {
            return true;
        }

        // Record fallback candidate
        if(fallback == null) {
            fallback = type;
        }

        return false;
    }
}
```

The allocator then invokes the API to allocate device memory of the selected type:

```java
protected DeviceMemory allocate(MemoryType type, long size) throws AllocationException {
    // Init memory descriptor
    var info = new VkMemoryAllocateInfo();
    info.allocationSize = oneOrMore(size);
    info.memoryTypeIndex = type.index();

    // Allocate memory
    VulkanLibrary lib = dev.library();
    PointerByReference ref = dev.factory().pointer();
    int result = lib.vkAllocateMemory(dev, info, null, ref);

    // Check allocated
    if(result != VulkanLibrary.SUCCESS) throw new AllocationException();

    // Create device memory
    return new DefaultDeviceMemory(new Handle(ref), dev, type, size);
}
```

Note that the actual size of the allocated memory may be larger than the requested length depending on how the hardware handles alignment.

The API for memory management is actually quite simple:

```java
interface Library {
    int  vkAllocateMemory(DeviceContext device, VkMemoryAllocateInfo pAllocateInfo, Pointer pAllocator, PointerByReference pMemory);
    void vkFreeMemory(DeviceContext device, DeviceMemory memory, Pointer pAllocator);
    int  vkMapMemory(DeviceContext device, DeviceMemory memory, long offset, long size, int flags, PointerByReference ppData);
    void vkUnmapMemory(DeviceContext device, DeviceMemory memory);
}
```

---

## Memory Pool

### Overview

With a basic memory framework in place we can now implement the memory pool.

The requirements for the pool are as follows:

- One pool per memory type.

- A pool grows as required.

- Memory can be pre-allocated to reduce the number of allocations and avoid fragmentation.

- Memory allocated by the pool can be destroyed by the application and potentially re-allocated.

- The allocator and pool(s) should provide useful statistics to the application, e.g. number of allocations, free memory, etc.

To support these requirements a second memory implementation is introduced for a _block_ of memory managed by the pool, requests are then served from a block with available free memory or new blocks are created on demand.

### Memory Blocks

A _block_ is a wrapper for a device memory instance and manages allocations from the block:

```java
class Block {
    private final DeviceMemory mem;
    private final List<BlockDeviceMemory> allocations = new ArrayList<>();
    private long next;

    class BlockDeviceMemory implements DeviceMemory {
        ...
    }

    void destroy() {
        mem.destroy();
        allocations.clear();
    }
}
```

Memory requests are allocated from the end of the block indicated by the `next` free-space offset:

```java
BlockDeviceMemory allocate(long size) {
    // Allocate from free space
    BlockDeviceMemory alloc = new BlockDeviceMemory(next, size);
    allocations.add(alloc);

    // Update free space pointer
    next += size;
    assert next <= mem.size();

    return alloc;
}
```

Memory allocated from the block is an inner class which is essentially a proxy to the device memory of that block:

```java
class BlockDeviceMemory implements DeviceMemory {
    private final long offset;
    private final long size;
    private boolean destroyed;

    @Override
    public Handle handle() {
        return mem.handle();
    }

    @Override
    public long size() {
        return size;
    }
    
    @Override
    public Optional<Region> region() {
        return mem.region();
    }

    @Override
    public Region map(long offset, long size) {
        mem.region().ifPresent(Region::unmap);
        return mem.map(offset, size);
    }

    @Override
    public boolean isDestroyed() {
        return destroyed || mem.isDestroyed();
    }

    @Override
    public void destroy() {
        destroyed = true;
    }
}
```

Notes:

* Since only one region mapping is allowed per memory instance the `map` method silently releases the previous mapping (if any).

* Destroyed allocations are not removed from the block but are only marked as released and can be reallocated (see below).

Finally the amount of available memory in a block is queried as follows:

```java
long free() {
    long total = allocations
        .stream()
        .filter(ALIVE)
        .mapToLong(DeviceMemory::size)
        .sum();
        
    return mem.size() - total;
}
```

Where `ALIVE` is a simple helper constant:

```java
Predicate<DeviceMemory> ALIVE = Predicate.not(DeviceMemory::isDestroyed);
```

### Pool

Each memory pool is comprised of a number of blocks from which memory requests are served:

```java
public class MemoryPool implements TransientObject {
    private final MemoryType type;
    private final List<Block> blocks = new ArrayList<>();
    private long total;

    public long free() {
        return blocks
            .stream()
            .mapToLong(Block::free)
            .sum();
    }
}
```

New blocks can be added to the pool either for pre-allocation or when the pool grows:

```java
void add(Block block) {
    blocks.add(block);
    total += block.size();
}
```

Memory can be allocated from a block with sufficient free space:

```java
public Optional<DeviceMemory> allocate(long size) {
    return blocks
        .stream()
        .filter(block -> block.remaining() >= size)
        .findAny()
        .map(block -> block.allocate(size));
}
```

Or destroyed memory can be reallocated:

```java
public Optional<DeviceMemory> reallocate(long size) {
    return blocks
        .stream()
        .flatMap(Block::allocations)
        .filter(DeviceMemory::isDestroyed)
        .filter(mem -> mem.size() >= size)
        .sorted(Comparator.comparingLong(DeviceMemory::size))
        .findAny()
        .map(mem -> mem.reallocate(size));
}
```

The `reallocate` method of the block memory (not shown) is a crude approach that simply resets the memory size to the request.
Any unused memory is orphaned until the block is released resulting in memory fragmentation (see below).

All memory allocations can be released back to the pool:

```java
public void release() {
    blocks
            .stream()
            .flatMap(Block::allocations)
            .filter(Block.ALIVE)
            .forEach(DeviceMemory::destroy);

    assert free() == total;
}
```

Destroying the pool also destroys the allocated memory blocks:

```java
public void destroy() {
    for(Block b : blocks) {
        b.destroy();
    }
    blocks.clear();
    total = 0;
    assert free() == 0;
}
```

Note that although the pool supports pre-allocation of blocks and memory instance reallocation, the overall memory will be subject to fragmentation.  

However we have decided that de-fragmentation is out-of-scope for a couple of reasons:

1. A de-fragmentation algorithm would be very complex to implement and test.

2. Applications would probably prefer pre-allocation and periodic pool flushes.

### Allocator

A specialised allocator can now be implemented that manages a group of memory pools:

```java
public class PoolAllocator extends Allocator implements TransientObject {
    private final Map<MemoryType, MemoryPool> pools = new ConcurrentHashMap<>();

    @Override
    public void destroy() {
        pools.values().forEach(MemoryPool::destroy);
        assert size() == 0;
        assert free() == 0;
    }
}
```

To service a memory request the allocator tries the following in order:

1. Find an existing block with sufficient free memory.

2. Or find a released allocation that can be reallocated.

3. Otherwise grow the pool and allocate from a new block.

This is implemented as follows:

```java
@Override
protected DeviceMemory allocate(MemoryType type, long size) throws AllocationException {
    MemoryPool pool = pools.computeIfAbsent(type, MemoryPool::new);
    return pool
        .allocate(size)
        .or(() -> pool.reallocate(size))
        .orElseGet(() -> create(type, size, pool));
}
```

Where the `create` method allocates a new memory block, adds it to the pool, and then allocates a memory instance from that new block:

```java
private DeviceMemory create(MemoryType type, long size, MemoryPool pool) throws AllocationException {
    DeviceMemory mem = super.allocate(type, size);
    Block block = new Block(mem);
    pool.add(block);
    return block.allocate(size);
}
```

Notes:

* Memory pools are created on-demand.

* The pool allocator also exposes methods (not shown) to query the various statistics and release or destroy the pools.

### Configuration

The `VkPhysicalDeviceLimits` structure contains two properties that can be used to configure memory allocation:

* `bufferImageGranularity` specifies the optimal _page size_ for memory blocks.

* `maxMemoryAllocationCount` is the maximum number of individual memory allocations that can be supported by the hardware.

The memory allocator is modified to include these two properties:

```java
public class Allocator {
    private final long granularity;
    private final int max;
    private int count;
}
```

Where:

* The `granularity` is the optimal page granularity.

* and `count` is the number of memory allocations capped at `max`.

The `allocate` method can now also be modified to take these properties into account when serving a memory request:

```java
protected DeviceMemory allocate(MemoryType type, long size) throws AllocationException {
    // Check maximum number of allocations
    if(count >= max) throw new AllocationException();

    // Quantise the requested size
    long pages = pages(size);

    // Init memory descriptor
    var info = new VkMemoryAllocateInfo();
    info.allocationSize = granularity * pages;
    ...

    // Create device memory
    ++count;
    return new DefaultDeviceMemory(...);
}
```

The local `pages` method quantises the requested memory size to the page granularity:

```java
protected long pages(long size) {
   return 1 + (size - 1) / granularity;
}
```

Finally the pool allocator overrides this method to allocate new blocks with a minimum number of pages:

```java
public class PoolAllocator extends Allocator {
    private final int pages;

    protected long pages(long size) {
        return Math.max(pages, super.pages(size));
    }
}
```

This allows the pool to over-allocate memory when a new block is required to reduce the overall number of actual allocations.

Note that the pagination approach results in memory allocations that are usually larger than the requested size.
However the page size is usually relatively small (1K on our development machine) and we assume that memory will usually be allocated using the pool-based implementation.

Finally a convenience factory method is added to instantiate and configure an allocator for the logical device:

```java
public static Allocator create(LogicalDevice dev) {
    // Retrieve supported memory types
    var props = dev.parent().memory();
    MemoryType[] types = MemoryType.enumerate(props);

    // Lookup hardware limits
    VkPhysicalDeviceLimits limits = dev.limits();
    int max = limits.maxMemoryAllocationCount;
    long page = limits.bufferImageGranularity;

    // Create allocator
    return new Allocator(dev, types, max, page);
}
```

---

## Summary

In this chapter the framework for memory allocation was implemented to support vertex buffers and texture images.

