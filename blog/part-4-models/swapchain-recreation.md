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

# Introduction

The `acquire` and `present` methods of the swapchain return _multiple_ result codes if it has become invalid, generally when the window has been resized or minimised.  These methods will be refactored to determine the appropriate response.

A swapchain that has become invalid will be indicated by a new exception, which can then be trapped to recreate the swapchain (and framebuffers) as required.

However the current implementation has several problems that should also be addressed as part of this change:

1. The swapchain has several properties that are dependant on the rendering surface, several of which entail logic to select the appropriate configuration.  This logic is currently spread across the surface, the swapchain, and its companion builder, which is messy and difficult to test.

2. The rendering surface is a plain `Handle` until it is composed with the physical device.  However this handle is not used elsewhere and should be encapsulated into the surface component.

3. The surface _properties_ are also dependant on the physical device making the existing class overly complex and hard to test.

The swapchain and surface will therefore be (hopefully) simplified and the logic to configure and recreate the swapchain will be factored out to new discrete components.

---

# Refactor

## Surface

First the surface class is refactored:

* The handle is encapsulated into the class and derived from the window explicitly.

* The various selection methods are removed (to be replaced with a cleaner mechanism below).

The resultant class is now simpler and makes the dependencies explicit:

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

Next the various surface properties are aggregated into a new abstraction:

```java
public interface Properties {
    VkSurfaceCapabilitiesKHR capabilities();
    List<VkSurfaceFormatKHR> formats();
    List<VkPresentModeKHR> modes();
}
```

With an internal implementation:

```java
private class LocalProperties implements Properties{
    private final PhysicalDevice device;

    public VkSurfaceCapabilitiesKHR capabilities() {
        var capabilities = new VkSurfaceCapabilitiesKHR();
        library.vkGetPhysicalDeviceSurfaceCapabilitiesKHR(device, VulkanSurface.this, capabilities);
        return capabilities;
    }

    public List<VkSurfaceFormatKHR> formats() {
        VulkanFunction<VkSurfaceFormatKHR[]> formats = (count, array) -> library.vkGetPhysicalDeviceSurfaceFormatsKHR(device, VulkanSurface.this, count, array);
        VkSurfaceFormatKHR[] array = VulkanFunction.invoke(formats, VkSurfaceFormatKHR[]::new);
        return List.of(array);
    }

    public List<VkPresentModeKHR> modes() {
        VulkanFunction<VkPresentModeKHR[]> function = (count, array) -> library.vkGetPhysicalDeviceSurfacePresentModesKHR(device, VulkanSurface.this, count, array);
        VkPresentModeKHR[] modes = VulkanFunction.invoke(function, VkPresentModeKHR[]::new);
        return List.of(modes);
    }
}
```

To minimise the number of API calls the properties are cached:

```java
public Properties properties(PhysicalDevice device) {
    return new LocalProperties(device) {
        private final VkSurfaceCapabilitiesKHR capabilities = super.capabilities();
        private final List<VkSurfaceFormatKHR> formats = super.formats();
        private final List<VkPresentModeKHR> modes = super.modes();
        ...
    };
}
```

This approach results in a much simpler and more cohesive implementation, and provides an abstraction that can more easily be unit-tested.

## Swapchain

A new exception is introduced indicating that the swapchain has been invalidated:

```java
public class Swapchain extends VulkanObject {
    private static final ReverseMapping<VkResult> MAPPING = ReverseMapping.mapping(VkResult.class);

    public static final class Invalidated extends VulkanException {
    }
}
```

The swapchain API methods can return _multiple_ success codes, indicating whether the swapchain has become invalid.  Since all Vulkan API methods that return `VkResult` are checked against `SUCCESS` in the library proxy, the swapchain API is refactored to return the underlying _integer_ success code, circumventing the usual validation logic:

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
    private ColourClearValue clear;

    public Builder() {
        init();
        usage(VkImageUsageFlag.COLOR_ATTACHMENT);
    }
}
```

The swapchain properties are initialised to sensible defaults _once_ by the `init` method:

```java
private void init() {
    info.imageFormat = VkFormat.UNDEFINED;
    info.preTransform = VkSurfaceTransformFlagKHR.IDENTITY_KHR;
    info.imageArrayLayers = 1;
    info.compositeAlpha = VkCompositeAlphaFlagKHR.OPAQUE;
    info.imageSharingMode = VkSharingMode.EXCLUSIVE;
    info.presentMode = DEFAULT_PRESENTATION_MODE;
    info.clipped = true;
}
```

And properties dependant on the surface can be initialised as required by a new method:

```java
public Builder init(VkSurfaceCapabilitiesKHR capabilities) {
    info.minImageCount = capabilities.minImageCount;
    info.preTransform = capabilities.currentTransform;
    info.imageExtent = capabilities.currentExtent;
    return this;
}
```

This ensures that all swapchain properties are initialised to valid values _before_ any configuration is applied.  The application can then override the properties as required whenever the swapchain is recreated, i.e. the builder can be initialised _once_ and reused.

---

# Recreation

## Swapchain Manager

The final piece of the jigsaw is a new component that is responsible for creating the swapchain on demand:

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
public void recreate() {
    swapchain.destroy();
    swapchain = build();
}
```

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

* the process of recreating the swapchain is factored out to the new manager.

* the swapchain and builder are now considerably simpler (and easier to test).

* the logic to configure the swapchain is factored out to cohesive and testable components (see below).

## Swapchain Configuration

The following sections cover implementation of the various swapchain configuration use cases:

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

The second constructor specifies a literal fallback value (ignoring the candidate list).

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

Note that if none of the candidates are available the swapchain falls back to the default `FIFO_KHR` mode which is guaranteed on all platforms.

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

As a bonus the selector utility can also be reused to choose the format for the depth-stencil attachment in the model demo:

```java
var filter = new FormatFilter(device::properties, true, Set.of(VkFormatFeatureFlags.DEPTH_STENCIL_ATTACHMENT));
var selector = new PrioritySelector<>(filter, first());
VkFormat format = selector.select(List.of(VkFormat.D32_SFLOAT, VkFormat.D32_SFLOAT_S8_UINT, VkFormat.D24_UNORM_S8_UINT));
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

## Framebuffers

The framebuffers must also be recreated when the swapchain becomes invalid, therefore another new component (equivalent to the swapchain manager) is introduced to manage a set of framebuffers:

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

The framebuffers are recreated on demand:

```java
public void build(Swapchain swapchain) {
	destroy();
	create(swapchain);
}
```

Delegating to the following code to rebuild a framebuffer for each colour attachment of the swapchain:

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

The view(s) for each attachment are aggregated by the following helper:

```java
private List<View> views(int index) {
	return pass
		.attachments()
		.stream()
		.map(attachment -> attachment.view(index))
		.toList();
}
```

Note that the `view` method of a colour attachment returns the swapchain image for the given framebuffer index, whereas the depth-stencil attachment returns a _single_ view shared across frames and the index is irrelevant.

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

The surface properties can be queried once a physical device has been selected:

```java
var properties = surface.properties(physical);
```

The basic swapchain properties are initialised _once_ in the code:

```java
var builder = new Swapchain.Builder()
    .clipped(true)
    .init(properties.capabilities());
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
		catch(Swapchain.Invalidated e) {
			recreate();
		}
	}
}
```

Recreating the swapchain and framebuffers as required:

```java
private void recreate() {
	// Wait for pending rendering tasks
	LogicalDevice device = manager.swapchain().device();
	device.waitIdle();

	// Recreate the swapchain
	Swapchain swapchain = manager.recreate();

	// Rebuild the framebuffers
	factory.build(swapchain);
}
```

Note that the task blocks until all pending rendering work has completed before recreating.

---

# Summary

In this chapter the Vulkan surface, swapchain, and builder were refactored to both simplify the code and to support recreation of the swapchain.

In particular the logic to select various configuration properties was factored out to a suite of helper classes.

The new `SwapchainManager` recreates the swapchain when it is invalidated (indicated by a new custom exception), which is usually handled in the rendering task.

