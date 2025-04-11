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
- [Other](#other)

---

## Overview

It has always been the intention at some point to re-implement the Vulkan native layer using the FFM (Foreign Function and Memory) library developed as part of project [Panama](https://openjdk.java.net/projects/panama/).  The reasons for delaying this rework are discussed in [code generation](/jove-blog/blog/part-1-intro/code-generation#alternatives) at the very beginning of the blog.

With the LTS release of JDK 21 the FFM library was elevated to a preview feature (rather than requiring an incubator build) and much more information and tutorials started to appear online, so it high time to replace JNA with the more future proofed (and safer) FFM solution.

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

1. Find the off-heap address of the the method via lookup service.

2. Build a `FunctionDescriptor` specifying the signature of the method.

3. Combine both of these to link a `MethodHandle` to the native method.

4. Invoke the handle with an array of the appropriate arguments.

The GLFW library must first be initialised which returns an integer success code:

```java
MethodHandle glfwInit = linker.downcallHandle(
    lookup.find("glfwInit").orElseThrow(),
    FunctionDescriptor.of(ValueLayout.JAVA_INT)
);
println("glfwInit=" + glfwInit.invoke());
```

The GLFW version string can now be retrieved:

```java
MethodHandle glfwGetVersionString = linker.downcallHandle(
    lookup.find("glfwGetVersionString").orElseThrow(),
    FunctionDescriptor.of(ValueLayout.ADDRESS);
);
MemorySegment versionAddress = (MemorySegment) glfwGetVersionString.invoke();
String version = versionAddress.reinterpret(Integer.MAX_VALUE).getString(0L);
println("glfwGetVersionString=" + version);
```

Notes:

* The `0L` argument in the `getString` helper is an offset into the off-heap memory.

* The returned address has to be _reinterpreted_ to expand the bounds of the off-heap memory, in this case to an undefined length for a pointer to a null-terminated character array.

Support for Vulkan can also be queried:

```java
MethodHandle glfwVulkanSupported = linker.downcallHandle(
    lookup.find("glfwVulkanSupported").orElseThrow(),
    FunctionDescriptor.of(ValueLayout.JAVA_BOOLEAN);
);
println("glfwVulkanSupported=" + glfwVulkanSupported.invoke());
```

Retrieving the supporting Vulkan extensions is slightly complex since the API method returns a pointer to a string-array and uses a _by reference_ integer to return the size of the array as a side-effect:

```java
MethodHandle glfwGetRequiredInstanceExtensions = linker.downcallHandle(
    lookup.find("glfwGetRequiredInstanceExtensions").orElseThrow(),
    FunctionDescriptor.of(ValueLayout.ADDRESS, ValueLayout.ADDRESS);
);
MemorySegment count = arena.allocate(ValueLayout.ADDRESS);
MemorySegment extensions = (MemorySegment) glfwGetRequiredInstanceExtensions.invoke(count);
```

The length of the array is extracted from the by-reference argument to resize the off-heap memory:

```java
int length = count.get(ValueLayout.JAVA_INT, 0L);
MemorySegment array = extensions.reinterpret(length * ValueLayout.ADDRESS.byteSize());
```

And each element is extracted and transformed to a Java string:

```java
for(int n = 0; n < length; ++n) {
    MemorySegment e = array.getAtIndex(ADDRESS, n);
    println("extension=" + e.reinterpret(Integer.MAX_VALUE).getString(0L));
}
```

Finally GLFW can be closed (a `void` method without parameters):

```java
MethodHandle glfwTerminate = linker.downcallHandle(
    lookup.find("glfwTerminate").orElseThrow(),
    FunctionDescriptor.ofVoid()
);
glfwTerminate.invoke();
```

Note that `reinterpret` is an unsafe operation that can result in a JVM crash for an out-of-bounds array or string length, since the actual size off the off-heap cannot be guaranteed by the JVM.

Additionally this method is _restricted_ and will generate a runtime warning, which can be suppressed with the following VM argument:

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

* In particular enumerations are implemented as static integer accessors (WTF!) 

* Implies throwing away the existing code generated enumerations and structures.

Alternatively an abstraction layer based on the existing hand-crafted API and code-generated classes would have the following advantages:

* The native API and structures are expressed in domain terms.

* Proper enumerations.

* Retains the existing API and code-generated enumerations and structures.

On the other hand:

* Substantial development effect.

* Proprietary solution with the potential for unknown problems further down the line.

* The FFM memory layout needs to be generated for _all_ structures.

Despite the advantages of `jextract` the hybrid solution is clearly preferable - the ugly bindings, lack of type safety, etc. were the very concerns that led us to abandon LWJGL in the first place.

> In any case the application and/or JOVE would _still_ need to transform domain types to/from the FFM equivalents even if `jextract` was used to generate the API.

The source code generated by `jextract` will be retained as it will be a useful resource during development, particularly for the memory layout of structures.

### Requirements

With this decision made (for better or worse) the requirements for the new framework can be determined:

* A factory mechanism that generates an FFM-based implementation of a native library, based on the existing GLFW and Vulkan API definitions.  The API should be expressed in _domain_ terms to avoid the type-safety and self-documenting concerns outlined above.

* A _marshalling_ framework that transforms supported Java types to/from the FFM equivalents.

* Support for by-reference parameters.

* Support for callbacks for GLFW device polling and the Vulkan diagnostics handler.

* Refactoring of the code generator to build the FFM memory layout for all structures and remove the JNA artifacts (imports and the `@FieldOrder` annotation).

The marshalling framework will need to support the following types:

* primitives

* strings

* structures

* domain types, i.e. reference types such as `Handle` or subclasses of `NativeObject`

* integer enumerations and bitfields

Which are used in the following use-cases:

* parameter

* return value

* array parameter

* structure field

* parameter returned by reference (sometimes referred to as an _in/out_ parameter)

* array elements returned by reference

These requirements form a matrix that can be illustrated as a table:

| type              | parameter | return value  | array     | by reference  | array by reference    | structure field   |
| ----              | --------- | ------------  | -----     | ------------  | ------------------    | ---------------   |
| primitive         | yes       | yes           | yes       | integer only  | ?                     | yes               |
| string            | yes       | yes           | yes       | no            | ?                     | yes               |
| structure         | yes       | yes           | yes       | yes           | yes                   | yes               |
| domain            | yes       | yes (usually) | yes       | pointers only | yes                   | yes               |
| enumerations      | yes       | yes           | ?         | ?             | ?                     | yes               |

Entries marked as ? are either out-of-scope or unclear at this point.

From the above we can make the following observations:

1. For the moment at least it seems logical to continue to treat by-reference integers and pointers as special cases.  They are already integral to JOVE (based on the existing JNA equivalents) and there is no Java analog for an by-reference integer in particular.

2. A structure is the only type that can be returned by-reference as a __single__ parameter.

3. The other by-reference use-cases imply some form of post-processing to 'copy' the off-heap memory into the structure or array.

4. GLFW makes heavy use of callbacks for device polling which will require substantial factoring to use FFM upcall stubs.

5. The `glfwGetRequiredInstanceExtensions` method is an edge-case (for reasons detailed later) and is also the only method in either library that returns an array, therefore this requirement is deferred for the moment and the Vulkan extensions are temporarily hard-coded.

From the above the required components for the new framework are:

* A _native library builder_ responsible for constructing the FFM implementation of a native API expressed in _domain_ terms.

* A set of _transformers_ for the supported types which are responsible for allocation of off-heap memory and marshalling to/from the equivalent FFM representation.

* A _native method_ class that composes an FFM method handle and the transformers for its parameters and optional return type.

* A custom annotation to identify by-reference parameters and logic to unmarshal these parameters after method invocation.

* Support for arrays of the supported types (including by-reference array parameters).

* Automated generation of structure memory layouts.

### Approach

Integration of the new framework into JOVE entails the following changes:

* Removal of the JNA library.

* Implementation of the new framework components outlined above.

* Implementation of transformers for the supported types.

* A global find-and-replace to refactor the GLFW and Vulkan APIs in terms of the new framework.

A 'big bang' approach always feels inherently risky simply due to the shear scale of the changes to be made.  Therefore the plan is to create a throwaway prototype targeted at a cut-down API (again using GLFW to begin with) such that we can focus on the development of the new framework without getting bogged down in compilation issues.  Once this is up and running _then_ we will integrate the code into JOVE and iteratively reintroduce each component refactoring accordingly.

As already noted, Vulkan is very front-loaded with complexity, so framework support will be implemented as we progress.  Once we are confident that the bulk of the challenges facing the new framework have been addressed, the prototype will be discarded and refactoring can continue with the existing suite of demo applications.

---

## Framework

### Native Library

The logical starting point is the factory class that builds a native library.

This involves the following steps:

1. Enumerate the native methods from the user defined API.

2. Create and link an FFM method handle for each identified API method.

3. Create a Java proxy that delegates API calls to the underlying FFM methods.

This process is implemented as follows:

```java
public class NativeLibraryBuilder {
    private final SymbolLookup lookup;
    private final Linker linker = Linker.nativeLinker();

    public <T> T build(Class<T> api) {
        if(!api.isInterface()) throw new IllegalArgumentException(...);
        Map<Method, NativeMethod> methods = methods(api);
        return proxy(api, methods);
    }
}
```

Where _api_ is the user defined interface specifying the API to be implemented.

The `NativeMethod` class is a wrapper for the resultant FFM handle:

```java
public class NativeMethod {
    private MemoryHandle handle;
    
    public Object invoke(Object[] args) {
        return handle.invokeWithArguments(args);
    }
}
```

The API methods are enumerated by reflecting the API and building a wrapper for each native method:

```java
private Map<Method, NativeMethod> methods(Class<?> api) {
    return Arrays
        .stream(api.getMethods())
        .filter(NativeLibraryBuilder::isNativeMethod)
        .collect(toMap(Function.identity(), this::build));
}
```

Where a native method is defined as a public, non-static method of the API:

```java
private static boolean isNativeMethod(Method method) {
    int modifiers = method.getModifiers();
    return !Modifier.isStatic(modifiers);
}
```

The overloaded `build` method finds the symbol (i.e. the off-heap address) of the method by name and temporarily creates an empty wrapper:

```java
public NativeMethod build(Method method) {
    MemorySegment address = lookup.find(method.getName()).orElseThrow(...);                
    return new NativeMethod(...);
}
```

Next an _invocation handler_ is created that delegates Java method calls to the underlying native method:

```java
private <T> T proxy(Class<T> api, Map<Method, NativeMethod> methods) {
    var handler = new InvocationHandler() {
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            NativeMethod delegate = methods.get(method);
            return delegate.invoke(args);
        }
    }
    ...
}
```

From which a Java proxy is created:

```java
ClassLoader loader = this.getClass().getClassLoader();
return (T) Proxy.newProxyInstance(loader, new Class<?>[]{api}, handler);
```

This returns an implementation of the user-defined API which delegates method calls to the underlying native method.

The approach of using a proxy enables the new framework to validate _all_ Vulkan return codes in one location, previously _every_ native call had to be wrapped with the `check` helper method to test the result which was ugly and slightly tedious.

First a handler for return values is added to the factory:

```java
public class NativeLibraryBuilder {
    private Consumer<Object> check = result -> {};
}
```

The invocation handler simply delegates return values to the configured handler after invocation:

```java
var handler = new InvocationHandler() {
    public Object invoke(...) {
        ...
        Object result = ...
        check.accept(result);
        return result;
    }
};
```

For convenience a specialised integer handler can be configured:

```java
public void setIntegerReturnHandler(IntConsumer handler) {
    Consumer<Object> wrapper = result -> {
        if(result instanceof Integer code) {
            handler.accept(code);
        }
    };
    setReturnHandler(wrapper);
}
```

Which is integrated into the Vulkan implementation accordingly:

```java
public static Vulkan create() {
    ...
    var factory = new NativeLibraryBuilder(registry);
    factory.setIntegerReturnHandler(Vulkan::check);
    ...
}
```

Nice.

### Native Method

Building a native method requires the parameter signature and return type in order to link the FFM method handle.  Looking ahead we will also need to construct native methods programatically (i.e. not using reflection) for the diagnostics handler and possibly other use cases.  Therefore this class will be agnostic to the source of the method signature and the library symbol.

First a builder is added to construct a native method:

```java
public static class Builder {
    private final Linker linker = Linker.nativeLinker();
    private MemorySegment address;
    private Class<?> returns;
    private final List<Class<?>> signature = new ArrayList<>();
}
```

Construction of the native method handle requires the symbol and an FFM function descriptor.  First the memory layout of the method is derived from the signature:

```java
public NativeMethod build() {
    MemoryLayout[] layouts = Arrays
        .stream(signature)
        .map(NativeLibraryBuilder::layout)
        .toArray(MemoryLayout[]::new);
        
    ...
}
```

Where `layout` is a temporary, quick-and-dirty mapper that just supports the primitive `int` type needed for a basic GLFW demonstration:

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

    if((returns == null) || (returns == void.class)) {
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
    FunctionDescriptor descriptor = descriptor(returns, layout);
    MethodHandle handle = linker.downcallHandle(address, descriptor);
    return new NativeMethod(handle);
}
```

The factory can now be updated to use this new builder to instantiate a native method for each API method:

``` java
public NativeMethod build(Method method) {
    MemorySegment address = ...

    var builder = new NativeMethod.Builder()
        .address(address)
        .returns(method.getReturnType());

    for(Parameter p : method.getParameters()) {
        builder.parameter(p.getType());
    }

    return builder.build();
}
```

To exercise the new framework a second prototype is created to test GLFW, using the following (very) cut-down API dependant only on integer primitives:

```java
interface API {
    int         glfwInit();
    void        glfwTerminate();
    int         glfwVulkanSupported();
    //String    glfwGetVersionString();
    //String[]  glfwGetRequiredInstanceExtensions(&int count);
}
```

A proxy implementation is generated using the new factory:

```java
try(Arena arena = Arena.ofConfined()) {
    var lookup = SymbolLookup.libraryLookup("glfw3", arena);
    var factory = new NativeLibraryBuilder(lookup);
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

* Maps and links the user-defined API methods to the native library.

* Maps the domain type of the parameters and the return value to the equivalent FFM memory layout (albeit only for primitive integers so far).

* Transparently delegates API calls to the underlying native methods.

### Native Transformer

With the basic framework in place the next step is to implement support for the remaining primitives and Java reference types, such that the API can be expressed in domain terms.

The first attempt used the functionality provided by the `MethodHandle` class (used extensively in FFM) which supports transformation of the incoming arguments and the method return value.  This initially seemed a perfect fit for the requirements, in particular the framework would only need custom handling for non-primitive types.  However in practice the code became convoluted very quickly, since the logic to modify a method argument is generally also implemented as a 'target' method handle, which entails fiddly lookup and binding code for all but the most trivial cases.

After some time we abandoned this approach in favour of pre/post-processing the arguments around invocation of the native method.  The code became much clearer and easier to unit-test, at the expense of more framework code and the overhead of additional processing on method invocation.  Of importance is that checks for error cases can still be made when the native method is being constructed.

> As it turns out some form of pre/post-processing would probably have been required anyway, since it hard to envisage how by-reference parameters could be implemented solely using the `MethodHandle` class.

The second attempt started with the concept of an argument _transformer_ which is responsible for marshalling a Java type to/from its FFM equivalent (inspired by JNA type converters).

After some further analysis of the required types to be supported by the framework, the following groups or _archetypes_ were identified:

* The identity transformer for arguments that do not require any transformation.  These are the types directly supported by FFM, namely Java primitives and possibly the `MemorySegment` type.

* A reference type transformer, either built-in helpers for common use-cases (e.g. string) or custom domain types.

* Structures

* Arrays

The new argument transformer is modelled on the above:

```java
public sealed interface Transformer permits IdentityTransformer, AddressTransformer {
    /**
     * @return Native memory layout
     */
    MemoryLayout layout();
}
```

The identity transformer is trivial:

```java
public record IdentityTransformer(MemoryLayout layout) implements Transformer {
}
```

The native method wrapper class is next refactored in terms of transformers:

```java
public class NativeMethod {
    private final MethodHandle handle;
    private final Transformer returns;
    private final Transformer[] signature;
}
```

The `signature` transformers are applied before invocation to marshal the arguments to FFM types:

```java
public Object invoke(Object[] args) {
    Object[] foreign = marshal(args, Arena.ofAuto());
    Object result = handle.invokeWithArguments(foreign);
    ...
}
```

The `marshal` method delegates responsibility to the various transformer archetypes:

```java
private Object[] marshal(Object[] args, Arena arena) {
    if(args == null) {
        return null;
    }

    Object[] transformed = new Object[args.length];
    for(int n = 0; n < args.length; ++n) {
        transformed[n] = switch(transformer) {
            case IdentityTransformer _ -> args[n];
            case AddressTransformer ref -> ref.marshal(args[n]);
            ...
        };
    }

    return transformed;
}
```

And similarly for the return value:

```java
return switch(returns) {
    case null -> null;
    case IdentityTransformer _ -> result;
    case AddressTransformer ref -> ref.unmarshal(result);
};
```

### Address Transformer

The _address transformer_ will be an extension point for the framework supporting both built-in and custom types reference types:

```java
public non-sealed interface AddressTransformer<T extends Object, R> extends Transformer {
    @Override
    default MemoryLayout layout() {
        return ValueLayout.ADDRESS;
    }

    /**
     * Marshals a reference argument to its native representation.
     * @param arg           Java argument
     * @param allocator     Off-heap allocator
     * @return Pointer
     */
    Object marshal(T arg, SegmentAllocator allocator);

    /**
     * Unmarshals a native value.
     * @param value Native value
     * @return Reference type
     * @throws UnsupportedOperationException if this type cannot logically be returned from a native method
     */
    T unmarshal(R value);
}
```

Where `T` is a Java or domain reference type and `R` is the corresponding FFM type (usually `MemorySegment`).

An application can now define custom transformers which are registered with the framework via a new _registry_ component:

```java
public class Registry {
    private final Map<Class<?>, Transformer> registry = new HashMap<>();

    public void add(Class<?> type, Transformer transformer) {
        registry.put(type, transformer);
    }
}
```

The transformer for a supported type can be looked up by class:

```java
public Transformer get(Class<?> type) {
    Transformer transformer = registry.get(type);
    if(transformer == null) {
        ...
    }
    else {
        return transformer;
    }
}
```

If a transformer is not found the registry scans the registered types to find a supported supertype:

```java
private Transformer find(Class<?> type) {
    return registry
        .keySet()
        .stream()
        .filter(e -> e.isAssignableFrom(type))
        .findAny()
        .map(registry::get)
        .orElseThrow(...);
```

And the mapping to the supertype transformer is registered as a side-effect:

```java
if(transformer == null) {
    Transformer supertype = find(type);
    add(type, supertype);
    return supertype;
}
```

The registry is integrated into the `NativeMethod` builder which is refactored to lookup the transformer for the given parameters and return value.  Finally the FFM function descriptor is derived indirectly via the `layout` method of the transformers:

```java
private static FunctionDescriptor descriptor(...) {
    MemoryLayout[] layouts = signature
            .stream()
            .map(Transformer::layout)
            .toArray(MemoryLayout[]::new);

    FunctionDescriptor descriptor = FunctionDescriptor.ofVoid(layouts);

    if(returns == null) {
        return descriptor;
    }
    else {
        return descriptor.changeReturnLayout(returns.layout());
    }
}
```

At this point support can be added for some of the simpler reference types required by JOVE, firstly Java strings:

```java
public class StringTransformer implements AddressTransformer<String, MemorySegment> {
    public Object marshal(String str, SegmentAllocator allocator) {
        return allocator.allocateFrom(str);
    }

    public String unmarshal(MemorySegment address) {
        return address.reinterpret(Integer.MAX_VALUE).getString(0L);
    }
}
```

A JOVE `Handle` is an immutable pointer:

```java
public final class Handle {
    private final MemorySegment address;

    public Handle(MemorySegment address) {
        this.address = address.asReadOnly();
    }

    public MemorySegment address() {
        return MemorySegment.ofAddress(address.address());
    }
}
```

With a companion transformer:

```java
public static class HandleTransformer implements AddressTransformer<Handle, MemorySegment> {
    public Object marshal(Handle arg, SegmentAllocator _) {
        return arg.address;
    }

    public Handle unmarshal(MemorySegment address) {
        return new Handle(address);
    }
}
```

And similarly for JOVE domain types:

```java
public interface NativeObject {
    Handle handle();

    public static class NativeObjectTransformer implements AddressTransformer<NativeObject, MemorySegment> {
        public Object marshal(NativeObject arg, SegmentAllocator _) {
            return arg.handle().address();
        }

        public NativeObject unmarshal(MemorySegment _) {
            throw new UnsupportedOperationException();
        }
    }
}
```

### Native Reference

The final framework component required for the basic GLFW prototype is support for the commonly used by-reference types (integer and pointer), which we continue to handle as a special case.

The following new type encapsulates the off-heap pointer and returned value:

```java
public class NativeReference<T> {
    private final VarHandle handle;
    private MemorySegment pointer;
    private T value;

    public T get() {
        ...
    }
}
```

The accessor _unmarshals_ the off-heap value from the pointer:

```java
public T get() {
    if(Objects.nonNull(pointer)) {
        value = unmarshal(handle.get(pointer, 0L));
    }

    return value;
}
```

Where `unmarshal` is a simple pass-through for the case of primitive references:

```java
protected T unmarshal(Object arg) {
    return (T) arg;
}
```

The companion transformer plugs the new type into the framework:

```java
public static final class NativeReferenceTransformer implements Transformer {
    public MemoryLayout layout() {
        return ADDRESS;
    }

    public MemorySegment marshal(NativeReference<?> ref, SegmentAllocator allocator) {
        return ref.allocate(allocator);
    }
}
```

Where the pointer is allocated on-demand:

```java
private MemorySegment allocate(SegmentAllocator allocator) {
    if(pointer == null) {
        pointer = allocator.allocate(ValueLayout.ADDRESS);
    }

    return pointer;
}
```

Finally the existing factory class is updated to create commonly used by-reference variables:

```
public static class Factory {
    public NativeReference<Integer> integer() {
        var ref = new NativeReference<Integer>(JAVA_INT);
        ref.set(0);
        return ref;
    }
}
```

A by-reference handle has some additional logic to check for an empty result:

```java
public NativeReference<Handle> pointer() {
    return new NativeReference<>(ADDRESS) {
        public Handle unmarshal(Object arg) {
            MemorySegment address = (MemorySegment) arg;
            if(MemorySegment.NULL.equals(address)) {
                return null;
            }
            else {
                return new Handle(address);
            }
        }
    };
}
```

This implementation is basically a replacement for the equivalent JNA classes and nicely requires minimal refactoring of the existing JOVE code.

> It is not yet clear how much overlap there will be (if any) between these relatively simple by-reference parameters and the more complicated use-cases (structures, arrays, etc).

The GLFW library is now fully supported by the new framework other than the following (deferred until later):

* Callback support for GLFW devices.

* The `glfwGetRequiredInstanceExtensions` edge case.

### Integration

At this point we feel confident that the framework is ready to be integrated into JOVE, with a few required workarounds:

* The code-generator requires significant work to remove JNA and generate the FFM memory layouts of the structures, which we prefer to defer until the bulk of the refactoring has been completed.  For the moment some text editor magic is used to remove the JNA dependencies and the layouts will be hand-crafted on demand.

* Similarly GLFW device polling is also temporarily hacked out.

* The new framework will attempt to build the _whole_ API at instantiation-time.  Therefore both the GLFW and Vulkan APIs are cut back to the bare minimum and the methods will be gradually reintroduced as we progress.

Remarkably this refactoring work went much more smoothly than anticipated, with only a handful of compilation problems.  Apparently the existing JOVE code did a decent job of abstracting over JNA.

Refactoring of the `Desktop` service resulted in the following:

```java
public static Desktop create() {
    // Init supported types
    var registry = Registry.create();
    registry.add(int.class, new IdentityTransformer(ValueLayout.JAVA_INT));
    registry.add(boolean.class, new IdentityTransformer(ValueLayout.JAVA_BOOLEAN));
    registry.add(String.class, new StringTransformer());
    registry.add(Handle.class, new HandleTransformer());
    registry.add(NativeObject.class, new NativeObjectTransformer());
    registry.add(NativeReference.class, new NativeAddressTransformer());

    // Load native library
    var factory = new NativeLibraryBuilder("glfw3", registry);
    DesktopLibrary lib = factory.build(DesktopLibrary.class);

    // Init GLFW
    int result = lib.glfwInit();
    if(result != 1) throw new RuntimeException(...);

    // Create desktop service
    return new Desktop(lib);
}
```

And the prototype was similarly refactored and extended to the point where the application window is created:

```java
void main() throws Exception {
    var desktop = Desktop.create();
    System.out.println("version=" + desktop.version());
    System.out.println("Vulkan=" + desktop.isVulkanSupported());

    Window window = new Window.Builder()
        .title("Prototype")
        .size(new Dimensions(1024, 768))
        .hint(Hint.CLIENT_API, 0)
        .hint(Hint.VISIBLE, 0)
        .build(desktop);
        
    desktop.terminate();
}
```

The approach from this point is to add Vulkan to the prototype and tackle each JOVE component in order starting with the `Instance` domain object.  This requires implementing support for the native structures, the subject of the next section.

---

## Structures

### Memory Layout

The first step is the introduction of the following replacement for the definition of a native structure with an FFM memory layout:

```java
public interface NativeStructure {
    /**
     * @return Memory layout of this structure
     */
    StructLayout layout();
}
```

With a partially implemented companion transformer:

```java
class StructureTransformer implements AddressTransformer<NativeStructure, MemorySegment> {
    private StructLayout layout;
    
    public MemoryLayout layout() {
        return layout;
    }

    public MemorySegment marshal(NativeStructure structure, SegmentAllocator allocator) {
        MemorySegment address = allocator.allocate(layout);
        ...
        return address;
    }
}
```

Structure transformers are generated via a builder:

```java
public static class Builder {
    private final Registry registry;
    
    public StructureTransformer build(Class<? extends NativeStructure> type) {
        var structure = type.getConstructor().newInstance();
        var layout = structure.layout();
        return new StructureTransformer(layout);
    }
}
```

Notes:

* This code mandates that _all_ structures have a default constructor, a common constraint for a marshalling framework.

* Exception handling has been omitted and helper methods silently inlined for brevity.

It is worth noting that the `marshal` method allocates and populates the off-heap memory on _every_ invocation, with the memory being discarded afterwards.  A future enhancement _could_ cache the memory address in the structure itself and implement some sort of synchronisation mechanism (which is the JNA approach) but this would be complex and is certainly not required at the moment.  In reality native structures are really only intended as _data carriers_ with scope during marshalling, therefore the simpler approach is much more palatable.

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

* These layouts are copied from the code generated by the `jextract` tool earlier.

* The JOVE code generator will need to track the byte alignment of each structure and inject padding as required.

* The `layout` is temporarily made a `default` method (throwing an exception) to allow the remaining structures to compile without errors.

### Integration

This section covers various modifications to the framework to support the Vulkan instance.

#### Vulkan Service

As a quick diversion, the following new type is introduced to compose the various top-level Vulkan components that were previously separate fields of the instance:

```java
public class Vulkan {
    private final VulkanLibrary lib;
    private final Registry registry;
    private final NativeReference.Factory factory;
}
```

With a `create` method to instantiate the library:

```java
public static Vulkan create() {
    var registry = new Registry();
    ...
    
    var factory = new NativeLibraryBuilder(registry);
    var lib = factory.build("vulkan-1", VulkanLibrary.class);
    return new Vulkan(lib, registry, new NativeReference.Factory());
}
```

This is equivalent to the `Desktop` class for the GLFW implementation and replaces the existing method in the `VulkanLibrary` interface (which had become a bit of dumping ground for helper methods and constants).

#### Empty Values

The framework now supports reference types that can be `null` for an unused or empty value, which FFM represents with the `MemorySegment.NULL` constant.  An `empty` value is added to the definition of an address transformer:

```java
interface AddressTransformer {
    default Object empty() {
        return MemorySegment.NULL;
    }
}
```

Since it is anticipated that marshalling will be invoked from other parts of the framework a helper is introduced that handles empty values in one location:

```java
class TransformerHelper {
    public static Object marshal(Object arg, Transformer transformer, SegmentAllocator allocator) {
        return switch(transformer) {
            case IdentityTransformer _ -> arg;
            
            case AddressTransformer ref -> {
                if(arg == null) {
                    yield ref.empty();
                }
                else {
                    yield ref.marshal(arg, allocator);
                }
            }
        };
    }
}
```

#### Enumerations

Both of the structures used to create the Vulkan instance are dependant on integer enumerations which requires another companion transformer:

```java
class IntEnumTransformer implements AddressTransformer<IntEnum, Integer> {
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

    public IntEnum unmarshal(Integer value) {
        if(value == 0) {
            return mapping.defaultValue();
        }
        else {
            return mapping.map(value);
        }
    }
}
```

Note that the return type and `empty` values are a native integer in this case.

Finally the registry is modified to factor out the logic that generates new transformers on demand:

```java
private Transformer create(Class<?> type) {
    if(IntEnum.class.isAssignableFrom(type)) {
        return new IntEnumTransformer((Class<? extends IntEnum>) type);
    }
    else
    if(NativeStructure.class.isAssignableFrom(type)) {
        var builder = new StructureTransformer.Builder(this);
        return builder.build((Class<? extends NativeStructure>) type);
    }
    else {
        Transformer supertype = find(type);
        add(type, supertype);
        return supertype;
    }
}
```

This tightly couples the registry to the transformer implementations and requires nasty casting, it works for the moment but may warrant some sort of factory approach later.

#### Vulkan Instance

The `Instance` domain class can now be refactored with the following temporary fiddles:

* The `VulkanLibrary` is cut down to just the instance methods

* The diagnostics handler is temporarily removed.

* Similarly the `function` method that retrieves the function pointers for handlers.

* Extensions and validation layers are uninitialised pending array support and a temporary `String[]` transformer is registered.

The instance API looks like this after some modifications detailed below:

```java
interface Library {
    int     vkCreateInstance(VkInstanceCreateInfo pCreateInfo, Handle pAllocator, NativeReference<Handle> pInstance);
    void    vkDestroyInstance(Instance instance, Handle pAllocator);
    int     vkEnumerateInstanceExtensionProperties(String pLayerName, NativeReference<Integer> pPropertyCount, @Returned VkExtensionProperties[] pProperties);
    int     vkEnumerateInstanceLayerProperties(IntegerReference pPropertyCount, @Returned VkLayerProperties[] pProperties);
    Handle  vkGetInstanceProcAddr(Instance instance, String pName);
}
```

Notes:

* Array parameters are now explicitly specified as Java arrays, previously these were implemented as the _first_ element due to the way that JNA handled arrays.

* The `@Returned` annotation denotes parameters that are returned by-reference (handled later).

The prototype can now be extended to instantiate Vulkan and configure the instance:

```java
Vulkan vulkan = Vulkan.create();

Instance instance = new Instance.Builder()
    .name("Prototype")
    .extension("VK_EXT_debug_utils")
    .layer(ValidationLayer.STANDARD_VALIDATION)
    .build(vulkan);
```

This code compiles and runs but does nothing since we are not actually populating the structures.

### Field Mappings

To populate the off-heap memory a _field mapping_ associates a structure field with its native representation:

```java
private record FieldMapping(VarHandle field, VarHandle foreign, Transformer transformer) {
}
```

Where `field` and `foreign` are handles to the structure field and the off-heap field respectively.

The builder for the structure transformer first creates a helper that builds each mapping:

```java
public StructureTransformer build(Class<? extends NativeStructure> type) {
    ...

    var instance = new Object() {
        private FieldMapping mapping(MemoryLayout member) {
            ...
        }
    }
}
```

And then enumerates the members of the structure layout and delegates to the helper:

```java
List<FieldMapping> mappings = layout
    .memberLayouts()
    .stream()
    .filter(member -> member instanceof ValueLayout))
    .map(instance::mapping)
    .toList();

return new StructureTransformer(layout, mappings);
```

Notes:

* For the moment only 'atomic' fields (primitives and reference types) are supported, i.e. members that are implemented as a `ValueLayout`.

* This includes pointers to other structures, e.g. the `pApplicationInfo` field of the instance descriptor.

* The structure layout explicitly specifies the order of the fields since it is derived from the Vulkan header, unlike reflection where the field order is undefined (hence the `@FieldOrder` annotation required for JNA structures).

The `mapping` method in the helper reflects the structure field by name:

```java
private FieldMapping mapping(MemoryLayout member) {
    String name = member.name().orElseThrow(...);
    Field field = type.getField(name);
    ...
}
```

And creates a handle to the field:

```java
VarHandle local = lookup.unreflectVarHandle(field);
```

Where `lookup` is a new property of the builder:

```java
public static class Builder {
    private final Lookup lookup = MethodHandles.lookup();
    ...
}
```

Next the handle for the off-heap field is extracted from the structure layout:

```java
PathElement path = PathElement.groupElement(field.getName());
VarHandle foreign = layout.varHandle(path);
```

And finally the appropriate transformer is looked up and the mapping is wrapped as a new instance:

```java
Transformer transformer = registry.get(field.getType());
return new FieldMapping(local, foreign, transformer);
```

The process of marshalling a structure field is:

1. Retrieve the value of the field from the structure.

2. Apply the transformer.

3. Set the transformed field in the off-heap memory.

This is implemented by the following method on the new mapping class:

```java
private record FieldMapping(...) {
    public void marshal(Object structure, MemorySegment address, SegmentAllocator allocator) {
        Object value = field.get(structure);

        Object transformed = switch(transformer) {
            case IdentityTransformer -> value;
            case ReferenceTransformr ref -> ref.marshal(value, allocator);
        }

        foreign.set(address, 0L, transformed);
    }
}
```

The last piece of the puzzle is the equivalent method in the structure transformer that allocates the off-heap memory and iterates over the mappings:

```java
public MemorySegment marshal(Object structure, SegmentAllocator allocator) {
    MemorySegment address = allocator.allocate(layout);
    for(var field : fields) {
        field.marshal(structure, address, allocator);
    }
    return address;
}
```

In the prototype the Vulkan instance should now be successfully instantiated and configured using the new framework.  A lot of work to get to the stage JOVE was at several years previous!

### Unmarshalling

To reduce the chance of breaking something the diagnostics handler will be reintroduced before we progress any further, this will require _unmarshalling_ of the diagnostic report which neatly completes the framework for structures.

#### Diagnostic Handler

Refactoring the diagnostic handler presents a few new challenges:

* The native methods to create and destroy a handler must be created programatically, since the address is a _function pointer_ instead of a symbol looked up from the native library.

* In particular the method signatures are not defined in the Vulkan API but must be derived from the documentation.

* Ideally these methods will reuse the existing framework, in particular the support for structures when creating the handler and unmarshalling diagnostic reports.

* A handler requires an FFM callback _stub_ to allow Vulkan to report diagnostic messages to the application.

The majority of the existing handler code remains as-is except for:

* Invocation of the create and destroy function pointers.

* Creation of the message callback.

* Unmarshalling of the diagnostic report structure to construct the `Message` record.

First the `function` method of the `Instance` is reintroduced, which turns out to be trivial:

```java
public Handle function(String name) {
    Handle handle = vulkan.library().vkGetInstanceProcAddr(this, name);
    if(handle == null) throw new IllegalArgumentException(...);
    return handle;
}
```

This is used to retrieve the function pointer that creates the handler:

```java
private Handle create(VkDebugUtilsMessengerCreateInfoEXT info) {
    Handle function = instance.function("vkCreateDebugUtilsMessengerEXT");
    ...
}
```

A native method is constructed for the create method:

```java
Class<?>[] signature = {
    Instance.class,
    VkDebugUtilsMessengerCreateInfoEXT.class,
    Handle.class,
    PointerReference.class
};

NativeMethod create = new NativeMethod.Builder(registry)
    .address(function.address())
    .returns(int.class)
    .parameters(signature)
    .build();
```

Which is invoked with the relevant arguments to create the handler:

```java
NativeReference<Handle> ref = instance.vulkan().factory().pointer();
Object[] args = {instance, info, null, ref};
Vulkan.check((int) create.invoke(args));
return ref;
```

The handler is destroyed in a similar fashion:

```java
protected void release() {
    // Lookup the function pointer
    Handle function = instance.function("vkDestroyDebugUtilsMessengerEXT");

    // Build the native method
    NativeTransformerRegistry registry = instance.vulkan().registry();
    Class<?>[] signature = {Instance.class, DiagnosticHandler.class, Handle.class};
    NativeMethod destroy = new NativeMethod.Builder(registry)
        .address(function.address())
        .parameters(signature)
        .build();

    // Destroy handler
    Object[] args = {instance, this, null};
    destroy.invoke(args);
}
```

This approach nicely reuses the existing framework rather than having to implement more FFM code from the ground up.

#### Callback

To build the diagnostic callback a new method is added to the existing class that provides the memory _address_ of the callback method itself:

```java
private record MessageCallback(Consumer<Message> consumer, StructureTransformer transformer) {
    MemorySegment address() {
        var signature = MethodType.methodType(
            boolean.class,
            int.class,
            int.class,
            MemorySegment.class,
            MemorySegment.class
        );
        MethodHandle handle = MethodHandles.lookup().findVirtual(MessageCallback.class, "message", signature);
        return link(handle.bindTo(this));
    }
}
```

Notes:

* Again the signature of the callback must be hard-coded based on the documentation.

* The resultant method handle is bound to the callback _instance_ in order to delegate diagnostic reports to the specific handler instance.

This handle is then linked to an up-call stub with a method signature matching the `message` method:

```java
private static MemorySegment link(MethodHandle handle) {
    var descriptor = FunctionDescriptor.of(
        JAVA_BOOLEAN,
        JAVA_INT,
        JAVA_INT,
        ADDRESS,
        ADDRESS
    );
    return Linker.nativeLinker().upcallStub(handle, descriptor, Arena.ofAuto());
}
```

Which has the following signature:

```java
public boolean message(int severity, int typeMask, MemorySegment pCallbackData, MemorySegment pUserData) {
```

The `build` method of the handler is modified to lookup the transformer for the message structure:

```java
public DiagnosticHandler build() {
    ...
    var transformer = (StructureTransformer) registry.get(VkDebugUtilsMessengerCallbackData.class);
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

The last required change is unmarshalling the diagnostic report structure received by the `message` callback method itself:

```java
public boolean message(...) {
    long size = transformer.layout().byteSize();
    var segment = pCallbackData.reinterpret(size);
    var data = (VkDebugUtilsMessengerCallbackData) transformer.unmarshal(segment);
    ...
}
```

Note that in this case the off-heap memory needs to be explicitly resized to match the expected structure (also note that this will cause a JVM crash if the new size out-of-bounds).

The `unmarshal` method is added to the structure transformer:

```java
class StructureTransformer implements AddressTransformer<NativeStructure, MemorySegment> {
    public NativeStructure unmarshal(MemorySegment address) {
        NativeStructure structure = constructor.get();
        for(var field : fields) {
            field.unmarshal(address, structure);
        }
        return structure;
    }
}
```

Where `constructor` is a factory that wraps the reflected default constructor to generate new instances on demand.

In turn each mapping populates the structure field from the off-heap memory:

```java
private record FieldMapping(...) {
    public void unmarshal(MemorySegment address, NativeStructure structure) {
        Object value = foreign.get(address, 0L);
        Object transformed = TransformerHelper.unmarshal(value, transformer);
        field.set(structure, transformed);
    }
}
```

Which delegates to a new method on the transformer helper:

```java
public static Object unmarshal(Object arg, Transformer transformer) {
    return switch(transformer) {
        case IdentityTransformer _ -> arg;

        case AddressTransformer def -> {
            if(MemorySegment.NULL.equals(arg)) {
                yield null;
            }
            else {
                yield def.unmarshal(arg);
            }
        }
    };
}
```

### Arrays

The diagnostics handler requires the debug extension and layer to be configured when the instance is created, which means implementing support for arrays (specifically an array of strings in this case).

A new transformer implementation is added for arrays:

```java
public final class ArrayTransformer implements Transformer {
    private final Transformer transformer;

    public MemoryLayout layout() {
        return ValueLayout.ADDRESS;
    }
}
```

Where the `transformer` marshals the _component type_ of the array.

First the off-heap memory is allocated for the array:

```java
public MemorySegment marshal(Object array, SegmentAllocator allocator) {
    MemoryLayout layout = transformer.layout();
    int length = Array.getLength(array);
    MemorySegment address = allocator.allocate(layout, length);

    ...
    
    return address;
}
```

And each element is marshalled accordingly:

```java
for(int n = 0; n < length; ++n) {
    Object value = Array.get(arg, n);
    
    if(value == null) {
        continue;
    }

    MemorySegment element = switch(transformer) {
        case IdentityTransformer _ -> value;
        case AddressTransformer ref -> ref.marshal(value);
        ...
    }
    
    address.setAtIndex(ValueLayout.ADDRESS, n, element);
}
```

Notes:

* This implementation only supports reference types for the moment (the extensions and layers are string arrays).

* The use of the `Array` helper to extract the array length and the elements is ugly but prevents having to cast the method argument, something that may be addressed later.

Finally the registry creates new array transformers as required:

```java
private Transformer create(Class<?> type) {
    if(type.isArray()) {
        Class<?> component = type.getComponentType();
        return new ArrayTransformer(get(component));
    }
    ...
}
```

Diagnostic handlers can now be attached to the instance as before, which is very useful given the invasive changes being made to JOVE.  A good test of the refactor is to temporarily remove the code that releases attached handlers when the instance is destroyed, which should result in Vulkan complaining about orphaned objects.

---

## Returned Parameters

### Returned Annotation

The next Vulkan component after the instance is the physical device which uses native methods that are highly dependant on by-reference parameters.  These pose a problem because by-reference and 'normal' parameters cannot be distinguished solely from the method signature.

JNA used the rather cumbersome `ByReference` interface which generally required _every_ structure to have _two_ implementations.  Additionally _every_ array is essentially a by-reference parameter as far as JNA was concerned, whereas we would prefer to separate the normal and by-reference cases.

A generic wrapper type could be introduced that would allow the framework to switch behaviour accordingly.  However the actual type of the reference would then only be known at runtime, meaning the framework would be unable to build and validate a transformer when the library is created.  The most obvious solution is to implement a custom annotation to explicitly declare by-reference parameters and still have access to the actual reference type.

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
    signature.add(new Parameter(transformer, returned));
    return this;
}
```

Where `Parameter` is an internal record that composes a transformer and the flag:

The provided arguments are _updated_ after invocation of the native method:

```java
public Object invoke(Object[] args) {
    Object[] foreign = marshal(args, Arena.ofAuto());
    Object result = handle.invokeWithArguments(foreign);
    update(foreign, args);
    return returns.apply(result);
}
```

Where by-reference arguments are _overwritten_ with the unmarshalled data:

```java
private void update(Object[] foreign, Object[] args) {
    if(args == null) {
        return;
    }

    for(int n = 0; n < args.length; ++n) {
        if(signature[n].returned && Objects.nonNull(args[n])) {
            switch(signature[n].transformer) {
                case StructureTransformer struct -> struct.unmarshal(foreign[n], args[n]);
                case ArrayTransformer array -> array.update(foreign[n], args[n]);
                ...
            }
        }
    }
}
```

Note that the framework assumes that the argument or array has already been instantiated prior to the method call, i.e. `null` parameters or array elements are ignored.

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

There is a subtle difference between a 'normal' structure array and one being returned by-reference.  Both cases require the off-heap array to be allocated as a contiguous memory block, however the memory _layout_ of a by-reference parameter __must__ be a _pointer_ (which has an `ADDRESS` layout).

Therefore the following helper determines the actual layout used for a given parameter:

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

The `minImageTransferGranularity` field is a _nested_ structure which can be seen from the memory layout:

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

In the native structure the `VkExtent3D` fields are essentially _embedded_ into the overall structure, whereas in the structure class this field is a normal pointer (there is no analog of embedding in Java).  The structure transformer will need to support both _atomic_ fields (i.e. primitives and pointer types) and nested structures.

The code that walks through the layout is first modified to include all members except for alignment padding:

```java
return layout
    .memberLayouts()
    .stream()
    .filter(Predicate.not(e -> e instanceof PaddingLayout))
    .map(instance::mapping)
    .toList();
```

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
}
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

---

## Final Touches

This section covers the implementation of the various edge-cases or features that were deferred.

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

* An unsupported component type will only be detected at runtime in the `get` method, but this seems an acceptable compromise.

* In fact the `extensions` method is the only such instance in the GLFW or Vulkan APIs.

* An invalid array length cannot be bounds checked since the off-heap memory must be resized using the unsafe `reinterpret` method, which can result in a JVM crash.

### Default Registry

TODO
- default transformer?
- factory approach?

### Desktop Callbacks

TODO

### Code Generation

TODO
