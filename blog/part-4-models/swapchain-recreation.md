---
title: Swapchain Recreation
---

---

# Contents

- [Introduction](#introduction)
- [Refactor](#refactor)
- [Recreation](#recreation)
- [Integration](#integration)
- [Summary](#summary)

---

TODO - minised -> pause render loop here?

# Introduction

The swapchain can become _invalidated_ when it no longer supports the rendering surface, generally when the window has been resized or minimised.  In this situation the properties of the surface must be refreshed and the swapchain recreated accordingly.

Note that swapchain recreation also encompasses rebuilding the attachment views and framebuffers.

This requirement implies the following:

* Refactoring of the swapchain to test whether the swapchain has become invalid.

* A new exception thrown by these methods such that the swapchain can be recreated.

* Some sort of 'holder' approach for the various components that can be recreated.

Most Vulkan device drivers _should_ generate errors when the swapchain becomes invalid, but this is not guaranteed.  It is recommended to also explicitly check for changes to the window (i.e. using GLFW event callbacks), therefore event handlers will also be implemented to explicitly handle resized and minimised windows.

Additionally the current implementation of the swapchain and rendering surface has several problems that will be addressed as part of this refactoring work:

1. The swapchain has several properties that are dependant on the rendering surface, e.g. the presentation mode must be selected from those supported by the hardware.  Some of these properties are currently hard-coded in the demo applications and will be replaced by a new configuration mechanism that ties in with swapchain recreation.

2. Similarly the swapchain class and its companion builder also have complex selection and state validation logic, making these classes brittle to change and difficult to test, e.g. configuration of the number of swapchain images.  Again this logic should be factored out.

3. Due to the fact that we have chosen to use the Vulkan integration in GLFW to instantiate the rendering surface (rather than using Vulkan extensions from the ground up), the surface class is inherently dependant on the Vulkan instance, physical device and the GLFW window.  However the existing code is overly complex and will be refactored to simplify configuration of the swapchain.

---

# Refactor

## Surface

First the surface class is simplified as follows:

* The GLFW window becomes an explicit dependency of the surface.

* The surface handle can now be encapsulated into the revised class (rather than being another responsibility of the user).

* The various methods supporting configuration of the swapchain are removed (to be replaced later).

The resultant class is now considerably simpler:

```java
public class VulkanSurface extends TransientNativeObject {
    private final Window window;
    private final Instance instance;
    private final Library library;

    public VulkanSurface(Window window, Instance instance, Library library) {
        Handle handle = window.surface(instance.handle());
        super(handle);
        ...
    }

    protected void release() {
        library.vkDestroySurfaceKHR(instance, this, null);
    }
}
```

The method that tests whether a given device-family supports presentation becomes a simple query:

```java
public boolean isPresentationSupported(PhysicalDevice device, Family family) {
    var supported = new IntegerReference();
    library.vkGetPhysicalDeviceSurfaceSupportKHR(device, family.index(), this, supported);
    return NativeBooleanTransformer.isTrue(supported.get());
}
```

The _provider_ for the surface properties is created _once_ when the physical device has been selected:

```java
public Supplier<Properties> properties(PhysicalDevice device) {
    return () -> new PropertiesInstance();
}
```

Where `PropertiesInstance` is the existing local implementation of the properties interface.

This results in a much simpler and more cohesive implementation of the rendering surface.

## Swapchain

A new exception is introduced indicating that the swapchain has been invalidated:

```java
public class Swapchain extends VulkanObject {
    private static final ReverseMapping<VkResult> MAPPING = ReverseMapping.mapping(VkResult.class);

    public static final class Invalidated extends VulkanException {
    }
}
```

The `acquire` and `present` of the swapchain return _multiple_ success codes, indicating whether the swapchain has become invalid.  Since all Vulkan API methods that return `VkResult` are checked against `SUCCESS` in the library proxy, the swapchain API is refactored to return the underlying _integer_ success code, circumventing the usual validation logic:

```java
public interface Library {
    int vkAcquireNextImageKHR(LogicalDevice device, Swapchain swapchain, ...);
    int vkQueuePresentKHR(WorkQueue queue, VkPresentInfoKHR pPresentInfo);
}
```

In the `acquire` method the returned code is programatically mapped to the result enumeration:

```java
var index = new IntegerReference();
int code = library.vkAcquireNextImageKHR(this.device(), this, Long.MAX_VALUE, semaphore, fence, index);
VkResult result = MAPPING.map(code);
```

And the different cases can now be handled appropriately, throwing the new exception if the swapchain has been invalidated:

```java
return switch(result) {
    case SUCCESS, SUBOPTIMAL_KHR -> index.get();
    case ERROR_OUT_OF_DATE_KHR -> throw new Invalidated(result);
    default -> throw new VulkanException(result);
}
```

The `present` method is refactored similarly:

```java
public static void present(Library library, WorkQueue queue, VkPresentInfoKHR info) throws Invalidated {
    int code = library.vkQueuePresentKHR(queue, info);
    VkResult result = MAPPING.map(code);
    if(result != VkResult.SUCCESS) {
        switch(result) {
            case ERROR_OUT_OF_DATE_KHR, SUBOPTIMAL_KHR -> throw new Invalidated(result);
            default -> throw new VulkanException(result);
        }
    }
}
```

## Swapchain Builder

The swapchain builder is simplified by removing the dependency on the surface and stripping out any selection logic:

```java
public static class Builder {
    private final VkSwapchainCreateInfoKHR info = new VkSwapchainCreateInfoKHR();
    private final Set<VkSwapchainCreateFlagKHR> flags = new HashSet<>();
    private final Set<VkImageUsageFlag> usage = new HashSet<>();
}
```

The swapchain properties are initialised to sensible defaults _once_ in the constructor:

```java
public Builder() {
    info.imageFormat = VkFormat.B8G8R8A8_UNORM;
    info.imageColorSpace = VkColorSpaceKHR.SRGB_NONLINEAR_KHR;
    info.preTransform = new EnumMask<>(VkSurfaceTransformFlagsKHR.IDENTITY_KHR);
    info.imageArrayLayers = 1;
    info.compositeAlpha = new EnumMask<>(VkCompositeAlphaFlagsKHR.OPAQUE_KHR);
    info.imageSharingMode = VkSharingMode.EXCLUSIVE;
    info.presentMode = DEFAULT_PRESENTATION_MODE;
    info.clipped = true;
}
```

This ensures that all swapchain properties are initialised to valid values _before_ any configuration is applied.  The application can then override the properties as required whenever the swapchain is recreated, i.e. the builder is initialised _once_ and reused.

---

# Recreation

## Swapchain Manager

The final piece of the jigsaw is a new 'holder' component that is responsible for creating the swapchain on demand:

```java
public class SwapchainManager implements TransientObject {
    private final LogicalDevice device;
    private final Properties properties;
    private final Builder builder;
    private final Collection<SwapchainConfiguration> configuration;
    private Swapchain swapchain;
}
```

The logic to configure the various swapchain properties is factored out to a new abstraction:

```java
public interface SwapchainConfiguration {
    /**
     * Configures a swapchain property.
     * @param builder       Swapchain builder
     * @param properties    Surface properties
     */
    void configure(Builder builder, Properties properties);
}
```

The swapchain is first instantiated in the constructor:

```java
public SwapchainManager(...) {
    ...
    this.swapchain = build();
}
```

And can be recreated on demand using the same mechanism:

```java
public Swapchain recreate() {
    device.waitIdle();
    release();
    swapchain = build();
    return swapchain;
}
```

Note that `recreate` blocks until all pending rendering work has completed.

The `build` method first applies the configuration and then generates a new swapchain via the builder:

```java
private Swapchain build() {
    for(var c : configuration) {
        c.configure(builder, properties);
    }

    return builder.build(device, properties);
}
```

This separates the concerns of the various collaborators that contribute to configuration of the swapchain:

* The process of recreating the swapchain is factored out to the new manager.

* The swapchain and builder are now considerably simpler.

* Logic to configure the swapchain is factored out to cohesive and testable components (detailed in the next section).

## Swapchain Configuration

### Presentation Mode

The presentation mode is selected from a prioritised list of candidates, filtered by those supported by the surface, falling back to a default value as appropriate.  The selected mode is then applied to the swapchain builder:

```java
public class PresentationModeSwapchainConfiguration implements SwapchainConfiguration {
    private final List<VkPresentModeKHR> modes;

    public void configure(Swapchain.Builder builder, Properties properties) {
        List<VkPresentModeKHR> available = properties.modes();
        ...
        builder.presentation(mode);
    }
}
```

Since there are several use-cases that repeat this pattern of selecting from a list with a fallback option, a new generic utility is introduced:

```java
public class PrioritySelector<T> {
    private final Predicate<T> filter;
    private final Function<List<T>, T> fallback;

    public PrioritySelector(Predicate<T> filter, Function<List<T>, T> fallback) {
        ...
    }

    public PrioritySelector(Predicate<T> filter, T fallback) {
        this(filter, _ -> fallback);
    }
}
```

The selection logic is then implemented as follows:

```java
public T select(List<T> candidates) {
    return candidates
        .stream()
        .filter(filter)
        .findAny()
        .or(() -> Optional.ofNullable(fallback.apply(candidates)))
        .orElseThrow();
}
```

The presentation mode can now be selected and applied using the new utility:

```java
public void configure(Swapchain.Builder builder, Properties properties) {
    List<VkPresentModeKHR> available = properties.modes();
    var selector = new PrioritySelector<>(available::contains, Swapchain.DEFAULT_PRESENTATION_MODE);
    VkPresentModeKHR mode = selector.select(modes);
    builder.presentation(mode);
}
```

Note that if none of the candidates are available the swapchain falls back to the default `FIFO_KHR` mode guaranteed on all platforms.

### Surface Format

Selecting the surface format of the swapchain also uses the new selector utility:

```java
public class SurfaceFormatSwapchainConfiguration implements SwapchainConfiguration {
    private final SurfaceFormatWrapper format;

    public void configure(Builder builder, Properties properties) {
        List<VkSurfaceFormatKHR> formats = properties.formats();
        var selector = new PrioritySelector<VkSurfaceFormatKHR>(format::equals, first());
        VkSurfaceFormatKHR selected = selector.select(formats);
        builder.format(selected);
    }
}
```

Where the following convenience wrapper compares the underlying structure by equality:

```java
public static class SurfaceFormatWrapper extends VkSurfaceFormatKHR {
    public SurfaceFormatWrapper(VkFormat format, VkColorSpaceKHR space) {
        this.format = format;
        this.colorSpace = space;
    }

    public boolean equals(Object obj) {
        return
            (obj == this) ||
            (obj instanceof VkSurfaceFormatKHR that) &&
            (this.format == that.format) &&
            (this.colorSpace == that.colorSpace);
    }
}
```

And `first` is a helper that selects the first candidate as the fallback:

```java
public static <T> Function<List<T>, T> first() {
    return List::getFirst;
}
```

### Image Count

The number of swapchain images (i.e. colour attachments) is configured by a _policy_ applied to the surface capabilities:

```java
public record ImageCountSwapchainConfiguration(ToIntFunction<VkSurfaceCapabilitiesKHR> policy) implements SwapchainConfiguration {
    public void configure(Builder builder, Properties properties) {
        int count = policy.applyAsInt(properties.capabilities());
        builder.count(count);
    }
}
```

The following common policies are provided built-in:

```java
public enum Policy implements ToIntFunction<VkSurfaceCapabilitiesKHR> {
    MIN,
    MAX;

    public int applyAsInt(VkSurfaceCapabilitiesKHR capabilities) {
        return switch(this) {
            case MIN -> capabilities.minImageCount;
            case MAX -> capabilities.maxImageCount;
        };
    }
}
```

### Sharing Mode

The sharing mode of the swapchain is `EXCLUSIVE` if a single queue family is used for both rendering and presentation, or `CONCURRENT` if swapchain images are shared across multiple queues.  The default is `EXCLUSIVE` which does not entail the overhead of ownership transfers.

The following configuration overrides the default behaviour as required:

```java
public class SharingModeSwapchainConfiguration implements SwapchainConfiguration {
    private final Set<Family> families;

    public void configure(Swapchain.Builder builder, Properties properties) {
        if(families.size() != 1) {
            builder.concurrent(families);
        }
    }
}
```

This delegates to a new helper on the swapchain builder that overrides the sharing mode to `CONCURRENT` and populates the queue families (which are otherwise unused):

```java
public Builder concurrent(Collection<Family> families) {
    info.imageSharingMode = VkSharingMode.CONCURRENT;
    info.queueFamilyIndexCount = families.size();
    info.pQueueFamilyIndices = families.stream().mapToInt(Family::index).toArray();
    return this;
}
```

### Swapchain Extents

The extents of the swapchain are initialised to the `currentExtent` of the surface capabilities, which is generally exactly the same dimensions as the window.

However there is a special case where the window manager allows (or forces) the application to set the extents, indicated by the maximum value.    In this case the application should query the window dimensions and set the extents accordingly.

This is further complicated by the fact that Vulkan only works with extents specified in _pixels_ whereas GLFW uses both pixels and _screen coordinates_, which are not necessarily the same (e.g. for high DPI monitors).

The following configuration logic overrides the basic behaviour of the swapchain builder to correctly handle these additional cases:

```java
public class ExtentSwapchainConfiguration implements SwapchainConfiguration {
    public void configure(Builder builder, Properties properties) {
    }
}
```

The `configure` method first handles the normal case:

```java
var capabilities = properties.capabilities();
VkExtent2D current = capabilities.currentExtent;
if(current.width < Integer.MAX_VALUE) {
    builder.extent(current);
}
else {
    ...
}
```

Otherwise the extents are derived from the dimensions of the window.

A new enumeration is added to the GLFW window class to support both measurement units:

```java
public enum Unit {
    PIXEL,
    SCREEN_COORDINATE
}
```

And the `size` accessor is modified by the addition of the `glfwGetFramebufferSize` API method for the `PIXEL` case:

```java
public Dimensions size(Unit unit) {
    var w = new IntegerReference();
    var h = new IntegerReference();
    switch(unit) {
        case PIXEL              -> library.glfwGetFramebufferSize(this, w, h);
        case SCREEN_COORDINATE  -> library.glfwGetWindowSize(this, w, h);
    }
    return new Dimensions(w.get(), h.get());
}
```

Finally the window dimensions are clamped to the min/max extents supported by the surface:

```java
Dimensions size = properties.surface().window().size(PIXEL);
var min = capabilities.minImageExtent;
var max = capabilities.maxImageExtent;
int w = Math.clamp(size.width(),  min.width,  max.width);
int h = Math.clamp(size.height(), min.height, max.height);
builder.extent(new Dimensions(w, h));
```

## Attachment Views

After the swapchain has been recreated the image views for all the attachments need to be rebuilt.

A `recreate` method is added to the definition of an attachment:

```java
public interface Attachment {
    /**
     * Recreates the image-view(s) of this attachment when the swapchain has become invalid.
     * @param device        Logical device
     * @param extents       Swapchain extents
     */
    void recreate(LogicalDevice device, Dimensions extents);
}
```

The template implementation first deletes the existing image views:

```java
public final void recreate(LogicalDevice device, Dimensions extents) {
    if(views != null) {
        release();
    }

    this.views = views(device, extents);
}
```

And then delegates to a new abstract method to rebuild them, which is the following for the colour attachment:

```java
protected List<View> views(LogicalDevice device, Dimensions extents) {
    return swapchain
        .attachments()
        .stream()
        .map(image -> View.of(device, image))
        .toList();
}
```

## Framebuffers

Lastly the framebuffers are recreated given the newly rebuilt image views of the attachments.

Another new component is introduced to manage the set of framebuffers:

```java
public class Framebuffer extends VulkanObject {
    public static class Factory implements TransientObject {
        private final RenderPass pass;
        private final List<Framebuffer> framebuffers = new ArrayList<>();
    
        public Framebuffer framebuffer(int index) {
            return framebuffers.get(index);
        }
    }
}
```

The framebuffers are deleted and recreated on demand:

```java
public void build(Swapchain swapchain) {
    destroy();
    create(swapchain);
}
```

A framebuffer is created for each swapchain image:

```java
private void create(Swapchain swapchain) {
    int count = swapchain.attachments();
    Dimensions extents = swapchain.extents();
    for(int n = 0; n < count; ++n) {
        List<View> views = views(n);
        Framebuffer buffer = create(views, extents);
        framebuffers.add(buffer);
    }
}
```

Where the image view(s) for each attachment in the render pass are aggregated by the following helper:

```java
private List<View> views(int index) {
    return pass
        .attachments()
        .stream()
        .map(attachment -> attachment.view(index))
        .toList();
}
```

Note that the `view` method of a colour attachment returns the swapchain image for the given framebuffer index, whereas the depth-stencil attachment returns a _single_ view which is shared across frames (the index is irrelevant).

---

# Integration

A JOVE application can now encapsulate the surface handle with a proper domain type:

```java
var surface = new VulkanSurface(window, instance, vulkan);
```

And selection of a physical device that supports presentation is now simpler:

```java
Selector selector = Selector.of(surface::isPresentationSupported);
```

The provider for the surface properties is created _once_ when the physical device has been selected:

```java
var properties = surface.properties(physical);
```

The basic swapchain properties are initialised _once_ via the builder:

```java
var builder = new Swapchain.Builder().clipped(true);
```

With the more complex configuration specified separately as appropriate for a given application:

```java
SwapchainConfiguration[] configuration = {
    new ImageCountSwapchainConfiguration(Policy.MIN),
    new SurfaceFormatSwapchainConfiguration(new SurfaceFormatWrapper(VkFormat.B8G8R8A8_UNORM, VkColorSpaceKHR.SRGB_NONLINEAR_KHR)),
    new PresentationModeSwapchainConfiguration(List.of(VkPresentModeKHR.MAILBOX_KHR)),
    new SharingModeSwapchainConfiguration(List.of(graphicsFamily, presentationFamily, ...)),
    new ExtentSwapchainConfiguration(),
    ...
};
```

The new manager composes the swapchain configuration and the surface properties:

```java
var manager = new SwapchainManager(device, properties, builder, List.of(configuration));
```

And finally the render task checks for an invalidated swapchain:

```java
public class RenderTask implements Runnable, TransientObject {
    private final SwapchainManager manager;
    private final Framebuffer.Factory factory;
    ...

    @Override
    public void run() {
        try {
            render();
        }
        catch(Invalidated e) {
            recreate();
        }
    }
}
```

Recreating the swapchain, attachment views, and framebuffers as required:

```java
public void recreate() {
    // Recreate swapchain
    Swapchain swapchain = manager.recreate();

    // Recreate attachment image-views
    Dimensions extents = swapchain.extents();
    LogicalDevice device = swapchain.device();
    for(Attachment attachment : framebuffers.pass().attachments()) {
        attachment.recreate(device, extents);
    }

    // Recreate framebuffers
    framebuffers.recreate(swapchain);
}
```

---

# Summary

In this chapter the Vulkan surface, swapchain, and builder were refactored to both simplify the code and to support recreation of the swapchain.  In particular the logic to select various configuration properties was factored out to a suite of helper classes.

New components and supporting framework were implemented to support recreation of the swapchain, attachment views, and framebuffers when the swapchain is invalidated (indicated by a new custom exception).

