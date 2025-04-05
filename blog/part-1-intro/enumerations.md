---
title: Enumerations Revisited
---

---

## Contents

- [Background](#background)
- [Solution](#solution)
- [Reverse Mapping](#reverse-mapping)
- [Bit Masks](#bit-masks)
- [Integration](#integration)
- [Conclusion](#conclusion)

---

## Background

When first using the code generated enumerations it became apparent that several requirements had been overlooked:

1. There is also the need to map a native value _to_ the corresponding enumeration constant, for example when retrieving the set of `VkQueueFlag` for a physical device.  The integer enumerations need to be _bi directional_ mappings.

2. Many of the enumerations are in fact bit-fields essentially representing a collection of boolean values, where the constant values specify individual bits.  This implies some mechanism is required to reduce a collection of constants to a bitfield (and again support the reverse operation).

3. A native enumeration is a `typedef` which is essentially a _synonym_ for an unsigned integer (rather than an actual 'type' as such).  However Java obviously has no analog of synonyms, therefore as things stand the enumerations _have_ to be represented as a primitive `int` in the code generated structures and API methods.

The implications of the last point in particular are:

* Type safety - There is nothing to prevent the use of the wrong enumeration or even an arbitrary integer.

* Purpose - The user can only determine the actual enumeration type from the documentation or by guessing its type from the field or parameter name.

One of the main reasons for abandoning LWJGL in favour of a bespoke solution was to provide type-safe Java enumerations, unfortunately the current solution is only halfway there at best.

For a library with a handful of enumerations these problems could be worked around, but for the large number of Vulkan enumerations something more practical is required that works in __all__ cases.

## Solution

JNA supports the notion of _type converters_ that are used to marshal data between Java and the native layer.  This mechanism is used to transform the basic types (primitives, strings, pointers, arrays, etc) but can also be extended to elevate custom domain types to first-class JNA citizens.  This seems an ideal solution to the type-safety problem and also encompasses the missing requirement for bi-directional enumerations.

However a separate converter would need to be implemented and registered for __every__ integer enumeration.  It is clearly preferable to avoid abusing the code generator by simply replicating the same solution in every enumeration class (essentially a semi-automated cut-and-paste bodge).

What is really needed is some sort of base type which can then be used to implement a common solution, but of course Java enumerations are special types that cannot be subclasses.  However - although it is not common practice - a Java enumeration _can_ implement an interface.  This technique is leveraged to define a common abstraction to support the above requirements using a single type converter.

> Indeed our IDE will not code-complete an interface on an enumeration presumably because it thinks it is not legal Java.

The [interface](https://github.com/stridecolossus/JOVE/blob/master/src/main/java/org/sarge/jove/util/IntEnum.java) itself is trivial and simply affirms the existing `value` method of the enumerations:

```java
public interface IntEnum {
    /**
     * @return Enum literal
     */
    int value();
}
```

The enumeration template used in the code generator is modified to implement the new interface as illustrated in the following example:

```java
public enum VkImageUsageFlag implements IntEnum {
    TRANSFER_SRC(1),
    TRANSFER_DST(2),
    ...

    private final int value;
    
    private VkImageUsageFlag(int value) {
        this.value = value;
    }

    @Override
    public int value() {
        return value;
    }
}
```

The missing requirements can now be addressed in terms of this new type which is common to __all__ integer enumerations.

First a JNA type converter is added to the interface:

```java
TypeConverter CONVERTER = new TypeConverter() {
    public Class<?> nativeType() {
        return Integer.class;
    }

    public Object toNative(Object value, ToNativeContext context) {
        if(value instanceof IntEnum e) {
            return e.value();
        }
        else {
            return 0;
        }
    }

    public Object fromNative(Object nativeValue, FromNativeContext context) {
        ...
    }
};
```

This converter is registered with a global JNA _type mapper_ in the Vulkan library:

```java
public interface VulkanLibrary ... {
    TypeMapper MAPPER = mapper();

    private static TypeMapper mapper() {
        var mapper = new DefaultTypeMapper();
        mapper.addTypeConverter(IntEnum.class, IntEnum.CONVERTER);
        return mapper;
    }
}
```

And finally the JNA library is configured with this type mapper at instantiation-time:

```java
static VulkanLibrary create() {
    ...
    return Native.load(name, VulkanLibrary.class, Map.of(Library.OPTION_TYPE_MAPPER, MAPPER));
}
```

The introduction of the type converter also neatly avoids the minor irritation of having to invoke the `value` method whenever an enumeration constant is used in code.

## Reverse Mapping

The following new type implements the bi-directional requirement:

```java
class ReverseMapping<E extends IntEnum> {
    private final Map<Integer, E> map;

    public ReverseMapping(Class<E> clazz) {
        E[] array = clazz.getEnumConstants();
        this.map = Arrays
            .stream(array)
            .collect(toMap(IntEnum::value, Function.identity(), (a, b) -> a));
    }
}
```

Note that some enumerations have duplicate or synonymous entries (e.g. `VkImageAspect`) which are silently ignored by the collector in the above constructor.

The enumeration constant for a given native value can now be looked up from the reverse mapping:

```java
public E map(int value) {
    E constant = map.get(value);
    if(constant == null) throw new IllegalArgumentException();
    return constant;
}
```

The `fromNative` method in the type converter can now be completed in terms of the new class:

```java
public Object fromNative(Object nativeValue, FromNativeContext context) {
    var type = ((? extends IntEnum) context.getTargetType();
    var mapping = new ReverseMapping<>(type);
    return mapping.map(value);
}
```

> Caching of the reverse mappings will probably be introduced later in the project.

## Bit Masks

For bitfields a second new type is introduced which parameterises an integer enumeration and wraps the underlying mask:

```java
public record EnumMask<E extends IntEnum>(int bits) {
}
```

A custom constructor can now be implemented to reduce a collection of constants:

```java
public EnumMask(Collection<E> values) {
    this(reduce(values));
}

private static int reduce(Collection<? extends IntEnum> values) {
    return values.stream().mapToInt(IntEnum::value).sum();
}
```

Transforming a mask to the corresponding enumeration constants is slightly more involved, the following method walks the positive bits of the mask mapping each to the corresponding constant:

```java
public Set<E> enumerate(ReverseMapping<E> reverse) {
    int range = Integer.SIZE - Integer.numberOfLeadingZeros(bits);
    return IntStream
        .range(0, range)
        .map(bit -> 1 << bit)
        .filter(value -> (value & bits) == value)
        .mapToObj(reverse::map)
        .collect(toSet());
}
```

Note that the bits of the mask are defined by powers of two rather than an index.

A second type converter (not shown) is implemented for the `EnumMask` type.

## Integration

Unfortunately the JNA type mapper needs to be applied to __every__ JNA structure instance to use integer enumerations, therefore the following intermediate base-class is introduced for the code-generated structures:

```java
public abstract class VulkanStructure extends Structure {
    protected VulkanStructure() {
        super(MAPPER);
    }
}
```

A final complication is that an unspecified value (i.e. `null` or zero) may not be a valid constant and/or native integer.  Therefore a _default value_ is introduced to the reverse mapping which is initialised in the constructor:

```java
class ReverseMapping<E extends IntEnum> {
    ...
    private final E def;

    private ReverseMapping(Class<E> clazz) {
        ...
        this.def = map.getOrDefault(0, array[0]);
    }
}
```

The default value is mapped from the enumeration constant represented by zero if present or is arbitrarily set to the first constant.

A `null` constant is now defensively handled in the type converter:

```java
public Object toNative(Object value, ToNativeContext context) {
    if(value instanceof IntEnum e) {
        return e.value();
    }
    else {
        return def.value();
    }
}
```

And similarly for an unspecified native value received from the native layer:

```java
public Object fromNative(Object nativeValue, FromNativeContext context) {
    var type = ((? extends IntEnum) context.getTargetType();
    var mapping = new ReverseMapping<>(type);
    if(nativeValue == 0) {
        return mapping.defaultValue();
    }
    else {
        return mapping.map(value);
    }
}
```

## Conclusion

Summary:

* Integer enumerations are now fully type-safe, self-documenting and transparently handled by JNA.

* Enumeration bitfields are supported by the new `EnumMask` class.

* The code generator is updated to support the new types.

