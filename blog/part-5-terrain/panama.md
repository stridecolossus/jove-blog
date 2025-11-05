---
title: Panama Integration
---

---

# Contents

- [Overview](#overview)
- [Design](#design)
- [Framework](#framework)
- [Structures](#structures)
- [Returned Parameters](#returned-parameters)
- [Conclusion](#conclusion)
- [Summary](#summary)

---

# Overview

It has always been the intention at some point to re-implement the Vulkan native layer using the FFM (Foreign Function and Memory) library developed as part of project [Panama](https://openjdk.java.net/projects/panama/).  The reasons for delaying this rework are discussed in [code generation](/jove-blog/blog/part-1-intro/code-generation#alternatives) at the very beginning of the blog.

With the LTS release of JDK 21 the FFM library was elevated to a preview feature (rather than requiring an incubator build) and much more information and tutorials started to appear online, so it is high time to replace JNA with the more future proofed (and safer) FFM solution.

A fundamental technical decision that must be addressed immediately is whether to use the `jextract` tool to generate the _bindings_ to the native libraries or to implement a bespoke solution.  To resolve this question the tool is used to generate the Vulkan API and the results are compared and contrasted with a throwaway, hard-crafted application built using FFM from first principles.

---

# Design

## Exploring Panama

The GLFW library is again used as the guinea-pig since this API is relatively simple in comparison to Vulkan, which is front-loaded with complexities such as marshalling of structures, by-reference types, callbacks for diagnostic handlers, etc.

First the native library is loaded using the `SymbolLookup` service:

```java
public class DesktopForeignDemo {
    void main() {
        try(Arena arena = Arena.ofConfined()) {
            Linker linker = Linker.nativeLinker();
            SymbolLookup lookup = SymbolLookup.libraryLookup("glfw3", arena);
            var demo = new DesktopForeignDemo(lookup);
            ...
        }
    }
}
```

The process of invoking a native method using FFM is:

1. Find the off-heap address of the the method via the lookup service.

2. Build a `FunctionDescriptor` specifying the signature of the method.

3. Combine both of these to link a `MethodHandle` to the native method.

4. Invoke the handle with an array of the appropriate arguments.

This process is illustrated in the following code which initialises GLFW and returns a success code:

```java
private void init() {
    MemorySegment symbol = lookup.find("glfwInit").orElseThrow();
    MethodHandle handle = linker.downcallHandle(symbol, FunctionDescriptor.of(ValueLayout.JAVA_INT));
    println("init=" + handle.invoke());
}
```

The GLFW version can be queried:

```java
private void version() {
    MemorySegment symbol = lookup.find("glfwGetVersionString").orElseThrow(),
    MethodHandle handle = linker.downcallHandle(symbol, FunctionDescriptor.of(ValueLayout.ADDRESS));
    MemorySegment address = (MemorySegment) handle.invoke();
    String version = versionAddress.reinterpret(Integer.MAX_VALUE).getString(0L);
    println("version=" + version);
}
```

Note that off-heap address has to be _reinterpreted_ to expand the bounds of the off-heap memory, in this case an undefined length for a pointer to a null-terminated character array.  The `0L` argument in the `getString` helper is an offset into the off-heap memory.

Support for Vulkan can also be tested:

```java
private void supported() {
    MemorySegment symbol = lookup.find("glfwVulkanSupported").orElseThrow(),
    MethodHandle handle = linker.downcallHandle(symbol, FunctionDescriptor.of(ValueLayout.JAVA_BOOLEAN));
    println("supported=" + handle.invoke());
}
```

Retrieving the supporting Vulkan extensions for the platform is slightly complex since the API returns a pointer to a string-array and uses a _by reference_ integer to return the size of the array as a side-effect:

```java
private void extensions() {
    MemorySegment symbol = lookup.find("glfwGetRequiredInstanceExtensions").orElseThrow(),
    MethodHandle handle = linker.downcallHandle(symbol, FunctionDescriptor.of(ValueLayout.ADDRESS, ValueLayout.ADDRESS));
    MemorySegment count = arena.allocate(ValueLayout.ADDRESS);
    MemorySegment extensions = (MemorySegment) handle.invoke(count);
    ...
}
```

The length of the array is extracted from the by-reference `count` to resize the off-heap memory:

```java
int length = count.get(ValueLayout.JAVA_INT, 0L);
MemorySegment array = extensions.reinterpret(length * ValueLayout.ADDRESS.byteSize());
```

And each element is then extracted and transformed to a Java string:

```java
for(int n = 0; n < length; ++n) {
    MemorySegment e = array.getAtIndex(ADDRESS, n);
    println("extension=" + e.reinterpret(Integer.MAX_VALUE).getString(0L));
}
```

Finally GLFW is closed:

```java
private void terminate() {
    MemorySegment symbol = lookup.find("glfwTerminate").orElseThrow(),
    MethodHandle handle = linker.downcallHandle(symbol, FunctionDescriptor.ofVoid());
    glfwTerminate.invoke();
}
```

Note that `reinterpret` is an _unsafe_ operation that can result in a JVM crash for an out-of-bounds memory access, since the actual memory size cannot be guaranteed by the JVM.  Additionally this method is _restricted_ and generates a runtime warning, which can be suppressed by the following VM argument:

```
--enable-native-access=ALL-UNNAMED
```

### Analysis

Generating the FFM bindings using `jextract` is relatively trivial and takes a couple of seconds:

`jextract --source \VulkanSDK\1.2.154.1\Include\vulkan\vulkan.h`

With some experience of the FFM library and the generated Vulkan bindings, we can attempt to extrapolate how JOVE could be refactored and make some observations.

The advantages of `jextract` are obvious:

* Proven and standard JVM technology.

* Automated (in particular the FFM memory layouts for native structures).

However there are disadvantages:

* The generated API is polluted by ugly internals such as field handles, structure sizes, helper methods, etc.

* API methods and structures can only be defined in terms of an FFM `MemorySegment` or primitives resulting in poor type safety and loss of meaning.

* Enumerations are implemented as static integer accessors (WTF!) 

* Implies throwing away the existing code generated enumerations and structures.

Alternatively an abstraction layer based on the existing hand-crafted API and code-generated classes would have the following advantages:

* The native API and structures are expressed in domain terms.

* Proper enumerations.

* Retains the existing API and code-generated enumerations and structures.

On the other hand:

* Substantial development effect.

* Proprietary solution with the potential for unknown problems further down the line.

* Memory layouts needs to be generated for _all_ structures.

Despite the advantages of `jextract` the hybrid solution is clearly preferable - the ugly bindings, lack of type safety, etc. were the very concerns that led us to abandon LWJGL in the first place.

> In any case the application and/or JOVE would _still_ need to transform domain types to/from the FFM equivalents even if `jextract` was used to generate the underlying bindings.

The source code generated by `jextract` will be retained as it will be a useful resource during development, particularly for the memory layout of structures.

### Requirements

With this decision made (for better or worse) the requirements are determined as the following:

* A mechanism that generates an FFM-based implementation of the existing GLFW and Vulkan API definitions.

* A _marshalling_ framework that transforms supported Java types to/from the FFM equivalents.

* Support for by-reference parameters.

* Callback support to enable GLFW device polling and the Vulkan diagnostics handler.

* Refactoring of the code generator to build the FFM memory layout for all structures.

The marshalling framework will need to support the following types:

* primitives

* strings

* structures

* domain types, i.e. reference types such as `Handle` or subclasses of `NativeObject`

* integer enumerations and bitfields

* arrays of these supported types.

From these requirements the following observations can be made concerning some of the trickier aspects:

1. For the moment at least it seems logical to continue to treat by-reference integers and pointers as special cases.  They are already integral to JOVE (based on the existing JNA equivalents) and in particular there is no Java analog for a by-reference integer.

2. The other by-reference use-cases imply some form of post-processing to 'copy' the off-heap memory back to the argument(s).

3. GLFW makes heavy use of callbacks for device polling which will require substantial factoring to use FFM upcall stubs.

4. The `glfwGetRequiredInstanceExtensions` method returns an array of strings, however the length of the array is specified as a by-reference integer parameter populated as a side-effect.  Therefore the returned extensions _cannot_ be modelled as a Java array (probably implying some sort of intermediate wrapper).  This is the only method in either library that returns an array, therefore this requirement is treated as an edge-case and deferred for the moment, and the Vulkan extensions are temporarily hard-coded

### Design

From the above the following framework components are identified:

* A _native library factory_ responsible for constructing the FFM implementation of a native API expressed in domain terms.

* A set of _transformers_ for the supported types responsible for off-heap memory allocation and marshalling to/from the equivalent FFM representation.

* A _native method_ class that composes an FFM method handle and the transformers for its parameters and optional return type.

* A mechanism to identify by-reference parameters and logic to unmarshal these parameters _after_ method invocation.

Additionally there are some offline tasks to be completed:

* Removal of the JNA library.

* A global find-and-replace to refactor the GLFW and Vulkan APIs in terms of the new framework.

* Automated generation of the FFM memory layouts for the Vulkan structures.

A 'big bang' approach always feels inherently risky simply due to the shear scale of the changes to be made.  Therefore the plan is to create a throwaway prototype targeted at a cut-down API (again using GLFW to begin with) such that we can focus on the development of the new framework without getting bogged down in compilation issues.  Once this is up and running _then_ we will integrate the code into JOVE and iteratively reintroduce each component and refactor accordingly.

As already noted, Vulkan is very front-loaded with complexity, so framework support will be implemented as we progress.  Once we are confident that the bulk of the challenges facing the new framework have been addressed, the prototype will be discarded and refactoring can continue with the existing suite of demo applications.

///////////////////

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
    .filter(Builder::isNativeMethod)
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

To exercise the new framework a prototype is created using the following (very) cut-down GLFW library, dependant only on integer primitives for the moment:

```java
interface Prototype {
    int     glfwInit();
    int     glfwVulkanSupported();
    void    glfwTerminate();
}
```

A proxy implementation is generated using the new factory:

```java
var builder = new NativeLibraryFactory();
Prototype api = builder.build(List.of(API.class));
```

Which can then be invoked to test the trivial API:

```java
api.glfwInit();
api.glfwVulkanSupported();
api.glfwTerminate();
```

Essentially all this does is intercept the method calls and dump them to the console.

## Native Method

In order to actually invoke the native library the wrapper class is reimplemented to compose an FFM _method handle_ rather than the reflected method itself:

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
MemoryLayout[] layouts = Arrays
    .stream(method.getParameterTypes())
    .map(this::layout)
    .toArray(MemoryLayout[]::new);
```

Where `layout` is a temporary, quick-and-dirty mapper that just supports the integer primitives used in the prototype:

```java
private static MemoryLayout layout(Class<?> type) {
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
private static FunctionDescriptor descriptor(Class<?> returns, MemoryLayout[] layout) {
    FunctionDescriptor descriptor = FunctionDescriptor.ofVoid(layouts);

    if(returns == null) || (returns == void.class)) {
        return descriptor;
    }
    else {
        return descriptor.changeReturnLayout(layout(returns));
    }
}
```

Finally the method handle is linked to the native library:

```java
public NativeMethod build() {
    MemoryLayout[] layout = ...
    FunctionDescriptor descriptor = descriptor(method.getReturnType(), layout);
    MethodHandle handle = linker.downcallHandle(descriptor).bindTo(address);
    return new NativeMethod(handle);
}
```

In the prototype the FFM `SymbolLookup` first loads the native library:

```java
SymbolLookup lookup = SymbolLookup.libraryLookup("glfw3", Arena.ofAuto());
var builder = new NativeLibraryFactory(lookup);
API api = ...
```

And should now invoke GLFW natively when executed.  Nice.

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

The factory is refactored to lookup the relevant transformers for the methods signature:

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

Where the temporary `layout` method now maps to a transformer:

```java
private static Transformer layout(Class<?> type) {
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
    FunctionDescriptor descriptor = FunctionDescriptor.ofVoid(layouts);

    // Append return layout
    if(returns == null) {
        return descriptor;
    }
    else {
        return descriptor.changeReturnLayout(returns.layout());
    }
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

The method arguments are marshalled according to the transformers before invocation:

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

To fully explore GLFW using the new framework the prototype will be extended to create a native window, which requires support for strings and a `Handle` to the window.

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
        return address.reinterpret(Integer.MAX_VALUE).getString(0L);
    }
}
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

And finally the temporary `layout` method is extended to the newly supported types:

```java
private static Transformer layout(Class<?> type) {
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

The prototype API is next extended further with the addition of the following methods:

```java
interface Prototype {
    String  glfwGetVersionString();
    void    glfwDefaultWindowHints();
    void    glfwWindowHint(int hint, int value);
    Handle  glfwCreateWindow(int w, int h, String title, Handle monitor, Handle shared);
    void    glfwDestroyWindow(Handle window);
}
```

The prototype itself can now also be extended accordingly:

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

The method to create the window passes `null` for some of the arguments, therefore the `Handle` transformer returns the special `MemorySegment.NULL` value in this case:

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

Before proceeding further the temporary `layout` bodge is replaced with a _registry_ that maps a domain type to its corresponding transformer:

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

* The `JAVA_BOOLEAN` layout is not included since `boolean` primitives turn out to be a bit of a special case (covered later in this chapter).

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

* Callback support for GLFW devices (temporarily hacked out).

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
    PrimitiveTransform.register(registry);
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

Although a basic FFM-based marshalling framework in now in place, as mentioned previously Vulkan is very front-loaded with complexities.

The first step instantiates the Vulkan instance using the following native API method:

```java
VkResult vkCreateInstance(VkInstanceCreateInfo* pCreateInfo, VkAllocationCallbacks* pAllocator, VkInstance* pInstance);
```

Implementing just this very first API call implies marshalling support for the following:

* structures - namely `VkInstanceCreateInfo` and `VkApplicationInfo`.

* string arrays - for the instance extensions and layers.

* integer enumerations - the `sType` fields of the structures and the `VkResult` return type.

* by-reference pointers - to 'return' the resultant `pInstance` handle.

Additionally FFM memory layouts for the required structures will need to be implemented.

The simpler dependencies will be dealt with first along with a few framework enhancements before moving on to structures.

## Framework

### Enumerations

Marhalling an integer enumeration is implemented as another companion transformer:

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
        foreign[n] = marshal(args[n], parameters[n], allocator);
    }
    return foreign;
}
```

Delegating to the following helper:

```java    
private static Object marshal(Object value, Transformer transformer, SegmentAllocator allocator) {
    if(arg == null) {
        return transformer.empty();
    }
    else {
        return transformer.marshal(value, allocator);
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

Note that the `unmarshal` function is disallowed since a domain type will never be returned from a native method.

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

The handle of the created instance is returned as a _by reference_ parameter, essentially a _pointer_ or a mutable address.

The following new template class implements a by-reference parameter that holds a _pointer_ to the off-heap memory:

```java
public abstract class NativeReference<T> {
    private T value;
    private MemorySegment pointer;

    public T get() {
        if(pointer != null) {
            value = update(pointer);
        }
        return value;
    }

    protected abstract T update(MemorySegment pointer);
}
```

A general `Pointer` dereferences the address of the handle:

```java
public class Pointer extends NativeReference<Handle> {
    protected T update(MemorySegment pointer) {
        MemorySegment address = pointer.get(ValueLayout.ADDRESS, 0L);
        if(MemorySegment.NULL.equals(address)) {
            return null;
        }
        else {
            return new Handle(address);
        }
    }
}
```

Although not required yet, the implementation for an integer reference is relatively trivial:

```java
public class IntegerReference extends NativeReference<Integer> {
    protected Integer update(MemorySegment pointer) {
        return pointer.get(ValueLayout.JAVA_INT, 0L);
    }
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

Note that the `unmarshal` method is disallowed since by-reference values can logically only ever be method parameters and never a return value.

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

Note that array parameters are now explicitly declared as Java arrays.  Previously these had to be specified as a __single__ instance since JNA mandated that a contiguous block was represented by the __first__ element of an array.

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
        return builder.build(List.of(api));
    }
```

Where `DefaultRegistry` is a helper that registers the various transformers required by Vulkan (basically all of the above).

### Return Codes

The approach of using a proxy means there is a central location where _all_ Vulkan return codes can be validated, whereas previously _every_ native call had to be wrapped with the `check` helper which was ugly and slightly tedious.

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
    if((code instanceof VkResult result) && (result != VkResult.SUCCESS)) {
        throw new VulkanException(result);
    }
};
factory.handler(check);
```

Nice.

## Structures

### Structure Layout

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
    public final VkStructureType sType = VkStructureType.INSTANCE_CREATE_INFO;
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

The field name is specified in the member layout:

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

Refactoring the diagnostic handler presents a few new challenges:

* The native methods to create and destroy a handler must be created programatically, since the address is a _function pointer_ instead of a symbol looked up from the native library.

* In particular the method signatures are not defined in the Vulkan API but must be derived from the documentation.

* A handler requires an FFM callback _stub_ to allow Vulkan to report diagnostic messages to the application.

The majority of the existing handler code remains as-is except for:

* Invocation of the create and destroy function pointers.

* Creation of the message callback.

* Unmarshalling of the diagnostic report structure to construct the `Message` record.

Although the diagnostic handler uses a different mechanism to lookup the memory address of the function pointers, the existing framework can be reused by the introduction of the following hand-crafted  library:

```java
private interface HandlerLibrary {
    VkResult vkCreateDebugUtilsMessengerEXT(Instance instance, VkDebugUtilsMessengerCreateInfoEXT pCreateInfo, Handle pAllocator, Pointer pHandler);
    void vkDestroyDebugUtilsMessengerEXT(Instance instance, DiagnosticHandler handler, Handle pAllocator);
}
```

Next the `function` method of the `Instance` is reintroduced, which turns out to be trivial:

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

Note that again the signature of the callback must be hard-coded based on the documentation:

```java
public boolean message(int severity, int typeMask, MemorySegment pCallbackData, MemorySegment pUserData)
```

The resultant method handle is then bound to the callback _instance_ in order to delegate diagnostic reports to the relevant handler:

```java
MethodHandle binding = handle.bindTo(this);
```

This handle is then linked to an up-call stub with a function descriptor matching the `message` method:

```java
private static MemorySegment link(MethodHandle handle) {
    var descriptor = FunctionDescriptor.of(
        JAVA_BOOLEAN,
        JAVA_INT,
        JAVA_INT,
        ADDRESS,
        ADDRESS
    );
    return Linker.nativeLinker().upcallStub(binding, descriptor, Arena.ofAuto());
}
```

The `build` method of the handler is modified to lookup the transformer for the message structure:

```java
public DiagnosticHandler build() {
    ...
    var transformer = registry.get(VkDebugUtilsMessengerCallbackData.class);
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
    var data = (VkDebugUtilsMessengerCallbackData) transformer.unmarshal(segment);
    ...
}
```

In this case the off-heap memory needs to be explicitly resized to match the expected structure.

Unmarshalling a structure field is essentially the inverse of the `marshal` method:

```java
record FieldMapping(...) {
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

Diagnostic handlers can now be attached to the instance as before, which is very useful given the invasive changes being made to JOVE.

A good test of the refactor is to temporarily remove the code that releases attached handlers when the instance is destroyed, which should result in Vulkan complaining about orphaned objects.

----

# Physical Device







---

//////////////////////

## Returned Parameters

### Returned Annotation

The next Vulkan component after the instance is the physical device which uses native methods that are highly dependant on by-reference parameters.  These pose a problem because by-reference and 'normal' parameters cannot be distinguished solely from the method signature.

JNA used the rather cumbersome `ByReference` interface which generally required _every_ structure to have _two_ implementations.  Additionally _every_ array is essentially a by-reference parameter as far as JNA is concerned, whereas we would prefer to separate the normal and by-reference cases.

The most obvious solution is to use a custom annotation to explicitly declare by-reference parameters, the annotation is a simple _marker_ with no declared functionality:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Returned {
}
```

Parameters returned by-reference are detected in the library builder when a native method is constructed:

```java
public NativeMethod build(Method method) {
    ...
    for(Parameter p : method.getParameters()) {
        boolean returned = Objects.nonNull(p.getAnnotation(Returned.class));
        builder.parameter(p.getType(), returned);
    }
    ...
}
```

The `parameter` method is overloaded to accept a flag indicating whether a parameter is returned by-reference:

```java
public Builder parameter(Class<?> type, boolean returned) {
    Transformer transformer = registry.get(type);
    signature.add(new NativeParameter(transformer, returned));
    return this;
}
```

```java
record NativeParameter(Transformer<?> transformer, boolean returned) {
    private MemoryLayout layout() {
        if(returned) {
            return AddressLayout.ADDRESS;
        }
        else {
            return transformer.layout();
        }
    }
}
```

```java
public Object invoke(Object[] args) {
    if(args == null) {
        return null;
    }

    Object[] foreign = marshal(args, Arena.ofAuto());
    Object result = handle.invokeWithArguments(foreign);
    update(foreign, args);
    return unmarshal(result);
}
```

```java
private void update(Object[] args, Object[] foreign) {
    for(int n = 0; n < args.length; ++n) {
        // Skip normal parameters
        NativeParameter p = signature[n];
        if(!p.returned) {
            continue;
        }

        // Skip empty elements
        if(Objects.isNull(args[n]) || MemorySegment.NULL.equals(foreign[n])) {
            continue;
        }

        // Overwrite by-reference argument
        ...
    }
}
```

Note that the framework assumes that the argument or array has already been instantiated prior to the method call, i.e. `null` parameters or array elements are ignored.

/////////////////////



### Integration

Using the new annotation the refactored library for the physical device is as follows:

```java
interface Library {
    int  vkEnumeratePhysicalDevices(Instance instance, NativeReference<Integer> pPhysicalDeviceCount, @Returned Handle[] devices);
    void vkGetPhysicalDeviceProperties(PhysicalDevice device, @Returned VkPhysicalDeviceProperties props);
    void vkGetPhysicalDeviceFeatures(Handle device, @Returned VkPhysicalDeviceFeatures features);
    void vkGetPhysicalDeviceQueueFamilyProperties(Handle device, NativeReference<Integer> pQueueFamilyPropertyCount, @Returned VkQueueFamilyProperties[] props);
    void vkGetPhysicalDeviceFormatProperties(PhysicalDevice device, VkFormat format, @Returned VkFormatProperties props);
    int  vkGetPhysicalDeviceSurfaceSupportKHR(PhysicalDevice device, int queueFamilyIndex, Handle surface, NativeReference<Integer> supported);
}
```

Other than refactoring the library the remainder of the `PhysicalDevice` class remains as-is.

/////////////////////

There are three by-reference use-cases to be addressed:

#### Structure

The `vkGetPhysicalDeviceFeatures` method accepts a `VkPhysicalDeviceProperties` structure which is _populated_ by the native layer.

A new method is added to the structure transformer to copy the off-heap data back to the structure:

```java
public void unmarshal(MemorySegment address, Object structure) {
    for(FieldMapping field : mappings) {
        field.unmarshal(address, structure);
    }
}
```

Structures are the only type that can be returned in this manner, validation (not shown) applies this restriction in the `parameter` method above.

#### Handle Array

The `vkEnumeratePhysicalDevices` method enumerates the hardware devices by populating an array of handles.

The native method eventually delegates to the following code to populate the array:

```java
class ArrayTransformer {
    public void update(MemorySegment address, Object[] array) {
        for(int n = 0; n < array.length; ++n) {
            MemorySegment element = address.getAtIndex(ValueLayout.ADDRESS, n);
            array[n] = transformer.unmarshal(element);
        }
    }
}
```

As a quick reminder: Vulkan employs the [two stage invocation](/jove-blog/blog/part-2-triangle/devices#improvements) pattern when returning by-reference arrays:

```java
public static Stream<PhysicalDevice> devices(Instance instance) {
    Vulkan vulkan = instance.vulkan();
    VulkanLibrary lib = vulkan.library();
    VulkanFunction<Handle[]> enumerate = (count, devices) -> lib.vkEnumeratePhysicalDevices(instance, count, devices);
    Handle[] handles = vulkan.invoke(enumerate, Handle[]::new);
    ...
}
```

The slightly improved `invoke` helper (moved to the new `Vulkan` class) implements this pattern:

```java
public <T> T invoke(VulkanFunction<T> function, IntFunction<T> supplier) {
    // Determine the result size
    NativeReference<Integer> count = factory.integer();
    function.get(count, null);

    // Instantiate the container
    int size = count.value();
    T data = supplier.apply(size);

    // Invoke again to populate the container
    if(size > 0) {
        function.get(count, data);
    }

    return data;
}
```

#### Structure Array

Finally `vkGetPhysicalDeviceQueueFamilyProperties` returns a by-reference array of structures.

There is a subtle difference between a 'normal' structure array and one being returned by-reference.  Both cases require the off-heap array to be allocated as a contiguous memory block, however the memory layout of a by-reference parameter __must__ be a pointer (i.e. has an `ADDRESS` layout).

Therefore the following helper determines the actual parameter layout depending on usage:

```java
record Parameter(Transformer transformer, boolean returned) {
    private MemoryLayout layout() {
        if(returned) {
            return ValueLayout.ADDRESS;
        }
        else {
            return transformer.layout();
        }
    }
}
```

In either case the structure array is populated by slicing the off-heap memory and unmarshalling each element:

```java
public void update(MemorySegment address, Object[] array) {
    long size = transformer.layout().byteSize();
    for(int n = 0; n < array.length; ++n) {
        MemorySegment element = address.asSlice(n * size, size);
        array[n] = transformer.unmarshal(element);
    }
}
```

### Nested Structures

There is another 'hidden' use-case that only became apparent during refactoring of the physical device.  Each device supports a number of queue families which are retrieved during enumeration, described by the following structure:

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

In the native structure the `VkExtent3D` fields are essentially _embedded_ into the parent structure, whereas in the structure class this field obviously has to be an object reference (there is no analog of embedding in Java).  The structure transformer will need to support both _atomic_ fields (i.e. primitives and pointer types) and nested structures.

The code that walks through the layout is first modified to include all members except for alignment padding:

```java
return layout
    .memberLayouts()
    .stream()
    .filter(Predicate.not(e -> e instanceof PaddingLayout))
    .map(instance::mapping)
    .toList();
```

/////////////

```java
interface FieldMapping {
    /**
     * Marshals this field to off-heap memory.
     * @param structure     Structure
     * @param address       Off-heap address
     * @param allocator     Allocator
     */
    void marshal(NativeStructure structure, MemorySegment address, SegmentAllocator allocator);

    /**
     * Unmarshals this field from off-heap memory.
     * @param address       Off-heap memory
     * @param structure     Structure
     */
    void unmarshal(MemorySegment address, NativeStructure structure);
}
```

////////////

To accommodate nested structures the existing field mapping is converted to an interface which is a delegate for the different cases:

```java
private interface FieldMapping {
    void marshal(Object value, MemorySegment address, SegmentAllocator allocator);
    Object unmarshal(MemorySegment address);
}
```

This is composed into the following simple record type with the structure field handle (which is common to both cases):

```java
private record StructureField(VarHandle field, FieldMapping mapping)
```

The `mapping` function can now switch behaviour according to the layout of each field:

```java
private StructureField mapping(MemoryLayout member) {
    ...

    FieldMapping mapping = switch(member) {
        case ValueLayout _ -> atomic(field);
        case StructLayout nested -> nested(field, nested);
        default -> throw new IllegalArgumentException(...)
    };

    return new StructureField(handle(field), mapping);
}
```

The existing code is moved to a new field mapping implementation for atomic fields:

```java
private FieldMapping atomic(Field field) {
    PathElement path = PathElement.groupElement(field.getName());
    VarHandle foreign = layout.varHandle(path);
    Transformer transformer = registry.get(field.getType());
    return new AtomicFieldMapping(foreign, transformer);
}
```

A nested structure recursively builds the mappings for the embedded fields:

```java
private FieldMapping nested(Field field, StructLayout nested) {
    Class<?> type = field.getType();
    List<StructureField> fields = build(child, nested);
    Supplier<?> factory = factory(type);
    return new StructureFieldMapping(factory, fields);
}
```

Where `factory` looks up the reflected default constructor for a given structure type.

A second field mapping implementation for nested structures recursively marshals the embedded fields:

```java
private record StructureFieldMapping(Supplier<?> constructor, List<StructureField> fields) implements FieldMapping {
    public void marshal(Object structure, MemorySegment address, SegmentAllocator allocator) {
        for(var field : fields) {
            field.marshal(structure, address, allocator);
        }
    }
}
```

And similarly for unmarshalling:

```java
public Object unmarshal(MemorySegment address) {
    var structure = constructor.get();
    unmarshal(address, structure);
    return structure;
}

public void unmarshal(MemorySegment address, Object structure) {
    for(var field : fields) {
        field.unmarshal(address, structure);
    }
}
```

Note that the field is _overwritten_ by a new instance of the nested structure.

TODO - concludes enumeration, including queue families and device features, all 3 by-ref cases

---

## Conclusion

This section covers the implementation of various edge-cases or features that were deferred above.

### Returned Arrays

A native method that returns an array is an edge-case for Java since the length of the resultant array cannot be determined from the return value itself.  Generally the length is provided separately, often as an integer-by-reference parameter in the same method.  Therefore a returned array __cannot__ be modelled as a Java array and must be represented by a new, intermediate type:

```java
public class ReturnedArray<T> {
    private final MemorySegment address;
    private final Registry registry;
}
```

Where `<T>` is the array component type.

The `get` method extracts the actual Java array given the length:

```java
public T[] get(int length, Class<? extends T> type) {
    ...
    return array;
}
```

The array is first allocated:

```java
T[] array = (T[]) Array.newInstance(type, length);
```

And then delegates to the relevant transformer to unmarshal the array elements:

```java
var transformer = (ArrayTransformer) registry.get(type.arrayType());
MemorySegment data = address.reinterpret(length * transformer.layout().byteSize());
transformer.update(data, array);
```

The new type has a companion transformer to plug into the framework:

```java
static final class ReturnedArrayTransformer implements AddressTransformer<ReturnedArray<?>, MemorySegment> {
    private final Registry registry;

    public Object marshal(ReturnedArray<?> arg, SegmentAllocator allocator) {
        throw new UnsupportedOperationException();
    }

    public ReturnedArray<?> unmarshal(MemorySegment address) {
        return new ReturnedArray<>(address, registry);
    }
}
```

Which is instantiated on demand in the `create` method of the registry:

```java
if(ReturnedArray.class.isAssignableFrom(type)) {
    return new ReturnedArrayTransformer(this);
}
```

The supported Vulkan extensions can now be retrieved from GLFW:

```java
public String[] extensions() {
    NativeReference<Integer> count = factory.integer();
    ReturnedArray<String> array = lib.glfwGetRequiredInstanceExtensions(count);
    return array.get(count.get(), String.class);
}
```

Notes:

* An unsupported component type will only be detected at runtime in the `get` method, but this seems acceptable given that the `extensions` method is the only such instance in the GLFW or Vulkan APIs.

* An invalid array length cannot be bounds checked since the off-heap memory must be resized using the unsafe `reinterpret` method, which can result in a JVM crash.

### Desktop Callbacks

TODO

### Code Generation

TODO

 and remove the JNA artifacts (imports and the `@FieldOrder` annotation).

### buffers

replace byte buffers with FFM equivalents
- device memory
- VK buffers
- Bufferable mechanism
- etc

---
 
## Summary

In this chapter

Elected to implement a custom FFM implementation and generator

proxy implementation

transformer framework

code generator
