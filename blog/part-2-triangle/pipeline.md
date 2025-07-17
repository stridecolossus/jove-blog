---
title: The Graphics Pipeline
---

---

## Contents

- [Overview](#overview)
- [Pipeline](#building-the-pipeline)
- [Shaders](#shaders)
- [Integration](#integration)

---

## Overview

The _graphics pipeline_ configures the various _stages_ executed by the hardware to perform rendering.

A pipeline stage is either a configurable _fixed function_ or a _programmable stage_ implemented by a _shader module_.

In this chapter we will implement the mandatory fixed-function and shader stages required for the triangle demo.

---

## Building the Pipeline

### Pipeline

The pipeline domain class itself is relatively trivial:

```java
public class Pipeline extends AbstractVulkanObject {
    private final PipelineLayout layout;

    @Override
    protected Destructor<Pipeline> destructor(VulkanLibrary lib) {
        return lib::vkDestroyPipeline;
    }
}
```

The _pipeline layout_ specifies the _resources_ (texture samplers, uniform buffers and push constants) used by the pipeline.  None of these are needed for the triangle demo but we are still required to specify a layout for the pipeline.

A bare-bones implementation is created to satisfy this stage of development:

```java
public class PipelineLayout extends AbstractVulkanObject {
    @Override
    protected Destructor<PipelineLayout> destructor(VulkanLibrary lib) {
        return lib::vkDestroyPipelineLayout;
    }
}
```

With a companion builder that is largely empty for the moment:

```java
public PipelineLayout build() {
    var info = new VkPipelineLayoutCreateInfo();
    VulkanLibrary lib = dev.library();
    PointerByReference layout = dev.factory().pointer();
    check(lib.vkCreatePipelineLayout(dev, info, null, layout));
    return new PipelineLayout(layout.getValue(), dev);
}
```

### Builder

In terms of the number of data structures required, configuration of the pipeline is probably the largest aspect of creating a Vulkan application.  Even for the simple case of the triangle rendering demo there are around a dozen descriptors that must be configured, although the bulk of the data is default or empty values.

Our goals for the pipeline are:

1. Abstract over the configuration of the various pipeline stages to minimise the amount of boiler-plate required for a given application.

2. Factor out each component to discrete classes to maximise maintainability and testability.

3. Provide a relatively fluid API for the developer.

The starting point is an outline builder for the pipeline itself:

```java
public static class Builder {
    private PipelineLayout layout;
    private RenderPass pass;
}
```

Next nested builders are added for the fixed-function pipeline stages:

```java
private final VertexInputStage input = new VertexInputStage();
private final InputAssemblyStage assembly = new InputAssemblyStage();
private final ViewportStage viewport = new ViewportStage();
private final RasterizerStage raster = new RasterizerStage();
private final ColourBlendStage blend = new ColourBlendStage();
```

And a set of programmable pipeline stages for the shaders:

```java
private final Map<VkShaderStage, ProgrammableShaderStage> shaders = new HashMap<>();
```

If the number of complexity of the pipeline stages was relatively small they could be implemented as inner classes of the main pipeline builder.  This is clearly not viable in this case, instead each pipeline stages is factored out to a separate source file with accessors to configure each stage, for example:

```java
public VertexInputStage input() {
    return input;
}
```

To avoid making this chapter overly long only the implementation of the mandatory pipeline stages needed for the triangle demo are detailed, with the other stages introduced as they become relevant in future chapters.  Note that there are also several optional pipeline stages (e.g. the depth-stencil buffer) that are also glossed over for the moment.

### Viewport

The one fixed-function that __must__ always be configured is the _viewport pipeline stage_ which defines the drawing regions of the rasterizer:

```java
public class ViewportStageBuilder extends AbstractPipelineStageBuilder<VkPipelineViewportStateCreateInfo> {
    private final List<Viewport> viewports = new ArrayList<>();
    private final List<Rectangle> scissors = new ArrayList<>();
}
```

A `Viewport` is a local type that composes a viewport rectangle and near/far rendering depths:

```java
public record Viewport(Rectangle rect, Percentile min, Percentile max) {
    void populate(VkViewport info) {
        info.x = rectangle.x();
        info.y = rectangle.y();
        info.width = rectangle.width();
        info.height = rectangle.height();
        info.minDepth = min.floatValue();
        info.maxDepth = max.floatValue();
    }
}
```

Where `Rectangle` is a trivial record type:

```java
public record Rectangle(int x, int y, int width, int height)
```

The following helper is added for the common case of a single viewport and scissor rectangle with the same dimensions:

```java
public ViewportStage viewportAndScissor(Viewport viewport) {
    viewports.add(viewport);
    scissor(viewport.rectangle);
    return this;
}
```

Each viewport generates a separate descriptor:

```java
void populate(VkViewport viewport) {
    viewport.x = rectangle.x();
    viewport.y = rectangle.y();
    viewport.width = rectangle.width();
    viewport.height = rectangle.height();
    viewport.minDepth = min.value();
    viewport.maxDepth = max.value();
    return viewport;
}
```

And finally the builder creates the overall descriptor for the viewport stage:

```java
VkPipelineViewportStateCreateInfo descriptor() {
    // Validate
    int count = viewports.size();
    if(count == 0) throw new IllegalArgumentException(...);
    if(scissors.size() != count) throw new IllegalArgumentException(...);

    // Add viewports
    var info = new VkPipelineViewportStateCreateInfo();
    info.viewportCount = count;
    info.pViewports = StructureCollector.pointer(viewports, new VkViewport(), Viewport::populate);

    // Add scissors
    info.scissorCount = count;
    info.pScissors = StructureCollector.pointer(scissors, new VkRect2D.ByReference(), VulkanHelper::populate);

    return info;
}
```

Note there are separate fields for the number of viewports and scissors but they __must__ have the same length.

---

## Shaders

### Shader Module

The other component that must be implemented to render the triangle is the programmable pipeline stage to support the _vertex_ and _fragment_ shaders:

```java
public class Shader extends AbstractVulkanObject {
    @Override
    protected Destructor<Shader> destructor(VulkanLibrary lib) {
        return lib::vkDestroyShaderModule;
    }
}
```

Shaders are created via a factory method that first converts the SPIV code to a byte-buffer:

```java
public static Shader create(LogicalDevice dev, byte[] code) {
    ByteBuffer bb = ByteBuffer
        .allocateDirect(code.length)
        .order(ByteOrder.nativeOrder())
        .put(code)
        .flip();
}
```

As usual the shader is created by populating a Vulkan descriptor, invoking the API, and creating the domain object:

```java
// Create descriptor
var info = new VkShaderModuleCreateInfo();
info.codeSize = code.length;
info.pCode = bb;

// Allocate shader
VulkanLibrary lib = dev.library();
PointerByReference shader = dev.factory().pointer();
check(lib.vkCreateShaderModule(dev, info, null, shader));

// Create shader
return new Shader(shader.getValue(), dev);
```

The API for shader modules consists of two methods:

```java
interface Library {
    int  vkCreateShaderModule(LogicalDevice device, VkShaderModuleCreateInfo info, Pointer pAllocator, PointerByReference shader);
    void vkDestroyShaderModule(LogicalDevice device, Shader shader, Pointer pAllocator);
}
```

The following helper is also added to load shader code from the file system:

```java
public static Shader load(LogicalDevice dev, InputStream in) throws IOException {
    byte[] code = in.readAllBytes();
    return create(dev, code);
}
```

A shader stage is configured by the following new type:

```java
public record ProgrammableShaderStage(VkShaderStage stage, Shader shader, String name)
```

With a companion builder:

```java
public static class Builder {
    private VkShaderStage stage;
    private Shader shader;
    private String name = "main";
}
```

The descriptor for a shader stage is populated as follows:

```java
void populate(VkPipelineShaderStageCreateInfo info) {
    info.stage = stage;
    info.module = shader.handle();
    info.pName = name;
}
```

Finally shader stages can be attached to the parent pipeline builder:

```java
public Builder shader(ProgrammableShaderStage shader) {
    VkShaderStage stage = shader.stage();
    if(shaders.containsKey(stage)) throw new IllegalArgumentException();
    shaders.put(stage, shader);
    return this;
}
```

### Conclusion

The parent builder can now be completed by first constructing the descriptor for the pipeline itself:

```java
public Pipeline build(LogicalDevice dev) {
    // Create descriptor
    var pipeline = new VkGraphicsPipelineCreateInfo();

    // Init layout
    if(layout == null) throw new IllegalArgumentException(...);
    pipeline.layout = layout.handle();

    // Init render pass
    if(pass == null) throw new IllegalArgumentException(...);
    pipeline.renderPass = pass.handle();
    ...
}
```

Next the fixed-function stages are retrieved from the various sub-builders:

```java
pipeline.pVertexInputState = input.build();
pipeline.pInputAssemblyState = assembly.build();
pipeline.pViewportState = viewport.build();
pipeline.pRasterizationState = raster.build();
pipeline.pDepthStencilState = depth.build();
pipeline.pColorBlendState = blend.build();
```

And similarly for the programmable stages:

```java
if(!shaders.containsKey(VkShaderStage.VERTEX)) throw new IllegalStateException(...);
pipeline.stageCount = shaders.size();
pipeline.pStages = StructureCollector.pointer(shaders.values(), new VkPipelineShaderStageCreateInfo(), ProgrammableShaderStage::populate);
```

Finally the pipeline is instantiated:

```java
VulkanLibrary lib = dev.library();
Pointer[] pipelines = new Pointer[1];
check(lib.vkCreateGraphicsPipelines(dev, null, 1, new VkGraphicsPipelineCreateInfo[]{pipeline}, null, pipelines));
return new Pipeline(pipelines[0], dev, layout);
```

The pipeline API looks like this:

```java
interface Library {
    int  vkCreatePipelineLayout(LogicalDevice device, VkPipelineLayoutCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference pPipelineLayout);
    void vkDestroyPipelineLayout(LogicalDevice device, PipelineLayout pipelineLayout, Pointer pAllocator);
    int  vkCreateGraphicsPipelines(LogicalDevice device, Pointer pipelineCache, int createInfoCount, VkGraphicsPipelineCreateInfo[] pCreateInfos, Pointer pAllocator, Pointer[] pPipelines);
    void vkDestroyPipeline(LogicalDevice device, Pipeline pipeline, Pointer pAllocator);
}
```

Note that Vulkan supports creation of multiple pipelines in one operation but the code is restricted to a single instance for the moment.

---

## Integration

### Vertex Shader

The vertex shader hard-codes the triangle vertices and passes the colour of each vertex through to the fragment shader:

```glsl
#version 450

layout(location = 0) out vec4 fragColour;

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(-0.5, 0.5),
    vec2(0.5, 0.5)
);

vec3 colours[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColour = vec4(colours[gl_VertexIndex], 1.0);
}
```

Notes:

* The built-in `gl_VertexIndex` variable is used to index into the two arrays.

* All shader code is copied as-is from the tutorial.

The colour for each vertex is simply passed through to the rasterizer by the fragment shader:

```glsl
#version 450

layout(location = 0) in vec4 fragColour;

layout(location = 0) out vec4 outColour;

void main() {
    outColour = fragColour;
}
```

The shaders are compiled off-line from GLSL to SPIV code using the `GLSLC` utility application (provided as part of the Vulkan SDK):

```
cd JOVE/src/test/resources/demo/triangle
glslc triangle.frag -o spv.triangle.frag
glslc triangle.vert -o spv.triangle.vert
```

### Configuration

A new configuration class is implemented to load and configure the two shaders:

```java
@Configuration
class PipelineConfiguration {
    @AutoWired private LogicalDevice dev;

    @Bean
    public Shader vertex() throws IOException {
        return Shader.load(dev, new FileInputStream("./src/main/resources/spv.triangle.vert"));
    }

    @Bean
    public Shader fragment() throws IOException {
        return Shader.load(dev, new FileInputStream("./src/main/resources/spv.triangle.frag"));
    }
}
```

Finally the rendering pipeline is configured using the various builders implemented in this chapter:

```java
@Bean
public Pipeline pipeline(RenderPass pass, Swapchain swapchain, Shader vertex, Shader fragment, PipelineLayout layout) {
    // Init pipeline
    var pipeline = new Pipeline.Builder()
        .layout(layout)
        .pass(pass);

    // Configure fixed functions
    pipeline.viewportAndScissor(swapchain.extents());
    ...
    
    // Configure shaders
    pipeline.shader(new ProgrammableShaderStage(VkShaderStage.VERTEX, vertex));
    pipeline.shader(new ProgrammableShaderStage(VkShaderStage.FRAGMENT, fragment));
    
    // Create pipeline
    return pipeline.build(dev);
}
```

Where the pipeline layout is empty for the moment:

```java
@Bean
PipelineLayout pipelineLayout() {
    return new PipelineLayout.Builder(dev).build();
}
```

---

## Summary

In this chapter we implemented a fluid interface comprising nested builders to configure the graphics pipeline and shader modules.

