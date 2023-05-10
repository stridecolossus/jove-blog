---
title: The Render Pass
---

---

## Contents

- [Overview](#overview)
- [Render Pass](#render-pass)
- [Dependency Injection](#dependency-injection)

---

## Overview

In this chapter we will develop the remaining components required to complete the presentation functionality started in the previous chapter.

A _render pass_ is comprised of the following:

* A set of _attachments_ that define the structure and format of the frame buffer images.

* One-or-more _sub passes_ that operate sequentially on the attachments.

* A number of _dependencies_ that specify how the sub-passes are linked to allow the hardware to synchronise the process.

A render pass with multiple sub-passes is generally used to implement post-processing effects.
Subpass dependencies are deferred to a later chapter as they are not needed for this stage of development.

A new framework library will also be introduced to address the convoluted structure of the demo application.

---

## Render Pass

### Attachments

An attachment is essentially a wrapper for the underlying Vulkan structure:

```java
public record Attachment(VkFormat format, VkSampleCount samples, LoadStore attachment, LoadStore stencil, VkImageLayout initialLayout, VkImageLayout finalLayout) {
    /**
     * Convenience wrapper for load-store operations.
     */
    public record LoadStore(VkAttachmentLoadOp load, VkAttachmentStoreOp store) {
    }
}
```

Where _attachment_ and _stencil_ are the load-store operations for the colour and depth-stencil attachments respectively.

When creating the render pass the contiguous array of attachments is populated by the following method:

```java
void populate(VkAttachmentDescription attachment) {
    attachment.format = format;
    attachment.samples = samples;
    attachment.loadOp = attachment.load;
    attachment.storeOp = attachment.store;
    attachment.stencilLoadOp = stencil.load;
    attachment.stencilStoreOp = stencil.store;
    attachment.initialLayout = initialLayout;
    attachment.finalLayout = finalLayout;
}
```

A convenience builder is implemented that initialises the attachment properties to sensible defaults:

```java
public static class Builder {
    private VkFormat format;
    private VkSampleCount samples = VkSampleCount.COUNT_1;
    private LoadStore attachment = DONT_CARE;
    private LoadStore stencil = DONT_CARE;
    private VkImageLayout initialLayout = VkImageLayout.UNDEFINED;
    private VkImageLayout finalLayout;
}
```

Where `DONT_CARE` is a convenience load-store constant:

```java
private static final LoadStore DONT_CARE = new LoadStore(VkAttachmentLoadOp.DONT_CARE, VkAttachmentStoreOp.DONT_CARE);
```

Finally a convenience factory method is added for the common case of a colour attachment for presentation:

```java
public static Attachment colour(VkFormat format) {
    return new Builder(format)
        .attachment(new LoadStore(VkAttachmentLoadOp.CLEAR, VkAttachmentStoreOp.STORE))
        .finalLayout(VkImageLayout.PRESENT_SRC_KHR)
        .build();
}
```

### Subpass

A _subpass_ is a mutable type comprising the attachments used in that stage of rendering:

```java
public class Subpass {
    private final List<Reference> colour = new ArrayList<>();
    private Reference depth;
}
```

A colour attachment can be added to the subpass:

```java
public Subpass colour(Reference ref) {
    this.colour.add(ref);
    return this;
}
```

And similarly for the single depth-stencil attachment:

```java
public Subpass depth(Reference ref) {
    this.depth = notNull(ref);
    return this;
}
```

Where a `Reference` is a wrapper for a referenced attachment used in the subpass:

```java
public static class Reference {
    private final Attachment attachment;
    private final VkImageLayout layout;
    private Integer index;
}
```

The descriptor for each attachment reference is populated thus:

```java
void populate(VkAttachmentReference ref) {
    ref.attachment = index;
    ref.layout = layout;
}
```

The purpose of the `index` field is detailed below.

Finally the descriptor for the subpass is populated as follows:

```java
void populate(VkSubpassDescription descriptor) {
    // Init descriptor
    descriptor.pipelineBindPoint = VkPipelineBindPoint.GRAPHICS;

    // Populate colour attachments
    descriptor.colorAttachmentCount = colour.size();
    descriptor.pColorAttachments = StructureCollector.pointer(colour, new VkAttachmentReference(), Reference::populate);

    // Populate depth attachment
    if(depth != null) {
        var ref = new VkAttachmentReference();
        depth.populate(ref);
        descriptor.pDepthStencilAttachment = ref;
    }
}
```

### Render Pass

The domain class for a render pass contains the overall set of attachments used by its sub-passes:

```java
public class RenderPass extends AbstractVulkanObject {
    private final List<Attachment> attachments;

    @Override
    protected Destructor<RenderPass> destructor(VulkanLibrary lib) {
        return lib::vkDestroyRenderPass;
    }
}
```

We prefer to specify the render pass by an object graph of subpass and attachments, whereas the underlying Vulkan descriptors use array indices to refer to attachments (and later on subpass dependencies).  However all of these are transient objects that have no relevance once the render pass has been instantiated.  Additionally the application does not care about the attachment indices, unlike (for example) vertex attributes which are dependant on the shader layout.

Therefore the `create` factory method first enumerates the overall set of attachment references used in the render pass:

```java
public static RenderPass create(DeviceContext dev, List<Subpass> subpasses) {
    List<Reference> references = subpasses
        .stream()
        .flatMap(Subpass::attachments)
        .toList();
}
```

Next the unique set of attachments is aggregated from the references:

```java
List<Attachment> attachments = references
    .stream()
    .map(Reference::attachment)
    .distinct()
    .toList();
```

Which is then used to patch the `index` of each reference before the various Vulkan descriptors are populated:

```java
for(Reference ref : references) {
    int index = attachments.indexOf(ref.attachment());
    ref.init(index);
}
```

This approach shields the application from having to be concerned about attachment and subpass dependency indices.

The resultant attachments and subpasses are added to the create descriptor for the render pass:

```java
// Init render pass descriptor
var info = new VkRenderPassCreateInfo();
info.flags = 0;         // Reserved

// Add attachments
info.attachmentCount = attachments.size();
info.pAttachments = StructureCollector.pointer(attachments, new VkAttachmentDescription(), Attachment::populate);

// Add sub-passes
info.subpassCount = subpasses.size();
info.pSubpasses = StructureCollector.pointer(subpasses, new VkSubpassDescription(), Subpass::populate);
```

And finally the render pass is instantiated:

```java
VulkanLibrary lib = dev.library();
PointerByReference pass = dev.factory().pointer();
check(lib.vkCreateRenderPass(dev, info, null, pass));
return new RenderPass(pass.getValue(), dev, attachments);
```

The API for the render pass is simple:

```java
interface Library {
    int  vkCreateRenderPass(LogicalDevice device, VkRenderPassCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference pRenderPass);
    void vkDestroyRenderPass(LogicalDevice device, RenderPass renderPass, Pointer pAllocator);
}
```

### Frame Buffers

The final component required for rendering is the _frame buffer_ which composes the attachments used in the render pass:

```java
public class FrameBuffer extends AbstractVulkanObject {
    private final RenderPass pass;
    private final List<View> attachments;
    private final Dimensions extents;

    @Override
    protected Destructor<FrameBuffer> destructor(VulkanLibrary lib) {
        return lib::vkDestroyFramebuffer;
    }
}
```

A frame buffer is created using the following factory that as usual first populates a Vulkan descriptor:

```java
public static FrameBuffer create(RenderPass pass, Dimensions extents, List<View> attachments) {
    var info = new VkFramebufferCreateInfo();
    info.renderPass = pass.handle();
    info.attachmentCount = attachments.size();
    info.pAttachments = Handle.toArray(attachments);
    info.width = extents.width();
    info.height = extents.height();
    info.layers = 1;
    ...
}
```

The frame buffer is then created via the API and wrapped by a new domain object instance:

```java
DeviceContext dev = pass.device();
VulkanLibrary lib = dev.library();
PointerByReference buffer = dev.factory().pointer();
check(lib.vkCreateFramebuffer(dev, info, null, buffer));
return new FrameBuffer(buffer.getValue(), dev, pass, attachments, extents);
```

Frame buffer functionality will be developed further when rendering is addressed in the next chapter.

The API for frame buffers is also simple:

```java
interface Library {
    int  vkCreateFramebuffer(LogicalDevice device, VkFramebufferCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference pFramebuffer);
    void vkDestroyFramebuffer(LogicalDevice device, FrameBuffer framebuffer, Pointer pAllocator);
}
```

### Integration

We now have all the components required to implement a double-buffered swapchain and render pass for the demo application.

```java
Swapchain swapchain = new Swapchain.Builder(dev, surface)
    .count(2)
    .build();
```

The render pass consists of a single colour attachment with BGRA pixel components:

```java
VkFormat format = new FormatBuilder()
    .components(FormatBuilder.BGRA)
    .bytes(1)
    .signed(false)
    .type(FormatBuilder.Type.NORM)
    .build();
```

Which is cleared before rendering and transitioned to a layout ready for presentation:

```java
Attachment attachment = new Attachment.Builder()
    .format(format)
    .load(VkAttachmentLoadOp.CLEAR)
    .store(VkAttachmentStoreOp.STORE)
    .finalLayout(VkImageLayout.PRESENT_SRC_KHR)
    .build();
```

The render pass for the triangle demo consists of a single sub-pass to render the colour attachment:

```java
return new RenderPass.Builder()
    .subpass()
        .colour(colour, VkImageLayout.COLOR_ATTACHMENT_OPTIMAL)
        .build()
    .build();
```

And finally a frame buffer is created:

```java
View view = swapchain.views().get(0);
FrameBuffer fb = FrameBuffer.create(pass, swapchain.extents(), view);
```

Although the swapchain supports multiple colour attachments the demo will use a single frame-buffer until a proper rendering loop is implemented later.

---

## Dependency Injection

### Background

Although we are perhaps half-way to our goal of rendering a triangle it is already apparent that the demo is becoming unwieldy:

* The main application code is one large, imperative method that is difficult to navigate.

* The nature of a Vulkan application means that the various collaborating components are inherently highly inter-dependant, we are forced to structure the logic based on the inter-dependencies resulting in convoluted and brittle code.

* All components are created and managed in a single source file rather than being factoring out to discrete, coherent classes.

* The maintenance situation will get worse as more code is added to the demo.

The obvious solution is to use _dependency injection_ freeing development to focus on each component in relative isolation.
For this we will use [Spring Boot](https://spring.io/projects/spring-boot) which is one of the most popular and best supported dependency injection frameworks.

Note that only the demo applications will be dependant on Spring and not the JOVE library itself.

### Project

The [Spring Initializr](https://start.spring.io/) is used to generate a POM and main class for a new Spring-based project:

```java
@SpringBootApplication
public class TriangleDemo {
    public static void main(String[] args) {
        SpringApplication.run(TriangleDemo.class, args);
    }
}
```

This should run the new application though it obviously doesn't do anything yet.

Spring Boot is essentially a _container_ for the components that comprise the application, and is responsible for handling the lifecycle of each component and _auto-wiring_ the dependencies at instantiation time.

When the application is started Spring performs a _component scan_ of the project to identify the _beans_ to be managed by the container, which by default starts at the package containing the main class.

We start by factoring out the various desktop components into a Spring _configuration_ class:

```java
@Configuration
class DesktopConfiguration {
    @Bean
    public static Desktop desktop() {
        Desktop desktop = Desktop.create();
        if(!desktop.isVulkanSupported()) throw new RuntimeException("Vulkan not supported");
        return desktop;
    }

    @Bean
    public static Window window(Desktop desktop) {
        return new Window.Builder()
            .title("TriangleDemo")
            .size(new Dimensions(1024, 768))
            .property(Window.Hint.DISABLE_OPENGL)
            .build(desktop);
    }

    @Bean
    public static Handle surface(Instance instance, Window window) {
        return window.surface(instance.handle());
    }
}
```

The `@Configuration` annotation denotes a class containing a number of `@Bean` methods that create components to be managed by the container.

Here we can see the benefits of using dependency injection:

* The code to instantiate each component is now factored out into neater, more concise factory methods.

* The dependencies of each component are _declared_ in the signature of each bean method and the container _injects_ the relevant arguments for us (or throws an error if a dependency cannot be resolved).

* Components are instantiated by the container in logical order inferred from the dependencies, e.g. for the troublesome surface handle.

* Note that Spring beans are generally singleton instances.

This should result in code that is both simpler to develop and (more importantly) considerably easier to maintain.  In particular we no longer need to be concerned about inter-dependencies when components are added or refactored.

However one disadvantage of this approach is that the swapchain cannot be easily recreated when it is invalidated, e.g. when the window is minimised or resized.  This functionality is deferred to a future chapter.

### Integration

The configuration class for the Vulkan library and instance is relatively trivial:

```java
@Configuration
class VulkanConfiguration {
    @Bean
    public static VulkanLibrary library() {
        return VulkanLibrary.create();
    }

    @Bean
    public static Instance instance(VulkanLibrary lib, Desktop desktop, @Value("${application.title}") String title) {
        // Create instance
        Instance instance = new Instance.Builder()
            .name(title)
            .extension(VulkanLibrary.EXTENSION_DEBUG_UTILS)
            .extensions(desktop.extensions())
            .layer(ValidationLayer.STANDARD_VALIDATION)
            .build(lib);
        
        // Attach diagnostics handler
        ...
            
        return instance;
    }
}
```

The `@Value` annotation retrieves a configuration property which is usually specified in the `application.properties` file:

```java
application.title: Triangle Demo
```

We refactor the window title in the desktop configuration class similarly.

The next configuration class factors out the code to instantiate the physical and logical devices:

```java
@Configuration
class DeviceConfiguration {
    private final Selector graphics = Selector.of(VkQueueFlag.GRAPHICS);
    private final Selector presentation;

    public DeviceConfiguration(Handle surface) {
        presentation = Selector.of(surface);
    }
}
```

In this case _constructor injection_ is used to inject the surface handle for the `presentation` queue selector.

The bean methods for the devices are straight-forward:

```java
@Bean
public PhysicalDevice physical(Instance instance) {
    return PhysicalDevice
        .devices(instance)
        .filter(graphics)
        .filter(presentation)
        .findAny()
        .orElseThrow(() -> new RuntimeException());
}

@Bean
public LogicalDevice device(PhysicalDevice dev) {
    return new LogicalDevice.Builder(dev)
        .extension(VulkanLibrary.EXTENSION_SWAP_CHAIN)
        .layer(ValidationLayer.STANDARD_VALIDATION)
        .queue(graphics.select(dev))
        .queue(presentation.select(dev))
        .build();
}
```

Finally the resultant work queues are exposed:

```java
@Bean
public Queue graphics(LogicalDevice dev) {
    return dev.queue(graphics.select(dev.parent()));
}

@Bean
public Queue presentation(LogicalDevice dev) {
    return dev.queue(presentation.select(dev.parent()));
}
```

The swapchain, render pass and frame buffers are refactored similarly.

### Cleanup

Spring offers another bonus when cleaning up the various Vulkan components on application termination.  The following bean processor releases all native JOVE objects:

```java
@Bean
static DestructionAwareBeanPostProcessor destroyer() {
    return new DestructionAwareBeanPostProcessor() {
        @Override
        public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {
            if(bean instanceof TransientNativeObject obj && !obj.isDestroyed()) {
                obj.destroy();
            }
        }
    };
}
```

Alternatively the container invokes _inferred_ public destructor methods named `close` or `shutdown` on registered beans, though we prefer the more explicit approach.

Finally the container also ensures that components are destroyed in the correct reverse order (inferred from the dependencies) removing another responsibility from the developer.

Nice.

---

## Summary

In this chapter the final components required for presentation were implemented and Spring Boot was introduced to manage the various inter-dependant Vulkan components.

