                ---
title: Panama Integration
---

---

# Contents

- [Overview](#overview)
- [Design](#design)
- [Implementation](#implementation)

---

# Overview

It had always been the intention at some point to re-implement the Vulkan native layer using the FFM (Foreign Function and Memory) library developed as part of project [Panama](https://openjdk.java.net/projects/panama/).  The reasons for delaying this rework are discussed in [code generation](/jove-blog/blog/part-1-intro/code-generation#alternatives) at the very beginning of this blog.

With the LTS release of JDK 21 the FFM library was elevated to a preview feature (rather than requiring an incubator build) and much more information and tutorials started to appear online, so it was time to replace JNA with the more future proofed (and safer) FFM solution.

A fundamental decision is whether to use the `jextract` tool to generate the Java bindings for the native libraries or to implement some custom solution.  To resolve this question we use the tool to generate the Vulkan API and then compare and contrast with a small, hard-crafted application built using FFM from first principles.

---

# Design

## Exploring Panama

The first step in gaining some familiarity with the FFM library is to build a throwaway application that uses a native library, and again GLFW is chosen as the guinea pig.  This API is relatively simple in comparison to Vulkan which is front-loaded with complexities such as marshalling of structures, by-reference types, callbacks for the diagnostic handler, etc.

The new demo application first loads the native library:

```java
public class DesktopForeignDemo {
    private final Linker linker = Linker.nativeLinker();
    private final SymbolLookup lookup;

    public static void main(String[] args) throws Throwable {
        try(Arena arena = Arena.ofConfined()) {
            // Init lookup service
            SymbolLookup lookup = SymbolLookup.libraryLookup("glfw3", arena);
            var demo = new DesktopForeignDemo(lookup);
            
            // Init GLFW
            ...
        }
    }
}
```

The process of invoking a native method using FFM is:

1. Lookup the address of the the method via the `SymbolLookup` service.
2. Build a `FunctionDescriptor` specifying the signature of the method.
3. Combine both of these to link a `MethodHandle` to the method symbol.
4. Invoke the method with an array of the appropriate arguments.

This is wrapped up into a quick-and-dirty helper:

```java
private Object invoke(String name, Object[] args, MemoryLayout returnType, MemoryLayout... signature) throws Throwable {
    MemorySegment symbol = lookup.find(name).orElseThrow();
    var descriptor = returnType == null ? FunctionDescriptor.ofVoid(signature) : FunctionDescriptor.of(returnType, signature);
    MethodHandle handle = linker.downcallHandle(symbol, descriptor);
    return handle.invokeWithArguments(args);
}
```

The GLFW library can now be initialised, which in this case returns an integer success code:

```java
System.out.println("glfwInit=" + demo.invoke("glfwInit", null, JAVA_INT));
```

Vulkan support can be queried:

```java
System.out.println("glfwVulkanSupported=" + demo.invoke("glfwVulkanSupported", null, ValueLayout.JAVA_BOOLEAN));
```

The GLFW version string can also be retrieved:

```java
MemorySegment version = (MemorySegment) demo.invoke("glfwGetVersionString", null, ADDRESS);
System.out.println("glfwGetVersionString=" + version.reinterpret(Integer.MAX_VALUE).getString(0L));
```

Note that the returned address has to be _reinterpreted_ to expand the bounds of the off-heap memory, in this case with an undefined length for a pointer to a null-terminated character array.

Retrieving the supporting Vulkan extensions is also slightly complex since the API method returns a pointer to a string-array and uses a _by reference_ integer to return the size of the array as a side-effect.  The array length parameter has a `ValueLayout.JAVA_INT` memory layout:

```java
MemorySegment count = arena.allocate(JAVA_INT);
```

Invoking the extensions method returns the array pointer and the length of the array can then be extracted from the reference:

```java
MemorySegment extensions = (MemorySegment) demo.invoke("glfwGetRequiredInstanceExtensions", new Object[]{count}, ADDRESS, ADDRESS);
int length = count.get(JAVA_INT, 0L);
System.out.println("glfwGetRequiredInstanceExtensions=" + length);
```

The returned pointer is reinterpreted as an array and each element is extracted and transformed to a Java string:

```java
MemorySegment array = extensions.reinterpret(length * ADDRESS.byteSize());
for(int n = 0; n < length; ++n) {
    MemorySegment e = array.getAtIndex(ADDRESS, n);
    System.out.println("  " + e.reinterpret(Integer.MAX_VALUE).getString(0L));
}
```

Finally GLFW can be closed:

```java
demo.invoke("glfwTerminate", null, null);
```

Note that the `0L` argument in the various getter methods is an offset into the off-heap memory.

## Analysis

Generating the FFM bindings using `jextract` is relatively trivial and takes a couple of seconds:

`jextract --source \VulkanSDK\1.2.154.1\Include\vulkan\vulkan.h`

With some experience of using FFM and the generated Vulkan API, we can attempt to extrapolate how JOVE could be refactored and make some observations.

The advantages of `jextract` are obvious:

* Proven and standard technology.

* Automated (in particular the FFM memory layouts for native structures).

However there are disadvantages:

* The generated API is polluted by ugly internals such as field handles, structure sizes, helper methods, etc.

* API methods and structures can only be defined in terms of FFM types or primitives (resulting in poor type safety and loss of meaning).

* In particular enumerations are implemented as static integer accessors (!) with all the concerns that led us to abandon LWJGL in the first place.

* Implies throwing away the existing code generated enumerations and structures.

Alternatively an abstraction layer based on the existing hand-crafted API and code-generated classes would have the following advantages:

* The native API and structures could be expressed in domain terms.

* Proper enumerations.

* Retains the existing API and code-generated enumerations and structures.

On the other hand:

* Substantial development effect to implement a custom framework that abstracts over the underlying FFM bindings.

* Proprietary solution with the potential for unknown problems further down the line.

* An FFM memory layout needs to be generated for _all_ structures.

Despite the advantages of `jextract` the hybrid solution is preferred:

* Retains the large number of existing code generated enumerations and structures.

* Enforced type-safety and self-documentating.

* The application and/or JOVE would _still_ need to transform domain types to/from the FFM equivalents even if `jextract` was used to generate the API.

The code generated by `jextract` will be retained as it will be a useful resource, particularly for the memory layout of structures.

## Solution

With this decision made (for better or worse) the requirements for the new framework can be analysed.

The following table summarises the types that will need to be supported:

| type                | layout        | memory layout       | return      | by reference    |
| ----                | ------        | -----------         | ------      | ------------    |
| primitive           | primitive     | n/a                 | yes         | no              |
| String              | address       | string              | yes         | no              |
| IntEnum             | int           | n/a                 | yes         | no              |
| BitMask             | int           | n/a                 | yes         | no              |
| integer reference   | address       | int                 | no          | special         |
| pointer reference   | address       | address             | no          | special         |
| Handle              | address       | n/a                 | yes         | no              |
| NativeObject        | address       | n/a                 | no          | no              |
| NativeStructure     | address       | structure           | yes         | yes             |
| arrays              | address       | component           | special     | yes             |

Where:

* _type_ is a Java or JOVE type.

* _layout_ is the corresponding FFM memory layout.

* _memory layout_ specifies the structure of the off-heap memory pointed to by the address (if any).

* _return_ indicates whether the native type can logically be returned from a native method.

* and _by reference_ indicates method parameters that can be returned by the native layer as a by-reference side effect (see below).

From the above the following observations can be made:

1. The supported types are a mixture of primitives, built-in Java types (strings, arrays) and JOVE types.  Converting between these types and the corresponding FFM equivalents will be the responsibility of a _native transformer_ implemented as a _companion_ class for each supported type.

2. Integer and pointer by-reference types will continue to be treated as special cases since these are already integral to JOVE (based on the existing JNA equivalents) and in particular there is no Java analog for an integer-by-reference.

3. Supported reference types (indicated by `address` in the above table) require an FFM pointer to off-heap memory, which needs to be allocated at some point.  Ideally this will be handled by the framework with memory being allocated on-demand by the transformer.  This also nicely allows supported types to be instantiated _without_ a dependency on an `Arena` (which would be annoying).

4. Structures will require a `StructLayout` derived from its fields (which will also need to be code-generated).

5. Structures and arrays are compound types that will require 'recursive' transformation to/from FFM.

6. An array can be returned from a native method, e.g. `glfwGetRequiredInstanceExtensions`.  This is problematic since the _length_ of the returned array is unknown and therefore cannot be modelled as a Java array.  Generally the length is returned as a by-reference integer in the same API method.  The new framework will disallow arrays as a return type and provide a custom workaround.  Note that returned arrays are only used by the GLFW library.

7. GLFW makes heavy use of callbacks for device polling which will require substantial factoring to FFM upcall stubs.

8. Finally there are a couple of edge-cases for by-reference parameters:

* A structure parameter can be returned by-reference from a native method, e.g. `vkGetPhysicalDeviceProperties`.  The application is responsible for allocating a pointer to off-heap memory for the structure, which is then _populated_ by the native layer.  After invocation the pointer is dereferenced and a structure instance is constructed from the returned data.

* Similarly for a by-reference array parameter, e.g. `vkEnumeratePhysicalDevices` returns an array of device handle addresses.  The application creates an empty array sized accordingly beforehand (generally via the _two stage invocation_ mechanism), allocates off-heap memory for the array, and unmarshals the elements _after_ invocation.  Note that the memory must be a contiguous block in this case.

The problem here is that one cannot determine whether a parameter is being passed _by value_ or _by reference_ solely from the method signature.  JNA used the rather cumbersome `ByReference` marker interface, which generally required _every_ structure having _two_ implementations to support both cases.  The most obvious solution is to implement a custom annotation to explicitly identify by-reference parameters and switch behaviour accordingly.

Note that the notion of transformers to map domain objects to/from FFM method arguments is already sort of provided by Panama.  Native methods are implemented as method handles which support the  mutation of the arguments and return value.  The original intention was to make use of this functionality, unfortunately it proved very difficult in practice for all but the most trivial cases (most examples do not get any further than simple static adapters probably for this reason).  However this is probably an option to revisit when the refactored code (and our understanding) is more mature.

## Framework

From the above the required components for the new framework are:

* A _native factory_ responsible for creating an implementation of a native API expressed in _domain_ terms.

* A set of _native transformers_ for the identified supported types above which are responsible for 1. allocation of off-heap memory and 2. converting to/from the equivalent FFM representation.

* A _native method_ class that composes an FFM method handle and the transformers for its parameters and optional return type.

* The by-reference parameter annotation and logic to unmarshal parameters after method invocation.

* Handling for arrays of the supported types (including by-reference array parameters).

* Automated generation of structure memory layouts.

---

# Implementation

## Overview

Integration of the new framework into JOVE requires the following changes as a bare minimum:

* Removal of the JNA library.

* Implementation of the new framework components outlined above.

* Refactoring of JOVE types that are explicitly dependant on native pointers (namely the `Handle` class).

* Implementation of transformers for the basic types needed to support the framework: strings, enumerations, integer/pointer by-reference.

* Refactoring of the GLFW and Vulkan APIs in terms of the new framework.

* A temporary hack to remove GLFW device polling.

This 'big bang' approach should (hopefully) get the refactored JOVE code to a point where it compiles but will certainly not run.

The plan is then to iteratively reintroduce each JOVE component to a new demo application and refactor accordingly.  As already noted, Vulkan is very front-loaded with complexity, so additional framework support will be addressed as we progress.

Once we are confident that the bulk of the challenges facing the new framework have been addressed, the temporary application can be discarded and we will continue refactoring with the existing test suite.

## Framework

### Native Factory

The logical starting point is the native factory:

```java
public class NativeFactory {
    public <T> T build(SymbolLookup lookup, Class<T> api) {
        if(!api.isInterface()) throw new IllegalArgumentException(...);
        Instance instance = new Instance(lookup);
        ...
    }
}
```

Where _api_ is the interface defining the native API and _instance_ is a local helper.

The factory enumerates the API methods via reflection and delegates to the helper to construct a native method:

```java
Map<Method, NativeMethod> methods = Arrays
    .stream(api.getMethods())
    .filter(NativeFactory::isNativeMethod)
    .collect(toMap(Function.identity(), instance::build));
```

Where appropriate API methods are identified by the following filter:

```java
private static boolean isNativeMethod(Method method) {
    int modifiers = method.getModifiers();
    return !Modifier.isStatic(modifiers);
}
```

The native method is initially an empty implementation:

```java
public class NativeMethod {
    private MemorySegment address;
    
    public Object invoke(Object[] args) {
        return null;
    }
}
```

The helper looks up the memory address of each native method and wraps it into a new instance:

```java
private class Instance {
    private final SymbolLookup lookup;

    public NativeMethod build(Method method) {
        MemorySegment address = lookup.find(method.getName()).orElseThrow(...);
        return new NativeMethod(address);
    }
};
```

The factory next creates an _invocation handler_ that delegates API calls to the appropriate native method instance:

```java
var handler = new InvocationHandler() {
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        NativeMethod delegate = methods.get(method);
        return delegate.invoke(args);
    }
};
```

And finally a proxy implementation of the API is created from the handler:

```java
ClassLoader loader = this.getClass().getClassLoader();
return (T) Proxy.newProxyInstance(loader, new Class<?>[]{api}, handler);
```

### Native Methods

To fully implement the native method class the builder requires the method signature and return type in order to link the FFM method handle.

Looking ahead we will also need to programatically construct native methods (i.e. not using reflection) for the diagnostics handler (and possibly for other use cases).  Therefore the native method class will be agnostic to the source of its address.  This is a little more work but better separates concerns between the various collaborating classes.

First a builder is added to construct a native method:

```java
public static class Builder {
    private final Linker linker = Linker.nativeLinker();
    private MemorySegment address;
    private final List<Class<?>> signature = new ArrayList<>();
    private Class<?> returnType;
}
```

The memory layout of the method is derived from its signature:

```java
private MemoryLayout[] layout() {
    return signature
        .stream()
        .map(this::layout)
        .toArray(MemoryLayout[]::new);
}
```

Where the temporary `layout` method maps a Java type to its corresponding FFM layout:

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

This in turn is used to derive the function descriptor:

```java
private FunctionDescriptor descriptor(MemoryLayout[] layout) {
    if(returnType == null) {
        return FunctionDescriptor.ofVoid(layout);
    }
    else {
        MemoryLayout m = layout(returnType);
        return FunctionDescriptor.of(m, layout);
    }
}
```

The `build` method links the method handle from the function descriptor and instantiates a new instance:

```java
public NativeMethod build() {
    MemoryLayout[] layout = layout();
    FunctionDescriptor descriptor = descriptor(layout);
    MethodHandle handle = linker.downcallHandle(address, descriptor);
    return new NativeMethod(handle);
}
```

The factory helper uses this builder to construct a native method instance for each API method:

```java
private NativeMethod build(MemorySegment address, Method method) {
    // Init method builder
    var builder = new NativeMethod.Builder()
        .address(address)
        .signature(method.getParameterTypes());

    // Set return type
    Class<?> returnType = method.getReturnType();
    if(returnType != void.class) {
        builder.returns(returnType);
    }

    // Construct native method
    return builder.build();
}
```

Finally the native method is refactored to delegate to the method handle:

```java
public class NativeMethod {
    private MemoryHandle handle;
    
    public Object invoke(Object[] args) {
        return handle.invoke(args);
    }
}
```

To exercise the new framework the following API defines a (very) cut-down GLFW library dependant only on integer primitives:

```java
interface API {
    int         glfwInit();
    void        glfwTerminate();
    int         glfwVulkanSupported();
    //String    glfwGetVersionString();
    //String[]  glfwGetRequiredInstanceExtensions(&int count);
}
```

From which a proxy implementation is generated using the new factory:

```java
try(Arena arena = Arena.ofConfined()) {
    var lookup = SymbolLookup.libraryLookup("glfw3", arena);
    var factory = new NativeFactory();
    API api = factory.build(lookup, API.class);
    ...
}
```

This is then used to exercise the basic GLFW calls:

```java
System.out.println("init="+api.glfwInit());
System.out.println("Vulkan="+api.glfwVulkanSupported());
desktop.glfwTerminate();
```

### Native Transformer

The final piece of the framework is the _native transformer_ which defines a Java or JOVE type that can be converted to/from an FFM type:

```java
public interface NativeTransformer<T, R> {
    /**
     * @return Type
     */
    Class<T> type();

    /**
     * @return Native memory layout
     */
    default MemoryLayout layout() {
        return ValueLayout.ADDRESS;
    }
}
```

Where `T` is a domain type and `R` is the corresponding FFM type.

A native transformer converts an object to its native representation:

```java
/**
 * Transforms the given value to its native representation.
 * @param value         Value to transform
 * @param allocator     Native allocator
 * @return Native value
 */
Object transform(T value, SegmentAllocator allocator);
}
```

And optionally transforms return values:
    
```java
/**
 * Provides a transformer for a return value of this native type.
 * @return Return value transformer
 */
Function<R, T> returns();
```

The framework is made configurable by the introduction of a _registry_ of supported transformers:

```java
public class TranformerRegistry {
    private final Map<Class<?>, NativeTransformer> registry = new HashMap<>();

    public void add(NativeTransformer transformer) {
        Class<?> type = transformer.type();
        if(type == null) throw new IllegalArgumentException(...);
        registry.put(type, transformer);
    }
}
```

The transformer for a given type can be looked up from the registry:

```java
public NativeTransformer transformer(Class<?> type) {
    NativeTransformer transformer = registry.get(type);
    if(transformer == null) {
        NativeTransformer derived = find(type);
        registry.put(type, derived);
        return derived;
    }
    else {
        return transformer;
    }
}
```

Or a transformer can be _derived_ for a subclass of a supported type (the majority of the JOVE domain types):

```java
private NativeTransformer find(Class<?> type) {
    return registry
        .values()
        .stream()
        .filter(e -> e.type().isAssignableFrom(type))
        .findAny()
        .orElseThrow(...);
}
```

Notes:

* A derived transformer is registered as a side-effect.

* Collections are explicitly disallowed (not shown) since the _component type_ of a collection cannot be determined due to type-erasure.

The native method class is modified by replacing the raw class types with transformers:

```java
public class NativeMethod {
    private final MethodHandle handle;
    private final NativeTransformer[] signature;
    private final NativeTransformer returns;
}    
```

And the `invoke` method transforms the arguments before invocation and unmarshals the return value:

```java
public Object invoke(Object[] args, SegmentAllocator allocator) {
    Object[] actual = transform(args, allocator);
    Object result = handle.invoke(actual);
    return unmarshalReturnValue(result);
}
```

Which delegates to the following helper:

```java
private Object[] transform(Object[] args, SegmentAllocator allocator) {
    if(args == null) {
        return null;
    }

    Object[] mapped = new Object[args.length];
    for(int n = 0; n < mapped.length; ++n) {
        mapped[n] = signature[n].transform(args[n], allocator);
    }

    return mapped;
}
```

And similarly for the return value:

```java
private Object unmarshalReturnValue(Object value) {
    if(returns == null) {
        return null;
    }
    else {
        return returns.returns().apply(value);
    }
}
```

The builder is refactored accordingly to lookup the transformer for each parameter (and the return value):

```java
public Builder parameter(Class<?> type) {
    NativeTransformer transformer = registry.transformer(type).orElseThrow(...);
    signature.add(transformer);
    return this;
}
```

In the builder the memory layout of the method signature is now derived from the transformers:

```java
private MemoryLayout[] layout() {
    return signature
        .stream()
        .map(p -> p.transformer)
        .map(NativeTransformer::layout)
        .toArray(MemoryLayout[]::new);
}
```

And similarly for the function descriptor:

```java
private FunctionDescriptor descriptor(MemoryLayout[] layout) {
    if(returns == null) {
        return FunctionDescriptor.ofVoid(layout);
    }
    else {
        return FunctionDescriptor.of(returns.layout(), layout);
    }
}
```

### Null Arguments

Thus far the framework assumes all values are non-null, which is fine for primitives but needs explicit handling for reference types.

A new method is added to the native transformer to convert an empty (i.e. `null`) domain value:

```java
public interface NativeTransformer {
    default Object empty() {
        return MemorySegment.NULL;
    }
}
```

And the native method switches accordingly when transforming the arguments:

```java
private Object[] transform(Object[] args, SegmentAllocator allocator) {
    ...
    for(int n = 0; n < mapped.length; ++n) {
        if(args[n] == null) {
            mapped[n] = signature[n].empty();
        }
        else {
            mapped[n] = signature[n].transform(args[n], allocator);
        }
    }
    ...
}
```

Finally a `null` return value is also handled:

```java
private Object marshalReturnValue(Object value) {
    if(returns == null) {
        return null;
    }
    else
    if(MemorySegment.NULL.equals(value)) {
        return null;
    }
    else {
        ...
    }
}
```

This should slightly simplify the implementation of the various transformers and hopefully makes the logic more explicit.

### Integration

/////////////////////

TODO....

To support the cut-down GLFW library a default registry factory method is added:

```java
public static NativeTransformerRegistry create() {
    var registry = new NativeTransformerRegistry();

    var primitives = Map.of(
        byte.class,     ValueLayout.JAVA_BYTE,
        char.class,     ValueLayout.JAVA_CHAR,
        boolean.class,  ValueLayout.JAVA_BOOLEAN,
        int.class,      ValueLayout.JAVA_INT,
        short.class,    ValueLayout.JAVA_SHORT,
        long.class,     ValueLayout.JAVA_LONG,
        float.class,    ValueLayout.JAVA_FLOAT,
        double.class,   ValueLayout.JAVA_DOUBLE
    );
    for(Class<?> type : primitives.keySet()) {
        ValueLayout layout = primitives.get(type);
        var transformer = new DefaultNativeTransformer<>(type, layout);
        registry.add(transformer);
    }
    
    return registry;
}
```

Where a `DefaultNativeTransformer` is a simple pass-through implementation:

```java
public class DefaultNativeTransformer<T, R> implements ReturnMapper<T, R> {
    private final Class<T> type;
    private final MemoryLayout layout;

    @Override
    public Object marshal(T value, SegmentAllocator __) {
        return value;
    }

    @Override
    public Object unmarshal(R value, Class<?> type) {
        return value;
    }
}
```

/////////////////////

The existing demo should still run with the minor benefit that the following method can now explicitly return a Java boolean:

```java
interface API {
    boolean glfwVulkanSupported();
}
```

## Framework Support

TODO - primitives!

### Overview

With the basic framework in place, the following sections cover common functionality that can be implemented before actually tackling Vulkan itself:

* The `Handle` wrapper class.

* By-reference types.

* Integer enumerations.

* Strings.

### Handles

A JOVE `Handle` is an immutable, opaque wrapper for an FFM address:

```java
public final class Handle {
    private final MemorySegment address;

    public Handle(MemorySegment address) {
        this.address = address.asReadOnly();
    }

    public Handle(long address) {
        this(MemorySegment.ofAddress(address));
    }

    public MemorySegment address() {
        return MemorySegment.ofAddress(address.address());
    }
}
```

With a companion native transformer:

```java
public static class HandleNativeTransformer implements NativeTransformer<Handle, MemorySegment> {
    public MemorySegment transform(Handle handle, SegmentAllocator allocator) {
        return handle.address;
    }

    public Function<MemorySegment, Handle> returns() {
        return Handle::new;
    }
}
```

### Reference Types

For the commonly used integer and pointer by-reference types the following new class replaces the corresponding JNA helpers:

```java
public abstract class NativeReference<T> {
    private MemorySegment address;

    /**
     * @return Data contained by this reference
     */
    public abstract T get();
}
```

Note that this new type is generic and will be used for both implementations required by Vulkan and GLFW, but allows other data types if needed in the future.

A native reference has two lifecycle methods, the first allocates the off-heap pointer (on demand) when a reference is used as a parameter:

```java
private MemorySegment allocate(SegmentAllocator allocator) {
    if(address == null) {
        address = allocator.allocate(ADDRESS);
    }

    return address;
}
```

And the second performs an implementation-specific update _after_ invocation of the method:

```java
protected abstract void update(MemorySegment address);
```

The following factory (replacing the previous `ReferenceFactory`) provides an integer-by-reference implementation:

```java
public static class Factory {
    public NativeReference<Integer> integer() {
        return new NativeReference<>() {
            private Integer value;
    
            public Integer get() {
                return address.get(ValueLayout.JAVA_INT, 0L);
            }
    
            protected void update(MemorySegment address) {
                value = address.get(ValueLayout.JAVA_INT, 0L);
            }
        };
    }
}
```

And similarly for a pointer-by-reference:

```java
public static NativeReference<Handle> pointer() {
    return new NativeReference<>() {
        private Handle handle;

        public Handle get() {
            return handle;
        }

        protected void update(MemorySegment address) {
            handle = new Handle(address.get(ADDRESS, 0L));
        }
    };
}
```

The companion transformer delegates to the new lifecycle methods:

```java
public static class NativeReferenceTransformer implements NativeTransformer<NativeReference, MemorySegment> {
    public Class type() {
        return NativeReference.class;
    }

    public MemorySegment transform(NativeReference ref, SegmentAllocator allocator) {
        return ref.allocate(allocator);
    }

    public Object empty(Class<? extends NativeReference> target) {
        throw new NullPointerException("A native reference cannot be NULL");
    }

    public Function<MemorySegment, NativeReference> returns(Class<? extends NativeReference> target) {
        throw new UnsupportedOperationException();
    }
}
```

Note that a native reference is enforced to be non-null and cannot be returned from a native method.

For the moment the application will explicitly invoke `update` and then `get` to dereference a value.  This will be handled more simply when we address by-reference structures and arrays later.

### Strings

A Java string is the only non-JOVE type required to support the new framework (other than primitives and arrays), therefore the transformer is a stand-alone class rather than a companion in this case:

```java
public class StringNativeTransformer implements NativeTransformer<String, MemorySegment> {
    public MemorySegment transform(String string, SegmentAllocator allocator) {
        return allocator.allocateFrom(string);
    }

    public Function<MemorySegment, String> returns(Class<? extends String> target) {
        return StringNativeTransformer::unmarshal;
    }

    public static String unmarshal(MemorySegment address) {
        return address.reinterpret(Integer.MAX_VALUE).getString(0L);
    }
}
```

### Enumerations

Integer enumerations are slightly different to the other JOVE types in that the values are reference types but the native representation is an integer:

```java
public class IntEnumNativeTransformer implements NativeTransformer<IntEnum, Integer> {
    public MemoryLayout layout() {
        return ValueLayout.JAVA_INT;
    }

    public Integer transform(IntEnum e, SegmentAllocator allocator) {
        return e.value();
    }
}
```

However the transformer needs to account for default or `null` enumeration constants, implying that it must also be privy to the actual _target_ type of a parameter or return value.  Therefore the existing transformer interface is modified to provide the target type accordingly, for the 'empty' case:

```java
public Integer empty(Class<? extends IntEnum> target) {
    return ReverseMapping
        .get(target)
        .defaultValue()
        .value();
}
```

And the return type:

```java
public Function<Integer, IntEnum> returns(Class<? extends IntEnum> target) {
    return value -> {
        ReverseMapping<?> mapping = ReverseMapping.get(target);
        if(value == 0) {
            return mapping.defaultValue();
        }
        else {
            return mapping.map(value);
        }
    };
}
```

This subclass support will also be used later when marshalling native structures.

The transformer for an enumeration bit mask is relatively trivial since the default value is always zero and the actual target type has no relevance:

```java
public class BitMaskNativeTransformer implements NativeTransformer<BitMask, Integer> {
    public MemoryLayout layout() {
        return ValueLayout.JAVA_INT;
    }

    public Integer transform(BitMask value, SegmentAllocator allocator) {
        return value.bits();
    }

    public Integer empty(Class<? extends BitMask> type) {
        return 0;
    }

    public Function<Integer, BitMask> returns(Class<? extends BitMask> target) {
        return BitMask::new;
    }
}
```

### Arrays

Array parameters are marshalled by a new transformer implementation:

```java
public class ArrayNativeTransformer implements NativeTransformer<Object[], MemorySegment> {
    private final TransformerRegistry registry;

    public MemorySegment transform(Object[] array, SegmentAllocator allocator) {
        ...
        return address;
    }

    public Function<MemorySegment, Object[]> returns(Class<? extends Object[]> target) {
        throw new UnsupportedOperationException();
    }
}
```

Note that returning an array via the transformer is disallowed for the reasons outlined [above](#solution), however a workaround is implemented in the following section.

Transforming an array first requires the transformer for the _component type_ of the array:

```java
Class<?> component = array.getClass().getComponentType();
NativeTransformer transformer = registry.get(component);
MemoryLayout layout = transformer.layout(component);
```

The array elements are then transformed:

```java
var elements = new MemorySegment[array.length];
for(int n = 0; n < array.length; ++n) {
    elements[n] = (MemorySegment) NativeTransformer.transform(array[n], component, transformer, allocator);
}
```

Next off-heap memory is allocated for the array:

```java
MemorySegment address = allocator.allocate(layout, array.length);
```

And finally the `setAtIndex` helper method is used to copy each transformed element to the off-heap memory:

```java
switch(layout) {
    case AddressLayout __ -> {
        for(int n = 0; n < array.length; ++n) {
            address.setAtIndex(ADDRESS, n, elements[n]);
        }
    }

    default -> throw new UnsupportedOperationException();
}
```

Notes:

* For the moment this implementation only handles arrays of reference types (such as a string or `Handle`), i.e. where the component type has an `ADDRESS` layout.  Support will be added later for arrays of primitives and structures.

* The transformation process of an array is essentially recursive.

Although the new transformer is publically available, in general it is more convenient to assume that a supported type can also be marshalled as an array.  Therefore the transformer registry is updated to automatically provide an array transformer for a supported component type:

```java
public class TransformerRegistry {
    private final ArrayNativeTransformer array = new ArrayNativeTransformer(this);

    public NativeTransformer get(Class<?> type) {
        NativeTransformer transformer = registry.get(type);
        if(transformer == null) {
            if(type.isArray()) {
                add(type, array);
                return array;
            }
            ...
        }
        ...
    }
}
```

### Returned Arrays

Although a returned array cannot be modelled as a Java array, the following special case return value wraps the address of the off-heap array:

```java
public class ArrayReturnValue<T> {
    private final MemorySegment address;
    private final TransformerRegistry registry;
}
```

When the length is known the actual array can be extracted from this wrapper, by first allocating a new instance:

```java
public T[] get(int length, IntFunction<T[]> factory) {
    T[] array = factory.apply(length);
    ...
    return array;
}
```

Next the transformer for the array component type is looked up from the registry:

```java
Class type = array.getClass().getComponentType();
NativeTransformer<T, MemorySegment> transformer = registry.get(type);
```

The off-heap memory is resized accordingly:

```java
long size = transformer.layout(type).byteSize();
MemorySegment segment = address.reinterpret(size * length);
```

And finally each element is transformed to the component type via the `returns` function of the transformer:

```java
Function<MemorySegment, T> mapper = transformer.returns(type);
for(int n = 0; n < length; ++n) {
    MemorySegment element = segment.getAtIndex(ValueLayout.ADDRESS, n);
    array[n] = mapper.apply(element);
}
```

The custom return type is integrated into the builder for a native method as a special case:

```java
public Builder returns(Class<?> type) {
    if(ArrayReturnValue.class.isAssignableFrom(type)) {
        Function<MemorySegment, ArrayReturnValue<?>> mapper = address -> new ArrayReturnValue<>(address, registry);
        this.returns = new ReturnType(ADDRESS, mapper);
        return this;
    }
    ...
}
```

In GLFW the method that queries the required Vulkan extensions can now be defined thus:

```java
ArrayReturnValue<String> glfwGetRequiredInstanceExtensions(NativeReference<Integer> count);
```

And the corresponding implementation in the `Desktop` extracts the array sized by the by-reference integer parameter:

```java
public String[] extensions() {
    NativeReference<Integer> length = factory.integer();
    ArrayReturnValue<String> array = lib.glfwGetRequiredInstanceExtensions(length);
    return array.get(length.get(), String[]::new);
}
```

The validation layers and extensions can now be fully implemented when creating the descriptor for the Vulkan instance (which was skipped earlier).

## Instance

### Refactoring

With JNA removed and the basics of the new framework in place, refactoring of the Vulkan API itself can begin which entails the following:

* A global find-and-replace for integer and pointer by-reference values.

* Adding the newly implemented transformers to a default registry.

* Refactoring all the Vulkan API libraries to match the new framework and replace any JNA-specific types.

* Temporarily hacking out the GLFW device polling code.

* Regeneration of the Vulkan structures to remove the JNA artifacts such as imports, the `FieldOrder` annotations, etc.

Remarkably this initial refactoring work went much more smoothly than anticipated, with only a handful of compilation problems that were deferred until later in the process (which are covered below).  Apparently the existing JOVE code did a decent job of abstracting over JNA.

The approach from this point is to instantiate the Vulkan library and tackle each JOVE component in order starting with the `Instance` domain object.

First the refactored instance library now looks like this:

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

* A parameter for an array of structures is now explicitly specified as a Java array, previously these were implemented as the _first_ element due to the way that JNA handled arrays.

* The `@Returned` annotation denotes parameters that are returned by-reference (handled later).

* The memory layout for the required structures will be hand-crafted until we are confident that the requirements for refactoring the code generator are clear.

### Vulkan

As a quick diversion, the following new type is introduced to compose the various top-level components of the Vulkan implementation that were previously separate fields of the instance:

```java
public class Vulkan {
    private final VulkanLibrary lib;
    private final NativeTransformerRegistry registry;
    private final ReferenceFactory factory;
}
```

With a `create` method to instantiate the library using the new framework:

```java
public static Vulkan create() {
    var registry = NativeTransformerRegistry.create();
    var factory = new NativeFactory(registry);
    var lib = factory.build("vulkan-1", VulkanLibrary.class);
    return new Vulkan(lib, registry, new ReferenceFactory());
}
```

This is equivalent to the `Desktop` class for the GLFW implementation and replaces the `create` method on the existing `VulkanLibrary` interface (which had become a bit of dumping ground for helper methods and constants).

The demo can now be extended to instantiate Vulkan and configure the instance:

```java
Vulkan vulkan = Vulkan.create();

Instance instance = new Instance.Builder()
    .name("VulkanTest")
    .extension("VK_EXT_debug_utils")
    .extension(extensions)
    .layer(ValidationLayer.STANDARD_VALIDATION)
    .build(vulkan);
```

However this new code will fail since it requires support for structures, the subject of the next section.

### Structures

A new type is introduced as the definition for all native structures supported by the new framework:

```java
public interface NativeStructure {
    /**
     * @return Memory layout of this structure
     */
    StructLayout layout();
}
```

The memory layout of a native structure could theoretically be derived from the structure class itself (via reflection).  However the fields of a reflected Java class are (irritatingly) unordered (hence the `@FieldOrder` annotation required by JNA structures).  Conveniently a structure `layout` explicitly declares the field order.  Eventually structure layouts will be constructed by the code generator (where the actual field order is available), for the moment the layouts are hand-crafted.

A second internal type transforms a structure _field_ to/from off-heap memory:

```java
interface FieldMapping {
    /**
     * Transforms a structure field to off-heap memory.
     * @param structure     Structure
     * @param address       Off-heap memory
     * @param allocator     Allocator
     */
    void transform(NativeStructure structure, MemorySegment address, SegmentAllocator allocator);

    /**
     * Populates a structure field from off-heap memory.
     * @param address       Off-heap memory
     * @param structure     Structure
     */
    void populate(MemorySegment address, NativeStructure structure);
}
```

An _atomic_ field mapping transforms a primitive or simple reference-type which involves the following steps:
1. Extract the value from the reflected structure field.
2. Apply the native transformer.
3. Use a _var handle_ to set the transformed value to the corresponding off-heap field.

This is implemented in a new implementation for atomic fields:

```java
record AtomicFieldMapping(Field field, NativeTransformer transformer, VarHandle handle) implements FieldMapping {
    public void transform(NativeStructure structure, MemorySegment address, SegmentAllocator allocator) {
        Object value = field.get(structure);
        Object foreign = NativeTransformer.transform(value, field.getType(), transformer, allocator);
        handle.set(address, 0L, foreign);
    }
}
```

Transforming a return value is the inverse process:

```java
public void populate(MemorySegment address, NativeStructure structure) {
    Object foreign = handle.get(address, 0L);
    Object value = transformer.returns(field.getType()).apply(foreign);
    field.set(structure, value);
}
```

Notes:

* The structure field of the domain type is retrieved and updated via reflection.

* A var handle works similarly with the advantage that validation is performed _once_ when it is instantiated (the same as method handles).

* The `populate` method delegates to the `returns` function of the associated transformer to convert _from_ off-heap memory to a primitive or domain type.

* Exception handling has been omitted and helper methods silently inlined for brevity.

A structure can be an object graph (as is the case with the `VkInstanceCreateInfo` descriptor for example), hence its transformation is essentially a recursive process:

```java
record StructureFieldMapping(List<FieldMapping> mappings) implements FieldMapping {
    public void transform(NativeStructure structure, MemorySegment address, SegmentAllocator allocator) {
        for(FieldMapping m : mappings) {
            m.transform(structure, address, allocator);
        }
    }

    public void populate(MemorySegment address, NativeStructure structure) {
        for(FieldMapping m : mappings) {
            m.populate(address, structure);
        }
    }
}
```

The following factory method enumerates the members of a structure and constructs field mappings appropriately:

```java
public static StructureFieldMapping build(Class<?> type, StructLayout layout, TransformerRegistry registry) {
    var builder = new Object() {
        public FieldMapping mapping(MemoryLayout member) {
            ...
        }
    };
    
    return layout
        .memberLayouts()
        .stream()
        .filter(e -> e.name().isPresent())
        .map(builder::mapping)
        .collect(collectingAndThen(toList(), StructureFieldMapping::new));
}
```

The `mapping` method determines the type of the field mapping depending on the layout of the member:

```java
public FieldMapping mapping(MemoryLayout member) {
    String name = member.name().get();
    Field field = type.getDeclaredField(name);

    return switch(member) {
        case ValueLayout __ -> ...
        case StructLayout struct -> ...
        case SequenceLayout seq -> ...
        default -> throw new RuntimeException(...);
    };
}
```

An atomic field is identified by a `ValueLayout` member, which is mapped via a _path_ in order to create the var handle:

```java
var path = PathElement.groupElement(field.getName());
VarHandle handle = layout.varHandle(path);
NativeTransformer transformer = registry.get(field.getType());
yield new AtomicFieldMapping(field, transformer, handle);
```

Further field mapping implementations will be added later to support other types such as arrays, nested structures, primitive arrays, etc.

Finally the companion transformer for native structures is implemented in terms of field mappings:

```java
class StructureNativeTransformer implements NativeTransformer<NativeStructure, MemorySegment> {
    private final TransformerRegistry registry;
    private final Map<Class<?>, Entry> entries = new HashMap<>();

    private record Entry(StructLayout layout, FieldMapping mapping) {
    }
}
```

Where an `Entry` caches the layout and mappings of a given type of structure.

The entry for a given structure can be looked up:

```java
private Entry entry(NativeStructure structure) {
    return entries.computeIfAbsent(structure.getClass(), _ -> register(structure));
}
```

Which retrieves the structure layout and generates the field mappings once on demand:

```java
private Entry register(NativeStructure structure) {
    StructLayout layout = structure.layout();
    FieldMapping mappings = StructureFieldMapping.build(structure.getClass(), layout, registry);
    return new Entry(layout, mappings);
}
```

The `layout` method of the transformer requires an _instance_ of a structure in order to retrieve the FFM memory layout:

```java
public MemoryLayout layout(Class<? extends NativeStructure> type) {
    Entry entry = entries.computeIfAbsent(type, _ -> register(create(type)));
    return entry.layout;
}
```

Where a new instance is created by the following factory:

```java
private static NativeStructure create(Class<?> type) {
    try {
        return (NativeStructure) type.getDeclaredConstructor().newInstance();
    }
    catch(Exception e) {
        throw new RuntimeException(...);
    }
}
```

This mandates that _all_ structures must have a default constructor which is a common constraint for a marshalling framework.

Transforming a structure involves:
1. Retrieve or construct the field mappings.
2. Allocate off-heap for the structure according to its layout.
3. Delegate to the field mappings to transform each field.

This is implemented in the transformer as follows:

```java
public MemorySegment transform(NativeStructure structure, SegmentAllocator allocator) {
    Entry entry = entry(structure);
    MemorySegment address = allocator.allocate(entry.layout);
    entry.mapping.transform(structure, address, allocator);
    return address;
}
```

And similarly for a structure returned from a native method:

```java
public Function<MemorySegment, NativeStructure> returns(Class<? extends NativeStructure> target) {
    return address -> {
        NativeStructure structure = create(target);
        Entry entry = entry(structure);
        entry.mapping.populate(address, structure);
        return structure;
    };
}
```

Notes:

* The memory address of a structure is allocated on _every_ invocation.  A future enhancement _could_ store the address in the structure itself (essentially caching) but this would require tracking modifications (tricky given the fields are public) and would probably also make structure arrays more complex.  In reality native structures are really only intended as _data carriers_ with a lifetime that only applies during marshalling, therefore the simpler approach is much more palatable.

* The same applies to the memory address for a transformed Java string which _could_ also be cached.

The transformer implementation means that _any_ reference to a structure in an API method or another structure results in the field mappings being generated for that class.  Therefore the code generator had to be refactored at this point to rebuild all Vulkan structures involving:

1. Removal of the JNA dependency.

2. Implementation the new `NativeStructure` interface in the code template.

3. Removal of the injection of the JNA `FieldOrder` annotation.

4. The addition of the FFM memory `layout` for each structure (which is temporarily empty as we will continue to hand-craft structure layouts for the moment).

### Instance Creation

The `Instance` domain class can now be refactored with the following temporary fiddles:

* The diagnostics handler is temporarily removed.

* Similarly for the `function` method that retrieves the function pointers for handlers.

* Extensions and validation layers are uninitialised pending array support.

The first step is to refactor the two structures used to configure the instance which requires:

* Refactoring any fields to match the new framework.

* Deriving the FFM structure layout.

* Adding alignment padding as required (see below).

The revised instance descriptor is as follows:

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

And the corresponding FFM memory layout:

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

```JAVA
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

These layouts were essentially copied from the equivalent code generated by the `jextract` tool earlier.  The code generator will need to track the byte alignment of each structure and inject padding as required.

In the demo application the Vulkan instance should now be successfully instantiated and configured using the new framework.

A lot of work to get to the stage JOVE was at several years previous!

### Diagnostic Handler

Refactoring the diagnostic handler presents a few new challenges:

* The native methods to create and destroy a handler must be created programatically, since the address is a _function pointer_ instead of a symbol looked up from the native library.

* Ideally these methods will reuse the existing framework, in particular the support for structures when creating the handler and unmarshalling diagnostic reports.

* A handler requires an FFM callback _stub_ to allow Vulkan to report diagnostic messages to the application.

The majority of the existing handler code remains as-is except for:

* Invocation of the create and destroy function pointers.

* Creation of the message callback.

* Unmarshalling a diagnostic report to a `Message` record.

First the `function` method of the `Instance` is reintroduced, which turns out to be trivial:

```java
public Handle function(String name) {
    Handle handle = vulkan.library().vkGetInstanceProcAddr(this, name);
    if(handle == null) throw new IllegalArgumentException(...);
    return handle;
}
```

This is used to retrieve and invoke the function pointer to create the handler:

```java
private Handle create(VkDebugUtilsMessengerCreateInfoEXT info) {
    Handle create = instance.function("vkCreateDebugUtilsMessengerEXT");
    NativeMethod method = create(create.address());
    PointerReference ref = create(method, info);
    return ref.handle();
}
```

Where the native method is constructed from the function pointer:

```java
private NativeMethod create(MemorySegment address) {
    Class<?>[] signature = {
        Instance.class,
        VkDebugUtilsMessengerCreateInfoEXT.class,
        Handle.class,
        PointerReference.class
    };

    return new NativeMethod.Builder(registry)
        .address(address)
        .returns(int.class)
        .signature(signature)
        .build();
}
```

And invoked with the relevant arguments:

```java
private PointerReference create(NativeMethod method, VkDebugUtilsMessengerCreateInfoEXT info) {
    PointerReference ref = instance.vulkan().factory().pointer();
    Object[] args = {instance, info, null, ref};
    Vulkan.check((int) invoke(method, args, registry));
    return ref;
}
```

Delegating to the following helper:

```java
private static Object invoke(NativeMethod method, Object[] args, NativeTransformerRegistry registry) {
    var allocator = new SegmentAllocator(Arena.ofAuto(), registry);
    return method.invoke(args, allocator);
}
```

The handler is destroyed in a similar fashion:

```java
protected void release() {
    Handle function = instance.function("vkDestroyDebugUtilsMessengerEXT");
    NativeTransformerRegistry registry = instance.vulkan().registry();
    Class<?>[] signature = {Instance.class, Handler.class, Handle.class};
    NativeMethod destroy = new NativeMethod.Builder(registry).address(address).signature(signature).build();
    Object[] args = {instance, this, null};
    invoke(destroy, args, registry);
}
```

This approach nicely reuses the existing framework rather than having to implement more FFM code from the ground up, and was the reason for the native method being intentionally agnostic to the source of the function address earlier.

To build the callback a new method is added to the existing class that provides the memory _address_ of the callback method itself:

```java
private record MessageCallback(Consumer<Message> consumer, StructureNativeTransformer transformer) {
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

Note that the method handle is bound to the callback _instance_ in order to delegate diagnostic reports to the specific handler.

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

Finally the `build` method of the handler is modified to store the callback address in the descriptor:

```java
public Handler build() {
    init();
    var callback = new MessageCallback(...);
    var info = populate(callback.address());
    Handle handle = create(info);
    return new Handler(handle, instance);
}
```

The last required change is unmarshalling the diagnostic report:

```java
public boolean message(int severity, int typeMask, MemorySegment pCallbackData, MemorySegment pUserData) {
    // Transform the message properties
    ...

    // Unmarshal the message structure
    var data = (VkDebugUtilsMessengerCallbackData) transformer.returns(VkDebugUtilsMessengerCallbackData.class).apply(pCallbackData);

    // Handle message
    Message message = new Message(level, types, data);
    consumer.accept(message);

    return false;
}
```

The last required changes is to unmarshal diagnostic reports in the `message` callback method:

```java
var clazz = VkDebugUtilsMessengerCallbackData.class;
var layout = transformer.layout(clazz);
var segment = pCallbackData.reinterpret(layout.byteSize());
var data = (VkDebugUtilsMessengerCallbackData) transformer.returns(clazz).apply(segment);
```

Note that in this case the off-heap memory needs to be explicitly resized to match the expected structure.

Diagnostic handlers can now be attached to the instance as before, which is important given the invasive changes being made to JOVE.  A good test of the refactor is to temporarily remove the code that releases attached handlers when the instance is destroyed, which should result in Vulkan complaining about orphaned objects.

### Method Success Codes

The approach of using a proxy for the native libraries enables the new framework to validate _all_ Vulkan return codes in one location, previously _every_ native call had to be wrapped which was ugly and slightly tedious.

First a handler for return values is added to the factory that generates the proxy:

```java
public class NativeFactory {
    private Consumer<Object> check = result -> { /* Ignored */ };
}
```

The invocation handler simply delegates return values to the configured handler:

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

As a convenience an integer success code handler can be configured:

```java
public void setIntegerReturnHandler(IntConsumer check) {
    Consumer<Object> handler = result -> {
        if(result instanceof Integer code) {
            check.accept(code);
        }
    };
    setReturnHandler(handler);
}
```

This new feature is utilised by the Vulkan implementation accordingly:

```java
public static Vulkan create() {
    ...
    var factory = new NativeFactory(registry);
    factory.setIntegerReturnHandler(Vulkan::check);
    ...
}
```

Nice.

## Physical Device

### Overview

The refactored library for the physical device is as follows:

```java
interface Library {
    int  vkEnumeratePhysicalDevices(Instance instance, NativeReference<Integer> pPhysicalDeviceCount, @Returned Handle[] devices);
    void vkGetPhysicalDeviceProperties(PhysicalDevice device, @Returned VkPhysicalDeviceProperties props);
    void vkGetPhysicalDeviceMemoryProperties(PhysicalDevice device, @Returned VkPhysicalDeviceMemoryProperties pMemoryProperties);
    void vkGetPhysicalDeviceFeatures(Handle device, @Returned VkPhysicalDeviceFeatures features);
    void vkGetPhysicalDeviceQueueFamilyProperties(Handle device, NativeReference<Integer> pQueueFamilyPropertyCount, @Returned VkQueueFamilyProperties[] props);
    void vkGetPhysicalDeviceFormatProperties(PhysicalDevice device, VkFormat format, @Returned VkFormatProperties props);
    int  vkGetPhysicalDeviceSurfaceSupportKHR(PhysicalDevice device, int queueFamilyIndex, Handle surface, IntegerReference supported);
}
```

There are several fresh challenges to be addressed here:

1. Methods with array parameters, e.g. `vkEnumeratePhysicalDevices`.

2. Parameters that are returned _by reference_ indicated by the new `@Returned` annotation, e.g. `vkGetPhysicalDeviceProperties`.

3. Methods that require _both_ of these to return a structure array by-reference, e.g. `vkGetPhysicalDeviceQueueFamilyProperties`.

Other than refactoring the device library the remainder of the `PhysicalDevice` class remains as-is.

### Integration

The physical device is added to the demo:

```java
PhysicalDevice dev = PhysicalDevice.devices(instance).toList().getFirst();
```

As a reminder: Enumerating the available physical devices uses the [two stage invocation](TODO) approach:

```java
public static Stream<PhysicalDevice> devices(Instance instance) {
    Vulkan vulkan = instance.vulkan();
    VulkanFunction<Handle[]> enumerate = (count, devices) -> vulkan.library().vkEnumeratePhysicalDevices(instance, count, devices);
    Handle[] handles = vulkan.invoke(enumerate, Handle[]::new);
    ...
}
```

The slightly refactored `invoke` helper implements the two-stage invocation pattern:

```java
public <T> T invoke(VulkanFunction<T> function, IntFunction<T> create) {
    // Determine the result size
    var count = factory.integer();
    function.enumerate(count, null);

    // Instantiate the container
    int size = count.value();
    T data = create.apply(size);

    // Invoke again to populate the container
    if(size > 0) {
        function.enumerate(count, data);
    }

    return data;
}
```

This code is a little neater since the `factory` for the integer reference is now a property of the `Vulkan` class and there is no longer any need to explicitly `check` each invocation.

This should allocate the array and off-heap memory for the device handles, however the demo will still fail since the array is a _by reference_ parameter but the code does not update the elements after invocation of the native method.

### Returned Annotation

A by-reference parameter is identified by the `@Returned` marker annotation which is checked in the native factory:

```java
public class NativeFactory {
    private NativeMethod build(MemorySegment address, Method method) {
        ...
        for(Parameter p : method.getParameters()) {
            boolean returned = p.isAnnotationPresent(Returned.class) || NativeReference.class.isAssignableFrom(p.getType());
            builder.parameter(p.getType(), returned);
        }
        ...
    }
}
```

The existing `NativeReference` helper type is refactored (not shown) to follow the same pattern.

The builder for a native method is modified with this additional flag:

```java
public Builder parameter(Class<?> type, boolean returned) {
    NativeTransformer mapper = registry.get(type);
    signature.add(new NativeParameter(type, mapper, returned));
    return this;
}
```

TODO...

////////////////

Where `NativeParameter` composes the transformer and `returned` flag for each parameter:

```java
private record NativeParameter(Class<?> type, NativeTransformer transformer, boolean returned)
```

Arguments that are returned by-reference can now be updated after invocation of the native method:

```java
public Object invoke(Object[] args, SegmentAllocator allocator) {
    Object[] actual = transform(args, allocator);
    Object result = execute(actual);
    update(args, actual);
    return transform(result);
}
```

The `update` method checks whether each parameter is returned by-reference and updates the argument accordingly:

```java
private void update(Object[] args, Object[] actual) {
    if(args == null) {
        return;
    }

    for(int n = 0; n < args.length; ++n) {
        NativeParameter p = signature[n];
        Object arg = args[n];
        if(p.returned && Objects.nonNull(arg)) {
            p.transformer.update(actual[n], arg[n]) {
        }
    }
}
```

Which in turn delegates to a new handler method on the native transformer:

```java
/**
 * Provides a handler to update a by-reference argument.
 * @return Update handler
 * @throws UnsupportedOperationException if the domain type cannot be returned by-reference
 */
default BiConsumer<R, T> update() {
    throw new UnsupportedOperationException();
}
```

This new framework addition will allow by-reference return types to be handled.


```java
```

////////////////

TODO

The next integration step is to retrieve the queue families supported by each physical device, which are returned as a by-reference array of structures:

```java
void vkGetPhysicalDeviceQueueFamilyProperties(Handle device, IntegerReference count, @Returned VkQueueFamilyProperties[] props);
```



```java
public BiConsumer<MemorySegment, NativeStructure> update() {
    return (address, structure) -> {
        Entry entry = entry(structure);
        var segment = address.reinterpret(entry.layout.byteSize());
        entry.mapping.populate(segment, structure);
    };
}
```

skip transform
array update

### Embedded Structures

There is one final complication when retrieving the queue family properties for a physical device:

```java
public class VkQueueFamilyProperties implements NativeStructure {
    public BitMask<VkQueueFlag> queueFlags;
    public int queueCount;
    public int timestampValidBits;
    public VkExtent3D minImageTransferGranularity = new VkExtent3D();
}
```

The `minImageTransferGranularity` field is an _nested_ structure (rather than a pointer) which can be seen from the memory layout:

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

Notes:

* The `minImageTransferGranularity` structure field is initialised to an empty instance in the constructor.
TODO - does this need to be done in the structure? could it be created by the framework?

* The nested layout is essentially cloned from the `VkExtent3D` structure.

TODO...

In the `build` method of the `FieldMapping` class a new clause is added to handle an embedded structure field:

```java
public FieldMapping mapping(MemoryLayout member) {
    ...
    return switch(member) {
        ...
        case StructLayout child -> {
            var type = field.getType();
            List<FieldMapping> children = build(child, type, registry);
            yield new StructureFieldMapping(field, children);
        }
    }
}
```

When transforming an embedded structure the new `StructureFieldMapping` implementation retrieves the field via reflection and recurses accordingly:

```java
record StructureFieldMapping(Field field, List<FieldMapping> mappings) implements FieldMapping {
    public void transform(NativeStructure parent, MemorySegment address, SegmentAllocator allocator) {
        NativeStructure child = (NativeStructure) field.get(parent);
        for(var m : mappings) {
            m.transform(child, address, allocator);
        }
    }
}
```

The inverse `transform` operation (not shown) works similarly.

...TODO

///////////////////////

TODO - cache update function

```java
private List<Family> families(Handle handle) {
    VulkanFunction<VkQueueFamilyProperties[]> function = (count, array) -> vulkan.library().vkGetPhysicalDeviceQueueFamilyProperties(handle, count, array);
    VkQueueFamilyProperties[] properties = vulkan.invoke(function, VkQueueFamilyProperties[]::new);
    ...
}
```

remove variant for structures



