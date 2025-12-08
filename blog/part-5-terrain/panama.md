---
title: Panama Integration
---

---

# Contents

- [Introduction](#introduction)
- [Approach](#approach)
- [Framework](#framework)
- [Vulkan Instance](#vulkan-instance)
- [Physical Device](#physical-device)
- [Code Generation](#code-generation)
- [Miscellaneous](#miscellaneous)
- [Summary](#summary)

---

# Introduction

It had always been the intention at some point to re-implement the _bindings_ to the Vulkan native layer using the FFM (Foreign Function and Memory) library developed as part of project [Panama](https://openjdk.java.net/projects/panama/).  The reasons for delaying this rework are discussed in [code generation](/jove-blog/blog/part-1-intro/code-generation#alternatives) at the very beginning of the blog.

With the LTS release of JDK 21 the FFM library was elevated to a preview feature (rather than requiring an incubator build) and much more information and tutorials started to appear online, so it was high time to replace JNA with the more future proofed (and safer) FFM solution.

The following section analyses the requirements for the refactoring work.  The remaining sections iteratively integrate the FFM solution into the existing code base.

---

# Approach

## Analysis

A fundamental technical decision that must be addressed immediately is whether to use the `jextract` tool to generate the bindings or to implement a bespoke solution.

We use the tool and compare and contrast with a theoretical custom implementation built using FFM from first principles.

Generating the FFM bindings using `jextract` is relatively trivial and takes a couple of seconds:

`jextract --source \VulkanSDK\1.2.154.1\Include\vulkan\vulkan.h`

The advantages of using the tool are obvious:

* Proven and standard JVM technology.

* Automated, in particular the FFM memory layouts for native structures.

However there are disadvantages:

* The generated API is polluted by ugly internals such as field handles, structure sizes, helper methods, etc.

* API methods and structures can only be defined in terms of an FFM `MemorySegment` or primitives resulting in poor type safety and loss of meaning.

* Enumerations are implemented as static integer accessors (WTF!) 

* Implies throwing away the existing code generated enumerations and structures.

Alternatively a bespoke solution based on the existing hand-crafted API and code-generated classes would have the following advantages:

* The native API and structures are expressed in domain terms.

* Proper enumerations.

* Retains the existing API and code-generated enumerations and structures.

On the other hand:

* Substantial development effect.

* Proprietary solution with the potential for unknown problems further down the line.

* Memory layouts needs to be generated for _all_ structures.

Despite the advantages of `jextract` the hybrid solution is clearly preferable: the ugly bindings, lack of type safety, etc. were the very concerns that led us to abandon LWJGL in the first place.

> In any case the application and/or JOVE would _still_ need to transform domain types to/from the FFM equivalents even if `jextract` was used to generate the underlying bindings.

The generated source code will be retained since it will be a useful resource during development.

## Requirements

With this decision made (for better or worse) the custom framework and refactoring will entail the following:

* A one-off task to remove all JNA dependencies from the code base.

* A mechanism that generates an FFM-based implementation of the existing GLFW and Vulkan API definitions.

* A _marshalling_ framework that transforms supported Java types to/from the FFM equivalents.

* Support for by-reference parameters.

* Callback support to enable GLFW device polling and the Vulkan diagnostics handler.

* Refactoring of the code generator to build the FFM memory layout for all structures.

The marshalling framework will need to support the following types:

* Primitives

* Strings

* Structures

* Domain types, i.e. reference types such as `Handle` or subclasses of `NativeObject`

* Integer enumerations and bitfields

* Arrays of these supported types.

There are also a couple of edge cases that are immediately apparent:

1. It seems logical to continue to treat by-reference integers and pointers as special cases.  They are already integral to JOVE (based on the existing JNA equivalents) and in particular there is no Java analog for a by-reference integer.

2. The `glfwGetRequiredInstanceExtensions` method cannot simply be mapped to a string array and is therefore hard-coded for the moment, see [Returned Arrays](#returned-arrays) below.

---

# Framework

## Native Library

Constructing the FFM implementation of a native API involves the following steps:

1. Enumerate the native methods from the user defined API.

2. Create and link an FFM method handle for each identified API method.

3. Create a Java proxy that delegates API calls to the underlying FFM methods.

This process is implemented by a new factory component:

```java
public class NativeLibraryFactory {
    public Object build(List<Class<?>> api) {
        ...
    }
}
```

Where _api_ is the user defined interfaces specifying the API to be implemented.

The API methods are enumerated by reflecting the library interfaces:

```java
Map<Method, NativeMethod> methods = api
    .stream()
    .map(Class::getMethods)
    .flatMap(Arrays::stream)
    .filter(NativeLibraryFactory::isNativeMethod)
    .collect(toMap(Function.identity(), NativeMethod::new));
```

Where a native method is defined as a public, non-static member of an interface:

```java
private static boolean isNativeMethod(Method method) {
    int modifiers = method.getModifiers();
    return Modifier.isAbstract(modifiers) && !Modifier.isStatic(modifiers);
}
```

For the moment the native method class is a simple wrapper that echoes the delegated method:

```java
class NativeMethod {
    private final Method method;
    
    public void invoke(Object[] args) {
        System.out.println(method + " " + Arrays.toString(args));
    }
}
```

Next an _invocation handler_ is created that delegates API calls to the relevant native method:

```java
var handler = new InvocationHandler() {
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        NativeMethod delegate = methods.get(method);
        return delegate.invoke(args);
    }
}
```

From which a Java proxy is created:

```java
ClassLoader loader = this.getClass().getClassLoader();
var interfaces = api.toArray(Class<?>[]::new);
return Proxy.newProxyInstance(loader, interfaces, handler);
```

To exercise the new framework a throwaway prototype is created using GLFW as the guinea-pig.  This API is considerably simpler than Vulkan, which is front-loaded with complexities such as marshalling of structures, by-reference parameters, arrays, etc.

First the following (very) cut-down GLFW library is defined, dependant only on integer primitives for the moment:

```java
interface Prototype {
    int     glfwInit();
    int     glfwVulkanSupported();
    void    glfwTerminate();
}
```

A proxy implementation is generated using the new factory:

```java
var factory = new NativeLibraryFactory();
Prototype api = (Prototype) factory.build(List.of(API.class));
```

Which can then be invoked to test the trivial API:

```java
api.glfwInit();
api.glfwVulkanSupported();
api.glfwTerminate();
```

Essentially all this does is intercept the method calls and dump them to the console.

## Native Method

In order to actually invoke the native library the wrapper class is reimplemented to compose an FFM _method handle_ rather than the reflected method:

```java
class NativeMethod {
    private final MethodHandle handle;
    
    public Object invoke(Object[] args) {
        try {
            return handle.invokeWithArguments(args);
        }
        catch(Throwable e) {
            throw new RuntimeException(...);
        }
    }
}
```

And a new `build` method is introduced in the factory to construct the native methods:

```java
public class NativeLibraryFactory {
    private final Linker linker = Linker.nativeLinker();
    private final SymbolLookup lookup;    

    private NativeMethod build(Method method) {
        ...
    }
}
```

Linking the method handle requires an FFM _function descriptor_ derived from the _signature_ of the native method.

First the memory layout of the method is derived from its parameters:

```java
private FunctionDescriptor descriptor(Method method) {
    MemoryLayout[] layouts = Arrays
        .stream(method.getParameterTypes())
        .map(this::layout)
        .toArray(MemoryLayout[]::new);

    ...       
}
```

Where `layout` is a temporary, quick-and-dirty mapper that just supports the integer primitives used in the prototype:

```java
private MemoryLayout layout(Class<?> type) {
    if(type == int.class) {
        return ValueLayout.JAVA_INT;
    }
    else {
        throw new IllegalArgumentException(...);
    }
}
```

The function descriptor can now be derived from the method parameters and optional return type:

```java
Class<?> returns = method.getReturnType();
if(returns == void.class) {
    return FunctionDescriptor.ofVoid(layouts);
}
else {
    return FunctionDescriptor.of(layout(returns), layouts);
}
```

Finally the method handle is linked to the native library:

```java
public NativeMethod build() {
    FunctionDescriptor descriptor = descriptor(method);
    MethodHandle handle = linker.downcallHandle(descriptor).bindTo(address);
    return new NativeMethod(handle);
}
```

In the prototype the native GLFW library is loaded by the FFM `SymbolLookup` class:

```java
SymbolLookup lookup = SymbolLookup.libraryLookup("glfw3", Arena.ofAuto());
var factory = new NativeLibraryFactory(lookup);
API api = ...
```

And the API calls should now delegate to GLFW natively when executed.  Cool.

To summarise, the new framework:

* Maps and links a user-defined Java API to a native library.

* Maps the parameters and return value of each API method to the equivalent FFM memory layout (albeit only for primitive integers so far).

* Transparently delegates API calls to the underlying native methods.

## Native Transformer

With the basic framework in place the next step is to implement marshalling support for the required types identified above.

The first attempt used the functionality provided by the `MethodHandle` class (used extensively in FFM) which supports transformation of the incoming arguments and the method return value.  This initially seemed a perfect fit, however in practice the code became convoluted very quickly.  The logic required to mutate an argument is generally also implemented as 'target' method handles, resulting in fiddly lookup and binding code for all but the most trivial cases.

After some time we abandoned this approach in favour of pre/post-processing the arguments around invocation of the native method.  The code became much clearer and easier to unit-test, at the expense of some additional processing.  Of importance is that error cases can still be trapped when the native method is being constructed.

> As it turns out some form of pre/post-processing would probably have been needed anyway, since it hard to envisage how by-reference parameters could be implemented solely using method handles.

The second attempt introduced the concept of an argument _transformer_ responsible for marshalling a Java type to its FFM equivalent:

```java
public interface Transformer<T, N> {
    /**
     * @return Memory layout of this type
     */
    MemoryLayout layout();
    
    /**
     * Marshals a method argument to its off-heap representation.
     * @param arg           Argument
     * @param allocator     Off-heap allocator
     * @return Off-heap argument
     */
    N marshal(T arg, SegmentAllocator allocator);

    /**
     * @return Transformer to unmarshal a native value
     */
    Function<N, T> unmarshal();
}
```

Where _T_ is the Java type and _N_ the native equivalent.

Primitives are marshalled as-is without transformation:

```java
public record PrimitiveTransformer<T>(ValueLayout layout) implements Transformer<T, T> {
    public T marshal(T arg, SegmentAllocator allocator) {
        return arg;
    }

    public Function<T, T> unmarshal() {
        return Function.identity();
    }
}
```

The native method class is now refactored again in terms of transformers rather than actual parameters:

```java
public class NativeMethod {
    private final MethodHandle handle;
    private final Transformer returns;
    private final Transformer[] parameters;
}
```

The factory is modified to lookup the relevant transformers for the methods signature:

```java
private NativeMethod build(Method method) {
    // Map parameters
    List<Transformer> parameters = Arrays
        .stream(method.getParameterTypes())
        .map(this::transformer)
        .toList();

    // Map return value
    Transformer returns = transformer(method.getReturnType());
    ...
}
```

Where the temporary `layout` method now returns a transformer:

```java
private static Transformer transformer(Class<?> type) {
    if(type == int.class) {
        return new PrimitiveTransformer<>(ValueLayout.JAVA_INT);
    }
    else {
        throw new IllegalArgumentException(...);
    }
}
```

And the function descriptor is now derived indirectly from the transformers:

```java
private FunctionDescriptor descriptor() {
    // Map method signature to FFM layout
    MemoryLayout[] layouts = signature
        .stream()
        .map(Transformer::layout)
        .toArray(MemoryLayout[]::new);

    // Init descriptor
    FunctionDescriptor descriptor = ...
}
```

Next the `invoke` method of the native method wrapper is modified to incorporate the additional marshalling logic:

```java    
public Object invoke(Object[] args) {
    Object[] foreign = marshal(args);
    Object result = handle.invokeWithArguments(foreign);
    return unmarshal(result);
}
```

The method arguments are marshalled by the transformers before invocation:

```java
private Object[] marshal(Object[] args) {
    if(args == null) {
        return null;
    }

    var allocator = Arena.ofAuto();
    Object[] foreign = new Object[args.length];
    for(int n = 0; n < args.length; ++n) {
        foreign[n] = parameters[n].marshal(args[n], allocator);
    }

    return foreign;
}
```

And similarly for the return value after invocation:

```java
private Object unmarshal(Object result) {
    if(returns == null) {
        return null;
    }
    else {
        return returns.unmarshal().apply(result);
    }
}
```

## Basic Transformers

To fully explore GLFW using the new framework the prototype will be extended to create a native window, which requires support for strings and the `Handle` to the window.

String values are supported by the following companion transformer:

```java
public class StringTransformer implements Transformer<String, MemorySegment> {
    public MemorySegment marshal(String str, SegmentAllocator allocator) {
        return allocator.allocateFrom(str);
    }

    public Function<MemorySegment, String> unmarshal() {
        return StringTransformer::unmarshal;
    }

    public static String unmarshal(MemorySegment address) {
        if(address.byteSize() == 0L) {
            return address.reinterpret(Integer.MAX_VALUE).getString(0L);
        }
        else {
            return address.getString(0L);
        }
    }
}
```

If the length of the string is unknown the off-heap memory is a _variable length array_ which is resized by the `reinterpret` method. Note that `reinterpret` is an _unsafe_ operation that can result in a JVM crash for an out-of-bounds memory access, since the actual memory size cannot be guaranteed by the JVM.  Additionally this method is _restricted_ and generates a runtime warning suppressed by the following VM argument:

```
--enable-native-access=ALL-UNNAMED
```

A JOVE `Handle` is reimplemented as an immutable wrapper for an FFM address:

```java
public record Handle(MemorySegment address) {
    public Handle {
        address = address.asReadOnly();
    }
}
```

With a companion transformer:

```java
public static class HandleTransformer implements Transformer<Handle, MemorySegment> {
    public MemorySegment marshal(Handle arg, SegmentAllocator allocator) {
        return arg.address;
    }

    public Function<MemorySegment, Handle> unmarshal() {
        return Handle::new;
    }
}
```

Finally the newly supported types are included in the layout mapper:

```java
private static Transformer transformer(Class<?> type) {
    if(type == int.class) {
        return new PrimitiveTransformer<>(ValueLayout.JAVA_INT);
    }
    else
    if(type == String.class) {
        return new StringTransformer();
    }
    else
    if(type == Handle.class) {
        return new HandleTransformer();
    }
    else {
        throw new IllegalArgumentException(...);
    }
}
```

The following methods are added to the prototype API:

```java
interface Prototype {
    String  glfwGetVersionString();
    void    glfwDefaultWindowHints();
    void    glfwWindowHint(int hint, int value);
    Handle  glfwCreateWindow(int w, int h, String title, Handle monitor, Handle shared);
    void    glfwDestroyWindow(Handle window);
}
```

And the prototype itself can now also be extended accordingly:

```java
// Initialise GLFW
int result = api.glfwInit();
if(result != 1) {
    throw new RuntimeException(...);
}

// Query some properties
println(api.glfwGetVersionString());
println(api.glfwVulkanSupported());

// Init window properties
api.glfwDefaultWindowHints();
api.glfwWindowHint(0x00020004, 1);      // Make window visible

// Create window
Handle window = api.glfwCreateWindow(640, 480, "Prototype", null, null);

// Cleanup
api.glfwDestroyWindow(window);
api.glfwTerminate();
```

The method to create the window passes `null` for some of the arguments, therefore the `Handle` transformer returns the special `NULL` constant in this case:

```java
public MemorySegment marshal(Handle arg, SegmentAllocator allocator) {
    if(arg == null) {
        return MemorySegment.NULL;
    }
    return arg.address;
}
```

This conditional logic will be replaced with a more holistic approach shortly.

## Registry

Before proceeding further the temporary layout mapper bodge is replaced with a _registry_ that maps a domain type to its corresponding transformer:

```java
public class Registry {
    private final Map<Class<?>, Transformer> registry = new HashMap<>();

    public Optional<Transformer> transformer(Class<?> type) {
        return Optional.ofNullable(registry.get(type));
    }
}
```

A transformer is mapped to a domain type using the following registration method:

```java
public <T> void register(Class<T> type, Transformer<? extends T, ?> transformer) {
    registry.put(type, transformer);
}
```

This allows transformer mappings to be managed programatically rather than hard-coded, and also provides a centralised point for the implementation of other use cases further down the line.

Transformers for the built-in primitive types are registered by the following helper:

```java
import static java.lang.foreign.ValueLayout.*

public record PrimitiveTransformer ... {
    public static void register(Registry registry) {
        final ValueLayout[] primitives = {
            JAVA_BYTE,
            JAVA_CHAR,
            JAVA_SHORT,
            JAVA_INT,
            JAVA_LONG,
            JAVA_FLOAT,
            JAVA_DOUBLE
        };

        for(ValueLayout layout : primitives) {
            var transformer = new PrimitiveTransformer<>(layout);
            Class carrier = layout.carrier();
            registry.register(carrier, transformer);
        }
    }
}
```

Notes:

* The Java primitive type is derived from the FFM layout via the `carrier` method.

* The `JAVA_BOOLEAN` layout is not included since `boolean` primitives turn out to be a bit of a special case, see [Native Boolean](#native-booleans)

Finally the factory is refactored to lookup transformers from the registry and the temporary code can be removed:

```java
private Transformer transformer(Method method, Class<?> type) {
    return registry
        .transformer(type)
        .orElseThrow(...);
}
```

## Integration

The GLFW library is now fully supported by the new framework, other than the following which are deferred until later:

* Callback support for GLFW devices is temporarily hacked out.

* The `glfwGetRequiredInstanceExtensions` edge case is hard-coded for the moment.

The framework is integrated into the `Desktop` service resulting in the following:

```java
public class Desktop implements TransientObject {
    private final DesktopLibrary library;

    Desktop(DesktopLibrary library) {
        this.library = library;
        init();
    }

    private void init() {
        int result = library.glfwInit();
        if(result != 1) {
            throw new RuntimeException(...);
        }
    }
}
```

Where the refactored `create` method registers the transformers required for GLFW and instantiates the proxy implementation:

```java
public static Desktop create() {
    // Init registry
    var registry = new Registry();
    PrimitiveTransformer.register(registry);
    registry.register(String.class, new StringTransformer());
    registry.register(Handle.class, new HandleTransformer());    
    
    // Build GLFW library
    var factory = new NativeLibraryFactory("glfw3", registry);
    var library = (DesktopLibrary) factory.build(List.of(DesktopLibrary.class, WindowLibrary.class));

    // Create desktop service
    return new Desktop(library);
}
```

The prototype is similarly refactored and extended to the point where the application window is created:

```java
void main() throws Exception {
    var desktop = Desktop.create();
    println(desktop.version());
    println(desktop.isVulkanSupported());

    Window window = new Window.Builder()
        .title("Prototype")
        .size(new Dimensions(1024, 768))
        .hint(Hint.CLIENT_API, 0)
        .hint(Hint.VISIBLE, 0)
        .build(desktop);
    
    window.destroy();    
    desktop.terminate();
}
```

----

# Vulkan Instance

## Introduction

Although a basic FFM-based marshalling framework is now in place, as mentioned previously Vulkan is very front-loaded with complexities.

The first step instantiates the Vulkan instance using the following native API method:

```java
VkResult vkCreateInstance(VkInstanceCreateInfo* pCreateInfo, VkAllocationCallbacks* pAllocator, VkInstance* pInstance);
```

Implementing just this first API call implies marshalling support for the following:

* structures - namely `VkInstanceCreateInfo` and `VkApplicationInfo`.

* string arrays - for the instance extensions and layers.

* integer enumerations - the `sType` fields of the structures and the `VkResult` return type.

* by-reference pointers - to 'return' the resultant `pInstance` handle.

Additionally FFM memory layouts for the required structures will need to be implemented.

The simpler dependencies will be dealt with first along with a few framework enhancements before moving on to structures.

## Framework

### Enumerations

Marshalling an integer enumeration is implemented as another companion transformer:

```java
class IntEnumTransformer implements Transformer<IntEnum, Integer> {
    private final ReverseMapping<?> mapping;

    public MemoryLayout layout() {
        return ValueLayout.JAVA_INT;
    }

    public Integer marshal(IntEnum e, SegmentAllocator allocator) {
        return e.value();
    }

    public Function<Integer, IntEnum> unmarshal() {
        return this::unmarshal;
    }

    private IntEnum unmarshal(Integer value) {
        if(value == 0) {
            return mapping.defaultValue();
        }
        else {
            return mapping.map(value);
        }
    }
}
```

Unlike previous transformers this implementation is dependant on a specific _type_ of enumeration, therefore the concept of a _transformer factory_ is introduced:

```java
public class Registry {
    @FunctionalInterface
    public interface Factory<T> {
        /**
         * Creates a transformer for the given type.
         * @param type Type
         * @return Transformer
         */
        Transformer<T, ?> transformer(Class<? extends T> type);
    }

    private final Map<Class<?>, Factory<?>> factories = new HashMap<>();
}
```

A transformer is generated from a registered factory if it is not already present in the registry:

```java
public Optional<Transformer> transformer(Class<?> type) {
    return Optional
        .ofNullable(registry.get(type))
        .or(() -> factory(type));
}
```

Where the following helper finds the factory for a given domain type:

```java
private Optional<Transformer> factory(Class<?> type) {
    return factories
        .keySet()
        .stream()
        .filter(base -> base.isAssignableFrom(type))
        .findAny()
        .map(factories::get)
        .map(factory -> factory.transformer(type));
}
```

### Empty Values

Almost all method arguments can legitimately be `null`, e.g. the monitor parameter is unused when creating a window.  However integer enumerations have a different definition of 'empty' since their native representation is a primitive integer.

To make handling empty values more robust, a new provider is added to the transformer that returns the _empty_ value for the native type:

```java
public interface Transformer<T, N> {
    default Object empty() {
        return MemorySegment.NULL;
    }
}
```

In the majority of cases (i.e. reference types) this is the `MemorySegment.NULL` value.

An integer enumeration returns the default value (usually zero) as its empty value:

```java
class IntEnumTransformer implements Transformer<IntEnum, Integer> {
    public Integer empty() {
        return mapping.defaultValue().value();
    }
}
```

Empty values are now handled in one location in the marshalling logic of the native method:

```java
private Object[] marshal(Object[] args) {
    ...
    for(int n = 0; n < args.length; ++n) {
        foreign[n] = Transformer.marshal(args[n], parameters[n], allocator);
    }
    return foreign;
}
```

Delegating to the following helper:

```java    
public Transformer {
    static Object marshal(Object value, Transformer transformer, SegmentAllocator allocator) {
        if(arg == null) {
            return transformer.empty();
        }
        else {
            return transformer.marshal(value, allocator);
        }
    }
}
```

And the existing check in the `HandleTransformer` can now be removed.

### String Arrays

To effectively verify that the instance is being created correctly in the prototype, the framework needs to support string arrays for extensions and layers.

A new operation is added to the transformer definition to generate an array transformer of that type:

```java
public interface Transformer<T, N> {
    /**
     * @return Transformer for an array of this native type
     */
    default Transformer<?, ?> array() {
        return new ArrayTransformer(this);
    }
}
```

The array transformer composes the transformer for the _component_ type of the array:

```java
public class ArrayTransformer implements Transformer<Object, MemorySegment> {
    private final Transformer component;

    public MemorySegment marshal(Object array, SegmentAllocator allocator) {
        ...
        return address;
    }

    public final Function<MemorySegment, Object> unmarshal() {
        throw new UnsupportedOperationException();
    }
}
```

Notes:

* The `unmarshal` function is disallowed since a Java array cannot be returned from a native method.

* The domain type parameter of the array transformer is simply `Object` to support primitive arrays later.

In the `marshal` method the off-heap memory is allocated according to the memory layout and length of the array:

```java
MemoryLayout layout = component.layout();
int len = Array.getLength(array);
MemorySegment address = allocator.allocate(layout, len);
```

Next a handle is obtained to iterate over the array elements:

```java
VarHandle = Transformer.removeOffset(layout.arrayElementVarHandle());
```

A handle has an offset (a _coordinate_ in FFM terms) that is always zero in our implementation, which is initialised once by the following helper:

```java
static VarHandle removeOffset(VarHandle handle) {
    return MethodHandles.insertCoordinates(handle, 1, 0L);
}
```

Each element is marshalled by the _component_ transformer and written to the off-heap memory:

```java
for(int n = 0; n < length; ++n) {
    // Get element
    Object element = Array.get(array, n);

    // Skip empty elements
    if(element == null) {
        continue;
    }

    // Transform to native type
    Object value = component.marshal(element, allocator);

    // Write off-heap memory
    handle.set(address, (long) index, value);
}
```

Note that for the moment this implementation only supports reference types, such as strings or handles.  Primitive and structure arrays have additional requirements that are addressed below.

Finally, a further step is added to the logic in the registry to return an array transformer:

```java
private Optional<Transformer> find(Class<?> type) {
    if(type.isArray()) {
        Class<?> component = type.getComponentType();
        return transformer(component).map(Transformer::array);
    }
    else {
        return factory(type);
    }
}
```

This recursive approach essentially means that registering a transformer automatically supports arrays of that type.

### Native Objects

The following transformer allows domain types derived from `NativeObject` to be used in structures and API methods:

```java
public static class NativeObjectTransformer implements Transformer<NativeObject, MemorySegment> {
    public MemorySegment marshal(NativeObject object, SegmentAllocator allocator) {
        return object.handle().address();
    }

    public Function<MemorySegment, NativeObject> unmarshal() {
        throw new UnsupportedOperationException();
    }
}
```

Note that a `NativeObject` will obviously never be returned from a native method.

Rather than registering transformers for _every_ JOVE type it makes more sense to allow the registry to fallback to a supertype where present:

```java
private Optional<Transformer> supertype(Class<?> type) {
    return registry
        .keySet()
        .stream()
        .filter(base -> base.isAssignableFrom(type))
        .findAny()
        .map(registry::get);
}
```

The method to find a transformer now becomes:

```java
private Optional<Transformer> find(Class<?> type) {
    if(type.isArray()) {
        ...
    }
    else {
        return supertype(type).or(() -> factory(type));
    }
}
```

In summary the process of determining the transformer for a given type now becomes:

1. Lookup the registered transformer if present.

2. Return an array transformer if the type is an array.

3. Derive a transformer from a supertype if available.

4. Generate a new transformer from the relevant factory if available.

5. Otherwise the type is not supported.

### Pointers

The handle of the created instance is returned as a _pointer_ specified as a by-reference `MemorySegment`, i.e. a mutable address.

The following new template class implements a by-reference parameter that holds the _pointer_ to the off-heap memory:

```java
public abstract class NativeReference<T> {
    private final AddressLayout layout;
    private MemorySegment pointer;
    private T value;
    
    protected abstract T update(MemorySegment pointer, AddressLayout layout);
}
```

The referenced value is retrieved from the off-heap memory on demand in the accessor:

```java
public T get() {
    if((value == null) && (pointer != null)) {
        value = update(pointer, layout);
    }

    return value;
}
```

Native references are marshalled by a new companion transformer:

```java
public static class NativeReferenceTransformer implements Transformer<NativeReference<?>, MemorySegment> {
    public MemorySegment marshal(NativeReference<?> ref, SegmentAllocator allocator) {
        return ref.allocate(allocator);
    }

    public Function<MemorySegment, NativeReference<?>> unmarshal() {
        throw new UnsupportedOperationException();
    }
}
```

Which allocates the off-heap pointer on demand:

```java
private MemorySegment allocate(SegmentAllocator allocator) {
    if(pointer == null) {
        pointer = allocator.allocate(ValueLayout.ADDRESS);
    }
    return pointer;
}
```

Note that a by-reference value can logically only ever be used as a method parameter.

A general `Pointer` can now be implemented for by-reference memory addresses:

```java
public class Pointer extends NativeReference<Handle> {
    /**
     * Default constructor for a simple pointer.
     */
    public Pointer() {
        super(AddressLayout.ADDRESS);
    }

    /**
     * Constructor for a memory block of the given size.
     * @param size Block size (bytes)
     */
    public Pointer(long size) {
        super(AddressLayout.ADDRESS.withTargetLayout(MemoryLayout.sequenceLayout(size, ValueLayout.JAVA_BYTE)));
    }
}
```

The layout of the off-heap address is retained in order to preserve the size of the memory, which avoids having to `reinterpret` memory segments later on.

A pointer dereferences the address of the handle:

```java
protected Handle update(MemorySegment pointer, AddressLayout layout) {
    MemorySegment address = pointer.get(layout, 0L);
    if(MemorySegment.NULL.equals(address)) {
        return null;
    }
    return new Handle(address);
}
```

Although not required yet, the implementation for an integer reference is relatively trivial to implement:

```java
public class IntegerReference extends NativeReference<Integer> {
    protected Integer update(MemorySegment pointer, AddressLayout layout) {
        return pointer.get(ValueLayout.JAVA_INT, 0L);
    }
}
```

Finally to support unit-testing the referenced value can be `set` explicitly overriding any subsequent updates.

### Integration

At this point we feel confident that the framework is ready to support Vulkan.

Refactoring the Vulkan portion of JOVE involves the following tasks:

* Remove JNA from the POM.

* Some text editor magic to strip the JNA dependencies and imports.

* A global find-and-replace to update integer and pointer references to the new classes.

* The new framework builds and validates the _whole_ API at instantiation-time, therefore the Vulkan APIs are stripped down to the minimum and reintroduced gradually as we progress.

* The code-generator will require significant rework to remove JNA and generate the FFM memory layouts of the structures.  For the moment we prefer to defer this work until the bulk of the refactoring is complete.  In the meantime some further text editor magic is used to remove the JNA dependencies and structure layouts will be hand-crafted as required.

Remarkably this refactoring work went much more smoothly than anticipated, with only a handful of compilation problems.  Apparently the existing JOVE code did a decent job of abstracting over JNA.

The approach from this point is to add Vulkan to the prototype and tackle each JOVE component in order starting with the `Instance` domain object.

After the refactoring work the instance library looks like this:

```java
interface Library {
    int     vkCreateInstance(VkInstanceCreateInfo pCreateInfo, Handle pAllocator, Pointer pInstance);
    void    vkDestroyInstance(Instance instance, Handle pAllocator);
    int     vkEnumerateInstanceExtensionProperties(String pLayerName, IntegerReference pPropertyCount, VkExtensionProperties[] pProperties);
    int     vkEnumerateInstanceLayerProperties(IntegerReference pPropertyCount, VkLayerProperties[] pProperties);
    Handle  vkGetInstanceProcAddr(Instance instance, String pName);
}
```

Note that array parameters are now explicitly declared as Java arrays, with the following benefits:

* Previously array parameters had to be specified as a __single__ instance since JNA mandated that a contiguous block was represented by the __first__ element of an array.

* The code to populate arrays in Vulkan structures can now properly use streams and collections, rather than the awkward approach JNA forced on the developer, where the array __had__ to be allocated from a temporary instance and then 'filled' element by element.

The final change is to refactor the factory method that instantiates the Vulkan library:

```java
public interface Vulkan {
    static Object create() {
        // Init API factory
        Registry registry = DefaultRegistry.create();
        var factory = new NativeLibraryFactory("vulkan-1", registry);

        // Enumerate APIs
        Class<?>[] api = {
            Instance.Library.class,
            // TODO...
        };

        // Build native Vulkan API
        return factory.build(List.of(api));
    }
```

Where `DefaultRegistry` is a helper that registers the various transformers required by Vulkan (basically all of the above).

### Return Codes

The approach of using a proxy means there is a convenient, central location where _all_ Vulkan return codes can be validated.  Previously _every_ native call had to be wrapped with the `check` helper which was ugly and slightly tedious.

First a handler for return values is added to the factory:

```java
public class NativeLibraryFactory {
    private Consumer<Object> check = result -> {};
}
```

Return values are intercepted and delegated to the configured handler after invocation:

```java
new InvocationHandler() {
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        NativeMethod delegate = methods.get(method);
        Object result = delegate.invoke(args);
        check.accept(result);
        return result;
    }
};
```

In the Vulkan `create` method a handler can now be configured that validates all return codes:

```java
Consumer<Object> check = code -> {
    if((code instanceof VkResult result) && (result != VkResult.VK_SUCCESS)) {
        throw new VulkanException(result);
    }
};
factory.handler(check);
```

Nice.

## Structures

### Structure Layout

With all the above framework changes in place, we can now finally address marshalling of structures to implement creation of the Vulkan instance.

The following interface is introduced as the supertype of all JOVE structures:

```java
public interface NativeStructure {
    /**
     * @return Memory layout of this structure
     */
    StructLayout layout();
}
```

After the text editor fiddling mentioned above the instance descriptor now looks like this:

```java
public class VkInstanceCreateInfo implements NativeStructure {
    public VkStructureType sType = VkStructureType.INSTANCE_CREATE_INFO;
    public Handle pNext;
    public int flags;
    public VkApplicationInfo pApplicationInfo;
    public int enabledLayerCount;
    public String[] ppEnabledLayerNames;
    public int enabledExtensionCount;
    public String[] ppEnabledExtensionNames;
}
```

With the following memory layout:

```java
public StructLayout layout() {
    return MemoryLayout.structLayout(
        JAVA_INT.withName("sType"),
        PADDING,
        POINTER.withName("pNext"),
        JAVA_INT.withName("flags"),
        PADDING,
        POINTER.withName("pApplicationInfo"),
        JAVA_INT.withName("enabledLayerCount"),
        PADDING,
        POINTER.withName("ppEnabledLayerNames"),
        JAVA_INT.withName("enabledExtensionCount"),
        PADDING,
        POINTER.withName("ppEnabledExtensionNames")
    );
}
```

The convenience `POINTER` constant defines a native pointer to other structures, strings, variable-length arrays, etc:

```java
public interface NativeStructure {
    AddressLayout POINTER = ValueLayout.ADDRESS.withTargetLayout(MemoryLayout.sequenceLayout(Integer.MAX_VALUE, ValueLayout.JAVA_BYTE));
}
```

And `PADDING` is used to align fields to the word size of a native structure (8 bytes by default):

```java
MemoryLayout PADDING = MemoryLayout.paddingLayout(4);
```

The layout for the application details descriptor is as follows:

```java
public StructLayout layout() {
    return MemoryLayout.structLayout(
        JAVA_INT.withName("sType"),
        PADDING,
        POINTER.withName("pNext"),
        POINTER.withName("pApplicationName"),
        JAVA_INT.withName("applicationVersion"),
        PADDING,
        POINTER.withName("pEngineName"),
        JAVA_INT.withName("engineVersion"),
        PADDING,
        JAVA_INT.withName("apiVersion"),
        PADDING
    );
}
```

Notes:

* The memory layouts and constants are copied from the code generated by the `jextract` tool earlier.

* The JOVE code generator will need to track the byte alignment of each structure and inject padding as required.

* The `layout` is temporarily made a `default` method (throwing an exception) to allow the remaining structures to compile without errors.

> We considered completely overhauling native structures by code-generating builders (as LWJGL does) to replace the fully mutable public fields.  However since structures are generally only used as internal data carriers the refactoring effort would probably outweigh any benefit of slightly better encapsulation.

### Structure Transformer

The transformer for a structure composes a list of _field mappings_ that marshal its fields to off-heap memory:

```java
public class StructureTransformer implements Transformer<NativeStructure, MemorySegment> {
    private final Supplier<NativeStructure> factory;
    private final GroupLayout layout;
    private final List<FieldMapping> mappings;

    public MemoryLayout layout() {
        return layout;
    }

    public MemorySegment marshal(NativeStructure structure, SegmentAllocator allocator) {
        ...
    }
}
```

The `marshal` method allocates off-heap memory given the structure layout and then iteratively marshals each field:

```java
public MemorySegment marshal(NativeStructure structure, SegmentAllocator allocator) {
    MemorySegment address = allocator.allocate(layout);
    for(FieldMapping f : mappings) {
        f.marshal(structure, address, allocator);
    }
    return address;
}
```

A _field mapping_ composes handles to the structure field and the transformer:

```java
class FieldMapping {
    private final VarHandle local;
    private final VarHandle foreign;
    private final Transformer transformer;
}
```

Where _local_ and _foreign_ are the handles to the structure field and the off-heap memory respectively.

The process of marshalling a field is:

1. Retrieve the field value from the _local_ handle.

2. Apply the transformer to marshal this value to its off-heap representation.

3. Write the transformed value to the off-heap memory via the _foreign_ handle.

This is implemented as follows:

```java
public void marshal(NativeStructure structure, MemorySegment address, SegmentAllocator allocator) {
    Object value = local.get(structure);
    Object result = Transformer.marshal(value, transformer, allocator);
    foreign.set(address, result);
}
```

### Transformer Factory

A structure has a class-specific transformer which is constructed by the following factory:

```java
public class StructureTransformerFactory implements Registry.Factory<NativeStructure> {
    private final Registry registry;

    public StructureTransformer transformer(Class<? extends NativeStructure> type) {
        ...
    }
}
```

First the constructor is retrieved which is used to instantiate new instances of the structure:

```java
Supplier<NativeStructure> factory = () -> type.getConstructor().newInstance();
```

Notes:

* Exception handling and helper methods have been omitted for brevity.

* This implementation assumes a _default constructor_ which is a common constraint for a marshalling framework.

* The _factory_ will also be used later when unmarshalling structures.

A transient instance is then created in order to retrieve the memory layout of the structure:

```java
NativeStructure instance = factory.get();
GroupLayout layout = instance.layout();
```

Next the field mappings are constructed from this layout (see below):

```java
var builder = new FieldMappingBuilder(type, layout);
List<FieldMapping> mappings = builder.build();
```

And finally these components are then composed into the transformer:

```java
return new StructureTransformer(factory, layout, mappings);
```

### Field Mappings

A field mapping is generated for each _atomic_ member of the layout:

```java
private List<FieldMapping> build() {
    return layout
        .memberLayouts()
        .stream()
        .filter(member -> member instanceof ValueLayout)
        .map(this::mapping)
        .toList();
}
```

In the `mapping` method the field name is extracted from the layout:

```java
String name = member.name().orElseThrow(...);
```

The handle to the _local_ structure field is then reflected from the structure class:

```java
Field field = structure.getField(name);
VarHandle local = MethodHandles.lookup().unreflectVarHandle(field);
```

Next a handle to the _foreign_ or off-heap field is retrieved from the layout:

```java
PathElement path = PathElement.groupElement(name);
VarHandle foreign = Transformer.removeOffset(layout.varHandle(path));
```

And finally the handles and transformer are composed into the mapping:

```java
var transformer = registry.transformer(type).orElseThrow(...);
return new FieldMapping(local, transformer, marshal);
```

Notes:

* Pointers to other structures (e.g. the `pApplicationInfo` field of the instance descriptor) are automatically handled by this mechanism.

* The structure layout conveniently specifies the order of the fields _explicitly_ since it must obviously match the native representation exactly (including alignment).  This is unlike reflection where the order is arbitrary, hence there is no need for anything like the `@FieldOrder` annotation that was required for JNA structures.

It is worth considering that the `marshal` method allocates and populates the off-heap memory on _every_ invocation, with the memory being discarded afterwards.  A future enhancement _could_ cache and reuse the memory address and/or implement some sort of synchronisation mechanism (which is the JNA approach), but this would be quite complex and is not required for the forseeable future.  In reality native structures only have scope during marshalling, therefore the simpler approach seems more palatable.

The prototype can now be extended to instantiate Vulkan and configure the instance:

```java
VulkanLibrary vulkan = VulkanLibrary.create();

Instance instance = new Instance.Builder()
    .name("Prototype")
    .extension(DiagnosticHandler.EXTENSION)
    .layer(Vulkan.STANDARD_VALIDATION)
    .build(vulkan);
```

## Diagnostic Handler

### Handler Library

Refactoring the diagnostic handler presents a couple of new challenges:

* The native methods to create and destroy a handler must be created programatically, since the address is a _function pointer_ instead of a symbol looked up from the native library.

* A handler requires an FFM callback _stub_ to allow Vulkan to report diagnostic messages to the application.

The majority of the existing handler code remains as-is except for:

* Invocation of the create and destroy function pointers.

* Creation of the message callback.

* Unmarshalling of the diagnostic report structure to construct the `Message` record.

Although the diagnostic handler uses a different mechanism to lookup the memory address of its methods, the existing framework can be reused by the introduction of the following hand-crafted library:

```java
private interface HandlerLibrary {
    VkResult vkCreateDebugUtilsMessengerEXT(Instance instance, VkDebugUtilsMessengerCreateInfoEXT pCreateInfo, Handle pAllocator, Pointer pHandler);
    void vkDestroyDebugUtilsMessengerEXT(Instance instance, DiagnosticHandler handler, Handle pAllocator);
}
```

First the `function` method of the `Instance` is reintroduced, which turns out to be trivial:

```java
public Optional<Handle> function(String name) {
    Handle function = vulkan.vkGetInstanceProcAddr(this, name);
    return Optional.ofNullable(function);
}
```

In the builder for the diagnostics handler the native methods are retrieved via a custom lookup implementation:

```java
private DiagnosticHandler create(VkDebugUtilsMessengerCreateInfoEXT info, Instance instance) {
    SymbolLookup lookup = name -> instance
        .function(name)
        .map(Handle::address);
        
    ...
}
```

The existing framework can now be used to construct a proxy implementation of the _pseudo_ library:

```java
var factory = new NativeLibraryFactory(lookup, registry);
var library = (HandlerLibrary) factory.build(List.of(HandlerLibrary.class));
```

A handler is instantiated by invoking the relevant function pointer:

```java
var handle = new Pointer();
library.vkCreateDebugUtilsMessengerEXT(instance, info, null, handle);
return new DiagnosticHandler(handle.get(), instance, library);
```

And finally the class itself is refactored accordingly:

```java
public class DiagnosticHandler extends TransientNativeObject {
    private final Instance instance;
    private final HandlerLibrary library;

    protected void release() {
        library.vkDestroyDebugUtilsMessengerEXT(instance, this, null);
    }
}
```

This approach nicely reuses the existing framework rather than having to implement more FFM code from the ground up.

### Callback Address

To build the diagnostic callback a new method is added to the existing class that provides the off-heap _address_ of the callback method itself:

```java
private record MessageCallback(Consumer<Message> consumer, Transformer<NativeStructure> transformer) {
    MemorySegment address() {
        var signature = MethodType.methodType(
            boolean.class,
            int.class,
            int.class,
            MemorySegment.class,
            MemorySegment.class
        );
        MethodHandle handle = MethodHandles.lookup().findVirtual(MessageCallback.class, "message", signature);
        ...
    }
}
```

Which corresponds to the signature of the callback defined in the Vulkan header:

```java
typedef VkBool32 (VKAPI_PTR *PFN_vkDebugUtilsMessengerCallbackEXT)(
	VkDebugUtilsMessageSeverityFlagBitsEXT           messageSeverity,
	VkDebugUtilsMessageTypeFlagsEXT                  messageTypes,
	const VkDebugUtilsMessengerCallbackDataEXT*      pCallbackData,
	void*                                            pUserData);
```

The resultant method handle is then bound to the callback _instance_ in order to delegate diagnostic reports to the relevant handler:

```java
MethodHandle binding = handle.bindTo(this);
```

This handle is then linked to an up-call stub with a function descriptor matching the callback signature:

```java
private static MemorySegment link(MethodHandle handle) {
    var descriptor = FunctionDescriptor.of(
        JAVA_BOOLEAN,
        JAVA_INT,
        JAVA_INT,
        ADDRESS,
        ADDRESS
    );
    return Linker.nativeLinker().upcallStub(binding, descriptor, Arena.global());
}
```

Note that that linker uses the _global_ arena in this case, otherwise the upcall fails.

The `build` method of the handler is modified to lookup the transformer for the message structure:

```java
public DiagnosticHandler build() {
    ...
    var transformer = registry.get(VkDebugUtilsMessengerCallbackDataEXT.class);
    var callback = new MessageCallback(consumer, transformer);
    var info = populate(callback.address());
    ...
}
```

And the address of the callback method is populated in the descriptor for the handler:

```java
private VkDebugUtilsMessengerCreateInfoEXT populate(MemorySegment callback) {
    var info = new VkDebugUtilsMessengerCreateInfoEXT();
    ...
    info.pfnUserCallback = new Handle(callback);
    return info;
}
```

A similar approach will be used when GLFW device listeners are reintroduced later.

### Diagnostic Report

The last required change is to unmarshal the diagnostic report received by the `message` callback:

```java
public boolean message(...) {
    long size = transformer.layout().byteSize();
    MemorySegment segment = pCallbackData.reinterpret(size);
    var data = (VkDebugUtilsMessengerCallbackDataEXT) transformer.unmarshal(segment);
    ...
}
```

TODO - requires fiddle for returned arrays

In this case the off-heap memory needs to be explicitly resized to match the expected structure.

Unmarshalling a structure field is essentially the inverse of the `marshal` method:

```java
class FieldMapping {
    public void unmarshal(MemorySegment address, NativeStructure structure) {
        Object arg = foreign.get(address);
        Object value = transformer.unmarshal().apply(arg);
        local.set(structure, value);
    }
}
```

And similarly for the structure itself in the transformer:

```java
public Function<MemorySegment, NativeStructure> unmarshal() {
    return address -> {
        NativeStructure structure = factory.get();
        for(FieldMapping f : fields) {
            f.unmarshal(address, structure);
        }
        return structure;
    };
}
```

Note that the `factory` constructed above is also used here to instantiate the new structure instance.

Diagnostic handlers can now be attached to the instance as before, which is very useful given the invasive changes being made to JOVE.  A good test of the refactor is to temporarily remove the code that releases attached handlers when the instance is destroyed, which should result in Vulkan complaining about orphaned objects.

There is a fair amount of hard-coded FFM logic here that will (hopefully) be simplified when callbacks for GLFW are addressed later.

----

# Physical Device

## Overview

The next Vulkan component to refactor is the physical device which uses native methods that are highly dependant on by-reference parameters:

1. Enumerating the available physical devices via the `vkEnumeratePhysicalDevices` API method returns a by-reference _array_ of device handles.

2. The queue families for each device are retrieved using `vkGetPhysicalDeviceQueueFamilyProperties` which returns a by-reference _structure array_ of the `VkQueueFamilyProperties` structure.

3. Querying device properties generally returns a _structure_ by-reference, for example `vkGetPhysicalDeviceFeatures`.

All three cases require marshalling logic to _update_ the method arguments _after_ invocation, essentially overwriting the structure fields or array elements provided by the caller.

## Framework

This introduces an ambiguity since by-reference and 'normal' parameters cannot be distinguished solely from the method signature.  JNA used the rather cumbersome `ByReference` interface which generally required _every_ structure to have _two_ implementations.  Additionally JNA essentially considered _every_ array as a by-reference parameter, where we instead prefer to make the intent explicit.

The most obvious solution is to use a custom annotation to explicitly declare by-reference parameters:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Updated {
}
```

Notes:

* This is a simple _marker_ interface with no declared functionality.

* This is probably a poor name, however by-reference is ugly in camel-case and would probably overlap with the JNA equivalent.

* The Vulkan API is manually refactored to declare by-reference parameters using the new annotation.

The refactored library for the physical device is now as follows:

```java
interface Library {
    VkResult    vkEnumeratePhysicalDevices(Instance instance, IntegerReference pPhysicalDeviceCount, @Updated Handle[] devices);
    void        vkGetPhysicalDeviceQueueFamilyProperties(Handle device, IntegerReference pQueueFamilyPropertyCount, @Updated VkQueueFamilyProperties[] pQueueFamilyProperties);
    void        vkGetPhysicalDeviceProperties(PhysicalDevice device, @Updated VkPhysicalDeviceProperties props);
    void        vkGetPhysicalDeviceMemoryProperties(PhysicalDevice device, @Updated VkPhysicalDeviceMemoryProperties pMemoryProperties);
    void        vkGetPhysicalDeviceFeatures(Handle device, @Updated VkPhysicalDeviceFeatures features);
    VkResult    vkEnumerateDeviceExtensionProperties(PhysicalDevice device, String layer, IntegerReference count, @Updated VkExtensionProperties[] extensions);
    VkResult    vkEnumerateDeviceLayerProperties(PhysicalDevice device, IntegerReference count, @Updated VkLayerProperties[] layers);
    void        vkGetPhysicalDeviceFormatProperties(PhysicalDevice device, VkFormat format, @Updated VkFormatProperties props);
}
```

Other than refactoring the library the remainder of the `PhysicalDevice` class remains as-is.

A _native parameter_ is introduced that composes the transformer and a flag indicating whether the annotation is present:

```java
private NativeParameter parameter(Parameter parameter) {
    Transformer transformer = registry.transformer(parameter.getType()).orElseThrow(...);
    final boolean updated = parameter.isAnnotationPresent(Updated.class);
    return new NativeParameter(transformer, updated);
}
```

A by-reference parameter is unmarshalled via a new operation on the transformer interface:

```java
public interface Transformer<T, N> {
    default BiConsumer<N, T> update() {
        throw new UnsupportedOperationException();
    }
}
```

The `update` function is extracted from the transformer as specified by the flag, both caching the function and throwing an exception if the type cannot be used by-reference:

```java
public static class NativeParameter {
    private final Transformer transformer;
    private final BiConsumer update;

    public NativeParameter(Transformer transformer, boolean updated) {
        if(updated) {
            this.update = transformer.update();
        }
        else {
            this.update = null;
        }
    }
}
```

And new step is added in the native method to perform the update after invocation:

```java
public Object invoke(Object[] args) {
    ...
    Object result = invoke(foreign);
    update(args, foreign);
    return result;
}
```

The `update` method delegates to the transformer to overwrite each by-reference parameter as required:

```java
private void update(Object[] args, Object[] foreign) {
    for(int n = 0; n < parameters.length; ++n) {
        // Skip empty arguments
        if(args[n] == null) {
            continue;
        }

        // Skip empty off-heap arguments
        if(MemorySegment.NULL.equals(foreign[n])) {
            continue;
        }

        // Skip normal parameters
        if(!parameters[n].isUpdated()) {
            continue;
        }

        // Overwrite by-reference arguments
        parameters[n].update.accept(foreign[n], args[n]);
    }
}
```

The framework now supports the general notion of by-reference parameters and the individual use-cases identified above can now be addressed.

Note that the `IntegerReference` and `Pointer` types are special cases that operate independently of the above.

## Updated Parameters

### Reference Arrays

The `vkEnumeratePhysicalDevices` method returns a by-reference array of device handles marshalled by the array transformer:

```java
public class ArrayTransformer implements Transformer<Object, MemorySegment> {
    private final Transformer component;
    private final VarHandle handle;

    public BiConsumer<MemorySegment, Object> update() {
        return (address, array) -> {
            int length = Array.getLength(array);
            update(address, array, length);
        };
    }
}
```

Which unmarshals and overwrites each element of the array:

```java
protected void update(MemorySegment address, Object array, int length) {
    for(int n = 0; n < length; ++n) {
        // Retrieve off-heap element
        Object foreign = handle.get(address, (long) n);
    
        // Skip empty elements
        if(MemorySegment.NULL.equals(foreign)) {
            continue;
        }
    
        // Unmarshal element
        Object element = component.unmarshal().apply(foreign);
    
        // Write to array
        Array.set(array, n, element);
    }
}
```

Notes:

* The array itself is assumed to have already been instantiated by the caller prior to invocation.  In this case the array is allocated by the `VulkanFunction` since this API method employs the _two stage invocation_ approach.

* The `unmarshal` operation of the component transformer is reused to unmarshal each element from the off-heap memory.

### Structure Arrays

An API method that returns a by-reference array of structures such as `vkGetPhysicalDeviceQueueFamilyProperties` requires the array to be treated as a _compound_ type:

1. The off-heap array is a contiguous block of memory.

2. Each element of the off-heap array is itself a compound block of memory.

The elements of an array with a reference component type (such as strings) can be accessed using a `VarHandle`.  Unfortunately FFM does not seem to provide an equivalent mechanism to access structure elements.  Array handles _can_ be retrieved for the individual __fields__ of a structure array but not for the elements themselves.  It is worth noting that for these cases the `jextract` equivalent generally abandons handles and accesses the off-heap memory directly, often using nasty hard-coded byte offsets.

Therefore the following abstraction is introduced to the array transformer to retrieve and modify the off-heap array elements for the different cases:

```java
protected interface ElementAccessor {
    Object get(MemorySegment address, int index);
    void set(MemorySegment address, int index, Object element);
}
```

The existing handle-based code is factored out for arrays with an _atomic_ component, i.e. types that are __not__ a structure, usually handles or strings:

```java
class DefaultElementAccessor implements ElementAccessor {
    private final VarHandle handle;

    public DefaultElementAccessor(MemoryLayout layout) {
        this.handle = Transformer.removeOffset(layout.arrayElementVarHandle());
    }

    public void set(MemorySegment address, int index, Object element) {
        handle.set(address, (long) index, element);
    }

    public Object get(MemorySegment address, int index) {
        return handle.get(address, (long) index);
    }
}
```

A specialised implementation can now be added for structures using _slices_ of the off-heap memory directly rather than a handle:

```java
private class StructureElementAccessor implements ElementAccessor {
    private final long size = layout.byteSize();

    public void set(MemorySegment address, int index, Object element) {
        MemorySegment slice = get(address, index);
        slice.copyFrom(element);
    }

    public MemorySegment get(MemorySegment address, int index) {
        return address.asSlice(index * size, size);
    }
}
```

Which is used when creating an array transformer for a structure:

```java
public Transformer<?, ?> array() {
    return new ArrayTransformer(this, new StructureElementAccessor());
}
```

Other than the new element accessor and some slight refactoring to the array transformer class the above plugs into the existing framework to deliver by-reference structure arrays.
This also provides the basis for further specialised implementations for primitive arrays later.

### Structures

The final use case handles a by-reference parameter for a __single__ structure, for example:

```java
public VkPhysicalDeviceProperties properties() {
    var properties = new VkPhysicalDeviceProperties();
    library.vkGetPhysicalDeviceProperties(this, properties);
    return properties;
}
```

The structure transformer implements the `update` operation accordingly:

```java
public class StructureTransformer implements Transformer<NativeStructure, MemorySegment> {
    public BiConsumer<MemorySegment, NativeStructure> update() {
        return this::update;
    }
}
```

Which iteratively unmarshals each structure field:

```java
private void update(MemorySegment address, NativeStructure structure) {
    for(FieldMapping f : mappings) {
        f.unmarshal(address, structure);
    }
}
```

Note that again this code assumes that the structure has been instantiated by the caller.

## Nested Structures

There is another 'hidden' use-case that only became apparent during refactoring of the physical device.

The queue families for a given device are specified by the following structure:

```java
public class VkQueueFamilyProperties implements NativeStructure {
    public EnumMask<VkQueueFlag> queueFlags;
    public int queueCount;
    public int timestampValidBits;
    public VkExtent3D minImageTransferGranularity;
}
```

The `minImageTransferGranularity` field is actually a _nested_ structure which can be seen from the memory layout:

```java
public StructLayout layout() {
    return MemoryLayout.structLayout(
        JAVA_INT.withName("queueFlags"),
        JAVA_INT.withName("queueCount"),
        JAVA_INT.withName("timestampValidBits"),
        MemoryLayout.structLayout(
            JAVA_INT.withName("width"),
            JAVA_INT.withName("height"),
            JAVA_INT.withName("depth")
        ).withName("minImageTransferGranularity")
    );
}
```

In the native structure the `VkExtent3D` fields are essentially _embedded_ into the parent structure, whereas in the Java class this field obviously __has__ to be a reference since there is no analog of embedding in Java.  The structure transformer will need to support both _atomic_ fields (i.e. primitives and references) and nested structures, which unfortunately implies another level of abstraction.

In the `StructureTransformerFactory` the code that walks through the layout is first modified to include all members except for alignment padding:

```java
private List<FieldMapping> build() {
    return layout
        .memberLayouts()
        .stream()
        .filter(Predicate.not(e -> e instanceof PaddingLayout))
        .map(instance::mapping)
        .toList();
}
```

The `mapping` method is refactored accordingly to switch behaviour depending on the type of each member:

```java
private FieldMapping mapping(MemoryLayout member) {
    ...
    FieldMarshal marshal = switch(member) {
        case ValueLayout _              -> value(path);
        case SequenceLayout sequence    -> ...
        case GroupLayout nested         -> nested(path, nested);
        default -> throw new IllegalArgumentException(...);
    };
    return new FieldMapping(local, transformer, marshal);
}
```

The `SequenceLayout` case covers fixed or variable length array fields which are dealt with [later](#sequence-layout).

The `FieldMapping` class is refactored to delegate to a new abstraction that is responsible for marshalling the different structure field cases:

```java
interface FieldMarshal {
    void marshal(Object value, Transformer transformer, MemorySegment address, SegmentAllocator allocator);
    Object unmarshal(MemorySegment address, Transformer transformer);
}
```

And the existing case for a `ValueLayout` member becomes:

```java
private FieldMarshal value(PathElement path) {
    VarHandle foreign = layout.varHandle(path);
    return new AtomicFieldMarshal(foreign);
}
```

Where the code to marshal an _atomic_ structure field is factored out to a new implementation:

```java
record AtomicFieldMarshal(VarHandle handle) implements FieldMarshal {
    public void marshal(Object value, Transformer transformer, MemorySegment address, SegmentAllocator allocator) {
        Object result = Transformer.marshal(value, transformer, allocator);
        handle.set(address, result);
    }

    public Object unmarshal(MemorySegment address, Transformer transformer) {
        Object value = handle.get(address);

        if(MemorySegment.NULL.equals(value)) {
            return null;
        }

        return transformer.unmarshal().apply(value);
    }
}
```

The mapping for a nested structure field can now be created as follows:

```java
private FieldMarshal nested(PathElement path, GroupLayout nested) {
    long offset = layout.byteOffset(path);
    long size = nested.byteSize();
    return new SliceFieldMarshal(offset, size);
}
```

Which uses a second field marshalling implementation operating on a _slice_ of the off-heap memory:

```java
class SliceFieldMarshal implements FieldMarshal {
    private final long offset;
    private final long size;

    protected MemorySegment slice(MemorySegment address) {
        return address.asSlice(offset, size);
    }
}
```

The nested structure field is marshalled by first delegating to the structure transformer and then _copying_ the result to the off-heap memory:

```java
public void marshal(Object value, Transformer transformer, MemorySegment address, SegmentAllocator allocator) {
    if(value == null) {
        return;
    }

    var result = (MemorySegment) transformer.marshal(value, allocator);
    
    MemorySegment slice = slice(address);
    slice.copyFrom(result);
}
```

And unmarshalling performs the inverse operation:

```java
public Object unmarshal(MemorySegment address, Transformer transformer) {
    return transformer.unmarshal().apply(slice(address));
}
```

There seems to be considerable overlap between the new `FieldMarshal` implementations and the equivalent `FieldAccessor` used in the array transformer.  A potential future improvement could look at whether both of these can be condensed into a single abstraction, but the above works fine for now.

## Sequence Layout

TODO

VkPhysicalDeviceMemoryProperties - actual arrays
VkPhysicalDeviceProperties - arrays -> string
VkDeviceQueueCreateInfo - primitive arrays

## Conclusion

At this point the FFM-based framework now fully supports the bulk of the requirements for both GLFW and Vulkan, with the addition of some [miscellaneous](#miscellaneous) deferred or edge cases covered at the end of the chapter.

The prototype is discarded and refactoring continues with the existing suite of demo application.

The last major chunk of work is refactoring of the code generator which is covered in the next section.

----

# Code Generation

## Background

not into detail since
refactor previous iteration based on JNA
parser -> metadata remains roughly same
possibly replace with proper grammar / AST approach, e.g. ANTLR

major change is introduction of generation of the FFM memory layout for structures

issues faced, ambiguities:

enumerations as integer or as bitfields, cannot be done in Java, one or other - not both
bitfield named FlagBits
but typedefs -> VkFlags end with Flags
but not consistent anyway
and also some typedefs for types that do not actually exist, e.g. VkInstanceCreateFlags in VkInstanceCreateInfo
basically a mess

C pointers and arrays are indistinguishable
ptr to single structure:		const VkApplicationInfo* pApplicationInfo;
ptr to array of structures:		const VkDeviceQueueCreateInfo* pQueueCreateInfos;
ptr to primitive array:			const float* pQueuePriorities;

ambiguous / weird mappings
e.g. shader pCode is a uint32_t* but is SPIV bytes WTF!
but pQueueFamilyIndices is a pointer to an int[]

## Type Mapping

To support the changes for structure generation, the existing type mapper is completely overhauled

TODO ~ tidy up the code!!!

use cases

primitives
standard C types, e.g. uint32_t
char*
char**
void*
VkBool32 special case (again)
pointers ~ plurality -> handle or array, ropey but works (just)
enumerations/masks - nasty
char[] -> String

summarise in table again? simpler than the code!!!



```java
record StructureField<T>(String name, T type, int length)
```

```java
public record NativeType(String name, MemoryLayout layout)
```

## Structure Generator

### Overview

The major change to the structure generator is the addition of the requirement to also generate the source code for the FFM memory layout.

This will entail:

* Refactoring the existing generator in terms of the revised type mapper.

* Generation of the memory layout from the mapped structure fields.

* Additional logic to correctly align data within memory.

The design for generation of the memory layout splits the process into two steps:

1. Generate an actual FFM memory layout from the structure fields.

2. Transform the layout to source code.

While this may seem a little backwards (and the process could easily be done in one transformation), generating an FFM layout will leverage the built-in validation for possible bugs such as invalid field or structure alignment, bad layouts, etc.  The resultant code should also be more coherent and easier to unit-test.

### Refactor

First the domain type of each field is determined from the type mapper:

```java
List<StructureField<NativeType>> fields = structure
	.fields()
	.stream()
	.map(this::map)
	.toList();
```

Where `map` TODO

```java
private StructureField<NativeType> map(StructureField<String> field) {
	TODO
}
```

The template arguments are now derived from the mapped fields:

```java
Map<String, Object> arguments = fields
	.stream()
	.map(StructureGenerator::arguments)
	.toList();
```

Which delegates to the following helper:

```java
private static Map<String, Object> arguments(StructureField<NativeType> field) {
	return Map.of(
		"name",	field.name(),
		"type",	field.type().name()
	);
}
```

And the final set of arguments is constructed, with the addition of the new `layout` argument (implemented below):

```java
return Map.of(
	"name",		structure.name(),
	"fields",	arguments,
	"layout",	layout
);
```

### Alignment

_Data alignment_ is the aligning of fields according to their _natural alignment_, meaning the memory location __must__ be some multiple of the _word_ size of the platform (8 bytes by default).  See [Wikipedia](https://en.wikipedia.org/wiki/Data_structure_alignment).

The natural alignment for each data type can be summarised as follows:

* bytes - none.

* short - even bytes only.

* int - either the lower or upper 4 bytes

* long / address - beginning of a word boundary.

This may require the injection of _padding_:

1. _between_ the fields of a structure.

2. or at the end of a structure accessed via a pointer (or implicitly as an array element).

Note that this second constraint does not apply to nested structures since their fields are essentially 'embedded' into the parent structure.

These alignment constraints are implemented by the following helper:

```java
class FieldAlignment {
	private final long word;

	private long total;
	private long max;

	public FieldAlignment() {
		this(8);
	}

	public FieldAlignment(int word) {
		if(!MathsUtility.isPowerOfTwo(word)) {
			throw new IllegalArgumentException(...);
		}
		this.word = requireOneOrMore(word);
	}
}
```

The `padding` method calculates the amount of padding required for each field:

```java
public long padding(MemoryLayout layout) {
	// Determine required padding for the next layout
	long alignment = layout.byteAlignment();
	long padding = (alignment - (total % word)) % alignment;

	// Track alignment
	total += padding + layout.byteSize();
	max = Math.max(max, alignment);

	return padding;
}
```

Note that this is a mutable class that accumulates the alignment as a side effect:

* The `total` is the overall size of the structure.

* And `max` tracks the size of its _largest_ member.

The padding required to align the overall structure is calculated by a second method:

```java
public long padding() {
	if(total == 0) {
		return 0;
	}
	return total % max;
}
```

### Layout Builder

A new component builds the FFM memory layout for a given structure:

```java
var builder = new LayoutBuilder();
GroupLayout group = builder.layout(structure.name(), structure.group(), fields);
```

The builder iterates through the structure and determines the memory layout of each field, injecting padding as required:

```java
public GroupLayout layout(String name, GroupType type, List<StructureField<NativeType>> fields) {
	// Build field layouts and inject alignment padding as required
	MemoryLayout[] layouts = fields
		.stream()
		.map(LayoutBuilder::layout)
		.gather(padding())
		.toArray(MemoryLayout[]::new);

	// Build group layout
	GroupLayout group = switch(type) {
		case STRUCT -> MemoryLayout.structLayout(layouts);
		case UNION	-> MemoryLayout.unionLayout(layouts);
	};

	return group;
}
```

The `layout` method extracts the relevant information from the mapped structure field:

```java
private static MemoryLayout layout(StructureField<NativeType> field) {
	MemoryLayout layout = field.type().layout();
	MemoryLayout actual = layout(layout, field.length());
	return actual.withName(field.name());
}
```

Where array fields are wrapped as a sequence layout:

```java
private static MemoryLayout layout(MemoryLayout layout, int length) {
	if(length == 0) {
		return layout;
	}
	else {
		return MemoryLayout.sequenceLayout(length, layout);
	}
}
```

The alignment helper is wrapped as a stream gatherer to inject padding into the resultant layout:

```java
private static Gatherer<MemoryLayout, FieldAlignment, MemoryLayout> padding() {
	return Gatherer.ofSequential(
		FieldAlignment::new,
		LayoutBuilder::inject,
		LayoutBuilder::append
	);
}
```

The `inject` method is an _integrator_ that applies the alignment logic for each field:

```java
private static boolean inject(FieldAlignment alignment, MemoryLayout layout, Downstream<? super MemoryLayout> downstream) {
	// Inject padding as required
	long padding = alignment.padding(layout);
	add(padding, downstream);

	// Append field layout
	downstream.push(layout);

	return true;
}
```

Where `add` injects padding into the resultant structure layout:

```java
private static void add(long alignment, Downstream<? super MemoryLayout> downstream) {
	if(alignment > 0) {
		var padding = MemoryLayout.paddingLayout(alignment);
		downstream.push(padding);
	}
}
```

Conveniently a gatherer has a _finisher_ which is used to append padding as required to align the structure to the size of the largest member:

```java
private static void append(FieldAlignment alignment, Downstream<? super MemoryLayout> downstream) {
	add(alignment.padding(), downstream);
}
```

### Layout Writer

A second new component builds the source code for the generated structure layout:

```java
var writer = new LayoutWriter();
String layout = writer.write(group);
```

The writer prints the layout of each structure member:

```java
String fields = group
	.memberLayouts()
	.stream()
	.map(this::member)
	.collect(joining(","));
```

Where `member` outputs the textual representation of the layout with a quoted field name declaration:

```java
private String member(MemoryLayout layout) {
	if(layout instanceof PaddingLayout padding) {
		return write(padding);
	}

	String name = layout.name().orElseThrow(...);
	return String.format("%s.withName(%s)", write(layout), quote(name));
}
```

The writer switches over the sealed type to recursively generate the source code for a given layout:

```java
protected String write(MemoryLayout layout) {
	return switch(layout) {
		case AddressLayout _			-> "POINTER";
		case ValueLayout value			-> write(value);
		case SequenceLayout sequence	-> write(sequence);
		case GroupLayout group			-> write(group);
		case PaddingLayout padding		-> write(padding);
	};
}
```

Primitive types are output as the classname of the carrier type:

```java
private static String write(ValueLayout value) {
	return "JAVA_" + value.carrier().getSimpleName().toUpperCase();
}
```

An array is represented by a sequence layout:

```java
private String write(SequenceLayout sequence) {
	long length = sequence.elementCount();
	MemoryLayout component = sequence.elementLayout();
	return String.format("MemoryLayout.sequenceLayout(%d, %s)", length, write(component));
}
```

And padding layouts use the convenience `PADDING` constant where applicable:

```java
private static String write(PaddingLayout padding) {
	if(padding.byteSize() == 4) {
		return "PADDING";
	}
	else {
		return String.format("MemoryLayout.paddingLayout(%d)", padding.byteSize());
	}
}
```

Finally the resultant source code is wrapped depending on whether the layout is a structure or a union:

```java
String type = switch(group) {
	case StructLayout _	-> "structLayout";
	case UnionLayout _	-> "unionLayout";
};
return String.format("MemoryLayout.%s(%s)", type, fields);
```

A couple of features have been omitted for brevity:

* Indentation.

* Structure layouts with a single field are inlined.

### Template

The generated arguments and layout are injected into the Velocity template as previously to generate the source code.

The template itself is now much simpler:

```java
package org.sarge.jove.platform.vulkan;

import static java.lang.foreign.ValueLayout.*;

import java.lang.foreign.*;

import org.sarge.jove.foreign.NativeStructure;
import org.sarge.jove.common.Handle;
import org.sarge.jove.util.EnumMask;
import org.sarge.jove.platform.vulkan.*;

/**
 * Vulkan structure.
 * This class has been code-generated.
 */
public class $name implements NativeStructure {
#foreach($field in $fields)
	public $field.type $field.name;
#end

	@Override
	public GroupLayout layout() {
		return $layout;
	}
}
```

The following example structure illustrates several of the mapping cases:

```c
typedef struct VkBufferCreateInfo {
    VkStructureType        sType;
    const void*            pNext;
    VkBufferCreateFlags    flags;
    VkDeviceSize           size;
    VkBufferUsageFlags     usage;
    VkSharingMode          sharingMode;
    uint32_t               queueFamilyIndexCount;
    const uint32_t*        pQueueFamilyIndices;
} VkBufferCreateInfo;
```

The generated source code (excluding imports and comments) for this structure is:

```java
public class VkBufferCreateInfo implements NativeStructure {
	public VkStructureType sType;
	public Handle pNext;
	public EnumMask<VkBufferCreateFlags> flags;
	public long size;
	public EnumMask<VkBufferUsageFlags> usage;
	public VkSharingMode sharingMode;
	public int queueFamilyIndexCount;
	public int[] pQueueFamilyIndices;

	@Override
	public GroupLayout layout() {
		return MemoryLayout.structLayout(
			JAVA_INT.withName("sType"),
			PADDING,
			POINTER.withName("pNext"),
			JAVA_INT.withName("flags"),
			PADDING,
			JAVA_LONG.withName("size"),
			JAVA_INT.withName("usage"),
			JAVA_INT.withName("sharingMode"),
			JAVA_INT.withName("queueFamilyIndexCount"),
			PADDING,
			POINTER.withName("pQueueFamilyIndices")
		);
	}
}
```

## Enumerations

The enumeration generator is largely unaffected by the Panama refactor, however there are a few modifications for other reasons:

A handy feature of the existing code was automatic generation of the `sType` field for top-level structures, leveraging the consistent naming convention in the `VkStructureType` enumeration.  However in more recent iterations of the Vulkan API there are several structures that deviate from this convention, or in some cases do not have an entry in the enumeration.  Therefore unfortunately this logic is removed and the field is populated programatically throughout.

Previously many of the enumerations contained a number of _synthetic_ constants that were omitted by the generator to reduce code size.  However more recent iterations of the API only seem to contain `MAX_ENUM` so this is less of a problem.  Also several of the enumerations would be empty other than this synthetic entry, so this logic is largely redundant and is therefore removed.

Finally, the generator truncates the constant names to remove the classname prefix.  This results in an edge case where the resultant name may have a leading numeric, which is obviously not a valid Java identifier.  The logic for this case is modified to 'walk' back one word in the truncated name, which is simpler than the previous behaviour that attempted to 'translate' the numeric.

## File Printer

```java
interface FilePrinter {
	/**
	 * Writes a generated source file.
	 * @param name		Filename
	 * @param source	Source code
	 */
	void print(String name, String source);

	/**
	 * Silent implementation.
	 */
	FilePrinter IGNORE = new FilePrinter() {
		@Override
		public void print(String name, String source) {
			// Ignored
		}
	};
}
```

TODO - default, comparison


---

# Miscellaneous

## Overview

This section covers the implementation of various edge-cases or features that were previously deferred.

## Native Booleans

The humble `boolean` caused some problems when JNA was used for the native layer, and FFM continues that trend.

Boolean values are largely only used by Vulkan in structures, in particular the `VkPhysicalDeviceFeatures` and (to a lesser extent) `VkPhysicalDeviceLimits` structures.  However the FFM `JAVA_BOOLEAN` assumes a one-byte layout (as used by Java) whereas Vulkan (and probably every other native library) represent a boolean as a four-byte integer.  These fields _could_ be implemented as primitive integers but that obfuscates their purpose and seems frankly lame.

Therefore the following specialised implementation is used in place of a primitive transformer based on the `JAVA_BOOLEAN` layout:

```java
public record NativeBooleanTransformer() implements Transformer<Boolean, Integer> {
    public static final int TRUE = 1;
    public static final int FALSE = 0;

    public static boolean isTrue(int value) {
        return value != FALSE;
    }

    public MemoryLayout layout() {
        return JAVA_INT;
    }

    public Integer marshal(Boolean value, SegmentAllocator allocator) {
        return value ? TRUE : FALSE;
    }

    public Object empty() {
        throw new UnsupportedOperationException();
    }

    public Function<Integer, Boolean> unmarshal() {
        return NativeBooleanTransformer::isTrue;
    }

    public Transformer<?, ?> array() {
        return PrimitiveTransformer.array(this, JAVA_INT);
    }
}
```

Note that this implementation treats a non-zero value as `true` but explicitly outputs integer `one` in this case.

This allows `boolean` values to be correctly identified as such in structures while correctly maintaining the four-byte integer alignment.

## Returned Arrays

The `glfwGetRequiredInstanceExtensions` API method returns an array of strings where the length is populated as a by-reference side-effect:

```c
GLFWAPI const char** glfwGetRequiredInstanceExtensions(uint32_t* count);
```

However the return value for the JOVE implementation __cannot__ be modelled as `String[]` since the length is unknown.

Initially we considered implementing some sort of adapter that encapsulated the underlying string array until the length was available, but this seemed a lot of effort for an edge-case.

There are only a couple of methods in GLFW that return an array (and none in Vulkan) so a more pragmatic approach is to simply return a handle to the array:

```java
interface DesktopLibrary {
    Handle glfwGetRequiredInstanceExtensions(IntegerReference count);
}
```

This can then be transformed to a string array by the following helper once the length has been determined:

```java
public static String[] array(MemorySegment address, int length) {
    MemorySegment array = address.reinterpret(ValueLayout.ADDRESS.byteSize() * length);

    return IntStream
        .range(0, length)
        .mapToObj(n -> array.getAtIndex(ValueLayout.ADDRESS, n))
        .map(StringTransformer::unmarshal)
        .toArray(String[]::new);
}
```

The method in the `Desktop` service to retrieve the required Vulkan extensions can now be implemented as follows:

```java
public List<String> extensions() {
    var count = new IntegerReference();
    Handle handle = library.glfwGetRequiredInstanceExtensions(count);
    String[] array = StringTransformer.array(handle.address(), count.get());
    return List.of(array);
}
```

## Desktop Callbacks

TODO

----
 
# Summary

In this chapter the JNA bindings to the native layer were replaced with an FFM based implementation.

We elected to implement a custom solution to generate the bindings using an implementation based on a Java proxy, rather than the using the `jextract` tool, for reasons that are documented above.

This solution employs a _registry_ of bindings that map JOVE and Java domain types to _transformers_ responsible for marshalling to/from the off-heap memory.

Additionally the code generator was also refactored to build the FFM memory layouts of the Vulkan structures.
