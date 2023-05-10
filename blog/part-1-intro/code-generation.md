---
title: Code Generating the Vulkan API
---

---

## Contents

- [Overview](#overview)
- [Code Generation](#generation-game)
- [Enumerations](#enumerations)
- [Structures](#structures)
- [Conclusion](#conclusion)

---

## Overview

### Background

The previous project was implemented using [LWJGL](https://www.lwjgl.org/) which provides Java bindings for the native OpenGL library (amongst others).  LWJGL had recently implemented a Vulkan port and we expected to be able to use the new bindings to get to grips with the Vulkan API.

However things did not work out as hoped.  Not at all.

Our progress through the [tutorial](https://vulkan-tutorial.com/) was glacially slow, with more time being spent trying to understand what LWJGL was up to, instead of learning how to use Vulkan.  Each step forward seemed to lead to some mystifying situation or road-block that side-tracked any progress.

There were several reasons for this:

* The Vulkan API makes heavy use of structures to configure Vulkan components, which are surfaced by LWJGL as Java classes with (as a nice touch) a fluid builder-like interface.  However all the internals are exposed as __public__ members, there are multiple setters for each field, and a slew of allocation methods implemented on every structure.  Finding the right field becomes error-prone and time-consuming.  This is repeated for every field, for every structure, for every use-case involving structures (which is almost all of them).

* The API also uses a large number of enumerations all of which are bundled as old school integer constants into a __single__ class (along with the API methods) resulting in several __thousand__ members.  Besides the likelihood of accidentally using the wrong enumeration and the obvious type safety issues, finding an enumeration constant is virtually impossible and makes the IDE code assist feature practically redundant.

* The paucity of decent (or indeed any) documentation, examples or tutorials only exacerbated matters: those we found were basically just C code masquerading as Java with little or no documentation or modularity, essentially a wall-o-code that was very difficult to navigate and practically worthless as an exemplar.

A couple of examples:

The first task for a Vulkan application is to create the _instance_ which involves populating a couple of structures and invoking the relevant API method.  The `VkApplicationInfo` structure specifies some properties of the application and the required API version.  The native structure has six fields whereas the LWJGL implementation has over 70 class members, the code assist popup in our IDE has scroll-bars, and this is one of the simplest structures.

One of the fields in this structure is the application name which is a string (mapping to a native pointer to a null-terminated character array).  In the LWJGL implementation we had to instantiate a memory stack object, invoke a static method on a different utility class passing the stack and the string to allocate an NIO buffer, and then pass _that_ to the structure.  Presumably there are logical reasons for this (off-heap performance or thread safety perhaps) but there was no clue as to _why_ this had to be done, whether there were alternatives, was the application responsible for releasing resources later, etc.

Obviously the LWJGL port was code-generated from the Vulkan header, and it is to be expected that there are oddities and compromises in the resultant API which could never be as 'clean' as a totally hand-crafted solution.  This is not intended to be a negative 'review' but an illustration of our concerns: everything took an age to implement, much of the code was basically a mystery, and we had barely scratched the surface of Vulkan.  Given how difficult and tedious it had been to implement even the most basic Vulkan demo using LWJGL, the prospect of doing the same for the much more complicated stuff later on was not appealing.  Eventually we just lost interest and gave up.

### Alternatives

Sometime later we were encouraged by a friend to make a second attempt.  Our first design decision was that unless LWJGL had materially changed we would look for alternative bindings.

Unfortunately there were none (that we could find) so our focus shifted to implementing custom bindings to the native Vulkan library.

Straight JNI we immediately discounted - no one in their right mind would choose to implement JNI bindings for an API as large as Vulkan.  It had also been (thankfully) many years since we wrote any C/C++ code and we certainly didn't intend starting now.  Additionally JNI offers nothing to support the large number of enumerations and structures used by Vulkan.

Project [Panama](https://openjdk.java.net/projects/panama/) is an on-going JSR for a programmatic pure-Java alternative to JNI, in particular it provides the `jextract` tool that generates an API from a native header file.  This is perfect for our requirements but there are some misgivings.  The API is _extremely_ complicated with a morass of code required to perform even the simplest call to the native layer.  Also at the time of writing the technology was still in a relatively immature and fluid state, required an incubator JVM build, and there was little in the way of tutorials or examples.

> If and when Panama becomes a fully fledged part of the JDK and there is more support in terms of documentation, tutorials, etc. then it it likely to be the best solution to our problem.

Next we considered SWIG which is the code-generation technology used by LWJGL.  Again we were not encouraged, proprietary descriptors are required to bind to the native layer and we have already covered our issues with the resultant code.

Finally we came across JNA (which was new to the author) and initial impressions were promising:

* The premise of auto-magically mapping Java interfaces to the native API seemed ideal, with no additional metadata or descriptors required.

* In particular the support for mapping C/C++ structures to Java classes was very appealing given the large number of structures that are used to configure a Vulkan application.

* The library seemed to have a large and active user base with plenty of posts on stack-overflow (for example).

* The documentation was generally excellent and there were plenty of tutorials and examples available.

We had a possible winner.

### JNA

To see whether JNA would suit our purposes we first exercised it against a simpler library.  We had already planned to use [GLFW](https://www.glfw.org/) for desktop related functionality (such as creating native windows, managing input devices, etc) and it integrates nicely with Vulkan as we will see later.

With a hand-crafted implementation of the JNA library the bulk of what would become the _desktop_ package of JOVE was developed in a couple of hours, the progress reflecting our initial positive impressions of JNA:

* Defining a Java interface to represent the native API was relatively simple with JNA providing logical mappings to marshal between Java and native types.

* Although structured data is not used much in GLFW using JNA structures was simple and straight-forward.

* JNA also supports callbacks specified as Java interfaces.

On a high we stripped LWJGL from the JOVE project and replaced it with hand-crafted JNA interfaces, enumerations and structures.  We progressed to the point of instantiating the logical device in the space of an hour or so without any of the road-blocks or surprises that LWJGL threw at us, despite the overhead of developing the API and structures as we went.

In particular:

* There are no mysterious management methods and marshalling to/from the native layer is largely transparent - the application name is simply a string.

* Other than the fact that JNA mandates that all structure fields are public the internal workings are generally hidden.

* Where we did come across problems or confusing situations there was plenty of documentation, examples and tutorials.

At this point we paused to take stock because of course JNA was only the technical half of the solution.  Vulkan is a complex API with a large number of enumerations and structures, some of which are also absolutely huge such as the `VkStructureType` enumeration or the `VkPhysicalDeviceLimits` structure.  We had shown that hand-crafting a fraction of the API could be done, but it would be very tedious and highly error-prone to attempt to do the same for the entire Vulkan library.

We needed a code generator.

---

## Generation Game

### Overview

Having decided that JNA was the way forward some mechanism was needed to actually generate the API from the Vulkan header file.

First some requirements and constraints were established for the scope of the generator:

1. The process will be considered complete once an acceptable proportion of the API has been generated, rather than attempting to cover every possible use-case.  i.e. avoid diminishing returns on the time and effort to cover every edge case.

2. That said the generated code will be treated as read-only, we will attempt to avoid fiddling the generated source code where possible.

3. The generator will be invoked manually rather than being a part of an automated build process, making things considerably simpler.

4. For future versions of the Vulkan API we assume Khronos will take the same approach as OpenGL, whereby new iterations of the API are extensions and additions rather than replacements for existing components, i.e. each release can be code generated separately.

5. Any tools and libraries should follow the general goal of being well-documented and supported.

We had already identified the [JNAeator](https://github.com/nativelibs4java/jnaerator) tool that generates JNA bindings from a native library, apparently perfect for our requirements.  Unfortunately this tool produced a seemingly random package structure with the generated code looking more like the nasty SWIG bindings than the nice, neat code we had hand-crafted.  It also appeared to be inactive and the fact that it used other libraries that no longer existed was not encouraging.

### CDT

Further research largely drew a blank, the only option seemed to be an obscure Eclipse component called CDT used for code assist in the IDE.  This is not an actual library as such, there is no Maven or project page for example, we had to include a couple of JAR files directly into the generator project.

CDT builds an AST (Abstract Source Tree) from a C/C++ source or header file, which is a node-tree representing the various elements of the code.  In the main class for the code generator header file is first loaded:

```java
FileContent content = FileContent.createForExternalFileLocation(args[0]);
```

Where the file location argument points to the `vulkan_core.h` header.

Next the AST is generated from the source file:

```java
IScannerInfo info = new ScannerInfo(new HashMap<>(), new String[0]);
IncludeFileContentProvider emptyIncludes = IncludeFileContentProvider.getEmptyFilesProvider();
IIndex index = EmptyCIndex.INSTANCE;
int options = ILanguage.OPTION_IS_SOURCE_UNIT;
IParserLogService log = new DefaultLogService();
IASTTranslationUnit unit = GPPLanguage.getDefault().getASTTranslationUnit(content, info, emptyIncludes, index, options, log);
```

Finally the AST is processed by a visitor to extract the relevant information:

```java
ASTVisitor visitor = new ASTVisitor() {
    ...
}
unit.accept(visitor);
```

We did eventually manage to use CDT to extract the information required for the code generator but the process was very painful:

* CDT is not a public library so the documentation was virtually non-existent.

* Most of the terminology and logic means little to someone who is not a compiler expert, attempting to extract the relevant information from the AST was a process of blind searching across mysteriously named fields and types.

* The AST also seems to require a lot of casting between different types of node that are all very similar but not quite the same.

The visitor processes the nodes representing enumerations and structures and hands off to the relevant helper method:

```java
public class HeaderVisitor extends ASTVisitor {
    public HeaderVisitor() {
        this.shouldVisitDeclSpecifiers = true;
    }

    @Override
    public int visit(IASTDeclSpecifier spec) {
        if(spec instanceof IASTEnumerationSpecifier enumeration) {
            parse(enumeration);
        }
        else
        if(spec instanceof IASTCompositeTypeSpecifier structure) {
            parse(structure);
        }

        return PROCESS_CONTINUE;
    }
}
```

Note that a __public__ class member has to be set in addition to overloading the relevant visitor method(s) to select the AST nodes to be visited - WTF!

### Enumerations

Enumerations are tackled first as these are the simplest of the components to be generated from the AST:

```java
private void parse(IASTEnumerationSpecifier enumeration) {
    String name = enumeration.getName().toString();

    Map<String, Long> values = Arrays
        .stream(enumeration.getEnumerators())
        .map(CPPASTEnumerator.class::cast)
        .map(HeaderVisitor::map)
        .collect(toMap(Entry::getKey, Entry::getValue, Long::sum, LinkedHashMap::new));

    ...
}
```

Each enumeration constant is a map entry comprising the name and value:

```java
private static Entry<String, Long> map(CPPASTEnumerator entry) {
    String name = entry.getName().toString();
    Long value = entry.getIntegralValue().numericalValue();
    return Map.entry(name, value);
}
```

Notes:

* The map of enumeration values is linked to retain the order of the entries.

* Enumeration values are represented here as a `long` value, which might seem surprising since C/C++ enumerations are sized to a native `int` (or shorter).  This reason for this will become clear shortly.

Finally the parser delegates to a processor:

```java
private void parse(IASTEnumerationSpecifier enumeration) {
    ...
    proc.enumeration(name, values);
}
```

The _processor_ is responsible for pre-processing the parsed enumeration before handing off to another component to generate the Java source file:

```java
public class Processor {
    private final Generator generator;

    public Generator(Generator generator) {
        this.generator = generator;
    }

    public void enumeration(String name, Map<String, Long> values) {
        ...
    }
}
```

The Vulkan enumeration constant names are often extremely long since (with a couple of exceptions) _every_ value is prefixed with the enumeration name.  This is because a native `typedef` is essentially a synonym rather than an actual 'type', hence each constant has to be globally unique.  This is obviously not the case for a Java enumeration where the constants only need to be distinct within that type, therefore we build a _prefix_ so that the names can be truncated:

```java
public void enumeration(String name, Map<String, Long> values) {
    String[] parts = StringUtils.splitByCharacterTypeCamelCase(name);
    String prefix = String.join(UNDERSCORE, parts).toUpperCase() + UNDERSCORE;
    ...
}
```

For example `VK_COMMAND_BUFFER_RESET_RELEASE_RESOURCES_BIT` becomes the much more manageable `RELEASE_RESOURCES`.

The generator then replaces the key names of the enumeration values as follows:

```java
LinkedHashMap<String, Object> transformed = new LinkedHashMap<>();
for(Entry<String, Long> entry : values.entrySet()) {
    // Transform key
    String key = StringUtils
        .removeStart(entry.getKey(), prefix)
        .transform(str -> StringUtils.removeStart(str, "VK_"))
        .transform(str -> StringUtils.remove(str, "_BIT"))
        .transform(Generator::ensureStartsAlpha);

    // Skip synthetic constants
    if(STRIP.contains(key)) {
        continue;
    }

    // Add value
    transformed.put(key, entry.getValue());
}
```

The second transform step strips names that begin with the `VK` prefix, e.g. for `VkResult` which does not follow the pattern of the other enumerations.

Most of the Vulkan enumerations also contain some _synthetic_ constants (e.g. `BEGIN_RANGE`) that are presumably used by the native layer but are irrelevant for our purposes, these are removed by the `STRIP` test.

The final transformation step handles an edge case for constants that would end up starting with a numeric, which of course would be an invalid Java name:

```java
private static String ensureStartsAlpha(String key) {
    char ch = key.charAt(0);
    if(Character.isDigit(ch)) {
        StringBuilder str = new StringBuilder();
        String digits = DIGITS[ch - '1'];
        str.append(digits);
        str.append(UNDERSCORE);
        str.append(key.substring(1));
        return str.toString();
    }
    else {
        return key;
    }
}
```

This case only applies for the enumerations that specify the number of image or view dimensions:

```java
private static final String[] DIGITS = {"ONE", "TWO", "THREE"};
```

The leading numeric is replaced by the corresponding token, for example the `VK_IMAGE_TYPE_2D` constant becomes `TWO_D`.

### Templates

To generate the source code we selected [Apache Velocity](https://velocity.apache.org/), an old but active template library ideal for this scenario, in particular providing support for collections.

The _generator_ is a wrapper for the Velocity engine:

```java
public class Generator {
    private final VelocityEngine engine = new VelocityEngine();

    public String generate(String name, Map<String, ? extends Object> data) {
        ...
    }
}
```

The engine is configured in the constructor:

```java
public Generator() {
    Properties props = new Properties();
    props.setProperty("resource.loader", "file");
    props.setProperty("file.resource.loader.class", "org.apache.velocity.runtime.resource.loader.FileResourceLoader");
    props.setProperty("file.resource.loader.path", "src/main/resources");
    props.setProperty("file.resource.loader.cache", "false");
    engine.init(props);
}
```

The `generate` method loads the template and injects a map of arguments:

```java
public String generate(String name, Map<String, ? extends Object> data) {
    // Load template
    Template template = engine.getTemplate(name);

    // Init context
    VelocityContext ctx = new VelocityContext(new HashMap<>(data));

    // Generate source
    StringWriter out = new StringWriter();
    template.merge(ctx, out);

    return out.toString();
}
```

In the pre-processor the template arguments for an enumeration are constructed as follows:

```java
Map<String, Object> map = new HashMap<>();
map.put("package", "org.sarge.jove.platform.vulkan");
map.put("name", name);
map.put("values", transformed);
```

And finally the generator applies the template to generate the source file:

```java
String source = proc.generate("enumeration.template.txt", map);
```

This is the Velocity template for an enumeration:

```java
package $package;

import org.sarge.jove.util.IntEnum;

public enum $name implements IntEnum {
    #foreach($entry in $values.entrySet())
        ${entry.key}($entry.value)#if($foreach.hasNext),#else;#end
    #end

    private final int value;
    
    private $name(int value) {
        this.value = value;
    }

    @Override
    public int value() {
        return value;
    }
}
```

Where:

* The tokens prefixed by the dollar character are the injected arguments.

* The various tokens prefixed by the hash character are Velocity directives whose purpose should be fairly self-evident.

* The purpose of the `IntEnum` is explained in the next chapter.

The line that actually generates a enumeration constant might be slightly confusing at first glance due to white-space constraints.  The following fragment expands the `if..else..end` directive to illustrate the logic, adding a comma between each constant and a semi-colon after the final value:

```java
#if($foreach.hasNext)
    ,
#else
    ;
#end
```

Using the generator an enumeration in the Vulkan header file such as:

```c
typedef enum VkImageUsageFlagBits {
    VK_IMAGE_USAGE_TRANSFER_SRC_BIT = 0x00000001,
    VK_IMAGE_USAGE_TRANSFER_DST_BIT = 0x00000002,
    VK_IMAGE_USAGE_SAMPLED_BIT = 0x00000004,
    VK_IMAGE_USAGE_STORAGE_BIT = 0x00000008,
    VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT = 0x00000010,
    VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT = 0x00000020,
    VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT = 0x00000040,
    VK_IMAGE_USAGE_INPUT_ATTACHMENT_BIT = 0x00000080,
    VK_IMAGE_USAGE_SHADING_RATE_IMAGE_BIT_NV = 0x00000100,
    VK_IMAGE_USAGE_FRAGMENT_DENSITY_MAP_BIT_EXT = 0x00000200,
    VK_IMAGE_USAGE_FLAG_BITS_MAX_ENUM = 0x7FFFFFFF
} VkImageUsageFlagBits;
```

Becomes the following Java enumeration:

```java
public enum VkImageUsageFlag implements IntEnum {
    TRANSFER_SRC(1),
    TRANSFER_DST(2),
    SAMPLED(4),
    STORAGE(8),
    COLOR_ATTACHMENT(16),     
    DEPTH_STENCIL_ATTACHMENT(32),
    TRANSIENT_ATTACHMENT(64),
    INPUT_ATTACHMENT(128),
    SHADING_RATE_IMAGE_NV(256),
    FRAGMENT_DENSITY_MAP_EXT(512);

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

Note that although the constants are represented as _long_ values in the AST the enumeration treats the values as _unsigned_ integers, i.e. some constants may have a value larger than integer `MAX_VALUE` and are thus represented as negative integer values.

### Structures

A simple POJO is defined for each field of a structure:

```java
public class StructureField {
    private final String name;
    private final String type;
    private final String path;
    private final int pointers;
    private final int array;
}
```

Where:
* _name_ is the field name.
* _type_ specifies the type of the field.
* _path_ is the Java package for the type (see below).
* _pointers_ is the number of pointers: either none, one (for a pointer), or two (pointer-to-pointer).
* and _array_ is the length for array types (or zero).

Structures are parsed by from the AST as follows:

```java
private void parse(IASTCompositeTypeSpecifier structure) {
    String name = structure.getName().toString();

    List<StructureField> fields = Arrays
        .stream(structure.getChildren())
        .skip(1)
        .map(IASTSimpleDeclaration.class::cast)
        .map(HeaderVisitor::field)
        .toList();

    generator.structure(name, fields);
}
```

Next the properties of each field are extracted and wrapped by the transient record:

```java
private static StructureField field(IASTSimpleDeclaration field) {
    // Extract structure name
    CPPASTDeclarator declarator = (CPPASTDeclarator) field.getDeclarators()[0];
    String name = declarator.getName().toString();

    // Extract field type
    String type = type(field.getDeclSpecifier()).replace("const", "").trim();

    // Determine number of pointers
    int pointers = (declarator.getPointerOperators() == null) ? 0 : declarator.getPointerOperators().length;

    // Determine array size
    int len = length(declarator);

    // Create field
    return new StructureField(name, type, pointers, len);
}
```

Extracting the type name was particularly messy:

```java
private static String type(IASTDeclSpecifier spec) {
    if(spec instanceof CPPASTNamedTypeSpecifier named) {
        return named.getName().toString();
    }
    else
    if(spec instanceof CPPASTSimpleDeclSpecifier simple) {
        return simple.getRawSignature();
    }
    else
    if(spec instanceof CPPASTElaboratedTypeSpecifier elaborated) {
        return elaborated.getName().toString();
    }
    else {
        throw new UnsupportedOperationException(spec.getClass().getName());
    }
}
```

The length of an array type is determined by the following helper:

```java
private static int length(CPPASTDeclarator declarator) {
    if(declarator instanceof CPPASTArrayDeclarator array) {
        return Integer.parseInt(array.getArrayModifiers()[0].getConstantExpression().toString());
    }
    else {
        return 0;
    }
}
```

The structure generator populates the argument map and invokes the template processor:

```java
public void structure(String name, List<StructureField> fields) {
    Map<String, Object> map = new HashMap<>();
    map.put("package", PACKAGE);
    map.put("imports", imports);
    map.put("name", name);
    map.put("fields", fields);
    proc.generate("structure.template.txt", map);
}
```

The `imports` argument is a list of the required imports for non-primitive structure fields:

```java
List<String> imports = fields
    .stream()
    .map(StructureField::getPath)
    .filter(Objects::nonNull)
    .distinct()
    .toList();
```

All top-level Vulkan structures have an `sType` field that identifies the type of the structure to the native layer.  Since the values in `VkStructureType` are highly logical and regular the name of the enumeration constant can be generated from the structure and injected into the template:

```java
boolean top = fields.stream().map(StructureField::getName).anyMatch("sType"::equals);
if(top) {
    String[] tokens = StringUtils.splitByCharacterTypeCamelCase(name);
    String prefix = String.join(UNDERSCORE, tokens).toUpperCase();
    String value = StringUtils.removeStart(prefix, "VK_");
    map.put("sType", value);
}
```

For example the type of the `VkApplicationInfo` structure is set to `APPLICATION_INFO`.  This saves the effort of having to manually populate this field when using the structure, a nice little bonus.

The template for a Vulkan structure is slightly more complicated than the enumerations:

```java
package $package;

import org.sarge.jove.platform.vulkan.core.VulkanStructure;
#foreach($import in $imports)
import $import;
#end
import com.sun.jna.Structure;
import com.sun.jna.Structure.FieldOrder;

/**
 * Vulkan structure.
 * This class has been code-generated.
 */
@FieldOrder({
#foreach($field in $fields)
    "$field.name"#if($foreach.hasNext),
#end
#end

})
public class $name extends VulkanStructure {
#foreach($field in $fields)
#if($field.name == "sType")
    public $field.type $field.name = VkStructureType.${sType};
#else
#if($field.array == 0)
    public $field.type $field.name;
#else
    public ${field.type}[] $field.name = new ${field.type}[${field.array}];
#end
#end
#end
}
```

Notes:

* All the generated structures are derived from the `VulkanStructure` base-class which is detailed in the next chapter.

* All fields in a JNA structure must also be declared in the `@FieldOrder` annotation.

* The `foreach` loop for the structure fields injects the value for the `sType` special case.

* JNA mandates that all array fields are initialised so that structure memory can be sized accordingly.

### Type Mapping

The final piece of functionality required for structures is a mechanism to map a native type to the corresponding Java or JNA types.  The mapping logic is implemented as a helper method on the `StructureField` class.

The mapping first determines the Java equivalent for native pointer types:

```java
private static String map(String type, int pointers) {
    if(pointers > 0) {
        return switch(type) {
            // Assume arbitrary pointer
            case "void" -> POINTER;

            // Strings
            case "char" -> switch(pointers) {
                case 1 -> "String";
                default -> POINTER;
            };

            // Otherwise ignore
            default -> type;
        };
    }
    ...
}
```

Where an arbitrary pointer is implemented by the JNA type:

```java
public static final String POINTER = "com.sun.jna.Pointer";
```

There are special case mappings for boolean values:

```java
if(type.equals("VkBool32")) {
    return "boolean";
}
```

And bit-fields which are represented as an unsigned integer:

```java
if(type.endsWith("Flags")) {
    return INT;
}
```

Otherwise the field type is mapped to a primitive or retained as-is:

```java
return PRIMITIVES.getOrDefault(type, type);
```

Where the primitives mapping is a lookup table defined as follows:

```java
public static final String BYTE = Byte.TYPE.getName();
public static final String INT  = Integer.TYPE.getName();
public static final String LONG = Long.TYPE.getName();

private static final Map<String, String> PRIMITIVES = Map.of(
    "int8_t",       BYTE,
    "uint8_t",      BYTE,
    "int32_t",      INT,
    "uint32_t",     INT,
    "uint64_t",     LONG,
    "int64_t",      LONG,
    "size_t",       LONG
);
```

The following table summarises the type mappings:

| native type       | pointers  | mapped type           |
| -----------       | -----     | -----------           |
| void              | >= 1      | Pointer               |
| any               | 1         | Pointer               |
| char              | 0         | byte                  |
| char              | 1         | String                |
| char              | > 1       | Pointer               |
| VkBool32          | 0         | boolean               |
| *Flags            | 0         | int                   |
| uint32_t (etc)    | 0         | int                   |

The mapping is applied in the constructor of the field:

```java
public StructureField(String name, String type, int pointers, int array) {
    String mapped = map(type, pointers);
    int index = mapped.lastIndexOf('.');
    
    if(index > 0) {
        this.type = mapped.substring(index + 1);
        this.path = mapped;
    }
    else {
        this.type = mapped;
        this.path = null;
    }
}
```

Notes:

* The constructor also determines the import _path_ (i.e. the package) for non-primitive types.

* The structure field is a POJO with old-school getters which is required by the Velocity template engine.

---

## Conclusion

Each generated source file is dumped out to a target folder:

```java
Path path = root.resolve(name + ".java");
Files.writeString(path, source);
```

Which is specified by an application argument:

```java
Path root = Paths.get(args[1]);
```

Note that __all__ the generated components reside in a single package in a separate source folder.

In the end we decided not to code generate the API methods for a variety of reasons:

* Unlike enumerations and structures which conform to a (relatively) simple set of 'rules', many of the Vulkan API methods have unique and complex behaviour.

* We also anticipate that there will be a degree of experimentation to determine the best JNA mapping approach for each API method.  Preferably this would be tackled as those methods are iteratively introduced to JOVE, rather than attempting to understand and implement the entire API up-front.

* Ideally related API methods will be grouped to break up the overall Vulkan API and to co-locate methods with the associated domain objects.  Obviously the C-based header has no notion of packaging (although the methods are grouped logically) so this would have to a manual process anyway.

* Finally the number of API methods is relatively small in comparison to the number of enumerations and structures.

Therefore there seems little point in attempting to develop code-generation logic that only applies to a subset of cases (and often only a single method parameter).

The generator ran in a matter of milliseconds so the code could be iteratively modified until an acceptable level of results was achieved.  As it turned out there were only two structures that did not automatically compile, and since these were for an extension we had never heard they were simply deleted.

At the time of writing (for Vulkan version 1.1.101.0) the generator produced 390 structures and 142 enumerations.  The API consisted of 91 methods (excluding extensions) so the decision to implement methods manually was not particularly onerous.

In retrospect far too much time was spent messing around with CDT, and it certainly does not adhere to the goal of using well documented third-party tools.  We probably ought to have tried to implement a custom header parser (our requirements are relatively simple) and this may be something to consider for future versions of the Vulkan API.
