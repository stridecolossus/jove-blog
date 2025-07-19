---
title: Panama Integration
---

---

## Contents

- [Overview](#overview)
- [Design](#design)
- [Framework](#framework)
- [Structures](#structures)
- [Returned Parameters](#returned-parameters)
- [Conclusion](#conclusion)
- [Summary](#summary)

---

## Overview

It has always been the intention at some point to re-implement the Vulkan native layer using the FFM (Foreign Function and Memory) library developed as part of project [Panama](https://openjdk.java.net/projects/panama/).  The reasons for delaying this rework are discussed in [code generation](/jove-blog/blog/part-1-intro/code-generation#alternatives) at the very beginning of the blog.

With the LTS release of JDK 21 the FFM library was elevated to a preview feature (rather than requiring an incubator build) and much more information and tutorials started to appear online, so it is high time to replace JNA with the more future proofed (and safer) FFM solution.

A fundamental technical decision that must be addressed immediately is whether to use the `jextract` tool to generate the _bindings_ to the native libraries or to implement a bespoke solution.  To resolve this question the tool is used to generate the Vulkan API and the results are compared and contrasted with a throwaway, hard-crafted application built using FFM from first principles.

---

## Design

### Exploring Panama

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

This process is illustrated in the following code which initialises GLFW returning an integer success code:

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

Notes:

* The `0L` argument in the `getString` helper is an offset into the off-heap memory.

* The returned address has to be _reinterpreted_ to expand the bounds of the off-heap memory, which in this case is an undefined length for a pointer to a null-terminated character array.

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

Note that `reinterpret` is an _unsafe_ operation that can result in a JVM crash for an out-of-bounds array or string length, since the actual size off the off-heap memory cannot be guaranteed by the JVM.  Additionally this method is _restricted_ and will generate a runtime warning, which can be suppressed by the following VM argument:

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

* API methods and structures can only be defined in terms of FFM types or primitives (resulting in poor type safety and loss of meaning).

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

* Support for callbacks for GLFW device polling and the Vulkan diagnostics handler.

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

4. The `glfwGetRequiredInstanceExtensions` method is an edge-case (for reasons detailed later) and is also the only method in either library that returns an array, therefore this requirement is deferred for the moment and the Vulkan extensions are temporarily hard-coded.

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

---

## Framework

### Native Library

Constructing the FFM implementation of a native API involves the following steps:

1. Enumerate the native methods from the user defined API.

2. Create and link an FFM method handle for each identified API method.

3. Create a Java proxy that delegates API calls to the underlying FFM methods.

This process is implemented by a new factory component:

```java
public class NativeLibraryFactory {
    private final SymbolLookup lookup;

    public <T> T build(Class<T> api) {
        if(!api.isInterface()) throw new IllegalArgumentException(...);
        ...
    }
}
```

Where _api_ is the user defined interface specifying the API to be implemented.

The API methods are enumerated by reflecting the API:

```java
private Map<Method, NativeMethod> methods(Class<?> api) {
    return Arrays
        .stream(api.getMethods())
        .filter(NativeLibraryFactory::isNativeMethod)
        .collect(toMap(Function.identity(), this::build));
}
```

Where a native method is defined as the public, non-static members of the API:

```java
private static boolean isNativeMethod(Method method) {
    int modifiers = method.getModifiers();
    return !Modifier.isStatic(modifiers);
}
```

For the moment the native method class is a simple wrapper for the underlying FFM handle:

```java
public class NativeMethod {
    private final MemoryHandle handle;
    
    public Object invoke(Object[] args) {
        return handle.invokeWithArguments(args);
    }
}
```

The `build` method finds the symbol (i.e. the off-heap address) of each method by name and creates a new instance:

```java
public NativeMethod build(Method method) {
    MemorySegment address = lookup.find(method.getName()).orElseThrow(...);                
    return new NativeMethod(...);
}
```

Next an _invocation handler_ is created that delegates API calls to the relevant native method:

```java
private InvocationHandler handler(Map<Method, NativeMethod> methods) {
    return new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            NativeMethod delegate = methods.get(method);
            return delegate.invoke(args);
        }
    };
}
```

From which a Java proxy is created:

```java
private static <T> T proxy(Class<T> api, InvocationHandler handler) {
    ClassLoader loader = api.getClassLoader();
    return (T) Proxy.newProxyInstance(loader, new Class<?>[]{api}, handler);
}
```

This creates an implementation of the user-defined API that transparently delegates method calls to the underlying native library.  Cool.

### Native Method

In order to link the method handle the `build` method requires an FFM function descriptor derived from the _signature_ of the native method.

First the memory layout of the method is derived from its parameters:

```java
public NativeMethod build(Method method) {
    MemoryLayout[] layouts = Arrays
        .stream(method.getParameterTypes())
        .map(NativeLibraryFactory::layout)
        .toArray(MemoryLayout[]::new);
        
    ...
}
```

Where `layout` is a temporary, quick-and-dirty mapper that just supports integer primitives for a basic GLFW demonstration:

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

Next the function descriptor is derived from the memory layout and the optional return type:

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

And finally the method handle is linked to the native library:

```java
public NativeMethod build() {
    MemoryLayout[] layout = ...
    FunctionDescriptor descriptor = descriptor(method.getReturnType(), layout);
    MethodHandle handle = linker.downcallHandle(descriptor).bindTo(address);
    return new NativeMethod(handle);
}
```

### Integration #1

To exercise the new framework a second prototype is created to test GLFW, using the following (very) cut-down API dependant only on integer primitives:

```java
interface API {
    int     glfwInit();
    int     glfwVulkanSupported();
    void    glfwTerminate();
}
```

A proxy implementation is generated using the new factory:

```java
try(Arena arena = Arena.ofConfined()) {
    var lookup = SymbolLookup.libraryLookup("glfw3", arena);
    var factory = new NativeLibraryFactory(lookup);
    API api = factory.build(API.class);
    ...
}
```

Which can then be invoked to test a few basic GLFW methods:

```java
System.out.println("init=" + api.glfwInit());
System.out.println("Vulkan=" + api.glfwVulkanSupported());
api.glfwTerminate();
```

To summarise, the new framework:

* Maps and links a user-defined Java API to a native library.

* Maps the parameters and return value of each API method to the equivalent FFM memory layout (albeit only for primitive integers so far).

* Transparently delegates API calls to the underlying native methods.

### Native Transformer

With the basic framework in place the next step is to implement marshalling support for the required types identified above.

The first attempt used the functionality provided by the `MethodHandle` class (used extensively in FFM) which supports transformation of the incoming arguments and the method return value.  This initially seemed a perfect fit, however in practice the code became convoluted very quickly.  The logic required to mutate an argument is generally also implemented as 'target' method handles, resulting in fiddly lookup and binding code for all but the most trivial cases.

After some time we abandoned this approach in favour of pre/post-processing the arguments around invocation of the native method.  The code became much clearer and easier to unit-test, at the expense of some additional processing.  Of importance is that error cases can still be trapped when the native method is being constructed.

> As it turns out some form of pre/post-processing would probably have been needed anyway, since it hard to envisage how by-reference parameters could be implemented solely using method handles.

The second attempt introduced the concept of an argument _transformer_ responsible for marshalling a Java type to its FFM equivalent:

```java
public sealed interface Transformer permits IdentityTransformer, DefaultTransformer {
    /**
     * @return Memory layout of this type
     */
    MemoryLayout layout();
}
```

Simple types such as primitives (and conceivably the `MemorySegment` type) are marshalled as-is without transformation:

```java
public record IdentityTransformer(ValueLayout layout) implements Transformer {}
```

A second implementation parameterises a transformer for reference types:

```java
public non-sealed interface DefaultTransformer<T> extends Transformer {
    @Override
    default MemoryLayout layout() {
        return ValueLayout.ADDRESS;
    }

    /**
     * Marshals an argument to its native representation.
     * @param arg           Argument
     * @param allocator     Off-heap allocator
     * @return Native argument
     */
    Object marshal(T arg, SegmentAllocator allocator);
    
    /**
     * Provides a function to unmarshal a native return value of this type.
     * @return Unmarshalling function
     * @throws UnsupportedOperationException if this type cannot logically be returned from a native method
     */
    default Function<? extends Object, T> unmarshal() {
        throw new UnsupportedOperationException();
    }
}
```

Not all reference types can be logically be returned from a native method, hence `unmarshal` provides a function to transform return values.

> Two transformer implementations is probably overkill but makes subsequent code is a little simpler, particularly for the generic transformer which would otherwise require nasty wildcards, casts, etc.

The native method class is now defined in terms of transformers rather than raw types:

```java
public class NativeMethod {
    private final MethodHandle handle;
    private final Transformer returns;
    private final Transformer[] signature;
}
```

The method arguments are marshalled before invocation:

```java
private Object[] marshal(Object[] args) {
    var allocator = Arena.ofAuto();
    Object[] transformed = new Object[args.length];
    for(int n = 0; n < args.length; ++n) {
        transformed[n] = switch(parameters[n]) {
            case IdentityTransformer _ -> args[n];
            case DefaultTransformer def -> def.marshal(args[n], allocator);
        };
    }
    return transformed;
}
```

And similarly for the return value:

```java
private Object unmarshal(Object result) {
    return switch(returns) {
        case null -> null;
        case IdentityTransformer _ -> result;
        case DefaultTransformer def -> def.unmarshal().apply(result);
    };
}
```

### Registry

Next the following new component is introduced that manages transformers for the supported types, which for the moment is essentially a lookup table:

```java
public class Registry {
    private final Map<Class<?>, Transformer> registry = new HashMap<>();

    /**
     * Looks up or creates the native transformer for the given domain type.
     * @param type Domain type
     * @return Native transformer
     * @throws IllegalArgumentException if the type is not supported
     */
    public Transformer transformer(Class<?> type) {
        Transformer transformer = registry.get(type);
        if(transformer == null) throw new IllegalArgumentException(...);
        return transformer;
    }

    /**
     * Registers a native transformer for the given type.
     * @param type              Java or domain type
     * @param transformer       Native transformer
     */
    public void add(Class<?> type, Transformer transformer) {
        registry.put(type, transformer);
    }
}
```

The factory is refactored to lookup the relevant transformers for the methods signature:

```java
private NativeMethod build(Method method) {
    ...
    
    // Map return value
    Transformer returns = registry.transformer(method.getReturnType());

    // Map parameters
    List<Transformer> parameters = Arrays
        .stream(method.getParameterTypes())
        .map(registry::transformer)
        .toList();

    ...
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

### Supported Types

String values are supported by the following companion transformer:

```java
public class StringTransformer implements DefaultTransformer<String> {
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

A JOVE `Handle` is reimplemented as an immutable wrapper for an FFM pointer:

```java
public record Handle(MemorySegment address) {
    /**
     * Constructor.
     * @param address Memory address
     */
    public Handle {
        address = address.asReadOnly();
    }

    /**
     * @return Copy of the underlying memory address
     */
    public MemorySegment address() {
        return MemorySegment.ofAddress(address.address());
    }
}
```

With a companion transformer:

```java
public static class HandleTransformer implements DefaultTransformer<Handle> {
    public MemorySegment marshal(Handle arg, SegmentAllocator allocator) {
        return arg.address;
    }

    public Function<MemorySegment, Handle> unmarshal() {
        return Handle::new;
    }
}
```

And similarly for JOVE domain types:

```java
public static class NativeObjectTransformer implements DefaultTransformer<NativeObject> {
    public MemorySegment marshal(NativeObject arg, SegmentAllocator allocator) {
        return arg.handle().address();
    }
}
```

Note that in this case the `unmarshal` method is disallowed since a JOVE object will never be returned from the native layer.

To support custom JOVE and domain types that inherit from `NativeObject` a second step is added to the registry:

```java
public Transformer transformer(Class<?> type) {
    Transformer transformer = registry.get(type);

    if(transformer == null) {
        Transformer created = derive(type);
        add(type, created);
        return created;
    }
    else {
        return transformer;
    }
}
```

The `derive` method scans the registry for an available transformer with a suitable supertype:

```java
private Transformer derive(Class<?> type) {
    return registry
        .keySet()
        .stream()
        .filter(e -> e.isAssignableFrom(type))
        .findAny()
        .map(registry::get)
        .orElseThrow(...);
```

Note that the mapping to the supertype transformer is registered as a side-effect.

### Native References

As outlined above, the commonly used by-reference types (integer and pointer) will continue to be handled as a special case.

The following template implementation has a _pointer_ to the off-heap value:

```java
public abstract class NativeReference<T> {
    private T value;
    private MemorySegment pointer;

    /**
     * @return Referenced value or {@code null} if not populated
     */
    public T get() {
        ...
        return value;
    }
    
    /**
     * Updates the value of this reference.
     * @param pointer Pointer
     * @return Value
     */
    protected abstract T update(MemorySegment pointer);
}
```

The companion transformer allocates the pointer on-demand:

```java
public static class NativeReferenceTransformer implements DefaultTransformer<NativeReference<?>> {
    @Override
    public MemorySegment marshal(NativeReference<?> ref, SegmentAllocator allocator) {
        if(ref.pointer == null) {
            ref.pointer = allocator.allocate(ValueLayout.ADDRESS);
        }

        return ref.pointer;
    }
}
```

After invocation the by-reference value can be retrieved from the off-heap pointer:

```java
public T get() {
    if((value == null) && (pointer != null)) {
        value = update(pointer);
    }

    return value;
}
```

A by-reference integer is implemented by the following specialisation:

```java
public static class IntegerReference extends NativeReference<Integer> {
    @Override
    protected Integer update(MemorySegment pointer) {
        return pointer.get(ValueLayout.JAVA_INT, 0L);
    }
}
```

And similarly for general pointers:

```java
public static class Pointer extends NativeReference<Handle> {
    @Override
    protected Handle update(MemorySegment pointer) {
        MemorySegment address = pointer.get(ValueLayout.ADDRESS, 0L);
        return new Handle(address);
    }
}
```

### Integration #2

The last step is to register the various transformers required to support the GLFW demo:

```java
public final class DefaultRegistry {
    public static Registry create() {
        Registry registry = new Registry();
        primitives(registry);
        registry.add(String.class, new StringTransformer());
        registry.add(NativeReference.class, new NativeReferenceTransformer());
        registry.add(Handle.class, new HandleTransformer());
        registry.add(NativeObject.class, new NativeObjectTransformer());
        return registry;
    }
}
```

Where the transformers for the primitives types are created by the following helper:

```java
private static void primitives(Registry registry) {
    ValueLayout[] primitives = {
        JAVA_BOOLEAN,
        JAVA_BYTE,
        JAVA_CHAR,
        JAVA_SHORT,
        JAVA_INT,
        JAVA_LONG,
        JAVA_FLOAT,
        JAVA_DOUBLE
    };

    for(ValueLayout layout : primitives) {
        var transformer = new IdentityTransformer(layout);
        Class carrier = layout.carrier();
        registry.add(carrier, transformer);
    }
}
```

The GLFW library is now fully supported by the new framework other than the following which are deferred until later:

* Callback support for GLFW devices is temporarily hacked out.

* The `glfwGetRequiredInstanceExtensions` edge case is hard-coded for the moment.

Refactoring of the `Desktop` service resulted in the following:

```java
public static Desktop create() {
    // Build GLFW library
    var registry = DefaultRegistry.create();
    var factory = new NativeLibraryFactory("glfw3", registry);
    DesktopLibrary lib = factory.build(DesktopLibrary.class);

    // Init GLFW
    int result = lib.glfwInit();
    if(result != 1) throw new RuntimeException(...);

    // Create desktop service
    return new Desktop(lib, new NativeReference.Factory());
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
        
    desktop.terminate();
}
```

---

## Structures

### Overview

At this point we feel confident that the framework is ready to support Vulkan, with a couple of workarounds:

* The code-generator will require significant rework to remove JNA and generate the FFM memory layouts of the structures.  For the moment we prefer to defer this work until the bulk of the refactoring has been completed.  In the meantime some text editor magic is used to remove the JNA dependencies and the structure layouts will be hand-crafted as required.

* The new framework builds and validates the _whole_ API at instantiation-time.  Therefore both the GLFW and Vulkan APIs are cut back to the bare minimum and the methods will be gradually reintroduced as we progress.

Remarkably the refactoring work went much more smoothly than anticipated, with only a handful of compilation problems.  Apparently the existing JOVE code did a decent job of abstracting over JNA.

The approach from this point is to add Vulkan to the prototype and tackle each JOVE component in order starting with the `Instance` domain object.  This requires implementing marshalling support for native structures, the main subject of this section.  However there are various tidying tasks and a few framework enhancements to be implemented first.

### Further Framework

#### Return Handler

The approach of using a proxy enables the new framework to validate _all_ Vulkan return codes in one location, whereas previously _every_ native call had to be wrapped with the `check` helper method which was ugly and slightly tedious.

First a handler for return values is added to the factory:

```java
public class NativeLibraryFactory {
    private Consumer<Object> check = result -> {};
}
```

Return values are simply delegated to the configured handler after invocation:

```java
new InvocationHandler() {
    public Object invoke(...) {
        ...
        Object result = ...
        check.accept(result);
        return result;
    }
};
```

The Vulkan library is refactored in terms of the new framework and a custom return handler validates the success code:

```java
static VulkanLibrary create() {
    // Init API factory
    Registry registry = DefaultRegistry.create();
    var factory = new NativeLibraryFactory("vulkan-1", registry);

    // Handle success codes
    Consumer<Object> handler = code -> {
        if((code instanceof VkResult result) && (result != VkResult.SUCCESS)) {
            throw new VulkanException(result);
        }
    };
    factory.setReturnValueHandler(handler);

    // Build API proxy
    return factory.build(VulkanLibrary.class);
}
```

Nice.

#### Empty Values

The framework now supports reference types that can legitimately be `null` for unused or empty values, which FFM represents with the `MemorySegment.NULL` constant.

An `empty` value is added to the definition of the transformer:

```java
interface DefaultTransformer<T> ... {
    default Object empty() {
        return MemorySegment.NULL;
    }
}
```

And a convenience helper is also added that switches behaviour accordingly:

```java
static Object marshal(Object arg, DefaultTransformer transformer, SegmentAllocator allocator) {
    if(arg == null) {
        return transformer.empty();
    }
    else {
        return transformer.marshal(arg, allocator);
    }
}
```

#### Enumerations

Both of the structures used to create the Vulkan instance are dependant on integer enumerations which requires the implementation of another companion transformer:

```java
class IntEnumTransformer implements DefaultTransformer<IntEnum> {
    private final ReverseMapping<?> mapping;

    public MemoryLayout layout() {
        return ValueLayout.JAVA_INT;
    }

    public Integer empty() {
        return mapping.defaultValue().value();
    }

    public Integer marshal(IntEnum e, SegmentAllocator allocator) {
        return e.value();
    }

    public Function<Integer, IntEnum> unmarshal() {
        return value -> {
            if(value == 0) {
                return mapping.defaultValue();
            }
            else {
                return mapping.map(value);
            }
        };
    }
}
```

Note that the return type and `empty` values are integers in this case.

The transformer for an enumeration bitfield is refactored similarly:

```java
public static class EnumMaskTransformer implements DefaultTransformer<EnumMask<?>> {
    public MemoryLayout layout() {
        return ValueLayout.JAVA_INT;
    }

    public Integer marshal(EnumMask<?> mask, SegmentAllocator allocator) {
        return mask.bits;
    }

    public Integer empty() {
        return 0;
    }

    public Function<Integer, EnumMask<?>> unmarshal() {
        return EnumMask::new;
    }
}
```

/////////////////

The new transformer for an `IntEnum` is specific to a given enumeration class (i.e. for the reverse mapping) which implies an intermediate mechanism to generate a transformer for a given subclass (as opposed to a singleton that handles _all_ instances of that class).  This mechanism will also be required shortly for structures which will be similarly class-specific.

Therefore the registry is extended to configure _transformer factories_ that are responsible for generating new transformers as required:

```java
public class Registry {
    @FunctionalInterface
    public interface Factory<T> {
        /**
         * Creates a new transformer for the given type.
         * @param type Domain type
         * @return Transformer
         */
        Transformer<T> create(Class<? extends T> type);
    }

    private final Map<Class<?>, Factory<?>> factories = new HashMap<>();

    public <T> void add(Class<T> type, Factory<T> factory) {
        factories.put(type, factory);
    }
}
```

The process of creating a transformer for a given type is now extended accordingly:

```java
private Transformer<?> find(Class<?> type) {
    return create(type)
        .or(() -> derive(type))
        .orElseThrow(() -> new IllegalArgumentException(...));
}
```

Where the `create` method instantiates a new transformer if it is supported by a configured factory:

```java
private Optional<Transformer<?>> create(Class<?> type) {
    return factories
        .keySet()
        .stream()
        .filter(e -> e.isAssignableFrom(type))
        .findAny()
        .map(factories::get)
        .map(factory -> factory.create(type));
}
```

In summary the process of determining the transformer for a given type now becomes:

1. Lookup the existing registered transformer if present.

2. Create a new transformer from the relevant factory if one matches.

3. Derive from a registered transformer if there is a suitable supertype present.

4. Otherwise the type is not supported.

Possible future enhancements:

* The mechanism to `derive` a supertype transformer could probably be encapsulated into a separate transformer factory.

* A factory is currently matched by class type, this may be expanded to a general predicate later if required.

////////////////

#### Structures

To support structures the following replacement definition is introduced that provides the FFM memory layout:

```java
public interface NativeStructure {
    /**
     * @return Memory layout of this structure
     */
    StructLayout layout();
}
```

A companion factory generates a new transformer instance for a given structure as required:

```java
class StructureTransformerFactory implements Registry.Factory<NativeStructure> {
    public Transformer<NativeStructure> create(Class<? extends NativeStructure> type) {
        ...
    }
}
```

Which for the moment does nothing other than allocate the off-heap memory:

```java
return new Transformer<>() {
    private final StructLayout layout = mappings.layout();

    public MemorySegment marshal(NativeStructure structure, SegmentAllocator allocator) {
        MemorySegment address = allocator.allocate(layout);
        ...
        return address;
    }

    public Function<MemorySegment, NativeStructure> unmarshal() {
        ...
    }
}
```

#### Arrays

Arrays are the final type missing from the framework and will be required for the extensions and validation layers of the Vulkan instance.

The following new implementation composes a transformer that marshals the _component type_ of the array:

```java
public class ArrayTransformer<T> implements Transformer<T[]> {
    private final Transformer<T> component;

    public MemoryLayout layout() {
        return ValueLayout.ADDRESS;
    }

    public MemorySegment marshal(T[] array, SegmentAllocator allocator) {
        ...
        return address;
    }
    
    public Function<?, T[]> unmarshal() {
        throw new UnsupportedOperationException();
    }
}
```

The `marshal` method first allocates the off-heap memory as a contiguous block given the layout of the component:

```java
MemoryLayout layout = component.layout();
MemorySegment address = allocator.allocate(layout, array.length);
```

Next a _var handle_ is created to access the elements of the array:

```java
var handle = Transformer.removeOffset(layout.arrayElementVarHandle());
```

The handle is essentially the same as using the various mutator methods on the `MemorySegment` class but is more convenient in this case.

Modifying the off-heap data requires a byte offset argument (a _coordinate_ in FFM terms) even though this value is almost always zero.  The `removeOffset` method is a convenience helper that initialises this argument once:

```java
public static VarHandle removeOffset(VarHandle handle) {
    return MethodHandles.insertCoordinates(handle, 1, 0L);
}
```

Finally each element is transformed and written to the off-heap memory using the handle:

```java
for(int n = 0; n < array.length; ++n) {
    Object element = Transformer.marshal(array[n], component, allocator);
    handle.set(address, (long) n, element);
}
```

Whilst this class is public and can be used directly by the application (if required) it seems logical that an array of a supported type should also be automatically marshalled.

The `find` method of the registry is modified to treat arrays as a special case:

```java
private Transformer<?> find(Class<?> type) {
    if(type.isArray()) {
        return array(type);
    }
    ...
}        
```

Which creates and automatically registers an array transformer for a given component type:

```java
private Transformer<?> array(Class<?> type) {
    Transformer<?> component = find(type.getComponentType());
    Transformer transformer = new ArrayTransformer<>(component);
    add(type, transformer);
    return transformer;
}
```

This will allow extensions and validation layers to be configured as string-arrays in the revised instance structure below.

Note that for the moment this implementation only supports arrays of reference types (e.g. an array of strings or handles).  Additional functionality will be introduced later to handle arrays of structures and primitives which have further constraints.

### Integration #3

The `Instance` domain class can now be refactored with the following temporary fiddles:

* The `VulkanLibrary` is cut down to just the relevant instance methods

* The diagnostics handler is temporarily removed.

The instance library now looks like this:

```java
interface Library {
    int     vkCreateInstance(VkInstanceCreateInfo pCreateInfo, Handle pAllocator, Pointer pInstance);
    void    vkDestroyInstance(Instance instance, Handle pAllocator);
    int     vkEnumerateInstanceExtensionProperties(String pLayerName, IntegerReference pPropertyCount, @Returned VkExtensionProperties[] pProperties);
    int     vkEnumerateInstanceLayerProperties(IntegerReference pPropertyCount, @Returned VkLayerProperties[] pProperties);
    Handle  vkGetInstanceProcAddr(Instance instance, String pName);
}
```

Notes:

* Array parameters are now explicitly specified as Java arrays.  Previously these were declared as a __single__ instance since JNA mandated that a contiguous block was represented by the __first__ element of an array.

* The `@Returned` annotation denotes parameters that are returned by-reference (handled later in this chapter).

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

* The extensions and validation layers are now defined as string-arrays which will be automatically handled by the array transformer.

> We considered completely overhauling native structures by code-generating builders (as LWJGL does) to replace the fully mutable public fields.  However since structures are generally only used as internal data carriers the refactoring effort would probably outweigh any benefit of slightly better encapsulation.

The prototype can now be extended to instantiate Vulkan and configure the instance:

```java
VulkanLibrary vulkan = VulkanLibrary.create();

Instance instance = new Instance.Builder()
    .name("Prototype")
    .extension("VK_EXT_debug_utils")
    .layer(ValidationLayer.STANDARD_VALIDATION)
    .build(vulkan);
```

This code compiles and runs but does nothing since we are not actually populating the structures.

### Field Mapping

Marshalling a structure requires the transformer factory to construct and iterate over the _field mappings_ between the Java structure and the off-heap memory layout.

First an instance of the structure is created such that the memory layout can be retrieved:

```java
public Transformer<NativeStructure> create(Class<? extends NativeStructure> type) {
    Constructor<? extends NativeStructure> constructor = type.getConstructor();
    Supplier<? extends NativeStructure> factory = () -> constructor.newInstance();
    NativeStructure instance = factory.get();
    StructLayout layout = instance.layout();
    ...
}
```

Notes:

* Exception handling and helper methods have been silently inlined for brevity.

* The `factory` is also used when unmarshalling structures later.

Next control is delegated to a helper which constructs the field mappings for the structure:

```java
Helper helper = new Helper(type, layout);
List<FieldMapping> fields = helper.build();
```

The `build` method iterates over the members of the memory layout and constructs each field mapping:

```java
private List<FieldMapping> build() {
    return layout
        .memberLayouts()
        .stream()
        .filter(Predicate.not(e -> e instanceof PaddingLayout))
        .map(this::mapping)
        .toList();
}
```

The structure field is reflected via the name configured in the structure layout:

```java
private FieldMapping mapping(MemoryLayout member) {
    String name = member.name().orElseThrow(...);
    Field field = type.getField(name);
    ...
}
```

Which is converted to a handle:

```java
VarHandle local = MethodHandles.lookup().unreflectVarHandle(field);
```

Similarly a handle is created to the off-heap field:

```java
PathElement path = PathElement.groupElement(name);
VarHandle foreign = Transformer.removeOffset(layout.varHandle(path));
```

And these are both composed with the transformer into a new type:

```java
Transformer<?> transformer = registry.transformer(field.getType());
return new FieldMapping(local, foreign, transformer);
```

The process of marshalling a structure field is:

1. Retrieve the value of the field from the structure.

2. Apply the transformer.

3. Set the transformed field in the off-heap memory.

Which is implemented as follows:

```java
private record FieldMapping(VarHandle local, VarHandle foreign, Transformer<?> transformer) {
    public void marshal(NativeStructure structure, MemorySegment address, SegmentAllocator allocator) {
        Object value = local.get(structure);
        Object result = Transformer.marshal(value, transformer, allocator);
        foreign.set(address, result);
    }
}
```

Where `local` and `foreign` are the handles to the structure field and off-heap field respectively.

The final step is to create the transformer which is essentially a compound adapter for the field mappings:

```java
return new Transformer<>() {
    public MemoryLayout layout() {
        return layout;
    }

    public MemorySegment marshal(NativeStructure structure, SegmentAllocator allocator) {
        MemorySegment address = allocator.allocate(layout);
        for(FieldMapping f : fields) {
            f.marshal(structure, address, allocator);
        }
        return address;
    }
};
```

Notes:

* Unmarshalling of structures is handled below when the diagnostic handler is integrated.

* Pointers to other structures (e.g. the `pApplicationInfo` field of the instance descriptor) are automatically handled by this mechanism recursively, since a field mapping composes a transformer which can also marshal further structures.

* The structure layout conveniently specifies the order of the fields _explicitly_ since it must obviously match the native representation exactly (including alignment).  This is unlike reflection where the order is arbitrary, hence there is no need for anything like the `@FieldOrder` annotation that was required for JNA structures.

It is worth considering that the `marshal` method allocates and populates the off-heap memory on _every_ invocation, with the memory being discarded afterwards.  A future enhancement _could_ cache and reuse the memory address and implement some sort of synchronisation mechanism (which is the JNA approach), but this would be quite complex and is not required for the forseeable future.  In reality native structures only have scope during marshalling, therefore the simpler approach seems more palatable.

In the prototype the Vulkan instance should now be successfully instantiated and configured using the new framework.  A lot of work to get to the stage JOVE was at several years previously.

### Unmarshalling

To reduce the chance of breaking something the diagnostics handler will be reintroduced now before progressing further, which requires _unmarshalling_ of the diagnostic report structure.

#### Diagnostic Handler

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

In the builder for the diagnostics handler the native methods can now be retrieved via a custom lookup service:

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
var lib = factory.build(HandlerLibrary.class);
```

The handler is instantiated by invoking the relevant function pointer:

```java
var handle = new Pointer();
lib.vkCreateDebugUtilsMessengerEXT(instance, info, null, handle);
return new DiagnosticHandler(handle.get(), instance, lib);
```

And finally the handler class itself is refactored accordingly:

```java
public class DiagnosticHandler extends TransientNativeObject {
    private final Instance instance;
    private final HandlerLibrary lib;

    @Override
    protected void release() {
        lib.vkDestroyDebugUtilsMessengerEXT(instance, this, null);
    }
}
```

This approach nicely reuses the existing framework rather than having to implement more FFM code from the ground up.

#### Callback Address

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

#### Diagnostic Report

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

---

//////////////////////

## Returned Parameters

### Returned Annotation

The next Vulkan component after the instance is the physical device which uses native methods that are highly dependant on by-reference parameters.  These pose a problem because by-reference and 'normal' parameters cannot be distinguished solely from the method signature.

JNA used the rather cumbersome `ByReference` interface which generally required _every_ structure to have _two_ implementations.  Additionally _every_ array is essentially a by-reference parameter as far as JNA is concerned, whereas we would prefer to separate the normal and by-reference cases.

A generic wrapper type could be introduced that would allow the framework to switch behaviour accordingly.  However the actual type of the reference would then not be known at runtime (due to type erasure) meaning the framework would be unable to build and validate a transformer when the library is created.

The most obvious solution is to implement a custom annotation to explicitly declare by-reference parameters and still have access to the actual reference type.

The annotation is a simple _marker_ interface with no declared functionality:

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
