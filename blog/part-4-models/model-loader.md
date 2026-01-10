---
title: Model Loader
---

---

## Contents

- [Overview](#overview)
- [Model Loader](#model-loader)
- [Indexed Models](#indexed-models)
- [Summary](#summary)

---

## Overview

In this chapter we will create an OBJ loader to construct a JOVE representation of the chalet model used in the tutorial.

We will then implement some improvements to reduce the memory footprint of the resultant model and loading times.

> In the real world we would use a more modern format that supported animation, skeletons, etc. but OBJ is relatively simple to implement and is used in the Vulkan tutorial.

---

## Model Loader

### File Format

An OBJ model is specified by a text-based data format consisting of a series of _commands_ that define a static model.  Each line starts with a _command token_ followed by a number of space-delimited arguments.

The most common commands are:

| command       | arguments     | purpose               | example           |
| -------       | ---------     | -------               | -------           |
| v             | x y x         | vertex position       | v 0.1 0.2 0.3     |
| vn            | x y z         | normal                | vn 0.4 0.5 0.6    |
| vt            | u v           | texture coordinate    | vt 0.7 0.8        |
| f             | see below     | face or triangle      | f 1//3 4//6 7//9  |
| o             | name          | object                | o head            |
| g             | name          | polygon group         | g body            |
| s             | flag (int)    | smoothing group       | s 1               |

The slightly more complicated _face_ command specifies the vertices of a polygon as a tuple of indices delimited by the slash character:

| example                   | polygon components                        |
| -------                   | ------------------                        |
| f 1 2 3                   | vertex only                                |
| f 1/2 3/4 5/6             | vertex and texture coordinates            |
| f 1//2 3//4 5//6          | vertex and normal                            |
| f 1/2/3 4/5/6 7/8/9       | vertex, texture coordinate, and normal    |

Note that component indices start at __one__ and can be negative to index from the end of the list.

The following example specifies a simple triangle with texture coordinates:

```
o object
v 0.1 0.2 0.3
v 0.4 0.5 0.6
v 0.7 0.8 0.9
vt 0 0
vt 0 1
vt 1 0
g group
f 1/1 2/2 3/3
```

Only the minimal functionality required for the chalet model will be implemented, with the following assumptions and constraints on the scope of the loader:

* Face primitives are assumed to be triangles.

* Only the above commands will be supported, others are silently ignored.

* The OBJ format specifies material descriptors that specify texture properties (amongst other features) however the texture image will be hard-coded in the demo.

### Vertex Data

Loading an OBJ model consists of the following steps:

1. Load the vertex data (positions, normals, texture coordinates).

2. Load the faces that index into this data to generate the model vertices.

3. Build the resultant JOVE model.

We first implement an intermediate data structure to hold the vertex data:

```java
public class ObjectModel {
    private final List<Point> positions = new VertexComponentList<>();
    private final List<Vector> normals = new VertexComponentList<>();
    private final List<Coordinate2D> coordinates = new VertexComponentList<>();
}
```

The custom list implementation looks up a vertex component by index (which starts at __one__ and can be negative):

```java
class VertexComponentList<T> extends ArrayList<T> {
    @Override
    public T get(int index) {
        if(index > 0) {
            return super.get(index - 1);
        }
        else
        if(index < 0) {
            return super.get(size() + index);
        }
        else {
            throw new IndexOutOfBoundsException(...);
        }
    }
}
```

### Model Loader

The process of loading and parsing the OBJ model is:

1. Create an instance of the above transient model.

2. Load each line of the OBJ file (skipping comments and empty lines).

3. Parse each command line and update the model accordingly.

4. Generate the resultant JOVE model(s).

The class outline for the loader comprises the transient data model and OBJ command parsers:

```java
public class ObjectModelLoader {
    private final Pattern tokenize = Pattern.compile("\\s+");
    private final Map<String, Parser> parsers = new HashMap<>();
    private final ObjectModel model = new ObjectModel();
}
```

The loader applies the above logic and delegates to a local method to parse each line:

```java
public List<IndexedMesh> load(Reader input) throws IOException {
    var reader = new LineNumberReader(input);
    try(reader) {
        reader
            .lines()
            .map(ObjectModelLoader::clean)
            .map(String::trim)
            .filter(Predicate.not(String::isEmpty))
            .forEach(this::parse);
    }
    catch(RuntimeException e) {
        ...
    }

    // Build model
    return model.build();
}
```

Where the `clean` method strips comments from each line:

```java
private static String clean(String line) {
    int index = line.indexOf('#');
    if(index >= 0) {
        return line.substring(0, index);
    }
    else {
        return line;
    }
}
```

In the parse method each line is tokenized and then delegated to a _parser_ for the command:

```java
private void parse(String line) {
    // Tokenize line
    String[] parts = tokenize.split(line);

    // Lookup command parser
    Parser parser = parsers.get(parts[0]);

    // Notify unknown commands
    if(parser == null) {
        handler.accept(line);
        return;
    }

    // Delegate
    parser.parse(parts);
}
```

A parser is defined as follows:

```java
public interface Parser {
    /**
     * Parses the given command.
     * @param tokens Command tokens
     * @throws NumberFormatException if the data cannot be parsed
     */
    void parse(String[] tokens);
}
```

Unsupported commands are handed off to an error handler which by default silently consumes the line:

```java
private Consumer<String> handler = line -> {};
```

Normally we would throw an exception for an unknown command but it seems more prudent to ignore by default given the generally flakey nature of OBJ files.

The following configuration options are also provided:

* An `add` method to register command parsers.

* A setter to override the error handler as required.

* The `IGNORE` parser which can be used to explicitly ignore commands.

### Vertex Components

We note that the process of parsing the various vertex components (position, normal, texture coordinate) is the same in all cases:

1. Parse the command arguments to a floating-point array.

2. Construct the relevant domain object from this array.

3. Add the resultant object to the model.

First an _array constructor_ is implemented on the relevant vertex components, for example both points and vectors delegate to the following tuple constructor:

```java
protected Tuple(float[] array) {
    if(array.length != SIZE) {
        throw new IllegalArgumentException(...);
    }
    x = array[0];
    y = array[1];
    z = array[2];
}
```

Next the common pattern is abstracted by implementing a new general parser for vertex components:

```java
class VertexComponentParser<T extends Bufferable> implements Parser {
    private final int size;
    private final Function<float[], T> constructor;
    private final VertexComponentList<T> list;
}
```

Where:

* _T_ is the component type.

* _constructor_ is a method reference to the _array constructor_ of that type.

* And _list_ is the relevant component list.

To continue the example, the parser for a vertex position is specified as follows:

```java
new VertexComponentParser<>(Point.SIZE, Point::new, model.positions());
```

The parse method follows the steps outlined above:

```java
public void parse(String[] tokens) {
    // Parse array
    float[] array = new float[size];
    for(int n = 0; n < size; ++n) {
        array[n] = Float.parseFloat(tokens[n + 1]);
    }

    // Construct object from array
    final T value = constructor.apply(array);

    // Add to model
    list.add(value);
}
```

Finally built-in parsers are registered for the common vertex components:

```java
public ObjectModelLoader() {
    add("v",  new VertexComponentParser<>(Point.SIZE, Point::new, model.positions()));
    add("vt", new VertexComponentParser<>(2, Coordinate2D::new, model.coordinates()));
    add("vn", new VertexComponentParser<>(Vector.SIZE, Vector::new, model.normals()));
}
```

### Face Parser

The parser for the face command iterates over the face vertices, which are assumed to be triangles:

```java
public void parse(String[] args, ObjectModel model) {
    for(int n = 0; n < 3; ++n) {
        ...
    }
}
```

Each vertex is a slash-delimited tuple of indices into the vertex data:

```java
String[] components = tokens[n + 1].split("/");
if((parts.length < 1) || (parts.length > 3)) {
    throw new IllegalArgumentException();
}
```

The vertex position is mandatory:

```java
Point pos = parse(components[0], model.positions());
Vertex vertex = new Vertex(pos);
```

Where the `parse` method is a simple helper to lookup a component from the given list:

```java
private static <T> T parse(String value, VertexComponentList<T> list) {
    int index = Integer.valueOf(value);
    return list.get(index);
}
```

The optional normal is the last component:

```java
if(components.length == 3) {
    Normal normal = parse(components[2], model.normals());
    vertex.add(normal);
}
```

And the optional texture coordinate is the middle component:

```java
if((components.length > 1) && !components[1].isEmpty()) {
    Coordinate coordinate = parse(components[1], model.coordinates());
    vertex.add(coordinate);
}
```

### Model Builder

The OBJ builder constructs the JOVE model(s) from the transient data:

```java
public class ObjectModel {
    ...
    private final List<Mesh> meshes = new ArrayList<>();
    private Mesh current;
}
```

The `start` method initialises the model layout according to the component data:

```java
public void start() {
    // Ignore leading group commands
    if(positions.isEmpty()) {
        return;
    }

    // Determine model layout
    var layout = new ArrayList<>();
    layout.add(Point.LAYOUT);
    if(!normals.isEmpty()) {
        layout.add(Normal.LAYOUT);
    }
    if(!coordinates().isEmpty()) {
        layout.add(Coordinate2D.LAYOUT);
    }

    // Start model
    current = new Mesh(layout.toArray(Layout[]::new));
    meshes.add(current);
}
```

Notes:

* Some OBJ files _start_ with a group declaration, so this method ignores the case where no vertex data has been added to the current group.

* This implementation does not validate each vertex generated by the face parser since the layout is not known up-front.

To complete the loader the remaining command parsers are registered:

```java
add("f", new FaceParser());
add("o", Parser.GROUP);
add("g", Parser.GROUP);
add("s", Parser.IGNORE);
```
Where the `GROUP` parser delegates to the `start` method to begin a new group.

---

## Indexed Models

### Duplicate Removal

From the tutorial we know that the chalet model has a large number of duplicate vertices.  An obvious improvement is to de-duplicate the model before rendering and introduce an _index buffer_ to  reduce the total amount of data, at the expense of a second buffer for the index itself.

First the mesh definition is modified to include an optional index buffer:

```java
public interface Mesh {
    /**
     * @return Index buffer
     */
    Optional<MeshData> index();
}
```

Next a specialised implementation is introduced for a mesh with an index:

```java
public class IndexedMesh extends MutableMesh {
    private final List<Integer> indices = new ArrayList<>();
    private boolean restart;
    
    public int count() {
        return indices.size();
    }

    public IndexedMesh add(int index) {
        if((index < 0) || (index >= super.count())) {
            throw new IndexOutOfBoundsException();
        }
        indices.add(index);
        return this;
    }
    
    public Optional<Index> index() {
        return Optional.of(new DefaultIndex());
    }
}
```

Although unused in the current demo, an index can be _restarted_ using a special case value (as an alternative to degenerate triangles):

```java
public IndexedMesh restart() {
    indices.add(-1);
    restart = true;
    return this;
}
```

The index buffer is generated in a similar fashion to the vertex buffer:

```java
private static class DefaultIndex implements MeshData {
    public int length() {
        return indices.size() * Integer.BYTES;
    }

    public void buffer(ByteBuffer buffer) {
        for(int n : indices) {
            buffer.putInt(n);
        }
    }
}
```

For the OBJ model a second specialisation performs vertex de-duplication:

```java
class RemoveDuplicateMesh extends IndexedMesh {
    private final Map<Vertex, Integer> map = new HashMap<>();
}
```

The de-duplication process for a given vertex is:

1. Lookup the index of the vertex from the `map` table.

2. If the vertex already exists add its index to the model.

3. Otherwise add the vertex __and__ its index to the model and register it in the lookup table.

This is implemented in the overloaded `add` method as follows:

```java
public DuplicateModelBuilder add(Vertex v) {
    Integer prev = map.get(v);
    if(prev == null) {
        // Register new vertex
        Integer n = vertices.size();
        map.put(v, n);

        // Add vertex
        super.add(v);
        index.add(n);
    }
    else {
        // Otherwise add index for existing vertex
        index.add(prev);
    }
    return this;
}
```

Note that this implementation assumes that the vertex class has a decently efficient hashing implementation.

The OBJ loader is refactored to use the new mesh implementation which reduces the size of the interleaved model from 30Mb to roughly 11Mb (5Mb for the vertex data and 6Mb for the index buffer).  Nice!

### Index Buffer

A Vulkan index buffer is similar to the existing implementation but requires a different `bind` method and additionally has a data type (for short or integer indices), therefore a new buffer sub-class is introduced:

```java
public class IndexBuffer extends VulkanBuffer {
    private final VkIndexType type;

    public IndexBuffer(VulkanBuffer buffer, VkIndexType type) {
        super(buffer);
        this.type = type;
        require(VkBufferUsageFlag.INDEX_BUFFER);
    }

    public Command bind(long offset) {
        return (api, cmd) -> api.vkCmdBindIndexBuffer(cmd, this, offset, type);
    }
}
```

The vertex buffer implementation is also factored out from the base-class:

```java
public class VertexBuffer extends VulkanBuffer {
    /**
     * Constructor.
     * @param buffer Underlying buffer
     * @throws IllegalStateException if this buffer is not a {@link VkBufferUsageFlag#VERTEX_BUFFER}
     */
    public VertexBuffer(VulkanBuffer buffer) {
        super(buffer);
        require(VkBufferUsageFlag.VERTEX_BUFFER);
    }

    /**
     * Creates a command to bind this buffer as a vertex buffer (VBO).
     * @param binding Binding index
     * @return Command to bind this buffer
     */
    public Command bind(int binding) {
        ...
    }
}
```

And similarly for buffers that are used as descriptor set resources such as uniform buffers:

```java
public class ResourceBuffer extends VulkanBuffer implements DescriptorResource {
    private final VkDescriptorType type;
    private final long offset;

    public ResourceBuffer(VulkanBuffer buffer, VkDescriptorType type, long offset) {
        super(buffer);
        this.type = type;
        this.offset = offset;
        require(map(type));
    }

    @Override
    public VkDescriptorBufferInfo build() {
        ...
    }
}
```

Where `map` determines the buffer usage flag for a given type of descriptor:

```java
public static VkBufferUsageFlag map(VkDescriptorType type) {
    return switch(type) {
        case UNIFORM_BUFFER -> VkBufferUsageFlag.UNIFORM_BUFFER;
        default -> throw new IllegalArgumentException();
    };
}
```

### Index Data Type

The indices are stored as 32-bit integers in the mesh:

```java
private class IntegerIndex implements Index {
    public final int length() {
        return indices.size() * bytes();
    }

    protected int bytes() {
        return Integer.BYTES;
    }

    public void buffer(ByteBuffer buffer) {
        for(int n : indices) {
            buffer.putInt(n);
        }
    }
}
```

However Vulkan supports shorter data types to allow more compact index buffers for smaller meshes.

A new method on the mesh index determines the smallest integer type required to support the index:

```java
public final int minimumElementBytes() {
    int vertices = IndexedMesh.super.count();
    if(vertices <= MathsUtility.unsignedMaximum(Byte.SIZE)) {
        return Byte.BYTES;
    }
    else
    if(vertices <= MathsUtility.unsignedMaximum(Short.SIZE)) {
        return Short.BYTES;
    }
    else {
        return Integer.BYTES;
    }
}
```

Where the `unsignedMaximum` utility calculates the largest number of vertices that can be indexed by a given type:

```java
public static long unsignedMaximum(int bits) {
    return (1L << bits) - 1;
}
```

An application can then select a smaller index type via the following adapter:

```java
public Index index(int bytes) {
    int min = minimumElementBytes();
    if(bytes < min) {
        throw new IllegalArgumentException(...);
    }

    return switch(bytes) {
        case Byte.BYTES     -> new ByteIndex();
        case Short.BYTES    -> new ShortIndex();
        case Integer.BYTES  -> this;
        default             -> throw new IllegalArgumentException(...);
    };
}
```

Where 16-bit indices are represented by `short` values:

```java
class ShortIndex extends IntegerIndex {
    protected int bytes() {
        return Short.BYTES;
    }

    public void buffer(ByteBuffer buffer) {
        for(int n : indices) {
            buffer.putShort((short) n);
        }
    }
}
```

And similarly for an index with less than 256 vertices:

```java
class ByteIndex extends IntegerIndex {
    protected int bytes() {
        return Byte.BYTES;
    }

    public void buffer(ByteBuffer buffer) {
        for(int n : indices) {
            buffer.put((byte) n);
        }
    }
}
```

A convenience helper is added to the index buffer class to map the data type accordingly:

```java
public static VkIndexType type(int size) {
    return switch(size) {
        case Byte.BYTES     -> VkIndexType.UINT8_EXT;
        case Short.BYTES    -> VkIndexType.UINT16;
        case Integer.BYTES  -> VkIndexType.UINT32;
        default -> throw new IllegalArgumentException(...);
    };
}
```

Finally, validation is added to verify that a given index buffer is supported by the hardware:

```
private static void validate(VulkanBuffer buffer, VkIndexType type) {
    var device = buffer.device();
    switch(type) {
        case UINT16 -> {
            // A short index is always supported
        }
        ...
    }
}
```

An index comprising 8-bit values requires a specific device feature:

```java
case UINT8_EXT -> device.features().require(INDEX_TYPE_UINT8);
```

The size of a 32-bit index is limited by the device:

```java
case UINT32 -> {
    int max = device.limits().get("maxDrawIndexedIndexValue");
    if(max >= 0) {
        long size = buffer.length() / Integer.BYTES;
        if(size > max) {
            throw new IllegalStateException(...);
        }
    }
}
```

Note that the index for the chalet model is sufficiently large that the demo uses the default 32-bit indices.

### Model Persistence

Although the OBJ loader and indexed mesh are relatively efficient, the process of loading the model is now quite slow (albeit only taking a second or so).  We could attempt to optimise the code but this is usually very time-consuming and often results in complexity leading to bugs.

Instead we note that as things stand the following steps in the loading process are repeated _every_ time we run the demo:

1. Load and parse the OBJ model.

2. De-duplicate the vertex data.

3. Transform to NIO buffers.

Ideally the above steps only need be performed _once_ since the intermediate data is discarded once the resultant vertex and index buffers have been constructed.  Therefore a custom persistence mechanism is introduced to write the final model to the file-system as an off-line activity which can then be loaded with minimal overhead.

A new component outputs a mesh to a `DataOutputStream` which supports both Java primitives and byte arrays:

```java
public class MeshLoader {
    private static final int VERSION = 1;

    private static void write(Mesh mesh, DataOutputStream out) throws IOException {
    }
}
```

The `write` method first writes the version number of the custom file-format (for later verification):

```java
out.writeInt(VERSION);
```

The model header is written next:

```java
out.writeUTF(mesh.primitive().name());
out.writeInt(mesh.count());
```

Followed by the vertex layout:

```java
List<Layout> layout = mesh.layout().layout();
out.writeInt(layout.size());
for(Layout e : layout) {
    out.writeInt(e.size());
    out.writeUTF(e.type().name());
    out.writeBoolean(e.signed());
    out.writeInt(e.bytes());
}
```

Note that the _type_ of each component in the layout is written as a string using the `writeUTF` method on the stream.

Next the vertex buffer is output:

```java
writeBuffer(mesh.vertices(), out);
```

Which uses the following helper to output the length of the data followed by the buffer as a byte-array:

```java
private static void writeBuffer(MeshData obj, DataOutputStream out) throws IOException {
    // Output length
    int length = obj.length();
    out.writeInt(length);

    // Stop if empty buffer
    if(len == 0) {
        return;
    }

    // Write buffer
    ByteBuffer buffer = ByteBuffer.allocate(length).order(ByteOrder.nativeOrder());
    obj.buffer(buffer);
    out.write(buffer.array());
}
```

And finally the optional index buffer is written using the same helper.

Next a new public method is added to the loader to read back the persisted mesh:

```java
public Mesh load(DataInputStream in) throws IOException {
    ...
}
```

The loader first verifies that the file version is supported:

```java
int version = in.readInt();
if(version > VERSION) {
    throw new UnsupportedOperationException(...);
}
```

Next the model header is loaded:

```java
Primitive primitive = Primitive.valueOf(in.readUTF());
int count = in.readInt();
```

Followed by the vertex layout:

```java
int num = in.readInt();
List<Layout> layout = new ArrayList<>();
for(int n = 0; n < num; ++n) {
    int size = in.readInt();
    Layout.Type type = Layout.Type.valueOf(in.readUTF());
    boolean signed = in.readBoolean();
    int bytes = in.readInt();
    layout.add(new Layout(size, type, signed, bytes));
}
```

Next the vertex and index buffers are loaded:

```java
MeshData vertices = loadBuffer(in);
MeshData index = loadBuffer(in);
```

And finally the mesh is instantiated:

```java
return new AbstractMesh(primitive, new CompoundLayout(layout)) {
    public int count() {
        return count;
    }

    public MeshData vertices() {
        return vertices;
    }

    public Optional<MeshData> index() {
        return Optional.of(index);
    }
};
```

The `loadBuffer` helper is the inverse of `writeBuffer` above:

```java
private static MeshData loadBuffer(DataInputStream in) throws IOException {
    // Read buffer size
    int length = in.readInt();
    
    // Check for empty buffer
    if(length == 0) {
        return null;
    }

    // Load bytes
    byte[] bytes = new byte[length];
    in.readFully(bytes);

    // Convert to buffer
    return MeshData.of(bytes);
}
```

There is still a fair amount of type conversions to/from byte-arrays in this code but the buffered model can now be loaded in a matter of milliseconds.  Result.

---

## Summary

In this chapter we implemented:

- A loader for an OBJ model.

- Index buffers.

- Model persistence.
