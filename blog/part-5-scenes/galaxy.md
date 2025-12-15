---
title: Galaxy Demo
---

---

## Contents

TODO

- [Overview](#overview)
- [Galaxy Model](#galaxy-model)
- [Rendering](#rendering)

---

## Overview

In this chapter we will create a new project to render a model of a galaxy.

Rather than attempting to generate a convincing galaxy from first principles (which would be a project in itself) the model will be derived from an image.  The general approach is:

1. Load a galaxy image.

1. Scale the image down to a manageable size.

1. Create a flat grid of points corresponding to the scaled image.

1. Select the vertex colour of each point from the image.

1. Discard those where the colour is lower than some threshold, i.e. ignore black or very faint 'stars'.

1. Randomly set the vertical position of each vertex to simulate the _thin disc_ and central _bulge_ of the galaxy.

The approach for rendering the resultant model is:

* Transform each point to a _billboard_ (i.e. a flat quad facing the camera).

* Render each 'star' as a translucent disc with the colour fading with distance from the centre of the quad.

Therefore the following new components will be required for this demo:

* Support for the _geometry_ shader.

* Implementation of the _colour blending_ pipeline stage to mix the colour of overlapping vertices.

Several further improvements will also be introduced during the course of this chapter.

---

## Galaxy Model

The galaxy model is created by a quick-and-dirty builder:

```java
class ModelBuilder {
    private final Random random = new Random();

    private int size = 64;
    private int threshold = 100;
    private float bulge = 0.1f;
    private float disc = 0.005f;
    private float scale = 1;
}
```

Which will create a point-cloud model with colours:

```java
public Mesh build(InputStream in) throws IOException {
    var mesh = new DefaultMesh(Primitive.POINTS, Point.LAYOUT, Colour.LAYOUT);
    ...
    return mesh;
}
```

For the moment the galaxy image is loaded and scaled down using the AWT image library:

```java
BufferedImage image = ImageIO.read(in);
BufferedImage scaled = new BufferedImage(size, size, image.getType());
Graphics2D g = scaled.createGraphics();
g.setRenderingHint(RenderingHints.KEY_INTERPOLATION, RenderingHints.VALUE_INTERPOLATION_BILINEAR);
g.drawImage(image, 0, 0, size, size, null);
g.dispose();
```

Next the image is iterated to lookup the colour of each pixel:

```java
Raster raster = scaled.getData();
float[] pixel = new float[3];
for(int x = 0; x < size; ++x) {
    for(int y = 0; y < size; ++y) {
        raster.getPixel(x, y, pixel);
        ...
    }
}
```

Pixels that are darker than a configured `threshold` are discarded:

```java
if(pixel[0] + pixel[1] + pixel[2] < threshold) {
    continue;
}
```

Next the vertex position is determine from the pixel coordinates:

```java
float px = scale(x);
float py = scale(y);
```

Where `scale` calculates a scaled coordinate about the model origin:

```java
private float scale(float pos) {
    return (pos / size - MathsUtil.HALF) * scale;
}
```

To generate a 3D galaxy model the distance __squared__ from the centre of the galaxy is calculated and the Z coordinate is randomised:

```java
float dist = px * px + py * py;
float range = dist < bulge ? bulge - dist : disc;
float z = random.nextFloat(-range, +range);
Point pos = new Point(px, py, z);
```

Where `bulge` is the radius of the central bulge, such that `bulge - dist` creates a poor mans spherical distribution.

Next the pixel colour is scaled to percentile values:

```java
for(int n = 0; n < 3; ++n) {
    pixel[n] = pixel[n] / 0xFF;
}
Colour col = Colour.of(pixel);
```

And finally the vertex is added to the model:

```java
Vertex vertex = Vertex.of(pos, col);
builder.add(vertex);
```

---

## Rendering

### Geometry Shader

The role of the geometry shader in this demo is to transform each point in the model to a billboard quad.

This is declared in the shader layout:

```glsl
layout(points) in;
layout(triangle_strip, max_vertices=4) out;
```

The shader retrieves the vertex position and generates the billboard vertices:

```glsl
void main() {
    vec4 pos = gl_in[0].gl_Position;
    vertex(pos, -1, +1);
    vertex(pos, -1, -1);
    vertex(pos, +1, +1);
    vertex(pos, +1, -1);
    EndPrimitive();
}
```

Which uses the following helper method:

```glsl
layout(binding=0) uniform Projection {
    mat4 projection;
};

const float SIZE = 0.025;

void vertex(vec4 pos, float x, float y) {
    gl_Position = projection * (pos + vec4(x * SIZE, y * SIZE, 0, 0));
    outCoords = vec2(x, y);
    outColour = inColour[0];
    EmitVertex();
}
```

Note:

* The incoming colour has to be defined as an array, in this case with a single element used for all generated vertices.

* The geometry shader only applies perspective projection since the vertex shader has already transformed the model into view space, i.e. the billboard faces the screen.

* The projection matrix remains as a uniform buffer since it is not a volatile object, and push constants are a limited resource.

The final geometry shader is as follows:

```glsl
#version 450

layout(binding=0) uniform Projection {
    mat4 projection;
};

layout(points) in;

layout(triangle_strip, max_vertices=4) out;

layout(location=0) in vec4[] inColour;

layout(location=0) out vec2 outCoords;
layout(location=1) out vec4 outColour;

const float SIZE = 0.025;

void vertex(vec4 pos, float x, float y) {
    gl_Position = projection * (pos + vec4(x * SIZE, y * SIZE, 0, 0));
    outCoords = vec2(x, y);
    outColour = inColour[0];
    EmitVertex();
}

void main() {
    vec4 pos = gl_in[0].gl_Position;
    vertex(pos, -1, +1);
    vertex(pos, -1, -1);
    vertex(pos, +1, +1);
    vertex(pos, +1, -1);
    EndPrimitive();
}
```

### Specialisation Constants

The `layout(constant_id=1) const float SIZE = 0.025`


The terrain shaders contain a number of hard coded parameters (such as the tesselation factor and height scalar) which would ideally be programatically configured (possibly from a properties file).

Additionally in general we prefer to centralise common or shared parameters to avoid hard-coding the same information in multiple locations or having to replicate shaders for different parameters.

Vulkan provides _specialisation constants_ for these requirements which parameterise a shader when it is instantiated.

For example in the evaluation shader the hard-coded height scale is replaced with the following constant declaration:

```glsl
layout(constant_id=1) const float HeightScale = 2.5;
```

Note that the constant also has a default value if it is not explicitly configured by the application.

The descriptor for a set of specialisation constants is constructed via a new factory method:

```java
public static VkSpecializationInfo constants(Map<Integer, Object> constants) {
    // Skip if empty
    if(constants.isEmpty()) {
        return null;
    }
    ...
}
```

Each constant generates a separate child descriptor:

```java
var info = new VkSpecializationInfo();
info.mapEntryCount = constants.size();
info.pMapEntries = StructureCollector.pointer(constants.entrySet(), new VkSpecializationMapEntry(), populate);
```

Where `populate` is factored out to the following function:

```java
var populate = new BiConsumer<Entry<Integer, Object>, VkSpecializationMapEntry>() {
    private int len = 0;

    @Override
    public void accept(...) {
        ...
    }
};
```

This function populates the descriptor for each entry and calculates the buffer offset and total length as a side-effect:

```java
// Init constant
int size = size(entry.getValue());
out.size = size;
out.constantID = entry.getKey();

// Update buffer offset
out.offset = len;
len += size;
```

Where `size` is a local helper:

```java
int size() {
    return switch(value) {
        case Integer n -> Integer.BYTES;
        case Float f -> Float.BYTES;
        case Boolean b -> Integer.BYTES;
        default -> throw new UnsupportedOperationException();
    };
}
```

The constants are then written to a data buffer:

```java
ByteBuffer buffer = BufferHelper.allocate(populate.len);
for(Object value : constants.values()) {
    switch(value) {
        case Integer n -> buffer.putInt(n);
        case Float f -> buffer.putFloat(f);
        case Boolean b -> converter.toNative(b, null);
        default -> throw new RuntimeException();
    }
}
```

And finally this data is added to the descriptor:

```java
info.dataSize = populate.len;
info.pData = buffer;
```

Notes:

* Only scalar (int, float) and boolean values are supported.

* Booleans are represented as integer values.

Specialisation constants are configured in the pipeline:

```java
public class ShaderStageBuilder {
    private VkSpecializationInfo constants;
    
    ...
    
    public ShaderStageBuilder constants(VkSpecializationInfo constants) {
        this.constants = notNull(constants);
        return this;
    }
 
    void populate(VkPipelineShaderStageCreateInfo info) {
        ...
        info.pSpecializationInfo = constants;
    }
}
```

The set of constants used in both tesselation shaders is initialised in the pipeline configuration class:

```java
class PipelineConfiguration {
    private final VkSpecializationInfo constants = Shader.constants(Map.of(0, 20f, 1, 2.5f));
}
```

Finally the relevant shaders are parameterised when the pipeline is constructed, for example:

```java
shader(VkShaderStage.TESSELLATION_EVALUATION)
    .shader(evaluation)
    .constants(constants)
    .build()
```

### Colour Blending

The fragment shader generates a circle from the quad:

```glsl
#version 450

layout(location=0) in vec2 inCoord;
layout(location=1) in vec4 inColour;

layout(location=0) out vec4 outColour;

layout(constant_id=2) const float RADIUS = 0.2;

void main() {
    float alpha = 1 - dot(inCoord, inCoord);
    if(alpha < RADIUS) {
        discard;
    }
    outColour = vec4(alpha);
}
```

To enable colour blending in the demo a new pipeline stage is implemented:

```java
public class ColourBlendPipelineStageBuilder extends AbstractPipelineStageBuilder<VkPipelineColorBlendStateCreateInfo> {
    private final VkPipelineColorBlendStateCreateInfo info = new VkPipelineColorBlendStateCreateInfo();

    public ColourBlendPipelineStageBuilder() {
        info.logicOpEnable = false;
        info.logicOp = VkLogicOp.COPY;
        Arrays.fill(info.blendConstants, 1);
    }
}
```

The global blending properties are configured as follows:

```java
public ColourBlendPipelineStageBuilder enable(boolean enabled) {
    info.logicOpEnable = enabled;
    return this;
}

public ColourBlendPipelineStageBuilder operation(VkLogicOp op) {
    info.logicOp = notNull(op);
    return this;
}

public ColourBlendPipelineStageBuilder constants(float[] constants) {
    System.arraycopy(constants, 0, info.blendConstants, 0, constants.length);
    return this;
}
```

The blending configuration for each colour attachment is implemented as a nested builder:

```java
public AttachmentBuilder attachment() {
    return new AttachmentBuilder();
}
```

Which specifies the logical operation between a fragment and the existing colour in the framebuffer attachment(s).

```java
public class AttachmentBuilder {
    private static final List<VkColorComponent> MASK = Arrays.asList(VkColorComponent.values());

    private boolean enabled;
    private List<VkColorComponent> mask = MASK;
    private final BlendOperationBuilder colour = new BlendOperationBuilder();
    private final BlendOperationBuilder alpha = new BlendOperationBuilder();

    private AttachmentBuilder() {
        colour.source(VkBlendFactor.SRC_ALPHA);
        colour.destination(VkBlendFactor.ONE_MINUS_SRC_ALPHA);
        alpha.source(VkBlendFactor.ONE);
        alpha.destination(VkBlendFactor.ZERO);
        attachments.add(this);
    }
}
```

The colour `mask` is used to specify which channels are subject to the blend operation, expressed as a simple string:

```java
public AttachmentBuilder mask(String mask) {
    this.mask = mask
        .chars()
        .mapToObj(Character::toString)
        .map(VkColorComponent::valueOf)
        .toList();
    return this;
}
```

The properties for the colour and alpha components are identical so a further nested builder configures both values:

```java
public class BlendOperationBuilder {
    private VkBlendFactor src;
    private VkBlendFactor dest;
    private VkBlendOp blend = VkBlendOp.ADD;

    private BlendOperationBuilder() {
    }

    public AttachmentBuilder build() {
        return AttachmentBuilder.this;
    }
}
```

The descriptor for the attachment is generated in the nested builder as follows:

```java
private void populate(VkPipelineColorBlendAttachmentState info) {
    // Init descriptor
    info.blendEnable = enabled;
    info.colorWriteMask = mask;

    // Init colour blending operation
    info.srcColorBlendFactor = colour.src;
    info.dstColorBlendFactor = colour.dest;
    info.colorBlendOp = colour.blend;

    // Init alpha blending operation
    info.srcAlphaBlendFactor = alpha.src;
    info.dstAlphaBlendFactor = alpha.dest;
    info.alphaBlendOp = alpha.blend;
}
```

And finally the descriptor for the pipeline stage can be constructed:

```java
@Override
VkPipelineColorBlendStateCreateInfo get() {
    // Init default attachment if none specified
    if(attachments.isEmpty()) {
        new AttachmentBuilder().build();
    }

    // Add attachment descriptors
    info.attachmentCount = attachments.size();
    info.pAttachments = StructureCollector.pointer(attachments, new VkPipelineColorBlendAttachmentState(), AttachmentBuilder::populate);

    return info;
}
```

Note that by convenience a single, default attachment is added if none are configured (at least one must be specified).

### Integration

The pipeline configuration can now be updated to apply an additive blending operation in the demo:

```java
.blend()
    .enable(true)
    .operation(VkLogicOp.COPY)
    .attachment()
        .colour()
            .destination(VkBlendFactor.ONE)
            .build()
        .build()
    .get();
```

---

## Further Improvements

### Device Limits



In the `build` method for the pipeline layout we determine the _maximum_ length of the push ranges:

```java
int max = ranges
    .stream()
    .mapToInt(PushConstantRange::length)
    .max()
    .orElse(0);
```

This is validated (not shown) against the hardware limit specified by the `maxPushConstantsSize` of the `VkPhysicalDeviceLimits` structure, this value is usually quite small (256 bytes on our development environment).




The `VkPhysicalDeviceLimits` structure specifies various limits supported by the hardware, this is wrapped by a new helper class:

```java
public class DeviceLimits {
    private final VkPhysicalDeviceLimits limits;
    private final DeviceFeatures features;
}
```

For convenience the supported device features are also incorporated into this new class and can be enforced as required:

```java
public void require(String name) {
    if(!features.features().contains(name)) {
        throw new IllegalStateException("Feature not supported: " + name);
    }
}
```

A device limit can be queried from the structure by name:

```java
public <T> T value(String name) {
    return (T) limits.readField(name);
}
```

The reason for implementing limits by name is two-fold:

1. The `readField` approach avoids the problem of the underlying JNA structure being totally mutable.

2. It is assumed that applications will prefer to query by name rather than coding for specific structure fields.

Some device limits are a _quantised_ range of permissible values which can be retrieved by the following helper:

```java
public float[] range(String name, String granularity) {
    // Lookup range bounds
    float[] bounds = value(name);
    float min = bounds[0];
    float max = bounds[1];

    // Lookup granularity step
    float step = value(granularity);

    // Determine number of values
    int num = (int) ((max - min) / step);

    // Build quantised range
    float[] range = new float[num + 1];
    for(int n = 0; n < num; ++n) {
        range[n] = min + n * step;
    }
    range[num] = max;

    return range;
}
```

Where _name_ is the limit and _granularity_ specifies the step size, e.g. `pointSizeRange` and `pointSizeGranularity` for the range of valid point primitives.

The device limits are lazily retrieved from the logical device:

```java
public class LogicalDevice ... {
    ...
    private final Supplier<DeviceLimits> limits = new LazySupplier<>(this::loadLimits);

    private DeviceLimits loadLimits() {
        VkPhysicalDeviceProperties props = parent.properties();
        return new DeviceLimits(props.limits, features);
    }

    @Override
    public DeviceLimits limits() {
        return limits.get();
    }
}
```

For example, the builder for an indirect draw command can now validate that the command configuration is supported by the hardware:

```java
DeviceLimits limits = buffer.device().limits();
int max = limits.value("maxDrawIndirectCount");
limits.require("multiDrawIndirect");
if(count > max) throw new IllegalArgumentException(...);
```

Although the validation layer would also trap this problem when the command is _executed_ the above code applies the validation at _instantiation_ time (which may be earlier).

---

## Summary

In this chapter we rendered a galaxy model and introduced:

* The geometry shader.

* The colour blending pipeline stage.

* Push constants.

* Shader specialisation constants.

* Support for device limits.
