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

It has always been the intention at some point to re-implement the Vulkan native layer using the FFM (Foreign Function and Memory) library developed as part of project [Panama](https://openjdk.java.net/projects/panama/).  The reasons for delaying this rework are discussed in [code generation](/jove-blog/blog/part-1-intro/code-generation#alternatives) at the very beginning of the blog.

With the LTS release of JDK 21 the FFM library was elevated to a preview feature (rather than requiring an incubator build) and much more information and tutorials started to appear online, so it high time to replace JNA with the more future proofed (and safer) FFM solution.

A fundamental decision is whether to use the `jextract` tool to generate the Java bindings for the native libraries or to implement a bespoke solution.  To resolve this question we use the tool to generate the Vulkan API and then compare and contrast with a small, hard-crafted application built using FFM from first principles.

---

# Design

## Exploring Panama

The first step in gaining some familiarity with the FFM library is to build a throwaway application that uses a native library, and again GLFW is chosen as the guinea pig.  This API is relatively simple in comparison to Vulkan, which is front-loaded with complexities such as marshalling of structures, by-reference types, callbacks for diagnostic handlers, etc.

The new demo application first loads the native library:

```java
public class DesktopForeignDemo {
    private final Linker linker = Linker.nativeLinker();
    private final SymbolLookup lookup;

    void main() throws Exception {
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
int init = demo.invoke("glfwInit", null, JAVA_INT);
System.out.println("glfwInit=" + init);
```

Vulkan support can be queried:

```java
boolean supported = demo.invoke("glfwVulkanSupported", null, ValueLayout.JAVA_BOOLEAN);
System.out.println("glfwVulkanSupported=" + supported);
```

The GLFW version string can also be retrieved:

```java
MemorySegment version = (MemorySegment) demo.invoke("glfwGetVersionString", null, ADDRESS);
MEmorySegment string = version.reinterpret(Integer.MAX_VALUE).getString(0L);
System.out.println("glfwGetVersionString=" + string);
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

TODO

did similar for Vulkan
not shown
as far as querying physical device queues
since covers most of the use cases
structures
arrays handles and structures
by reference structure / arrays


## Analysis

Generating the FFM bindings using `jextract` is relatively trivial and takes a couple of seconds:

`jextract --source \VulkanSDK\1.2.154.1\Include\vulkan\vulkan.h`

With some experience of using FFM and the generated Vulkan API, we can attempt to extrapolate how JOVE could be refactored and make some observations.

The advantages of `jextract` are obvious:

* Proven and standard JVM technology.

* Automated (in particular the FFM memory layouts for native structures).

However there are disadvantages:

* The generated API is polluted by ugly internals such as field handles, structure sizes, helper methods, etc.

* API methods and structures can only be defined in terms of FFM types or primitives (resulting in poor type safety and loss of meaning).

* In particular enumerations are implemented as static integer accessors (!) with all the concerns that led us to abandon LWJGL in the first place.

* Implies throwing away the existing code generated enumerations and structures.

Alternatively an abstraction layer based on the existing hand-crafted API and code-generated classes would have the following advantages:

* The native API and structures can be expressed in domain terms.

* Proper enumerations.

* Retains the existing API and code-generated enumerations and structures.

On the other hand:

* Substantial development effect to implement a custom framework that abstracts over the underlying FFM bindings.

* Proprietary solution with the potential for unknown problems further down the line.

* An FFM memory layout needs to be generated for _all_ structures.

Despite the advantages of `jextract` the hybrid solution is preferred for the reasons given above, in particular type safety and self-documenting code.
In any case the application and/or JOVE would _still_ need to transform domain types to/from the FFM equivalents even if `jextract` was used to generate the API.

The source code generated by `jextract` will be retained as it will be a useful resource during development, particularly for the memory layout of structures.

## Solution

With this decision made (for better or worse) the requirements for the new framework can be analysed.

The following table summarises the domain types that will need to be supported:

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

* _type_ is a Java or JOVE domain type.

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

6. An array can be returned from some native methods, e.g. `glfwGetRequiredInstanceExtensions`.  This is problematic since the _length_ of the returned array is unknown and therefore cannot be modelled as a Java array.  Generally the length is returned as a by-reference integer in the same API method.  The new framework will disallow arrays as a return type and provide a custom workaround.  Note that returned arrays are only used by the GLFW library.

7. GLFW makes heavy use of callbacks for device polling which will require substantial factoring to FFM upcall stubs.

8. Finally there are a couple of edge-cases for by-reference parameters:

* A structure parameter can be returned by-reference from a native method, e.g. `vkGetPhysicalDeviceProperties`.  The application is responsible for allocating a pointer to off-heap memory for the structure which is _populated_ by the native layer.  The framework then constructs a new structure instance from the returned data _after_ invocation.

* Similarly for a by-reference array parameter, e.g. `vkEnumeratePhysicalDevices` returns an array of device handles.  The application creates an empty array sized accordingly beforehand (generally via the _two stage invocation_ mechanism), allocates off-heap memory for the array, and unmarshals the elements after the method call.  Note that the memory must be a contiguous block in this case.

The problem here is that one cannot determine whether a parameter is being passed _by value_ or _by reference_ solely from the method signature.  JNA used the rather cumbersome `ByReference` marker interface, which generally required _every_ structure to have _two_ implementations to support both cases.  The most obvious solution is to implement a custom annotation to explicitly declare by-reference parameters and switch behaviour accordingly.

Note that the notion of transformers to map domain objects to/from FFM method arguments is already sort of provided by Panama.  Native methods are implemented as _method handles_ which support mutation of the arguments and the return value.  The original intention was to make use of this functionality, unfortunately it proved very difficult in practice for all but the most trivial cases (most examples do not get any further than simple static adapters, probably for this reason).  However this is probably an option to revisit when the refactored code (and our understanding) is more mature.

## Framework

From the above the required components for the new framework are:

* A _native library builder_ responsible for creating an implementation of a native API expressed in _domain_ terms.

* A set of _transformers_ for the supported types identified above, which are responsible for allocation of off-heap memory and converting to/from the equivalent FFM representation.

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

This 'big bang' approach should (hopefully) get the refactored JOVE library to a point where it compiles but will certainly not run.

The plan is then to iteratively reintroduce each JOVE component to a new, temporary demo application and refactor accordingly.  As already noted, Vulkan is very front-loaded with complexity, so additional framework support will be addressed as we progress.

Once we are confident that the bulk of the challenges facing the new framework have been addressed, the temporary code can be discarded and we will continue refactoring with the existing suite of demo applications.

TODO...
create another new demo up to creation of the logical device
GLFW and Vulkan
implement basic framework 
extend framework to cover use cases as we progress
...TODO

## Framework

TODO...

With JNA removed and the basics of the new framework in place, refactoring of the Vulkan API itself can begin which entails the following:

* A global find-and-replace for the integer and pointer by-reference values.

* Adding the newly implemented transformers to a default registry.

* Refactoring all the Vulkan API libraries to match the new framework and replace any JNA-specific types.

* Temporarily hacking out the GLFW device polling code.

* Regeneration of the Vulkan structures to remove JNA artifacts such as imports, the `FieldOrder` annotations, etc.

Remarkably this initial refactoring work went much more smoothly than anticipated, with only a handful of compilation problems that were fiddled until later in the process (which are covered below).  Apparently the existing JOVE code did a decent job of abstracting over JNA.

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

* The memory address of a structure is allocated on _every_ invocation.  A future enhancement _could_ store the address in the structure itself (essentially caching) but this would require tracking modifications (tricky given the fields are public) and would probably also make structure arrays more complex.  In reality native structures are really only intended as _data carriers_ with a lifetime that only applies during marshalling, therefore the simpler approach is much more palatable.

* The same applies to the memory address for a transformed Java string which _could_ also be cached.

The transformer implementation means that _any_ reference to a structure in an API method or another structure results in the field mappings being generated for that class.  Therefore the code generator had to be refactored at this point to rebuild all Vulkan structures involving:

1. Removal of the JNA dependency.

2. Implementation the new `NativeStructure` interface in the code template.

3. Removal of the injection of the JNA `FieldOrder` annotation.

4. The addition of the FFM memory `layout` for each structure (which is temporarily empty as we will continue to hand-craft structure layouts for the moment).

...TODO

### Native Library

The logical starting point is the factory class that builds the native library.

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

Where _api_ is a user defined interface that specifies the native library methods to be implemented and a `NativeMethod` is a wrapper for the resultant FFM handle.

The API methods are enumerated by reflecting the public methods of the user-defined API and building the native method wrapper:

```java
private Map<Method, NativeMethod> methods(Class<?> api) {
    return Arrays
        .stream(api.getMethods())
        .filter(NativeLibraryBuilder::isNativeMethod)
        .collect(toMap(Function.identity(), this::build));
}
```

Where native methods are defined as the public, non-static methods of the API interface:

```
private static boolean isNativeMethod(Method method) {
    int modifiers = method.getModifiers();
    return !Modifier.isStatic(modifiers);
}
```

The overloaded `build` method looks up the symbol (i.e. the off-heap address) of the method by name and temporarily creates an empty wrapper:

```java
public NativeMethod build(Method method) {
    MemorySegment address = lookup.find(method.getName()).orElseThrow(...);                
    return new NativeMethod(); // TODO
}
```

Next an invocation handler is created that delegates Java method calls to the underlying native method:

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

### Native Method

Building a native method requires the parameter signature and return type in order to link the FFM method handle.  Looking ahead we will also need to construct native methods programatically (i.e. not using reflection) for the diagnostics handler and possibly other use cases.  Therefore this class will be agnostic to the source of the methods address.

First a builder is added to construct a native method:

```java
public static class Builder {
    private final Linker linker = Linker.nativeLinker();
    private MemorySegment address;
    private Class<?> returnType;
    private final List<Class<?>> signature = new ArrayList<>();
}
```

Construction of the native method handle requires the symbol address and an FFM function descriptor.  First the FFM memory layout of the method is derived from the signature:

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

    if(returns == void.class) {
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

The library factory can now be updated to use this new builder to instantiate a native method for each API method:

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

The native method class itself is trivial at this stage:

```java
public class NativeMethod {
    private MemoryHandle handle;
    
    public Object invoke(Object[] args) {
        try {
            return handle.invokeWithArguments(args);
        }
        catch(Exception e) {
            ...
        }
    }
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

* Maps the domain type of the parameters and the return value to the equivalent FFM native layout (albeit only for primitive integers so far).

* Transparently delegates API calls to the underlying native methods.

Sweet.

### Native Transformer

With the basic framework in place the next step is to implement support for the remaining primitives and Java reference types, such that the API can be expressed in domain terms (just for GLFW to begin with).

The first attempt used the functionality provided by the `MethodHandle` class (used extensively in FFM) which supports transformation of the incoming arguments and the method return value.  This initially seemed a perfect fit for the requirements, in particular the framework would only need custom handling for non-primitive types.  However in practice the code became convoluted very quickly, since the logic to modify a method argument is generally also implemented as a 'target' method handle, which entails fiddly lookup and binding code for all but the most trivial cases.

After some time we abandoned this approach in favour of pre/post-processing the arguments before invoking the native method.  The code became much clearer and easier to unit-test, at the expense of more framework code and the overhead of additional processing on method invocation.  Of importance is that checks for error cases can still be made when the native method is being constructed.

> As it turns out some form of pre/post-processing would probably have been required anyway, since it hard to envisage how by-reference parameters could be implemented using just the `MethodHandle` class.

The second attempt started with the concept of an argument _transformer_ which is responsible for marshalling a Java type to/from its FFM equivalent (inspired by JNA type converters).

After some further analysis of the required types to be supported by the framework, the following groups or _archetypes_ were identified:

* The identity transformer for arguments that do not require any transformation.  These are the types directly supported by FFM, namely Java primitives and possible the `MemorySegment` type.

* A reference type transformer, either built-in helpers for common use-cases (e.g. string) or custom domain types.

* Structures

* Arrays

The new argument transformer is modelled on the above:

```java
public interface Transformer {
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

The `signature` transformers are applied before invocation to marshal the arguments to FFM types and `returns` unmarshals the return value:

```java
public Object invoke(Object[] args) {
    Object[] foreign = marshal(args, Arena.ofAuto());
    Object result = handle.invokeWithArguments(foreign);
    if(returns == null) {
        return null;
    }
    else {
        returns.unmarshal(result);
    }
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
            case ReferenceTransformer ref -> ref.marshal(args[n]);
            ...
        };
    }

    return transformed;
}
```

### Reference Types

The reference transformer will be an extension point for the framework supporting both built-in and custom types:

```java
public non-sealed interface ReferenceTransformer<T extends Object, R> extends Transformer {
    @Override
    default MemoryLayout layout() {
        return ValueLayout.ADDRESS;
    }

    /**
     * Marshals a Java argument to its native representation.
     * @param arg           Java argument
     * @param allocator     Off-heap allocator
     * @return Native argument
     */
    Object marshal(T arg, SegmentAllocator allocator);

    /**
     * Unmarshals a native value.
     * @param value Native value
     * @return Java or domain value
     * @throws UnsupportedOperationException if this type cannot logically be returned from a native method
     */
    T unmarshal(R value);
}
```

Where `T` is a domain type and `R` is the corresponding FFM type.

An application can now define custom transformers which are registered with the framework via the a new _registry_ component:

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
        Transformer supertype = find(type);
        add(type, supertype);
        return supertype;
    }
    else {
        return transformer;
    }
}
```

If a transformer is not present the registry scans the registered types to find a supported supertype:

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

Note that the mapping to the supertype transformer is registered as a side-effect.

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

At this point we can also refactor the some of the other reference types required by JOVE, firstly Java strings:

```java
public class StringTransformer implements ReferenceTransformer<String, MemorySegment> {
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

    public Handle(long address) {
        this(MemorySegment.ofAddress(address));
    }

    public MemorySegment address() {
        return MemorySegment.ofAddress(address.address());
    }
}
```

With a companion transformer:

```java
public static class HandleTransformer implements ReferenceTransformer<Handle, MemorySegment> {
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

    public static class NativeObjectTransformer implements ReferenceTransformer<NativeObject, MemorySegment> {
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

The final framework component required for the basic GLFW prototype is support for by-reference integers and pointers.

For the moment the following new type implements these relatively simple 'atomic' by-reference types:

```java
public abstract class NativeReference<T> {
    private MemorySegment pointer = MemorySegment.NULL;

    protected MemorySegment pointer(SegmentAllocator allocator) {
        if(MemorySegment.NULL.equals(pointer)) {
            pointer = allocator.allocate(ValueLayout.ADDRESS);
        }

        return pointer;
    }

    /**
     * @return Referenced value
     */
    public abstract T get();
}
```

The `pointer` method allocates the underlying address on demand, which is invoked by the companion transformer:

```java
public static class NativeReferenceTransformer implements ReferenceTransformer<NativeReference, MemorySegment> {
    public Object marshal(NativeReference ref, SegmentAllocator allocator) {
        return ref.pointer(allocator);
    }

    public NativeReference unmarshal(MemorySegment address) {
        throw new UnsupportedOperationException();
    }
}
```

New by-reference integer instances are created by the refactored factory:

```java
public static class Factory {
    /**
     * @return New integer-by-reference
     */
    public NativeReference<Integer> integer() {
        return new NativeReference<>() {
            @Override
            public Integer get() {
                return super.pointer.get(ValueLayout.JAVA_INT, 0L);
            }
        };
    }
}
```

And similarly for by-reference pointers which return an immutable `Handle` instance:

```java
public NativeReference<Handle> pointer() {
    return new NativeReference<>() {
        @Override
        public Handle get() {
            MemorySegment address = super.pointer.get(ValueLayout.ADDRESS, 0L);
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

This implementation is basically an analog of the equivalent JNA classes (e.g. `IntByReference`) and nicely requires minimal refactoring of the existing JOVE code.

> It is not yet clear how much overlap there will be (if any) between the relatively simple atomic by-reference parameters and the more complicated use-cases (structures, arrays and structure-arrays).  These new classes may eventually be superceded by a transformer _archetype_ for all by-reference types.  In particular it seems a bit daft to have to implement but disallow the `unmarshal` method.

Most of the GLFW library is now supported by the new framework other than the following:

* The `glfwGetRequiredInstanceExtensions` method is somewhat of an edge case since it returns a string-array (the only method is either library that does so).

* Callback support for GLFW devices.

At this point we feel confident that the framework is ready to be integrated into JOVE.  The `Desktop` service is refactored using the new framework resulting in the following:

```java
public static Desktop create() {
    // Init supported types
    var registry = Registry.create();
    registry.add(int.class, new IdentityTransformer(ValueLayout.JAVA_INT));
    registry.add(boolean.class, new IdentityTransformer(ValueLayout.JAVA_BOOLEAN));
    registry.add(String.class, new StringTransformer());
    registry.add(Handle.class, new HandleTransformer());
    registry.add(NativeObject.class, new NativeObjectTransformer());
    registry.add(NativeReference.class, new NativeReferenceTransformer());

    // Load native library
    var factory = new NativeLibraryBuilder(..., registry);
    DesktopLibrary lib = factory.build(DesktopLibrary.class);

    // Init GLFW
    int result = lib.glfwInit();
    if(result != 1) throw new RuntimeException(...);

    // Create desktop service
    return new Desktop(lib, new NativeReference.Factory());
}
```

And the prototype is similarly refactored and extended to the stage of creating the application window:

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
}
```

Is it now time to turn our attention to Vulkan which means implementing support for native structures.

## Structures

TODO
Exception handling has been omitted and helper methods silently inlined for brevity.

### Instance Creation

The first step is the introduction of the following replacement for the definition of a native structure:

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
class StructureTransformer implements ReferenceTransformer<NativeStructure, MemorySegment> {
    private StructLayout layout;
    
    public MemoryLayout layout() {
        return layout;
    }

    public MemorySegment marshal(NativeStructure structure, SegmentAllocator allocator) {
        MemorySegment address = allocator.allocate(layout);
        // TODO - marshalling!
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

Note that the above code mandates that _all_ structures have a default constructor which is a common constraint for a marshalling framework.

Refactoring the various Vulkan structures will eventually require the following changes to the code generator:

* Removal of any JNA artifacts from the template: imports and the `@FieldOrder` annotation.

* Implement the new structure interface.

* Minor changes to the mapping logic, e.g. replace any JNA pointers with `Handle`.

* Generation of the FFM memory layout for each structure based on the metadata derived from the native header.

For the moment the two structures used to create the Vulkan instance are manually fiddled to implement the new interface but do not require any field-level changes.

The memory layout for the instance descriptor is as follows:

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

These layouts were copied from the equivalent code generated by the `jextract` tool earlier.  The code generator will also need to track the byte alignment of each structure and inject padding as required.

### Integration

As a quick diversion, the following new type is introduced to compose the various top-level components of the Vulkan implementation that were previously separate fields of the instance:

```java
public class Vulkan {
    private final VulkanLibrary lib;
    private final Registry registry;
    private final NativeReference.Factory factory;
}
```

With a `create` method to instantiate the library using the new framework:

```java
public static Vulkan create() {
    var registry = new Registry();
    ...
    
    var factory = new NativeLibraryBuilder(registry);
    var lib = factory.build("vulkan-1", VulkanLibrary.class);
    return new Vulkan(lib, registry, new NativeReference.Factory());
}
```

This is equivalent to the `Desktop` class for the GLFW implementation and replaces the `create` method on the `VulkanLibrary` interface (which had become a bit of dumping ground for helper methods and constants).

Both of the structures used to create the Vulkan instance are dependant on integer enumerations which requires another companion transformer:

```java
class IntEnumTransformer implements ReferenceTransformer<IntEnum, Integer> {
    private final ReverseMapping<?> mapping;

    public MemoryLayout layout() {
        return ValueLayout.JAVA_INT;
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

Note that the return type is a native integer in this case.

Finally the registry is modified to factor out the logic that generates new transformers on demand:

```java
private Transformer create(Class<?> type) {
    if(IntEnum.class.isAssignableFrom(type)) {
        var actual = (Class<? extends IntEnum>) type;
        return new IntEnumTransformer(actual);
    }
    else
    if(NativeStructure.class.isAssignableFrom(type)) {
        var builder = new StructureTransformer.Builder(this);
        return builder.build((Class<? extends NativeStructure>) type);
    }
    else {
        Transformer ref = find(type);
        add(type, ref);
        return ref;
    }
}
```

This tightly couples the registry to the transformer implementations and requires nasty casting, it works for the moment but may warrant some sort of factory approach later.

The `Instance` domain class can now be refactored with the following temporary fiddles:

* The `VulkanLibrary` is chopped back to just the instance methods

* The diagnostics handler is temporarily removed.

* Similarly for the `function` method that retrieves the function pointers for handlers.

* Extensions and validation layers are uninitialised pending array support and a temporary `String[]` transformer is registered that returns the `MemorySegment.NULL` constant.

The prototype can now be extended to instantiate Vulkan and configure the instance:

```java
Vulkan vulkan = Vulkan.create();

Instance instance = new Instance.Builder()
    .name("Prototype")
    .extension("VK_EXT_debug_utils")
    .layer(ValidationLayer.STANDARD_VALIDATION)
    .build(vulkan);
```

This code compiles and runs but either does nothing or fails since we are not actually populating the structures.

### Field Mappings

A _field mapping_ associates a structure field with its native representation:

```java
private record FieldMapping(VarHandle field, VarHandle foreign, Transformer transformer) {
}
```

Where `field` and `foreign` are handles to the structure field and the corresponding off-heap field respectively.

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

* For the moment only 'atomic' fields (primitives and reference types) are supported, i.e. members that are implemented as a `ValueLayout` or an `AddressLayout`.

* This includes pointers to other structures, e.g. the `pApplicationInfo` field of the instance descriptor.

* The structure layout also nicely explicitly specifies the order of the fields since it is derived from the Vulkan header (unlike reflection where the field order is undefined, hence the `@FieldOrder` annotation required for JNA structures).

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

Next the handle for the off-heap field is constructed from the structure layout:

```java
PathElement path = PathElement.groupElement(field.getName());
VarHandle foreign = layout.varHandle(path);
```

And finally the transformer to marshal between the fields is looked up and wrapped into a new instance:

```java
Transformer transformer = registry.get(field.getType());
return new FieldMapping(local, foreign, transformer);
```

The process for marshalling a structure field is:

1. Retrieve the value of the field from the structure.

2. Apply the transformer.

3. Set the transformed field in the off-heap memory.

Which is implemented by the following method on the new class:

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

And the last piece of the puzzle is the equivalent method in the structure transformer that allocates the off-heap memory and iterates over the mappings:

```java
public MemorySegment marshal(NativeStructure structure, SegmentAllocator allocator) {
    public MemorySegment marshal(Object structure, SegmentAllocator allocator) {
        MemorySegment address = allocator.allocate(layout);
        for(var field : fields) {
            field.marshal(structure, address, allocator);
        }
        return address;
    }
}
```

In the prototype the Vulkan instance should now be successfully instantiated and configured using the new framework.

A lot of work to get to the stage JOVE was at several years previous!




```java
public static Map<Class<?>, IdentityTransformer> primitives() {
    List<ValueLayout> primitives = List.of(
        JAVA_BOOLEAN,
        JAVA_BYTE,
        JAVA_CHAR,
        JAVA_SHORT,
        JAVA_INT,
        JAVA_LONG,
        JAVA_FLOAT,
        JAVA_DOUBLE
    );

    return primitives
        .stream()
        .collect(toMap(ValueLayout::carrier, IdentityTransformer::new));
}
```



///////////////////////



///////////////////////


----

# Diagnostic Handler

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

TODO...

```java
public boolean message(int severity, int typeMask, MemorySegment pCallbackData, MemorySegment pUserData) {
```

...TODO

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

Finally the `build` method of the handler is modified to lookup the transformer for the message structure:

```java
public DiagnosticHandler build() {
    ...
    var transformer = (StructureNativeTransformer) registry.get(VkDebugUtilsMessengerCallbackData.class);
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
The last required change is to unmarshal the diagnostic report structure received by the `message` callback method itself:

```java
public boolean message(...) {
    long size = transformer.layout().byteSize();
    var segment = pCallbackData.reinterpret(size);
    var data = (VkDebugUtilsMessengerCallbackData) transformer.returns().apply(segment);
    ...
}
```

In this case the off-heap memory needs to be explicitly resized to match the expected structure.

Diagnostic handlers can now be attached to the instance as before, which is important given the invasive changes being made to JOVE.  A good test of the refactor is to temporarily remove the code that releases attached handlers when the instance is destroyed, which should result in Vulkan complaining about orphaned objects.

### Method Success Codes

The approach of using a proxy for the native libraries enables the new framework to validate _all_ Vulkan return codes in one location, previously _every_ native call had to be wrapped which was ugly and slightly tedious.

First a handler for return values is added to the factory that generates the proxy:

```java
public class NativeFactory {
    private Consumer<Object> check = result -> {};
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

The physical device is added to the test code:

```java
PhysicalDevice dev = PhysicalDevice.devices(instance).toList().getFirst();
```

As a reminder: enumerating the available physical devices uses the [two stage invocation](TODO) approach:

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

### Nested Structures

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



