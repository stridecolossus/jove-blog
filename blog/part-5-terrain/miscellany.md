---
title: Miscellany
---

---

## Contents

This chapter introduces several largely unrelated Vulkan components that logically follow on from the previous terrain demo chapter:

- [Buffer Helper](#buffer-helper)
- [Push Constants](#push-constants)
- [Shader Constants](#shader-constants)
- [Pipeline Derivation](#pipeline-derivation)
- [Pipeline Cache](#pipeline-cache)
- [Queries](#queries)

---

## Buffer Helper

All of the following sections rely on the `BufferHelper` which is a new utility class for NIO buffers.

Most Vulkan functionality assumes __direct__ NIO buffers which are created by the following convenience factory:

```java
public final class BufferHelper {
    /**
     * Native byte order for a bufferable object.
     */
    public static final ByteOrder NATIVE_ORDER = ByteOrder.nativeOrder();

    private BufferHelper() {
    }

    public static ByteBuffer allocate(int len) {
        return ByteBuffer.allocateDirect(len).order(NATIVE_ORDER);
    }
}
```

The utility class also converts an NIO buffer to a byte array:

```java
public static byte[] array(ByteBuffer bb) {
    if(bb.isDirect()) {
        bb.rewind();
        int len = bb.limit();
        byte[] bytes = new byte[len];
        for(int n = 0; n < len; ++n) {
            bytes[n] = bb.get();
        }
        return bytes;
    }
    else {
        return bb.array();
    }
}
```

And the reverse operation:

```java
public static ByteBuffer buffer(byte[] array) {
    ByteBuffer bb = allocate(array.length);
    if(bb.isDirect()) {
        for(byte b : array) {
            bb.put(b);
        }
    }
    else {
        bb.put(array);
    }
    return bb;
}
```

Direct NIO buffers generally do not support the optional bulk methods, hence this new helper and the various `isDirect` tests in the code.

Existing code that converts arrays to/from byte buffers is refactored using the new utility methods, e.g. shader SPIV code.

---

## Push Constants

### Introduction

Push constants are an alternative and more efficient mechanism for transferring arbitrary data to shaders with some constraints:

* The maximum amount of data is usually relatively small.

* The data is stored within a command buffer _instance_ which is generally updated per frame.

* Push constants have alignment restrictions on the size and offset of each element.

See [vkCmdPushConstants](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdPushConstants.html).

### Range

A _push constant range_ specifies a portion of the data and the shader stages where it can be used:

```java
public record Range(int offset, int size, Set<VkShaderStage> stages) {
    void populate(VkPushConstantRange range) {
        range.stageFlags = new BitMask<>(stages);
        range.size = size;
        range.offset = offset;
    }
}
```

The _offset_ and _size_ of a push constant range must be a multiple of four bytes which is validated in the constructor by the following helper:

```java
static void validate(int size) {
    if((size % 4) != 0) throw new IllegalArgumentException();
}
```

The push constant class itself has a set of ranges and a backing NIO data buffer:

```java
public final class PushConstant {
    public record Range {
        ...
    }

    private final ByteBuffer data;
    private final List<Range> ranges;
}
```

The constructor sizes and allocates the data buffer according to the ranges:

```java
PushConstant(List<Range> ranges) {
    int len = ranges.stream().mapToInt(Range::max).reduce(0, Integer::max);
    this.data = BufferHelper.allocate(len);
    this.ranges = List.copyOf(ranges);
}
```

Where `max` is used to determine the overall size of the data buffer:

```java
int max() {
    return offset + size;
}
```

Notes:

* Multiple ranges can be specified which allows the application to update some or all of the push constants at different shader stages and also enables the hardware to perform optimisations.

* Ranges can overlap, hence the need to determine the `max` buffer length.

* However the entire backing buffer must be covered by at least one range (validation not shown).

A range of the push constants is updated in the render sequence by the following command:

```java
private final class UpdateCommand implements Command {
    private final Range range;
    private final PipelineLayout layout;
    private final BitMask<VkShaderStage> stages;

    @Override
    public void record(VulkanLibrary lib, Command.Buffer buffer) {
        data.position(range.offset);
        lib.vkCmdPushConstants(buffer, layout, stages, range.offset, range.size, data);
    }
}
```

And the new API method is added to the pipeline layout library:

```java
void vkCmdPushConstants(Buffer commandBuffer, PipelineLayout layout, int stageFlags, int offset, int size, ByteBuffer pValues);
```

Note that the data buffer is rewound before updates are applied, generally Vulkan seems to automatically rewind buffers as required (e.g. for updating the uniform buffer) but not in this case.

And update commands are created by a factory method:

```java
public Command update(Range range, PipelineLayout layout) {
    if(!ranges.contains(range)) throw new IllegalArgumentException();
    if(layout.push() != this) throw new IllegalArgumentException();
    return new UpdateCommand(range, layout);
}
```

### Pipeline Layout

A push constant is added to the pipeline layout and the associated builder is modified to configure the ranges:

```java
public static class Builder {
    private final List<Range> ranges = new ArrayList<>();

    public Builder add(Range range) {
        ranges.add(range);
        return this;
    }
}
```

The push constant is instantiated and the ranges added to the pipeline layout:

```java
PushConstant push;
if(ranges.isEmpty()) {
    push = PushConstant.NONE;
}
else {
    push = new PushConstant(ranges);
    info.pushConstantRangeCount = ranges.size();
    info.pPushConstantRanges = StructureCollector.pointer(ranges, new VkPushConstantRange(), Range::populate);
}
    
...
    
return new PipelineLayout(layout.getValue(), dev, push);
```

Where `NONE` is the empty push constant.

### Integration

To use the push constants in the terrain demo the uniform buffer is first replaced with the following layout declaration in the vertex shader:

```glsl
layout(push_constant) uniform Matrices {
    mat4 model;
    mat4 view;
    mat4 projection;
};
```

In the pipeline configuration we remove the uniform buffer and replace it with a single push constant range sized to the three matrices:

```java
@Bean
static Range range() {
    int len = 3 * Matrix.IDENTITY.length();
    return new Range(0, len, Set.of(VkShaderStage.VERTEX));
}
```

Which is registered with the pipeline:

```java
@Bean
PipelineLayout layout(DescriptorSet.Layout layout, PushConstant.Range range) {
    return new PipelineLayout.Builder()
        .add(layout)
        .add(range)
        .build(dev);
}
```

Next a command is created to update the whole of the push constants buffer:

```java
@Bean
static UpdateCommand update(Range range, PipelineLayout layout) {
    PushConstant push = layout.push();
    return push.update(range, layout);
}
```

In the camera configuration the push constants are updated once per frame:

```java
@Bean
public Task matrix(PushUpdateCommand update) {
    ByteBuffer data = update.data();
    return () -> {
        data.rewind();
        Matrix.IDENTITY.buffer(data);
        cam.matrix().buffer(data);
        projection.buffer(data);
    };
}
```

And finally the update command is added to the render sequence before starting the render pass.

---

## Shader Constants

The terrain shaders contain a number of hard coded parameters (such as the tesselation factor and height scalar) which would ideally be programatically configured, possibly from a properties file.
Additionally in general we prefer to centralise common or shared parameters to avoid hard-coding the same information in multiple locations or having to replicate shaders for different parameters.

Vulkan provides _specialisation constants_ for these requirements which parameterise a shader when it is instantiated.
For example in the evaluation shader the hard-coded height scale is replaced with the following constant declaration:

```glsl
layout(constant_id=1) const float HeightScale = 2.5;
```

Note that the constant also has a default value if it is not explicitly configured by the application.

First a new type is implemented as a wrapper for a set of constants:

```java
public final class SpecialisationConstants {
    private final Map<Integer, Constant> constants;
}
```

Where a specialisation constant is defined as follows:

```java
public sealed interface Constant {
    /**
     * @return Size of this constant (bytes)
     */
    int size();

    /**
     * Writes this constant to the given buffer.
     * @param bb Buffer
     */
    void buffer(ByteBuffer bb);
}
```

We can now implement the supported constant types, starting with integer values:

```java
record IntegerConstant(int value) implements Constant {
    @Override
    public int size() {
        return Integer.BYTES;
    }

    @Override
    public void buffer(ByteBuffer bb) {
        bb.putInt(value);
    }
}
```

Floating-point values:

```java
record FloatConstant(float value) implements Constant {
    @Override
    public int size() {
        return Float.BYTES;
    }

    @Override
    public void buffer(ByteBuffer bb) {
        bb.putFloat(value);
    }
}
```

And finally boolean values:

```java
record BooleanConstant(boolean value) implements Constant {
    @Override
    public int size() {
        return Integer.BYTES;
    }

    @Override
    public void buffer(ByteBuffer bb) {
        int n = NativeBooleanConverter.toInteger(value);
        bb.putInt(n);
    }
}
```

A convenience builder (not shown) is also implemented to construct a set of shader constants.

The Vulkan descriptor for a set of specialisation constants consists of two components:

1. A data buffer containing the actual values of the constants.

2. A list of entries specifying the constant identifiers and offsets into the data buffer.

First the constants are transformed to an array of entries:

```java
VkSpecializationInfo build() {
    Populate populate = new Populate();
    VkSpecializationMapEntry entries = StructureCollector.pointer(constants.entrySet(), new VkSpecializationMapEntry(), populate::populate);
    ...
}
```

Where the `Populate` helper class builds the entry for each constant and calculates the offset and the overall buffer length as a side-effect:

```java
private static class Populate {
    private int len;

    void populate(Entry<Integer, Constant> entry, VkSpecializationMapEntry out) {
        // Determine the size of this constant
        Constant constant = entry.getValue();
        int size = constant.size();

        // Populate the entry for this constant
        out.constantID = entry.getKey();
        out.offset = len;
        out.size = size;

        // Calculate the overall buffer length
        len += size;
    }
}
```

Now the data buffer can be allocated and populated from the constant values:

```java
var bb = BufferHelper.allocate(populate.len);
for(Constant c : constants.values()) {
    c.buffer(bb);
}
```

And finally the Vulkan descriptor for the set of constants is constructed:

```java
var info = new VkSpecializationInfo();
info.mapEntryCount = constants.size();
info.pMapEntries = entries;
info.dataSize = populate.len;
info.pData = bb;
return info;
```

Specialisation constants are configured in the programmable shader stages of the pipeline:

```java
public class ProgrammableShaderStage {
    private SpecialisationConstants constants;

    void populate(VkPipelineShaderStageCreateInfo info) {
        ...
        if(constants != null) {
            info.pSpecializationInfo = constants.build();
        }
    }
}
```

A set of specialisation constants can be now be configured for use in both tesselation shaders:

```java
class PipelineConfiguration {
    private final SpecialisationConstants constants = new SpecialisationConstants(Map.of(0, 20f, 1, 2.5f));
}
```

And finally the relevant shaders are parameterised when the pipeline is constructed, for example:

```java
shader(VkShaderStage.TESSELLATION_EVALUATION)
    .shader(evaluation)
    .constants(constants)
    .build()
```

## Pipeline Derivation

### Overview

The wireframe terrain model is useful for visually testing the tesselation shader but we would like to be able to toggle between filled and wireframe modes.  The _polygon mode_ is a property of the rasterizer pipeline stage which implies the demo needs _two_ pipelines to switch between modes.  As things stand we _could_ use the same builder to create two pipelines with the second overriding the polygon mode.

However Vulkan supports _derivative_ pipelines which provide a hint to the hardware that a derived (or child) pipeline shares common properties with its parent, potentially improving performance when the pipelines are instantiated and when switching bindings in the render sequence.

### Derivative Pipelines

A pipeline that allows derivatives (i.e. the parent) is identified by a flag at instantiation-time:

```java
public static class Builder {
    private final Set<VkPipelineCreateFlag> flags = new HashSet<>();
    private Handle base;

    ...

    public Builder flag(VkPipelineCreateFlag flag) {
        this.flags.add(flag);
        return this;
    }
    
    public Builder parent() {
        return flag(VkPipelineCreateFlag.ALLOW_DERIVATIVES);
    }
}
```

Where `parent` configures a pipeline that can be derived from.

Note that the set of `flags` is also added to the pipeline domain object.

Vulkan offers two mutually exclusive methods to derive pipelines:

1. Derive from an _existing_ pipeline instance.

2. Create an array of pipelines where derivative _peer_ pipelines are specified by index with the array.

A pipeline derived from an existing parent instance is specified by the following new method on the builder:

```java
public Builder derive(Pipeline base) {
    check(base.flags());
    this.base = base.handle();
    return this;
}
```

Where the local `derive` helper validates that the parent supports derivatives and marks the pipeline as a derivative:

```java
private void derive(Set<VkPipelineCreateFlag> flags) {
    if(!flags.contains(VkPipelineCreateFlag.ALLOW_DERIVATIVES)) throw new IllegalStateException(...);
    this.flags.add(VkPipelineCreateFlag.DERIVATIVE);
}
```

The base pipeline is populated in the `build` method:

```java
info.basePipelineHandle = base;
```

### Pipeline Peers

To support peer derivatives the pipeline builder is refactored to create an array of pipelines:

```java
public static List<Pipeline> build(List<Builder> builders, PipelineCache cache, DeviceContext dev) {
    // Build array of descriptors
    VkGraphicsPipelineCreateInfo[] array = StructureCollector.array(builders, new VkGraphicsPipelineCreateInfo(), Builder::populate);

    // Allocate pipelines
    VulkanLibrary lib = dev.library();
    Pointer[] handles = new Pointer[array.length];
    check(lib.vkCreateGraphicsPipelines(dev, cache, array.length, array, null, handles));

    // Create pipelines
    Pipeline[] pipelines = new Pipeline[array.length];
    for(int n = 0; n < array.length; ++n) {
        Builder builder = builders.get(n);
        pipelines[n] = new Pipeline(handles[n], dev, builder.layout, builder.flags);
    }
    return Arrays.asList(pipelines);
}
```

And the existing `build` method becomes a helper to instantiate a single pipeline.

A derivative peer pipeline can now be configured by a second `derive` overload:

```java
public Builder derive(Builder parent) {
    if(parent == this) throw new IllegalStateException();
    derive(parent.flags);
    this.parent = notNull(parent);
    return this;
}
```

The peer index is patched in the `build` method after the array has been populated:

```java
for(int n = 0; n < array.length; ++n) {
    Builder parent = builders.get(n).parent;
    if(parent == null) {
        continue;
    }
    int index = builders.indexOf(parent);
    if(index == -1) throw new IllegalArgumentException();
    array[n].basePipelineIndex = index;
}
```

Note that this second derivative approach currently does __not__ clone the pipeline builder configuration implying there may still be a large amount of code duplication (pipeline layout, the various stages, etc).  This may be something for future consideration.

### Integration

In the demo we now have several alternatives to switch between polygon modes, the configuration is modified to derive the wire-frame alternative from the existing pipeline:

```java
@Bean
public List<Pipeline> pipelines(...) {
    // Init main pipeline
    Pipeline pipeline = new Pipeline.Builder()
        ...
        .build();

    // Derive wireframe pipeline
    Pipeline wireframe = pipeline
        ...
        .derive(pipeline)
        .rasterizer()
            .polygon(VkPolygonMode.LINE)
            .build()
        .build();
}
```

In the render configuration the beans for the pipeline and command sequence are modified to generate new instances on each invocation rather than the previous singleton:

```java
private final AtomicInteger index = new AtomicInteger();

@Bean()
public RenderSequence(Pipeline[] pipelines) {
}
```

TODO 

And a toggle handler is bound to the space bar to allow the application to switch between the pipelines at runtime. Cool.

---

## Pipeline Cache

A _pipeline cache_ stores the results of pipeline construction and can be reused between pipelines and between runs of an application, allowing the hardware to possibly optimise pipeline construction.  

A pipeline cache is a trivial opaque domain class:

```java
public class PipelineCache extends AbstractVulkanObject {
    @Override
    protected Destructor<PipelineCache> destructor(VulkanLibrary lib) {
        return lib::vkDestroyPipelineCache;
    }
}
```

A cache object is instantiated using a factory method:

```java
public static PipelineCache create(DeviceContext dev, byte[] data) {
    // Build create descriptor
    VkPipelineCacheCreateInfo info = new VkPipelineCacheCreateInfo();
    if(data != null) {
        info.initialDataSize = data.length;
        info.pInitialData = BufferHelper.buffer(data);
    }

    // Create cache
    VulkanLibrary lib = dev.library();
    PointerByReference ref = dev.factory().pointer();
    check(lib.vkCreatePipelineCache(dev, info, null, ref));

    // Create domain object
    return new PipelineCache(ref.getValue(), dev);
}
```

Where `data` is the previously persisted cache (the data itself is platform specific).

The following method retrieves the cache data after construction of the pipeline (generally before application termination):

```java
public ByteBuffer data() {
    DeviceContext dev = super.device();
    VulkanFunction<ByteBuffer> func = (count, data) -> dev.library().vkGetPipelineCacheData(dev, this, count, data);
    IntByReference count = dev.factory().integer();
    return func.invoke(count, BufferHelper::allocate);
}
```

Caches can also be merged such that a single instance can be reused across multiple pipelines:

```java
public void merge(Collection<PipelineCache> caches) {
    DeviceContext dev = super.device();
    VulkanLibrary lib = dev.library();
    check(lib.vkMergePipelineCaches(dev, this, caches.size(), NativeObject.array(caches)));
}
```

Finally we add a new API library for the cache:

```java
int  vkCreatePipelineCache(DeviceContext device, VkPipelineCacheCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference pPipelineCache);
int  vkMergePipelineCaches(DeviceContext device, PipelineCache dstCache, int srcCacheCount, Pointer pSrcCaches);
int  vkGetPipelineCacheData(DeviceContext device, PipelineCache cache, IntByReference pDataSize, ByteBuffer pData);
void vkDestroyPipelineCache(DeviceContext device, PipelineCache cache, Pointer pAllocator);
```

To persist a cache we implement a simple loader:

```java
public static class Loader implements ResourceLoader<InputStream, PipelineCache> {
    private final DeviceContext dev;

    @Override
    public InputStream map(InputStream in) throws IOException {
        return in;
    }

    @Override
    public PipelineCache load(InputStream in) throws IOException {
        byte[] data = in.readAllBytes();
        return create(dev, data);
    }

    public void write(PipelineCache cache, OutputStream out) throws IOException {
        byte[] array = BufferHelper.array(cache.data());
        out.write(array);
    }
}
```

TODO - cache manager, file source helper, integration

---

## Queries

### Overview

Vulkan provides a _query_ API that allows an application to perform the following:

* _occlusion_ queries.

* retrieve pipeline statistics.

* inject timestamps into a command sequence.

* and some extensions that are out-of-scope for this project.

A _query_ is generally implemented as a pair of commands that wrap a segment of a rendering sequence, except for timestamps which consist of a single atomic command.
Queries are allocated from a _query pool_ which is essentially an array of available _slots_ for the results.

After execution the results of the query can be retrieved on-demand to an arbitrary NIO buffer or copied asynchronously to a Vulkan buffer.

First a new JNA library is created for the query API:

```java
interface Library {
    int  vkCreateQueryPool(DeviceContext device, VkQueryPoolCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference pQueryPool);
    void vkDestroyQueryPool(DeviceContext device, Pool queryPool, Pointer pAllocator);
    void vkCmdResetQueryPool(Buffer commandBuffer, Pool queryPool, int firstQuery, int queryCount);
    void vkCmdBeginQuery(Buffer commandBuffer, Pool queryPool, int query, BitMask<VkQueryControlFlag> flags);
    void vkCmdEndQuery(Buffer commandBuffer, Pool queryPool, int query);
    void vkCmdWriteTimestamp(Buffer commandBuffer, VkPipelineStage pipelineStage, Pool queryPool, int query);
    int  vkGetQueryPoolResults(DeviceContext device, Pool queryPool, int firstQuery, int queryCount, long dataSize, ByteBuffer pData, long stride, BitMask<VkQueryResultFlag> flags);
    void vkCmdCopyQueryPoolResults(Buffer commandBuffer, Pool queryPool, int firstQuery, int queryCount, VulkanBuffer dstBuffer, long dstOffset, long stride, BitMask<VkQueryResultFlag> flags);
}
```

### Queries

The outline class for the query framework is as follows:

```java
public interface Query {
    /**
     * Default implementation for a measurement query wrapping a portion of the render sequence.
     */
    interface DefaultQuery extends Query {
    }

    /**
     * Timestamp query.
     */
    interface Timestamp extends Query {
    }

    public static class Pool extends AbstractVulkanObject {
        private final VkQueryType type;
        private final int slots;
    }
}
```

A query pool is instantiated via a factory method:

```java
public static Pool create(DeviceContext dev, VkQueryType type, int slots, VkQueryPipelineStatisticFlag... stats) {
    // Init create descriptor
    var info = new VkQueryPoolCreateInfo();
    info.queryType = notNull(type);
    info.queryCount = oneOrMore(slots);
    info.pipelineStatistics = new BitMask<>(stats);

    // Instantiate query pool
    PointerByReference ref = dev.factory().pointer();
    VulkanLibrary lib = dev.library();
    check(lib.vkCreateQueryPool(dev, info, null, ref));

    // Create pool
    return new Pool(ref.getValue(), dev, type, slots);
}
```

Note that the `stats` collection parameter is only valid for a `PIPELINE_STATISTICS` query.

Query instances can then be allocated from the pool, the default implementation is used to wrap a segment of the rendering sequence:

```java
public DefaultQuery query(int slot) {
    return new DefaultQuery() {
        @Override
        public Command begin(VkQueryControlFlag... flags) {
            int mask = new BitMask<>(flags);
            return (lib, buffer) -> lib.vkCmdBeginQuery(buffer, Pool.this, slot, mask);
        }

        @Override
        public Command end() {
            return (lib, buffer) -> lib.vkCmdEndQuery(buffer, Pool.this, slot);
        }
    };
}
```

Similarly for timestamp queries:

```java
public Timestamp timestamp(int slot) {
    return new Timestamp() {
        @Override
        public Command timestamp(VkPipelineStage stage) {
            return (lib, buffer) -> lib.vkCmdWriteTimestamp(buffer, stage, Pool.this, slot);
        }
    };
}
```

Finally query slots (or the entire pool) must be reset before execution:

```java
public Command reset(int start, int num) {
    return (lib, buffer) -> lib.vkCmdResetQueryPool(buffer, this, start, num);
}
```

Note that this command __must__ be invoked before the start of a render pass.

### Results Builder

Retrieval of the results of a query is somewhat complicated:

* There are two supported mechanisms: on-demand or asynchronous copy to a buffer.

* Multiple results can be retrieved in one operation by specifying a _range_ of query slots, i.e. essentially an array of results.

* Results can be integer or long data types.

Therefore configuration of the results is specified by a builder instantiated from the query pool:

```java
public class Pool ... {
    public ResultBuilder result() {
        return new ResultBuilder(this);
    }
}
```

The builder specifies the data type and the range of query slots to be retrieved:

```java
public static class ResultBuilder {
    private final Pool pool;
    private int start;
    private int count;
    private long stride = Integer.BYTES;
    private final Set<VkQueryResultFlag> flags = new HashSet<>();
}
```

A convenience setter is provided to initialise the results to a long data type:

```java
public ResultBuilder longs() {
    flag(VkQueryResultFlag.LONG);
    stride(Long.BYTES);
    return this;
}
```

Since query results can be retrieved using two different mechanisms the builder provides __two__ build methods.
For the case where the results are retrieved on-demand the application provides an NIO buffer:

```java
public Consumer<ByteBuffer> build() {
    int mask = validate();

    DeviceContext dev = pool.device();
    Library lib = dev.library();

    return buffer -> {
        check(lib.vkGetQueryPoolResults(dev, pool, start, count, size, buffer, stride, mask));
    };
}
```

The `validate` method checks that the `stride` is a multiple of the specified data type and builds the flags bit-mask:

```java
private BitMask<VkQueryResultFlag> validate() {
    // Validate query range
    if(start + count > pool.slots) {
        throw new IllegalArgumentException();
    }

    // Validate stride
    long multiple = flags.contains(VkQueryResultFlag.LONG) ? Long.BYTES : Integer.BYTES;
    if((stride % multiple) != 0) {
        throw new IllegalArgumentException();
    }

    // Build flags mask
    return new BitMask<>(flags);
}
```

The second variant generates a command that is injected into the render sequence to asynchronously copy the results to a given Vulkan buffer:

```java
public Command build(VulkanBuffer buffer, long offset) {
    buffer.require(VkBufferUsageFlag.TRANSFER_DST);
    int mask = validate();
    return (lib, cmd) -> {
        lib.vkCmdCopyQueryPoolResults(cmd, pool, start, count, buffer, offset, stride, mask);
    };
}
```

### Integration

The query framework is used in the terrain demo to further verify the tesselation process using a pipeline statistics query.

First a query pool is created that counts the number of vertices generated by the tesselator:

```java
@Bean
public Pool queryPool(LogicalDevice dev) {
    return new Pool.Builder()
        .type(VkQueryType.PIPELINE_STATISTICS)
        .statistic(VkQueryPipelineStatisticFlag.VERTEX_SHADER_INVOCATIONS)
        .slots(1)
        .build(dev);
}
```

A query instance is then allocated from the pool:

```java
@Bean
public Query query(Query.Pool pool) {
    return pool.query(1);
}
```

The query wraps the render pass in the command sequence:

```java
buffer
    .add(pool.reset())
    .add(frame.begin())
        .add(query.begin())
            ...
        .add(query.end())
    .add(FrameBuffer.END)
```

Again note the pool is `reset` before the start of the pass.

Finally the query results are configured as a frame listener to dump the vertex count to the console:

```java
@Bean
public FrameListener queryResults() {
    ByteBuffer results = BufferHelper.allocate(Integer.BYTES);
    Consumer<ByteBuffer> consumer = pool.result().build();
    return () -> {
        consumer.accept(results);
        System.out.println(results.getInt());
    };
}
```

The terrain demo application should now output the number of the vertices generated on each frame.  Obviously this is a very crude implementation just to illustrate the query framework.

---



