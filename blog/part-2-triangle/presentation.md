---
title: Presentation
---

---

## Contents

- [Overview](#overview)
- [Framework](#framework-enhancements)
- [Rendering Surface](#rendering-surface)
- [Swapchain](#swapchain)
- [Improvements](#improvements)

---

## Overview

The next couple of chapters will cover development of the various components required for _presentation_ of rendered frames to the display.

The first of these is the _swapchain_ which is the controller for the presentation process.  The swapchain comprises of a number of image _attachments_ that are the target of the rendering process.  Generally an application employs a double or triple-buffer strategy whereby a completed frame is presented to the screen while the next is being rendered, the buffers are swapped, and the process repeats for the next frame.  Additionally Vulkan is designed to allow these activities to be performed in parallel.

To configure presentation we take advantage of the Vulkan integration in GLFW.  The _desktop_ library is extended to create an application window that supports a Vulkan _rendering surface_.  This demo will also build upon the previous chapter to select a device that supports presentation to that surface.

The following new components are required:

* A GLFW window.

* The Vulkan _surface_ derived from the window.

* Selection of a device that supports presentation to the surface.

* The swapchain domain class itself including new types to support the image attachments.

First several new framework components are introduced that will be used in the new presentation functionality.

> Note that Vulkan extensions _could_ have been used to implement the surface from the ground up but it makes sense to take advantage of the platform-independant implementation offered by GLFW.  The disadvantage of this approach is that the logic becomes a little convoluted (as will be seen) since the surface, instance and physical device become slightly inter-dependant, but this seems an acceptable trade-off.

---

## Framework Enhancements

### Native Handles

Most of the Vulkan domain objects developed thus far have a _handle_ which is the JNA native pointer for that object.

There are a couple of issues with this approach:

* A JNA pointer is mutable which potentially breaks the class if the underlying pointer is exposed.

* Vulkan API methods are defined in terms of pointers which is not type-safe.

* Additional code is required to extract the handle from the domain object.

Therefore we introduce several new framework classes to resolve (or at least mitigate) these problems.

The `Handle` class is an immutable, opaque wrapper for a JNA pointer:

```java
public final class Handle {
    private final Pointer ptr;

    public Handle(Pointer ptr) {
        this.ptr = new Pointer(Pointer.nativeValue(ptr));
    }

    @Override
    public int hashCode() {
        return ptr.hashCode();
    }

    @Override
    public boolean equals(Object obj) {
        return
            (obj == this) ||
            (obj instanceof Handle that) &&
            this.ptr.equals(that.ptr);
    }

    @Override
    public String toString() {
        return ptr.toString();
    }
}
```

Domain objects that contain a handle are refactored to implement the following new abstraction:

```java
public interface NativeObject {
    Handle handle();
}
```

And a new JNA type converter is implemented to transform a native object to the underlying JNA pointer:

```java
TypeConverter CONVERTER = new TypeConverter() {
    @Override
    public Class<?> nativeType() {
        return Pointer.class;
    }

    @Override
    public Object toNative(Object value, ToNativeContext context) {
        if(value instanceof NativeObject obj) {
            return obj.handle().toPointer();
        }
        else {
            return null;
        }
    }

    @Override
    public Object fromNative(Object nativeValue, FromNativeContext context) {
        throw new UnsupportedOperationException();
    }
};
```

Registering this converter allows all domain objects to be used directly in API methods which is more type-safe, better documented, and requires less code.

Notes:

* The `fromNative` method is disallowed since domain objects will never be instantiated via a native method.

* A further type converter is registered for the `Handle` class which is more convenient for Vulkan structure fields and some API methods.

* The `toArray` helper (not shown) on the new interface converts a collection of objects to a native pointer-to-array type as a contiguous memory block.

* The GLFW library is refactored similarly.

### Transient Objects

The new abstraction is extended for domain objects that are managed by the application:

```java
public interface TransientNativeObject extends NativeObject {
    /**
     * Destroys this object.
     * @throws IllegalStateException if this object has already been destroyed
     */
    void destroy();

    /**
     * @return Whether this object has been destroyed
     */
    boolean isDestroyed();
}
```

The following template implementation encapsulates a handle and manages the process of releasing resources on destruction:

```java
public abstract class AbstractTransientNativeObject implements TransientNativeObject {
    private final Handle handle;
    private boolean destroyed;

    @Override
    public Handle handle() {
        return handle;
    }

    @Override
    public final boolean isDestroyed() {
        return destroyed;
    }

    @Override
    public void destroy() {
        if(destroyed) throw new IllegalStateException(...);
        release();
        destroyed = true;
    }

    /**
     * Releases this object.
     */
    protected abstract void release();
}
```

### Abstract Vulkan Object

We also note that the majority of the Vulkan domain objects are derived from the logical device and share the following properties:

* A handle.

* A reference to the logical device.

* A _destructor_ method, e.g. `vkDestroyRenderPass`

This seems a valid case for a further intermediate base-class that abstracts this common pattern:

```java
public abstract class AbstractVulkanObject extends AbstractTransientNativeObject {
    private final LogicalDevice dev;

    protected AbstractVulkanObject(Pointer handle, LogicalDevice dev) {
        super(new Handle(handle));
        this.dev = dev;
    }
}
```

To destroy this object the following abstraction for a _destructor_ is introduced:

```java
@FunctionalInterface
public interface Destructor<T extends AbstractVulkanObject> {
    /**
     * Destroys this object.
     * @param dev           Logical device
     * @param obj           Native object to destroy
     * @param allocator     Vulkan memory allocator (always {@code null})
     */
    void destroy(DeviceContext dev, T obj, Pointer allocator);
}
```

The new class provides the API method used to destroy the object:

```java
protected abstract Destructor<?> destructor(VulkanLibrary lib);
```

Which is used in the overridden `destroy` method:

```java
@Override
public void destroy() {
    // Destroy this object
    Destructor destructor = destructor(dev.library());
    destructor.destroy(dev, this, null);

    // Delegate
    super.destroy();
}
```

---

## Rendering Surface

### Application Window

Implementation of the presentation process starts with a new domain class for the application window:

```java
public class Window extends TransientNativeObject {
    private final Desktop desktop;

    Window(Handle window, Desktop desktop) {
        super(window);
        this.desktop = desktop;
    }

    @Override
    public void destroy() {
        lib.glfwDestroyWindow(handle);
    }
}
```

A companion builder configures the properties of the window:

```java
public static class Builder {
    private String title;
    private Dimensions size;
    private final Map<Hint, Integer> hints = new HashMap<>();
}
```

The visual properties of the window (_hints_ in GLFW parlance) are copied from the header and wrapped into the following enumeration:

```java
public enum Hint {
    RESIZABLE(0x00020003),
    DECORATED(0x00020005),
    AUTO_ICONIFY(0x00020006),
    MAXIMISED(0x00020008),
    CLIENT_API(0x00022001);

    private final int hint;
}
```

The `CLIENT_API` hint suppresses the automatic creation of an OpenGL rendering surface which is the default behaviour (the _hint_ argument is exposed for this reason).

In the `build` method the windows hints are applied _prior_ to creating the window:

```java
public Window build(Desktop desktop) {
    DesktopLibrary lib = desktop.library();
    lib.glfwDefaultWindowHints();
    for(var entry : hints.entrySet()) {
        Hint hint = entry.getKey();
        int arg = entry.getValue();
        lib.glfwWindowHint(hint, arg);
    }
    ...
}
```

Next the window itself is instantiated:

```java
Handle window = lib.glfwCreateWindow(size.width(), size.height(), descriptor.title(), null, null);
if(window == null) throw new RuntimeException(...);
```

And wrapped into the domain object:

```java
return new Window(window, desktop);
```

A handle to the Vulkan rendering surface can then be retrieved from the window via a new accessor:

```java
public Handle surface(Handle instance) {
    DesktopLibrary lib = desktop.library();
    var ref = new PointerByReference();
    int result = lib.glfwCreateWindowSurface(instance, this, null, ref);
    if(result != 0) throw new RuntimeException(...);
    return ref.get();
}
```

Finally the API methods are added to the GLFW library:

```java
Handle  glfwCreateWindow(int w, int h, String title, Handle monitor, Window shared);
void    glfwDestroyWindow(Window window);
void    glfwDefaultWindowHints();
void    glfwWindowHint(int hint, int value);
int     glfwCreateWindowSurface(Handle instance, Handle window, Handle allocator, PointerByReference surface);
```

Notes:

* This is a bare-bones implementation sufficient for the triangle demo.

* This implementation ignores display monitors for the moment.

In the demo application a native window can now be created that supports Vulkan presentation:

```java
Window window = new Window.Builder()
    .title("demo")
    .size(new Dimensions(1280, 760))
    .hint(Hint.CLIENT_API, 0)
    .build(desktop);
```

And the handle to the Vulkan rendering surface is retrieved:

```java
Handle surface = window.surface(instance.handle());
```

### Device Selection

The next step is to select the physical device that supports presentation to the rendering surface, handled by a second device selector implementation:

```java
public static Selector presentation(Handle surface) {
    return new Selector() {
        @Override
        protected boolean matches(PhysicalDevice device, Family family) {
            return device.isPresentationSupported(surface, family);
        }
    };
}
```

Which matches a queue family against the surface handle:

```java
public boolean isPresentationSupported(Handle surface, Family family) {
    var supported = new IntegerByReference();
    vulkan.vkGetPhysicalDeviceSurfaceSupportKHR(this, family.index(), surface, supported);
    return supported.get() == 1;
}
```

The new API method is added to the library for the physical device:

```java
VkResult vkGetPhysicalDeviceSurfaceSupportKHR(PhysicalDevice device, int queueFamilyIndex, Handle surface, IntegerByReference supported);
```

### Rendering Surface

Configuration of the swapchain is dependant on the capabilities of the graphics hardware such as supported image formats, presentation modes, etc. which are dependant on the selected physical device.  As mentioned above, taking advantage of the platform-independant `glfwCreateWindowSurface` method couples the rendering surface, the device _and_ implicitly the Vulkan instance.  Therefore a new component is introduced that composes these dependencies:

```java
public class Surface extends TransientNativeObject {
    private final PhysicalDevice dev;
    private final Instance instance;

    public Surface(Handle surface, PhysicalDevice dev) {
        super(surface);
        this.dev = dev;
        this.instance = dev.instance();
    }

    @Override
    protected void release() {
        VulkanLibrary lib = dev.library();
        lib.vkDestroySurfaceKHR(instance, this, null);
    }
}
```

The new class provides a number of accessors that are used to configure the swapchain.  The _surface capabilities_ specify minimum and maximum constraints on various aspects of the hardware, such as the number of frame buffers, the maximum dimensions of the image views, etc:

```java
public VkSurfaceCapabilitiesKHR capabilities() {
    VulkanLibrary lib = dev.library();
    var caps = new VkSurfaceCapabilitiesKHR();
    check(lib.vkGetPhysicalDeviceSurfaceCapabilitiesKHR(dev, this, caps));
    return caps;
}
```

The supported _image formats_ are retrieved using the two-stage approach:

```java
public Collection<VkSurfaceFormatKHR> formats() {
    VulkanLibrary lib = instance.library();
    StructureVulkanFunction<VkSurfaceFormatKHR> func = (count, array) -> lib.vkGetPhysicalDeviceSurfaceFormatsKHR(dev, this, count, array);
    IntByReference count = instance.factory().integer();
    var formats = func.invoke(count, VkSurfaceFormatKHR::new);
    return Arrays.asList(formats);
}
```

Finally the swapchain supports a number of available _presentation modes_ (at least one) which can be configured by the application:

```java
public Set<VkPresentModeKHR> modes() {
    VulkanLibrary lib = instance.library();
    VulkanFunction<int[]> func = (count, array) -> lib.vkGetPhysicalDeviceSurfacePresentModesKHR(dev, this, count, array);
    IntByReference count = instance.factory().integer();
    int[] array = func.invoke(count, int[]::new);
    ...
}
```

The presentation modes are returned as an integer array which is transformed to the corresponding enumeration:

```java
var mapping = new ReverseMapping<>(VkPresentModeKHR.class);
return Arrays
    .stream(array)
    .mapToObj(mapping::map)
    .collect(toSet());
```

A new JNA library is created for the various surface related API methods:

```java
public interface Library {
    int  vkGetPhysicalDeviceSurfaceCapabilitiesKHR(PhysicalDevice device, Surface surface, VkSurfaceCapabilitiesKHR caps);
    int  vkGetPhysicalDeviceSurfaceFormatsKHR(PhysicalDevice device, Surface surface, IntByReference count, VkSurfaceFormatKHR formats);
    int  vkGetPhysicalDeviceSurfacePresentModesKHR(PhysicalDevice device, Surface surface, IntByReference count, int[] modes);
    void vkDestroySurfaceKHR(Instance instance, Surface surface, Handle allocator);}
```

Note that the API is expressed in terms of domain objects rather than JNA pointers thanks to the framework introduced earlier.

### Integration

The new presentation functionality is integrated into the demo application to select an appropriate device and create a rendering surface that will be used by the swapchain in the next section.

A second selector is added to find a device that supports that supports the requirements of the triangle demo:

```java
Selector graphics = Selector.queue(VkQueueFlag.GRAPHICS);
Selector presentation = Selector.presentation(handle);
```

Which are used to select the device from those available:

```java
PhysicalDevice physical = PhysicalDevice
    .devices(instance)
    .filter(graphics)
    .filter(presentation)
    .findAny()
    .orElseThrow();
```

The queue families are then extracted from the selected device:

```java
Family graphicsFamily = graphics.select(physical);
Family presentationFamily = presentation.select(physical);
```

Note that the resultant families could actually refer to the same object depending on the hardware implementation.

Finally the rendering surface handle is wrapped by the new domain object:

```java
Surface surface = new Surface(handle, instance, physical);
```

In summary, the inter-dependencies between the various components when creating the rendering surface are as follows:

1. Check whether Vulkan is supported by the hardware: `glfwVulkanSupported`

2. Create the Vulkan instance and an application window (with OpenGL disabled).

3. Retrieve a handle to the surface from the window given the instance.

4. Select a physical device that supports presentation to the surface: `vkGetPhysicalDeviceSurfaceSupportKHR`

5. Create the surface domain object.

With the rendering surface in place we can move onto the swapchain itself.

---

## Swapchain

### Images

Vulkan creates the images for the colour attachments when the swapchain is instantiated:

```java
public class Image implements NativeObject {
    private final Handle handle;
    private final LogicalDevice dev;
    private final Descriptor descriptor;
}
```

The _image descriptor_ specifies the static properties of an image:

```java
record Descriptor(VkImageType type, VkFormat format, Extents extents, Set<VkImageAspect> aspects, int levels, int layers)
```

Where the _extents_ are the dimensions of the image:

```java
record Extents(Dimensions dimensions, int depth)
```

And `Dimensions` is a simple record for the size of an arbitrary 2D rectangle:

```java
public record Dimensions(int width, int height)
```

A convenience builder is also implemented to construct an image descriptor.

### Image Views

An _image view_ is the entry-point for operations on an image such as layout transforms, sampling, etc:

```java
public class View extends AbstractVulkanObject {
    private final Image image;
    
    @Override
    protected Destructor<View> destructor(VulkanLibrary lib) {
        return lib::vkDestroyImageView;
    }
}
```

Notes:

* The swapchain images are created and managed by Vulkan and are not explicitly destroyed by the application.

* However the application is responsible for creating (and destroying) the image views.

* The view class initially has no functionality but will be expanded later, in particular when addressing clearing attachments.

An image view is constructed by a builder as usual:

```java
public static class Builder {
    private final Image image;
    private VkImageViewType type;
    private VkComponentMapping mapping = DEFAULT_COMPONENT_MAPPING;
}
```

The `build` method populates a Vulkan descriptor for the view and invokes the API method:

```java
public View build() {
    // Build view descriptor
    var info = new VkImageViewCreateInfo();
    info.viewType = type;
    info.format = image.descriptor().format();
    info.image = image.handle();
    info.components = mapping;
    info.subresourceRange = ...

    // Allocate image view
    LogicalDevice dev = image.device();
    VulkanLibrary lib = dev.library();
    PointerByReference handle = dev.factory().pointer();
    check(lib.vkCreateImageView(dev, info, null, handle));

    // Create image view
    return new View(handle.getValue(), dev, image);
}
```

The `subresourceRange` field of the create descriptor specifies a subset of the image aspects, mipmap levels, etc. accessible to the view.  For this demo the whole image will be used so the descriptor is hard-coded for the moment:

```java
var range = new VkImageSubresourceRange();
range.aspectMask = new BitMask<>(image.descriptor().aspects());
range.baseMipLevel = 0;
range.levelCount = 1;
range.baseArrayLayer = 0;
range.layerCount = 1;
info.subresourceRange = range;
```

The component mapping specifies the swizzle for the RGBA colour components of the view:

```java
private static final VkComponentMapping DEFAULT_COMPONENT_MAPPING = identity();

private static VkComponentMapping identity() {
    var swizzle = VkComponentSwizzle.IDENTITY;
    var identity = new VkComponentMapping();
    identity.r = swizzle;
    identity.g = swizzle;
    identity.b = swizzle;
    identity.a = swizzle;
    return identity;
}
```

Finally a new JNA library is added for image views:

```java
interface Library {
    int  vkCreateImageView(LogicalDevice device, VkImageViewCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference pView);
    void vkDestroyImageView(LogicalDevice device, View imageView, Pointer pAllocator);
}
```

### Swapchain

With the above in place the swapchain itself can now be implemented:

```java
public class Swapchain extends AbstractVulkanObject {
    private final VkFormat format;
    private final Dimensions extents;
    private final List<View> views;

    @Override
    protected Destructor<Swapchain> destructor(VulkanLibrary lib) {
        return lib::vkDestroySwapchainKHR;
    }

    @Override
    protected void release() {
        for(View view : views) {
            view.destroy();
        }
    }
}
```

The swapchain is another highly configurable domain object created via a builder:

```java
public static class Builder {
    private final LogicalDevice dev;
    private final VkSwapchainCreateInfoKHR info = new VkSwapchainCreateInfoKHR();
    private final Surface surface;
    private final VkSurfaceCapabilitiesKHR caps;
}
```

The surface capabilities are queried from the surface in the constructor:

```java
public Builder(LogicalDevice dev, Surface surface) {
    this.dev = dev;
    this.surface = surface;
    this.caps = surface.capabilities();
    init();
}
```

The constructor also initialises the swapchain properties to sensible defaults via the setters based on the capabilities of the surface:

```java
private void init() {
    extent(caps.currentExtent.width, caps.currentExtent.height);
    count(caps.minImageCount);
    transform(caps.currentTransform);
    format(DEFAULT_FORMAT);
    space(DEFAULT_COLOUR_SPACE);
    arrays(1);
    mode(VkSharingMode.EXCLUSIVE);
    usage(VkImageUsage.COLOR_ATTACHMENT);
    alpha(VkCompositeAlphaFlagKHR.OPAQUE);
    mode(DEFAULT_PRESENTATION_MODE);
    clipped(true);
}
```

Constructing the swapchain comprises three steps:

1. Create the swapchain.

2. Retrieve the image handles.

3. Create a view for each image.

Instantiating the swapchain is relatively trivial:

```java
public Swapchain build() {
    VulkanLibrary lib = dev.library();
    ReferenceFactory factory = dev.factory();
    PointerByReference chain = factory.pointer();
    check(lib.vkCreateSwapchainKHR(dev, info, null, chain));
    ...
}
```

Next the handles to the swapchain images are retrieved:

```java
VulkanFunction<Pointer[]> func = (count, array) -> lib.vkGetSwapchainImagesKHR(dev, chain.getValue(), count, array);
IntByReference count = factory.integer();
Pointer[] handles = func.invoke(count, Pointer[]::new);
```

The images all share the same descriptor:

```java
Dimensions extents = new Dimensions(info.imageExtent.width, info.imageExtent.height);
Descriptor descriptor = new Descriptor.Builder()
    .format(info.imageFormat)
    .extents(extents)
    .aspect(VkImageAspect.COLOR)
    .build();
```

Which is used when creating the view for each image:

```java
List<View> views = Arrays
    .stream(handles)
    .map(Handle::new)
    .map(handle -> new Image(handle, dev, descriptor))
    .map(image -> new View.Builder(image).build())
    .toList();
```

Finally the swapchain domain object is instantiated:

```java
return new Swapchain(chain.getValue(), dev, info.imageFormat, extents, views);
```

A new JNA library is added for the swapchain:

```java
interface Library {
    int         vkCreateSwapchainKHR(LogicalDevice device, VkSwapchainCreateInfoKHR pCreateInfo, Pointer pAllocator, PointerByReference pSwapchain);
    void        vkDestroySwapchainKHR(LogicalDevice device, Swapchain swapchain, Pointer pAllocator);
    int         vkGetSwapchainImagesKHR(LogicalDevice device, Swapchain swapchain, IntByReference pSwapchainImageCount, Pointer[] pSwapchainImages);
    VkResult    vkAcquireNextImageKHR(LogicalDevice device, Swapchain swapchain, long timeout, Semaphore semaphore, Fence fence, IntByReference pImageIndex);
    VkResult    vkQueuePresentKHR(Queue queue, VkPresentInfoKHR pPresentInfo);
}
```

### Presentation

Presentation is comprised of a pair of new methods on the swapchain.

The index of the next swapchain image to be rendered is retrieved as follows:

```java
public int acquire(Semaphore semaphore, Fence fence) throws SwapchainInvalidated {
    DeviceContext dev = super.device();
    VulkanLibrary lib = dev.library();
    IntByReference index = dev.factory().integer();
    VkResult result = lib.vkAcquireNextImageKHR(dev, this, Long.MAX_VALUE, semaphore, fence, index);
    ...
}
```

The _semaphore_ and _fence_ are synchronisation primitives that are covered in a later chapter, for the moment these values are `null` in the demo.

Acquiring the swapchain image is one of the few API methods that can return _multiple_ success codes (see [vkAcquireNextImageKHR](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkAcquireNextImageKHR.html)).  Therefore in this case the API method returns a `VkResult` and the usual `check` test is replaced by the following:

```java
return switch(result) {
    case SUCCESS, SUBOPTIMAL_KHR -> index.getValue();
    case ERROR_OUT_OF_DATE_KHR -> throw new SwapchainInvalidated(result);
    default -> throw new VulkanException(result);
};
```

The `SwapchainInvalidated` exception indicates that the window has been resized or minimised.  Note that a _suboptimal_ return value is considered as valid in this case.

When an image has been rendered it can then be presented to the surface, which requires population of a Vulkan descriptor for the presentation task:

```java
public void present(Queue queue, int index, Semaphore semaphore) {
    // Create presentation descriptor
    var info = new VkPresentInfoKHR();

    // Populate swap-chain
    info.swapchainCount = 1;
    info.pSwapchains = NativeObject.toArray(List.of(this));
    
    ...
}
```

The image(s) to be presented are stored as a contiguous memory block which maps to a native pointer-to-integer-array:

```java
int[] array = new int[]{index.getValue()};
Memory mem = new Memory(array.length * Integer.BYTES);
mem.write(0, array, 0, array.length);
info.pImageIndices = mem;
```

Finally the presentation task is added to the relevant work queue:

```java
VulkanLibrary lib = dev.library();
VkResult result = lib.vkQueuePresentKHR(queue, info);
switch(result) {
    case ERROR_OUT_OF_DATE_KHR, SUBOPTIMAL_KHR -> throw new SwapchainInvalidated(result);
    default -> check(result.value());
}
```

Notes:

* The API supports presentation of multiple swapchains in one operation but for the moment the code is restricted to a single instance.

* Similarly the `pImageIndices` array consists of a single image index.

* The presentation task descriptor is created on every invocation of the `present` method which will probably be cached later.

* In this case a sub-optimal swapchain is __not__ considered valid.

---

## Improvements

### Format Builder

When developing the swapchain we realised that finding an image `VkFormat` can be difficult given the size of the enumeration.  However the naming convention is highly consistent and it is therefore possible to specify the format _name_ programatically and lookup the relevant enumeration constant.

The following helper class allows an application to specify the various components of the required format:

```java
public class FormatBuilder {
    public static final String RGBA = "RGBA";

    private String components = RGBA;
    private int count = 4;
    private int bytes = 4;
    private Type type = Type.FLOAT;
    private boolean signed = true;
}
```

The `Type` field is an enumeration of the Vulkan component types:

```java
public enum Type {
    INT,
    FLOAT,
    NORM,
    SCALED,
    RGB
}
```

The builder constructs the format name and looks up the resultant enumeration constant:

```java
public VkFormat build() {
    // Build component layout
    var layout = new StringBuilder();
    int size = bytes * Byte.SIZE;
    for(int n = 0; n < count; ++n) {
        layout.append(template.charAt(n));
        layout.append(size);
    }

    // Build format string
    char ch = signed ? 'S' : 'U';
    String format = String.format("%s_%c%s", layout, ch, type.name());

    // Lookup format
    return VkFormat.valueOf(format);
}
```

The demo can now be refactored to build the image format rather than having to find it manually in the enumeration:

```java
VkFormat format = new FormatBuilder()
    .template("BGRA")
    .bytes(1)
    .signed(true)
    .type(FormatBuilder.Type.RGB)
    .build();
```

Which maps to the `B8G8R8A8_SRGB` format used by the colour attachment.

### Vulkan Booleans

Development of the swapchain was the first time that a boolean value (whether the swapchain images are clipped) was used in a JNA based library.  We encountered a curious problem that stumped us for some time.  See [this](https://stackoverflow.com/questions/55225896/jna-maps-java-boolean-to-1-integer) stack-overflow question.

In summary: a Vulkan boolean is represented as zero (for false) or one (for true) - so far so logical.
But by default JNA maps a Java boolean to zero for false but __minus one__ for true!  WTF!

There are a lot of boolean values used across Vulkan requiring a global solution to over-ride the default JNA mapping (which is unfortunately hard coded).
Again a custom JNA type converter comes to the rescue:

```java
public final class NativeBooleanConverter implements TypeConverter {
    private static final int TRUE = 1;
    private static final int FALSE = 0;

    @Override
    public Class<?> nativeType() {
        return Integer.class;
    }
}
```

A Java boolean is marshalled from a native integer where a non-zero value is true:

```java
@Override
public Boolean fromNative(Object nativeValue, FromNativeContext context) {
    if(nativeValue instanceof Integer n) {
        return n == TRUE;
    }
    else {
        return false;
    }
}
```

But explicitly marshals a boolean as either one or zero:

```java
@Override
public Integer toNative(Object value, ToNativeContext context) {
    if(value instanceof Boolean b) {
        return b ? TRUE : FALSE;
    }
    else {
        return FALSE;
    }
}
```

This type converter is registered with the JNA library solving the mapping problem for API methods and structures, and also has the side-benefit of being more type-safe and self-documenting.

> As it turns out the JNA `W32APITypeMapper` helper class probably already solves this issue.

---

## Summary

This chapter covered the implementation of various components to support presentation:

* The GLFW window.

* The Vulkan rendering surface and device selection.

* Implementation of the swapchain and image/views.

* A number of framework improvements to abstract common patterns.

