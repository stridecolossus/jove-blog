---
title: The Vulkan Instance
---

---

## Contents

- [Overview](#overview)
- [Instance](#vulkan-instance)
- [Diagnostics Handlers](#diagnostics-handler)
- [Testing Issues](#testing-issues)

---

## Overview

The first step in the development of JOVE is to create the Vulkan _instance_ which the root object for everything that follows.

Creating the instance involves the following steps:

1. Instantiate the Vulkan API using JNA.

2. Specify the API extensions required for the application.

3. Populate a JNA structure descriptor to configure the requirements of the instance.

4. Invoke the API to create the instance given this descriptor.

5. Create the domain object encapsulating the new instance.

This will require the following components:

* Implementation of a JNA library to create/destroy the instance.

* An instance domain object.

* The relevant code generated structures used to specify the requirements of the instance.

As already mentioned in the [code generation](/JOVE/blog/part-1-intro/code-generation) chapter the GLFW library will be used in future chapters to manage windows, input devices, etc. and the tutorial also uses GLFW.  However another compelling reason to use this library is that it integrates neatly with Vulkan, in particular providing a platform-independant mechanism for determining the required extensions for the instance.

Therefore to create the instance for a given platform we also require:

* A second JNA library for GLFW.

* Additional domain objects for extensions and validation layers.

The diagnostics extension to support logging and error reporting will also be implemented which will become _very_ helpful in subsequent chapters.

Finally we will address some issues around unit-testing Vulkan and JNA based code.

---

## Vulkan Instance

### Vulkan Library

We start by defining a JNA library for management of the Vulkan instance:

```java
interface VulkanLibrary extends Library {
    /**
     * Creates a vulkan instance.
     * @param info             Instance descriptor
     * @param allocator        Allocator
     * @param instance         Returned instance
     * @return Result
     */
    int vkCreateInstance(VkInstanceCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference pInstance);

    /**
     * Destroys the vulkan instance.
     * @param instance         Instance handle
     * @param allocator        Allocator
     */
    void vkDestroyInstance(Pointer instance, Pointer pAllocator);
}
```

This JNA interface maps to the following methods defined in the `vulkan_core.h` header:

```java
typedef VkResult (VKAPI_PTR *PFN_vkCreateInstance)(const VkInstanceCreateInfo* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkInstance* pInstance);
typedef void (VKAPI_PTR *PFN_vkDestroyInstance)(VkInstance instance, const VkAllocationCallbacks* pAllocator);
```

Notes:

* As detailed in the previous chapter we have intentionally decided to hand-craft the native methods rather than attempting to code-generate the API.

* Only the API methods actually needed at this stage are declared in the JNA library.

* We have intentionally retained the Hungarian notation used in Vulkan parameter names.

* The handle to the newly created instance is returned as a JNA `PointerByReference` object which maps to a native `VkInstance*` return-by-reference type.

* The `pAllocator` parameter in the API methods is out-of-scope for JOVE and is always set to `null`.

To instantiate the API itself the following factory method is added to the library:

```java 
static VulkanLibrary create() {
    String name = switch(Platform.getOSType()) {
        case Platform.WINDOWS -> "vulkan-1";
        case Platform.LINUX -> "libvulkan";
        default -> throw new UnsupportedOperationException();
    }

    return Native.load(name, VulkanLibrary.class);
}
```

### Instance

With the library in place we can implement our first domain object:

```java
public class Instance {
    private final VulkanLibrary lib;
    private final Pointer handle;

    private Instance(VulkanLibrary lib, Pointer handle) {
        this.lib = notNull(lib);
        this.handle = notNull(handle);
    }

    VulkanLibrary library() {
        return lib;
    }

    public Pointer handle() {
        return handle;
    }

    public void destroy() {
        lib.vkDestroyInstance(handle, null);
    }
}
```

Creating the instance involves populating a couple of JNA structures and invoking the create API method, this is an ideal scenario for a builder:

```java
public static class Builder {
    private String name;
    private Version ver = new Version(1, 0, 0);

    public Builder name(String name) {
        this.name = notEmpty(name);
        return this;
    }

    public Builder version(Version ver) {
        this.ver = notNull(ver);
        return this;
    }

    public Instance build(VulkanLibrary lib) {
        ...
    }
}
```

The `Version` number is a simple record:

```java
public record Version(int major, int minor, int patch) implements Comparable<Version> {
    /**
     * @return Packed version integer
     */
    public int toInteger() {
        return (major << 22) | (minor << 12) | patch;
    }

    @Override
    public int compareTo(Version that) {
        return this.toInteger() - that.toInteger();
    }
}
```

In the `build` method the code-generated structures are used for the first time to populate the application and engine details:

```java
public Instance build(VulkanLibrary lib) {
    var app = new VkApplicationInfo();
    app.pApplicationName = name;
    app.applicationVersion = ver.toInteger();
    app.pEngineName = "JOVE";
    app.engineVersion = new Version(1, 0, 0).toInteger();
    app.apiVersion = VulkanLibrary.VERSION.toInteger();
    ...
}
```

The `apiVersion` field is the version of the Vulkan implementation required by the application, a new constant is added to the library:

```java
public interface VulkanLibrary {
    Version VERSION = new Version(1, 1, 0);
}
```

Next the instance descriptor is populated, which for the moment is just the application details:

```java
var info = new VkInstanceCreateInfo();
info.pApplicationInfo = app;
```

The relevant API method is invoked to create the instance:

```java
PointerByReference handle = new PointerByReference();
check(lib.vkCreateInstance(info, null, handle));
```

Finally the domain object is created to wrap the handle and the library:

```java
return new Instance(lib, handle.getValue());
```

The `check` method wraps a Vulkan API call and validates the result code:

```java
public interface VulkanLibrary {
    int SUCCESS = VkResult.SUCCESS.value();
    
    static void check(int result) {
        if(result != SUCCESS) {
            throw new VulkanException(result);
        }
    }
}
```

Where `VulkanException` is a custom exception class that builds an informative message for a Vulkan error code:

```java
public class VulkanException extends RuntimeException {
    public final int result;

    public VulkanException(int result) {
        super(String.format("%s[%d]", reason(result), result));
        this.result = result;
    }

    private static String reason(int result) {
        try {
            return IntEnum.mapping(VkResult.class).map(result).name();
        }
        catch(IllegalArgumentException e) {
            return "Unknown error code";
        }
    }
}
```

### Extensions and Validation Layers

There are two other pieces of information to be supplied when creating the instance: extensions and validation layers.

An _extension_ is a platform or vendor-specific addition to the Vulkan API, e.g. swapchain support, GPU-specific functions, etc.

A _validation layer_ is a hook or interceptor for API methods providing additional functionality, e.g. diagnostics, Steam overlay, etc.

These new properties are added to the builder:

```java
public static class Builder {
    private final Set<String> extensions = new HashSet<>();
    private final Set<String> layers = new HashSet<>();
    ...

    public Builder extension(String ext) {
        Check.notEmpty(ext);
        extensions.add(ext);
        return this;
    }

    public Builder layer(ValidationLayer layer) {
        Check.notNull(layer);
        layers.add(layer.name());
        return this;
    }
}
```

The relevant fields of the instance descriptor are populated in the `build` method, which were left as defaults in the previous iteration:

```java
// Populate required extensions
info.ppEnabledExtensionNames = new StringArray(extensions.toArray(String[]::new));
info.enabledExtensionCount = extensions.size();

// Populate required layers
info.ppEnabledLayerNames = new StringArray(layers.toArray(String[]::new));
info.enabledLayerCount = layers.size();
```

Note the use of the JNA `StringArray` helper class that maps a Java array-of-strings to a native pointer-to-pointers (more specifically a `const char* const*` type).

Finally a simple record type is implemented for validation layers:

```java
public record ValidationLayer(String name, int version) {
    /**
     * Standard validation layer.
     */
    public static final ValidationLayer STANDARD_VALIDATION = new ValidationLayer("VK_LAYER_LUNARG_standard_validation");
}
```

The purpose of the standard validation layer is discussed below.

### Required Extensions

Generally two extensions are required to actually perform rendering:

- the general surface: `VK_KHR_surface` 

- and a platform-specific implementation, e.g. `VK_KHR_xcb_surface` for Linux or `VK_KHR_win32_surface` for Windows.

To determine the extensions required for the local hardware we will take advantage of the platform-independant Vulkan support provided by the GLFW library.

First a new package and JNA library are implemented:

```java
interface DesktopLibrary extends Library {
    /**
     * Initialises GLFW.
     * @return Success code
     */
    int glfwInit();

    /**
     * Terminates GLFW.
     */
    void glfwTerminate();

    /**
     * @return Whether Vulkan is supported on this platform
     */
    boolean glfwVulkanSupported();

    /**
     * Enumerates the required Vulkan extensions for this platform.
     * @param count Number of results
     * @return Vulkan extensions (pointer to array of strings)
     */
    Pointer glfwGetRequiredInstanceExtensions(IntByReference count);
}
```

The _desktop_ service abstracts over the underlying GLFW implementation:

```java
public class Desktop {
    public static Desktop create() {
    }

    private final DesktopLibrary lib;

    Desktop(DesktopLibrary lib) {
        this.lib = notNull(lib);
    }

    public boolean isVulkanSupported() {
        return lib.glfwVulkanSupported();
    }

    public String[] extensions() {
        IntByReference size = new IntByReference();
        Pointer ptr = lib.glfwGetRequiredInstanceExtensions(size);
        return ptr.getStringArray(0, size.getValue());
    }
}
```

The service is created and initialised in a similar fashion to the Vulkan API:

```java
public static Desktop create() {
    // Determine library name
    String name = switch(Platform.getOSType()) {
        case Platform.WINDOWS -> "glfw3";
        case Platform.LINUX -> "libglfw";
        default -> throw new UnsupportedOperationException(...);
    };

    // Load native library
    DesktopLibrary lib = Native.load(name, DesktopLibrary.class);

    // Init GLFW
    int result = lib.glfwInit();
    if(result != 1) throw new RuntimeException(...);

    // Create desktop
    return new Desktop(lib);
}
```

Notes:

* The new package is called `desktop` rather than using an ugly GLFW acronym.

* In reality the new GLFW framework was already in place before this stage of the tutorial, but we will continue to introduce functionality as it becomes relevant.

* The pros and cons of leveraging the GLFW-Vulkan integration are discussed in the next chapter.

Finally we can create our first demo application to instantiate a Vulkan instance for the local hardware:

```java
public class InstanceDemo {
    public static void main(String[] args) throws Exception {
        // Open desktop
        Desktop desktop = Desktop.create();
        if(!desktop.isVulkanSupported()) throw new RuntimeException("Vulkan not supported");

        // Init Vulkan
        VulkanLibrary lib = VulkanLibrary.create();

        // Create instance
        Instance instance = new Instance.Builder()
            .name("InstanceDemo")
            .extensions(desktop.extensions())
            .build(lib);

        // Cleanup
        instance.destroy();
        desktop.destroy();
    }
}
```

A convenience method is also added to the instance builder to add the array of extensions returned by GLFW.

---

## Diagnostics Handler

### Overview

Vulkan implements the `STANDARD_VALIDATION` layer that provides an excellent error and diagnostics reporting mechanism, offering comprehensive logging as well as identifying common problems such as orphaned object handles, invalid parameters, performance warnings, etc.  This functionality is not mandatory but its safe to say it is _highly_ recommended during development, so we will address it now before we progress any further.

However there is a complication: the reporting mechanism is not a core part of the API but is itself an extension.  The relevant function pointers must be looked up from the instance and the associated data structures can only determined from the Vulkan documentation.

Registering a diagnostics handler consists of the following steps:

1. Specify the diagnostics reporting requirements in a descriptor.

2. Create a JNA callback to be invoked by Vulkan to report a diagnostics message.

3. Lookup the function pointer to create the handler.

4. Invoke the create function with the descriptor and callback as parameters to attach the handler to the instance.

Our design for the the process of reporting a message is as follows:

1. Vulkan invokes the callback to report a diagnostics message.

2. The callback arguments are aggregated into a message record.

3. And this message is delegated to a consumer configured by the application.

The following new components are required:

* A builder to specify the diagnostics reporting requirements for the application.

* Additional functionality to lookup the function pointers to create and destroy handlers.

* The JNA callback invoked by Vulkan to report messages.

* A `Message` record that aggregates the diagnostics report.

Note that we will still attempt to implement comprehensive argument and logic validation throughout JOVE to trap errors at source.  Although this means the code is often essentially replicating the validation layer the development overhead is usually worth the effort, it is considerably easier to diagnose an exception at the root of a problem, rather than an error message with limited context that may occur later in the application (particularly during rendering).

### Handler

First a new domain object is implemented for a diagnostics handler with inner classes for messages and the callback:

```java
public class Handler {
    private final Pointer handle;
    private final Instance instance;

    private Handler(Pointer handle, Instance instance) {
        this.handle = notNull(handle);
        this.instance = notNull(instance);
    }
    
    public record Message { ... }
    
    static class MessageCallback { ... }
}
```

The message is essentially a convenience wrapper for a diagnostics report:

```java
public record Message(
    VkDebugUtilsMessageSeverity severity,
    Collection<VkDebugUtilsMessageType> types,
    VkDebugUtilsMessengerCallbackData data
)
```

Vulkan reports diagnostics messages via a callback function which is implemented by a JNA `Callback` interface:

```java
static class MessageCallback implements Callback {
    private final Consumer<Message> consumer;

    MessageCallback(Consumer<Message> consumer) {
        this.consumer = consumer;
    }

    /**
     * Callback handler method.
     * @param severity          Severity
     * @param type              Message type(s) mask
     * @param pCallbackData     Data
     * @param pUserData         Optional user data (always {@code null})
     * @return Whether to continue execution (always {@code false})
     */
    public boolean message(int severity, int type, VkDebugUtilsMessengerCallbackData pCallbackData, Pointer pUserData) {
        ...
        return false;
    }
}
```
Notes:

* The signature of the callback method is derived from the documentation, as an extension it is not specified in the API header.

* A JNA callback must contain a __single__ public method but this is not enforced at compile-time.

* The `pUserData` parameter is optional data returned to the callback to correlate state, this is largely redundant for an OO application and is always `null` in our implementation.

On invocation the callback first transforms the _severity_ and the _types_ bit-field to the corresponding enumerations:

```java
VkDebugUtilsMessageSeverity severityEnum = IntEnum.mapping(VkDebugUtilsMessageSeverity.class).map(severity);
Collection<VkDebugUtilsMessageType> typesEnum = IntEnum.mapping(VkDebugUtilsMessageType.class).enumerate(type);
```

The relevant data is then composed into a message instance and delegated to the handler:

```java
Message message = new Message(severityEnum, typesEnum, pCallbackData);
consumer.accept(message);
```

A custom `toString` implementation is added to the `Message` record to build a human-readable representation of a diagnostics report:

```java
public String toString() {
    String compoundTypes = types.stream().map(Enum::name).collect(joining("-"));
    StringJoiner str = new StringJoiner(":");
    str.add(severity.name());
    str.add(compoundTypes);
    if(!data.pMessage.contains(data.pMessageIdName)) {
        str.add(data.pMessageIdName);
    }
    str.add(data.pMessage);
    return str.toString();
}
```

For example (excluding the message text):

```
ERROR:VALIDATION:Validation Error: [ VUID-VkDeviceQueueCreateInfo-pQueuePriorities-00383 ] ...
```

### Configuration

Configuring and instantiating the diagnostics handler is again implemented via a builder:

```java
public static class Builder {
    private final Instance instance;
    private final Set<VkDebugUtilsMessageSeverity> severity = new HashSet<>();
    private final Set<VkDebugUtilsMessageType> types = new HashSet<>();
    private Consumer<Message> consumer = System.err::println;
}
```

Note that by default diagnostics messages are simply dumped to the error console.

The `build` method first populates the descriptor for the handler:

```java
public Handler build() {
    var info = new VkDebugUtilsMessengerCreateInfoEXT();
    info.messageSeverity = new BitMask<>(severity);
    info.messageType = new BitMask<>(types);
    info.pfnUserCallback = new MessageCallback(consumer);
    info.pUserData = null;
}
```

Next the API method is looked up from the parent instance:

```java
Function create = instance.function("vkCreateDebugUtilsMessengerEXT");
```

Which is a new factory on the instance class:

```java
public Function function(String name) {
    Pointer ptr = lib.vkGetInstanceProcAddr(this, name);
    if(ptr == null) throw new RuntimeException(...);
    return Function.getFunction(ptr);
}
```

And the new API method is added to the library:

```java
interface VulkanLibrary {
    /**
     * Looks up an instance function.
     * @param instance      Vulkan instance
     * @param pName         Function name
     * @return Function pointer
     */
    Pointer vkGetInstanceProcAddr(Pointer instance, String pName);
}
```

Next the function to create the handler is invoked with the appropriate arguments:

```java
Pointer parent = instance.handle();
PointerByReference ref = new PointerByReference();
Object[] args = {parent, info, null, ref};
check(create.invokeInt(args));
```

And finally the domain object is instantiated:

```java
return new Handler(ref.getValue(), instance);
```

Notes:

* The JNA `invokeInt` method calls an arbitrary function pointer with the given array of arguments.

* The name and signature of the function is specified in the [Vulkan documentation](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/html/vkspec.html#vkCreateDebugUtilsMessengerEXT).

* As a convenience the builder initialises the severity and message types to appropriate defaults if not specified by the application (not shown).

### Integration

To attach a diagnostics handler in the demo the relevant extension and the validation layer must be enabled:

```java
Instance instance = new Instance.Builder()
    .name("InstanceDemo")
    .extension(VulkanLibrary.EXTENSION_DEBUG_UTILS)
    .extensions(desktop.extensions())
    .layer(ValidationLayer.STANDARD_VALIDATION)
    .build(lib);
```

Where the name of the diagnostics extension is defined in the main library:

```java
public interface VulkanLibrary {
    String EXTENSION_DEBUG_UTILS = "VK_EXT_debug_utils";
}
```

A handler can now be configured and attached to the instance:

```java
instance
    .handler()
    .severity(VkDebugUtilsMessageSeverity.WARNING)
    .severity(VkDebugUtilsMessageSeverity.ERROR)
    .type(VkDebugUtilsMessageType.GENERAL)
    .type(VkDebugUtilsMessageType.VALIDATION)
    .build();
```

From now on when we screw things up we should receive error messages on the console.

To properly clean up resources the handler is released when the instance is destroyed:

```java
public void destroy() {
    Function destroy = instance.function("vkDestroyDebugUtilsMessengerEXT");
    Pointer parent = instance.handle();
    Object[] args = {parent, handle, null};
    destroy.invoke(args);
}
```

---

## Testing Issues

### Background

To unit-test the code that creates the instance we might start with the following (naive) approach:

```java
private VulkanLibrary lib;

@BeforeEach
void before() {
    lib = mock(VulkanLibrary.class);
}

@Test
void build() {
    // Init expected create descriptor
    var expected = new VkInstanceCreateInfo();
    ...

    // Init create method
    var handle = new PointerByReference();
    when(lib.vkCreateInstance(expected, null, handle)).thenReturn(0);

    // Build instance
    Instance instance = new Instance.Builder()
        .name("test")
        .extension("extension")
        .build(lib);

    // Check created instance        
    assertNotNull(instance);
    assertEquals(handle.getValue(), instance.handle());
    ...
}
```

This test is designed to check that the builder invokes the correct API method with the expected arguments and `VkInstanceCreateInfo` descriptor.

However by default all methods in the mocked library will return zero (which of course is the Vulkan success return code) so the test passes whether we include the `when` clause or not and essentially proves nothing.

A better approach is to introduce a post-condition that explicitly validates the expected method invocation:

```java
verify(lib).vkCreateInstance(expected, null, handle);
```

However this introduces further problems:

### Reference Types

The Vulkan API (and many other native libraries) make extensive use of _by-reference_ types to return data, with the actual return value of the method generally representing an error code.  For example `vkCreateInstance` returns the `handle` of the newly created instance via a JNA `PointerByReference` type.

Mercifully this approach is virtually unknown in Java but it does pose an awkward problem when we come to testing: the usual Java unit-testing frameworks (JUnit, Mockito) are designed around the return value of a method generally being the important part and any error conditions modelled by exceptions.

If a `PointerByReference` is created in the test it will be a different instance to the one created in the `build` method itself, it would be difficult to determine whether the test was legitimately successful or only passed by luck.

The `verify` statement could be refactored to use Mockito matchers:

```java
verify(lib).vkCreateInstance(..., isA(PointerByReference.class));
```

But this only checks that the argument was not `null` rather than verifying the actual value.  Additionally __every__ argument would then have to be an argument matcher which just adds more development complexity and obfuscates the test.

Alternatively a Mockito _answer_ could be used to initialise the reference argument, but that would require tedious and repetitive code for _every_ unit-test that exercises API methods with by-reference return values (which is pretty much all the methods that instantiate Vulkan components).

To resolve (or at least mitigate) this issue we introduce the _reference factory_ that is responsible for generating by-reference objects used in API calls:

```java
public interface ReferenceFactory {
    /**
     * @return New integer-by-reference
     */
    IntByReference integer();

    /**
     * @return New pointer-by-reference
     */
    PointerByReference pointer();
}
```

The reference factory and/or individual methods can then be mocked as appropriate for a given unit-test.

The default implementation simply creates new instances on demand:

```java
public interface ReferenceFactory {
    ...
    ReferenceFactory DEFAULT = new ReferenceFactory() {
        @Override
        public IntByReference integer() {
            return new IntByReference();
        }

        @Override
        public PointerByReference pointer() {
            return new PointerByReference();
        }
    };
}
```

A default reference factory is added to the instance builder and the existing code is refactored accordingly:

```java
public static class Builder {
    private ReferenceFactory factory = ReferenceFactory.DEFAULT;
    ...

    public Instance build(VulkanLibrary lib) {
        ...
        // Create instance
        PointerByReference handle = factory.pointer();
        check(lib.vkCreateInstance(info, null, handle));
    
        // Create instance domain wrapper
        return new Instance(handle.getValue(), lib, factory);
    }
}
```

### Structure Equality

The unit-test also asserts that the code constructs the expected Vulkan structures to be passed to the API method.  Unfortunately it turns out that two JNA structures with the same data are __not__ equal and the above `verify` test fails even though the code is working correctly.  A review of the JNA source code shows that the structure class essentially violates the Java `equals` contract assumed by the testing frameworks.

Over-riding the default implementation of `equals` would be risky since structures are compared by equality within JNA itself.  Also note that Mockito does not support stubbing of the basic `Object` methods (equals, to-string, etc) since these are also used internally.

Therefore we take the simpler (if more dubious) option of bastardising the `equals` method of a sub-class of the expected structure:

```java
var expected = new VkInstanceCreateInfo() {
    @Override
    public boolean equals(Object obj) {
        return dataEquals((Structure) obj);
    }
};
```

The `dataEquals` JNA method compares the native representation of the structure.  This is slightly repetitive since the same bodge has to implemented for _every_ case where a test needs to verify Vulkan descriptors but we have not been able to find a neater solution.

Using the reference factory and the structure equality bodge the final unit-test looks like this:

```java
private VulkanLibrary lib;
private ReferenceFactory factory;

@BeforeEach
void before() {
    lib = mock(VulkanLibrary.class);
    factory = mock(ReferenceFactory.class);
    when(factory.pointer()).thenReturn(new PointerByReference(new Pointer(1)));
}

@Test
void build() {
    // Build instance
    Instance instance = new Instance.Builder()
        .name("test")
        .extension("extension")
        .factory(factory)
        .build(lib);

    // Check created instance        
    assertNotNull(instance);
    ...

    // Init expected create descriptor
    var expected = new VkInstanceCreateInfo() {
        @Override
        public boolean equals(Object obj) {
            return dataEquals((Structure) obj);
        }
    };
    ...

    // Check API
    verify(lib).vkCreateInstance(expected, null, factory.pointer());
}
```

Note that we have made the decision to have API methods return result codes as an integer rather than the more explicit `VkResult` enumeration, taking advantage of the fact that `VK_SUCCESS` maps to integer zero (the default returned by mocked API methods that are not stubbed).  This avoids forcing having to explicitly stub the result code in every unit-test at the expense of a slightly less type-safe approach.  Note that we _could_ override the default Mockito answer for the entire mocked API but this is a little messy to implement for minimal benefit.

In general from now on testing is not covered unless there is a specific point-of-interest, it can be assumed that unit-tests are developed in-parallel with the main code.

---

## Summary

In this first chapter we:

- Instantiated the Vulkan API and GLFW library.

- Created a Vulkan instance with the required extensions and layers.

- Attached a diagnostics handler.

- Mitigated some of the issues around unit-testing Vulkan API methods.
