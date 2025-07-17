---
title: Vulkan Devices
---

---

## Contents

- [Overview](#overview)
- [Physical Devices](#physical-devices)
- [Logical Device](#logical-device)
- [Improvements](#improvements)

---

## Overview

Hardware components that support Vulkan are represented by a _physical device_ which defines the capabilities of that component (rendering, data transfer, etc).  In general there will be a single physical device (i.e. the GPU) or perhaps also an on-board graphics device for a laptop.

A _logical device_ is an instance of a physical device and is the central component for all subsequent Vulkan functionality.  The logical device exposes a number of asynchronous _work queues_ that are used for various tasks such as invoking rendering, transferring data to/from the hardware, etc.

Creating the logical device consists of the following steps:

1. Enumerate the physical devices available on the hardware.

2. Select one that satisfies the requirements of the application.

3. Create a logical device for the selected physical device.

4. Retrieve the required work queues.

Additionally some framework enhancements are introduced in this chapter to support common patterns that appeared during design and development.

---

## Physical Devices

### Devices and Queues

First a new domain object is defined for the physical device and its supported work queues:

```java
public class PhysicalDevice {
    private final Pointer handle;
    private final Instance instance;
    private final List<Family> families;
}
```

The physical device specifies a number of _queue families_ that define the capabilities of work that can be performed by that device:

```java
public record Queue(Pointer handle, Family family) {
    public record Family(int index, int count, Set<VkQueueFlag> flags) {
        public Family {
            flags = Set.copyOf(flags);
        }
    }
}
```

Where _index_ is a numeric identifier and _count_ is the number of queue instances.

Queues can also be blocked until all work has been processed by that queue:

```java
public record Queue {
    public void waitIdle(VulkanLibrary lib) {
        check(lib.vkQueueWaitIdle(handle));
    }
}
```

### Device Enumeration

To enumerate the available physical devices the `vkEnumeratePhysicalDevices` API method is invoked _twice_:

1. Once to retrieve the number of available devices via an integer-by-reference value (the array parameter is set to `null`).

2. And again to retrieve the actual array of device handles.

This process is wrapped in the following factory method:

```java
public static Stream<PhysicalDevice> devices(Instance instance) {
    // Determine array length
    IntByReference count = instance.factory().integer();
    check(api.vkEnumeratePhysicalDevices(instance.handle(), count, null));

    // Allocate array
    Pointer[] array = new Pointer[count.getValue()];

    // Retrieve array
    if(array.length > 0) {
        check(api.vkEnumeratePhysicalDevices(instance.handle(), count, array));
    }

    // Create devices
    return Arrays.stream(array).map(ptr -> create(ptr, instance));
}
```

The `create` method first retrieves the array of queue families for each device:

```java
private static PhysicalDevice create(Pointer handle, Instance instance) {
    // Count number of families
    VulkanLibrary lib = instance.library();
    IntByReference count = instance.factory().integer();
    lib.vkGetPhysicalDeviceQueueFamilyProperties(handle, count, null);

    // Retrieve families
    VkQueueFamilyProperties[] array;
    if(count.getValue() > 0) {
        array = (VkQueueFamilyProperties[]) new VkQueueFamilyProperties().toArray(count.getValue());
        lib.vkGetPhysicalDeviceQueueFamilyProperties(handle, count, array[0]);
    }
    else {
        array = Array.newInstance(VkQueueFamilyProperties.class, 0);
    }
    
    ...
}
```

Again the same API method is invoked twice to retrieve the queue families.  However in this case the JNA `toArray` factory method is invoked on an _instance_ of a `VkQueueFamilyProperties` structure to allocate the array and the __first__ element is passed to the API (i.e. a pointer to an array of structures).  This common pattern will be abstracted at the end of the chapter.

Another helper is implemented to create a queue family domain object:

```java
private static Family family(int index, VkQueueFamilyProperties props) {
    var mapping = new ReverseMapping<>(VkQueueFlag.class);
    Set<VkQueueFlag> flags = props.queueFlags.enumerate(mapping);
    return new Family(index, props.queueCount, flags);
}
```

Which is used to transform the array of queue families:

```java
List<Family> families = IntStream
    .range(0, props.length)
    .mapToObj(n -> family(n, props[n]))
    .toList();
```

And finally the device is instantiated:

```java
return new PhysicalDevice(handle, instance, families);
```

The device exposes a set of properties that specify the name and type of the device, hardware limits, etc:

```java
public VkPhysicalDeviceProperties properties() {
    var props = new VkPhysicalDeviceProperties();
    VulkanLibrary lib = instance.library();
    lib.vkGetPhysicalDeviceProperties(PhysicalDevice.this, props);
    return props;
}
```

The new API methods used thus far as added to the JNA library:

```java
int  vkEnumeratePhysicalDevices(Pointer instance, IntByReference count, Pointer[] devices);
void vkGetPhysicalDeviceProperties(Pointer device, VkPhysicalDeviceProperties props);
void vkGetPhysicalDeviceQueueFamilyProperties(Pointer device, IntByReference count, VkQueueFamilyProperties props);
```

The following temporary code can now be added to the demo to dump the available devices:

```java
PhysicalDevice
    .devices(instance)
    .map(PhysicalDevice::properties)
    .map(props -> props.deviceName)
    .map(String::new)
    .forEach(System.out::println);
```

### Device Selection

To select a physical device that supports the requirements of a given application a filter is applied to the queue families supported by each device.  However the mechanism used by Vulkan requires this test to be applied twice: once to select the device matching the filter, and again to retrieve the matched queue family when configuring the logical device.  After trying several approaches we settled on the design described below which determines the matching family as a _side effect_ of selecting the device (the same approach is used in the tutorial). This is generally something to avoid but seems acceptable in this case and is at least relatively simple from the perspective of the application.

The filter is essentially a binary predicate on the device and its supported queues:

```java
public static abstract class Selector implements Predicate<PhysicalDevice> {
    /**
     * Matches this selector.
     * @param device Physical device
     * @param family Queue family
     * @return Whether matched
     */
    protected abstract boolean matches(PhysicalDevice device, Family family);
}
```

The filter enumerates the queue families provided by each device and records the matched results:

```java
public static abstract class Selector implements Predicate<PhysicalDevice> {
    private final Map<PhysicalDevice, Family> results = new HashMap<>();

    @Override
    public boolean test(PhysicalDevice device) {
        return device
            .families
            .stream()
            .filter(family -> matches(device, family))
            .peek(match -> results.put(device, match))
            .findAny()
            .isPresent();
    }
}
```

Note the `peek` clause in the stream processor which records matching queue families as a side-effect.

The matched family can be queried later from the selected device:

```java
public Family select(PhysicalDevice device) {
    Family family = results.get(device);
    if(family == null) throw new UnsupportedOperationException();
    return family;
}
```

The following factory method creates a selector that matches a queue family by a set of required properties:

```java
public static Selector family(VkQueueFlag... flags) {
    return new Selector() {
        private final Set<VkQueueFlag> required = Set.of(flags);

        @Override
        protected boolean matches(PhysicalDevice device, Family family) {
            return family.flags().containsAll(required);
        }
    };
}
```

Additional selector implementations will be added in subsequent chapters.

---

## Logical Device

### Domain Object

The first cut domain object for the logical device is as follows:

```java
public class LogicalDevice {
    private final Pointer handle;
    private final PhysicalDevice parent;
    private final VulkanLibrary lib;
    private final Map<Family, List<Queue>> queues;

    public void destroy() {
        lib.vkDestroyDevice(handle, null);
    }
}
```

The whole device can also be blocked until all queues have finished processing:

```java
public void waitIdle() {
    check(lib.vkDeviceWaitIdle(handle));
}
```

The logical device is another object that is highly configurable and therefore constructed via a builder:

```java
public static class Builder {
    private final PhysicalDevice parent;
    private final Set<String> extensions = new HashSet<>();
    private final Set<String> layers = new HashSet<>();
    private final Map<Family, RequiredQueue> queues = new HashMap<>();
}
```

Note that extensions and validation layers can be configured at both the instance and device level, however more recent Vulkan implementations will ignore layers specified at the device level (both are retained for backwards compatibility).

The work queues required by the logical device are specified by a new transient type:

```java
public record RequiredQueue(Family family, List<Percentile> priorities)
```

Where:

- The _family_ is one of the queue families specified by the parent physical device.

- The _priorities_ is a list of percentile values that specify the required number of queues in the family and their relative priorities.

- A `Percentile` is a custom type for a `0..1` floating-point number.

The `build` method populates the descriptor for the logical device:

```java
public LogicalDevice build() {
    // Create descriptor
    var info = new VkDeviceCreateInfo();

    // Add required extensions
    info.ppEnabledExtensionNames = new StringArray(extensions.toArray(String[]::new));
    info.enabledExtensionCount = extensions.size();

    // Add validation layers
    info.ppEnabledLayerNames = new StringArray(layers.toArray(String[]::new));
    info.enabledLayerCount = layers.size();

    // Add queue descriptors
    info.queueCreateInfoCount = queues.size();
    info.pQueueCreateInfos = StructureCollector.pointer(queues.values(), new VkDeviceQueueCreateInfo(), RequiredQueue::populate);
    ...
}
```

JNA requires a native array to be a contiguous memory block as opposed to a Java array where the memory address of each element is arbitrary.
Here we introduce the `StructureCollector` helper class (detailed at the end of the chapter) which handles the transformation of a Java collection to a contiguous JNA structure array.

The `populate` method is invoked by the helper to 'fill' the JNA structure from the domain object:

```java
private void populate(VkDeviceQueueCreateInfo info) {
    info.queueCount = array.length;
    info.queueFamilyIndex = family.index();
    info.pQueuePriorities = build();
}
```

The queue priorities is another array field that must be a contiguous memory block (mapping to the native `const float*` type) constructed by the following helper:

```java
private Pointer build() {
    Percentile[] array = priorities.toArray(Percentile[]::new);
    Memory mem = new Memory(array.length * Float.BYTES);
    for(int n = 0; n < array.length; ++n) {
        mem.setFloat(n * Float.BYTES, array[n].floatValue());
    }
}
```

Finally the builder invokes the API to instantiate the logical device:

```java
Instance instance = parent.instance();
VulkanLibrary lib = instance.library();
ReferenceFactory factory = instance.factory();
PointerByReference handle = factory.pointer();
check(lib.vkCreateDevice(parent, info, null, handle));
```

### Work Queues

Next the work queues are retrieved from the new device:

```java
Map<Family, List<Queue>> map = queues
    .values()
    .stream()
    .flatMap(required -> queues(handle.getValue(), required))
    .collect(groupingBy(Queue::family));
```

Which uses the following helper to instantiate an array of queues for each entry:

```java
private Stream<Queue> queues(Pointer dev, RequiredQueue required) {
    // Init library
    Instance instance = parent.instance();
    Library lib = instance.library();
    PointerByReference ref = instance.factory().pointer();

    // Retrieve queues
    int count = required.priorities.size();
    Queue[] queues = new Queue[count];
    for(int n = 0; n < count; ++n) {
        lib.vkGetDeviceQueue(dev, required.family.index(), n, ref);
        queues[n] = new Queue(ref.getValue(), required.family);
    }

    return Arrays.stream(queues);
}
```

And finally the builder creates the domain object for the new device:

```java
return new LogicalDevice(handle.getValue(), parent, map);
```

### Integration

The new API methods are implemented as an inner class of the logical device:

```java
interface Library {
    int  vkCreateDevice(Pointer physicalDevice, VkDeviceCreateInfo pCreateInfo, Pointer pAllocator, PointerByReference device);
    void vkDestroyDevice(Pointer device, Pointer pAllocator);
    void vkGetDeviceQueue(Pointer device, int queueFamilyIndex, int queueIndex, PointerByReference pQueue);
    int  vkDeviceWaitIdle(Pointer device);
    int  vkQueueWaitIdle(Pointer queue);
}
```

The various API interfaces are then aggregated into the overall Vulkan library:

```java
public interface VulkanLibrary extends Library, DeviceLibrary, ...
```

Where convenient intermediate interfaces group multiple JNA libraries, e.g. for the various device libraries:

```java
interface DeviceLibrary extends Instance.Library, PhysicalDevice.Library, LogicalDevice.Library
```

From now we take this approach of implementing API libraries as inner classes of the companion domain object, co-locating each API with its associated class and reducing the number of super-interfaces for the root Vulkan library.

In the demo a selector is defined to find a physical device that supports rendering:

```java
Selector selector = Selector.queue(VkQueueFlag.GRAPHICS);
```

From which the appropriate device can be selected:

```java
PhysicalDevice physical = PhysicalDevice
    .devices(instance)
    .filter(selector)
    .findAny()
    .orElseThrow();
```

Next the logical device is constructed with a rendering queue:

```java
LogicalDevice device = new LogicalDevice.Builder(gpu)
    .extension(VulkanLibrary.EXTENSION_SWAP_CHAIN)
    .layer(ValidationLayer.STANDARD_VALIDATION)
    .queue(selector)
    .build();
```

And finally the actual queue instance can be retrieved:

```java
Queue queue = device.queue(selector).getFirst();
```

---

## Improvements

### Two-Stage Invocation

When enumerating the physical devices we first came across API methods that are invoked __twice__ to retrieve data from Vulkan.

The process is generally:

1. Invoke the API method with an integer-by-reference _count_ to determine the size of the results, with the data argument set to `null`.

2. Allocate the 'container' accordingly, usually an array of structures or pointer handles.

3. Invoke again passing both the count and the allocated object to populate the returned results.

This is a common pattern across the Vulkan API which we rather grandiosely refer to as _two-stage invocation_.

The following interface abstracts an API method that employs two-stage invocation:

```java
@FunctionalInterface
public interface VulkanFunction<T> {
    /**
     * Vulkan API method that retrieves data using the <i>two-stage invocation</i> approach.
     * @param count         Size of the data
     * @param data          Returned data or {@code null} to retrieve the size of the data
     * @return Vulkan result code
     */
    int enumerate(IntByReference count, T data);
}
```

The API method to enumerate the physical devices can now be defined thus:

```java
VulkanFunction<Pointer[]> func = (count, devices) -> lib.vkEnumeratePhysicalDevices(instance, count, devices);
```

Two-stage invocation of the function is encapsulated in the following helper:

```java
default <T> T invoke(IntByReference count, IntFunction<T> factory) {
    // Invoke to determine the size of the data
    check(enumerate(count, null));

    // Instantiate the data object
    int size = count.getValue();
    T data = factory.apply(size);

    // Invoke again to populate the data object
    if(size > 0) {
        check(enumerate(count, data));
    }

    return data;
}
```

The process of enumerating the physical devices can now be refactored to the following more concise code:

```java
public static Stream<PhysicalDevice> devices(Instance instance) {
    VulkanFunction<Pointer[]> func = (count, devices) -> instance.library().vkEnumeratePhysicalDevices(instance, count, devices);
    IntByReference count = instance.factory().integer();
    Pointer[] handles = func.invoke(count, Pointer[]::new);
    ...
}
```

For an array of JNA structures a second, slightly different implementation is needed since the array __must__ be a contiguous block of memory allocated using the `toArray` helper:

```java
interface StructureVulkanFunction<T extends Structure> extends VulkanFunction<T> {
    default T[] invoke(IntByReference count, T identity) {
        // Invoke to determine the length of the array
        check(enumerate(count, null));
    
        // Instantiate the structure array
        T[] array = (T[]) identity.toArray(count.getValue());
    
        // Invoke again to populate the array (note passes first element)
        if(array.length > 0) {
            check(enumerate(count, array[0]));
        }
    
        return array;
    }
}
```

Notes:

* The _identity_ is an instance of the structure used to allocate the resultant array.

* In this case the API method accepts a pointer to a structure array which maps to the __first__ element of the allocated Java array.

As an example, the code to retrieve the queue families for a physical device now becomes:

```java
StructureVulkanFunction<VkQueueFamilyProperties> func = (count, array) -> instance.library().vkGetPhysicalDeviceQueueFamilyProperties(handle, count, array);
IntByReference count = instance.factory().integer();
VkQueueFamilyProperties[] props = func.invoke(count, VkQueueFamilyProperties::new);
```

### Structure Collector

Vulkan makes heavy use of structures to configure a variety of objects, however _arrays_ of JNA structures pose a number of challenges:

* Unlike a standard POJO an array of JNA structures __must__ be allocated using the JNA `toArray` factory method to create a contiguous block of memory.

* Many API methods expect a pointer-to-array, i.e. the __first__ element of the array.

* Additionally there are edge cases where `null` is a valid argument.

* Obviously the size of the array must be known _before_ it can be populated, which imposes constraints on how the data is handled (in particular whether Java streams can be employed).

* We would prefer a solution that employs internal iteration (similar to streams) rather than having to implement loops throughout the code-base.

* JNA does not support cloning or copying of structures out-of-the-box therefore data must be represented by custom or transient data types (in any case JNA structures are really only intended as 'carriers' for marshalling to/from native structures).

None of these are particularly difficult issues to overcome, but the number of cases where this occurs makes the code tedious to develop, error-prone, and less testable.

The following helper allocates and populates a JNA structure array from an arbitrary collection of domain objects, providing a common solution to address these issues:

```java
public final class StructureCollector {
    public static <T, R extends Structure> R[] array(Collection<T> data, R identity, BiConsumer<T, R> populate) {
        // Check for empty data
        if(data.isEmpty()) {
            return null;
        }

        // Allocate contiguous array
        @SuppressWarnings("unchecked")
        R[] array = (R[]) identity.toArray(data.size());

        // Populate array
        Iterator<T> itr = data.iterator();
        for(R element : array) {
            populate.accept(itr.next(), element);
        }
        assert !itr.hasNext();

        return array;
    }
}
```

Where:

* _T_ is the domain type, e.g. a `RequiredQueue` for the logical device.

* _R_ is the component type of the resultant array, e.g. `VkDeviceQueueCreateInfo`.

* And _identity_ is an instance of the structure used to allocate the array.

The `populate` method 'fills' an element of the array from the corresponding domain object (generally implemented as a hidden helper method on the domain class).  Note that this implementation uses an iterator over the incoming data since there is no simple means of instantiating a generic array and using a simpler loop to walk both.

The following alternative is provided for the case where the API method requires a pointer-to-array argument:

```java
public static <T, R extends Structure> R pointer(Collection<T> data, Supplier<R> identity, BiConsumer<T, R> populate) {
    // Construct array
    R[] array = array(data, identity, populate);

    // Convert to pointer-to-array
    if(array == null) {
        return null;
    }
    else {
        return array[0];
    }
}
```

Note that the `pointer` variant requires the structure to be a JNA `ByReference` type, otherwise an exception is thrown when the structure array is marshalled.

A structure that is accessed _by reference_ is bizarrely identified by JNA via a marker interface, as opposed to (say) a flag on the structure itself.  This forces the developer to implement an entirely new class that provides no additional public functionality, and yet application code is _still_ required to determine whether the structure is being passed by value or by reference.

Therefore the code generator is modified to identify whether each structure is used by reference (i.e. it is used as a structure field with a pointer) and implements the marker interface accordingly, working on the assumption that a given structure will _always_ be used similarly.  The only instance where this assumption breaks down is the `VkRect2D` structure which is used both as a value __and__ a reference type, so unfortunately we are forced to manually implement a separate class to support both use-cases.

In the logical device the new helper class is used to build the array of required queue descriptors:

```java
info.pQueueCreateInfos = StructureCollector.pointer(queues.entrySet(), new VkDeviceQueueCreateInfo(), Builder::populate);
```

Finally a generalised custom stream collector is provided which is more convenient in some cases:

```java
/**
 * Helper - Creates a collector that constructs a contiguous array of JNA structures.
 * @param <T> Data type
 * @param <R> Resultant structure type
 * @param identity      Identity constructor
 * @param populate      Population function
 * @param chars         Collector characteristics
 * @return Structure collector
 * @see #array(Collection, Supplier, BiConsumer)
 */
public static <T, R extends Structure> Collector<T, ?, R[]> collector(Supplier<R> identity, BiConsumer<T, R> populate, Characteristics... chars) {
    BinaryOperator<List<T>> combiner = (left, right) -> {
        left.addAll(right);
        return left;
    };
    Function<List<T>, R[]> finisher = list -> array(list, identity, populate);
    return Collector.of(ArrayList::new, List::add, combiner, finisher, chars);
}
```

---

## Summary

In this chapter we:

* Enumerated the physical devices available on the local hardware and selected one appropriate for the demo application.

* Created the logical device and work queues used in subsequent chapters

* Added supporting functionality for _two stage invocation_ and population of structure arrays.

