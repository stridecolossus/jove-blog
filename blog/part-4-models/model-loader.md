---
title: Model Loader
---

---

## Contents

- [Overview](#overview)
- [Model Loader](#model-loader)
- [Indexed Models](#indexed-models)

---

## Overview

In this chapter we will create an OBJ loader to construct a JOVE representation of the chalet model used in the tutorial.

We will then implement some improvements to reduce the memory footprint of the resultant model and loading times.

> In the real world we would use a more modern format that supported animation, skeletons, etc. but the OBJ is relatively simple to implement and is used in the Vulkan tutorial.

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
| s             | flag          | smoothing group       | s 1               |

The _face_ command specifies the vertices of a polygon as a tuple of indices delimited by the slash character (the normal and texture coordinate are optional):

| example                   | position | texture | normal |
| -------                   | -------- | ------- | ------ |
| f 1 2 3                   | yes      | no      | no     |
| f 1/2 3/4 5/6             | yes      | yes     | no     |
| f 1//2 3//4 5//6          | yes      | no      | yes    |
| f 1/2/3 4/5/6 7/8/9       | yes      | yes     | yes    |

Example for a simple triangle with texture coordinates:

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
    private final List<Coordinate2D> coords = new VertexComponentList<>();
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
    private Set<String> comments = Set.of("#");
    private final Map<String, Parser> parsers = new HashMap<>();
    private final ObjectModel model = new ObjectModel();
}
```

The loader applies the above logic and delegates to a local method to parse each line:

```java
public Stream<Mesh> load(Reader r) throws IOException {
    // Parse OBJ model
    try(LineNumberReader in = new LineNumberReader(r)) {
        in.lines()
            .map(String::trim)
            .filter(Predicate.not(String::isBlank))
            .filter(Predicate.not(this::isComment))
            .forEach(this::parse);
    }
    catch(Exception e) {
        throw new IOException(...);
    }

    // Construct models
    return model.build();
}
```

The `isComment` method checks for lines that start with a comment token:

```java
private boolean isComment(String line) {
    return comments.stream().anyMatch(line::startsWith);
}
```

In the parse method each line is tokenized and then delegated to a _parser_ for the command:

```java
private void parse(String line) {
    // Tokenize line
    String[] parts = StringUtils.split(line);

    // Lookup parser
    Parser parser = parsers.get(parts[0]);
    if(parser == null) {
        handler.accept(line);
        return;
    }
    
    // Delegate
    parser.parse(parts, model);
}
```

A parser is defined as follows:

```java
public interface Parser {
    /**
     * Parses the given command.
     * @param args         Arguments (including the command token)
     * @param model        OBJ model
     * @throws NumberFormatException is the data cannot be parsed
     */
    void parse(String[] args, ObjectModel model);
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
    if(array.length != SIZE) throw new IllegalArgumentException(...);
    x = array[0];
    y = array[1];
    z = array[2];
}
```

Next the common pattern is abstracted by implementing a new general parser for vertex components:

```java
class VertexComponentParser<T extends Bufferable> implements Parser {
    private final float[] array;
    private final Function<float[], T> ctor;
    private final Function<ObjectModel, VertexComponentList<T>> mapper;
}
```

Where:

* _T_ is the component type.

* _ctor_ is a method reference to the _array constructor_ of that type.

* And _mapper_ retrieves the relevant component list.

To continue the example, the parser for a vertex position is specified as follows:

```java
new VertexComponentParser<>(Point.SIZE, Point::new, model::positions);
```

The parse method follows the steps outlined above:

```java
public void parse(String[] args, ObjectModel model) {
    // Convert to array
    for(int n = 0; n < array.length; ++n) {
        array[n] = Float.parseFloat(args[n + 1]);
    }

    // Create component
    T value = ctor.apply(array);

    // Add to model
    VertexComponentList<T> list = mapper.apply(model);
    list.add(value);
}
```

Finally built-in parsers are registered for the common vertex components:

```java
public ObjectModelLoader() {
    add("v",  new VertexComponentParser<>(Point.SIZE, Point::new, model::positions));
    add("vt", new VertexComponentParser<>(2, Coordinate2D::new, model::coordinates));
    add("vn", new VertexComponentParser<>(Vector.SIZE, Vector::new, model::normals));
}
```

### Face Parser

The parser for the face command iterates over the face vertices, which are assumed to be triangles:

```java
public void parse(String[] args, ObjectModel model) {
    for(int n = 0; n < 3; ++n) {
        String face = args[n + 1];
        ...
    }
}
```

Each vertex is a slash-delimited tuple of indices into the vertex data:

```java
String[] parts = StringUtils.splitPreserveAllTokens(face, '/');
if((parts.length == 0) || (parts.length > 3)) throw new IllegalArgumentException();
```

The vertex position is mandatory:

```java
Point pos = parse(parts[0], model.positions());
```

Where the `parse` method is a simple helper to lookup a component from the given list:

```java
private static <T> T parse(String value, VertexComponentList<T> list) {
    int index = Integer.valueOf(value);
    return list.get(index);
}
```

Next a vertex is created and added to the model:

```java
var vertex = new MutableVertex(pos);
model.add(vertex);
```

The `MutableVertex` is a custom implementation with an optional normal and texture coordinate:

```java
private static class MutableVertex extends Vertex {
    private Normal normal;
    private Coordinate2D coord;

    @Override
    public void buffer(ByteBuffer bb) {
        super.buffer(bb);
        if(normal != null) {
            normal.buffer(bb);
        }
        if(coord != null) {
            coord.buffer(bb);
        }
    }
}
```

The middle component of the tuple is the optional texture coordinate:

```java
if((parts.length > 1) && !parts[1].isEmpty()) {
    Coordinate2D coord = parse(parts[1], model.coordinates());
    vertex.coord = coord;
}
```

And the last component is the optional normal:

```java
if(parts.length == 3) {
    Normal normal = parse(parts[2], model.normals());
    vertex.normal = normal;
}
```

### Model Builder

The OBJ builder constructs the JOVE model(s) from the transient data:

```java
public class ObjectModel {
    ...
    private final List<DefaultMesh> models = new ArrayList<>();
    private DefaultMesh model;

    public List<DefaultMesh> build() {
        buildCurrentGroup();
        return new ArrayList<>(models);
    }
}
```

The `buildCurrentGroup` method initialises the layout of the current model and instantiates the next group:

```java
private void buildCurrentGroup() {
    // Init model layout
    var layout = new ArrayList<Layout>();
    layout.add(Point.LAYOUT);
    if(!normals.isEmpty()) {
        layout.add(Normal.LAYOUT);
    }
    if(!coords.isEmpty()) {
        layout.add(Coordinate2D.LAYOUT);
    }

    // Start new model
    model = new DefaultMesh(new CompoundLayout(layout));
    models.add(model);
}
```

Although not required for the current demo application the OBJ format supports construction of multiple models from a single file.  The following method builds the JOVE model for the current group and then resets the transient OBJ model:

```java
public void start() {
    // Ignore if current group is empty
    if(current.isEmpty()) {
        return;
    }

    // Build current model group
    buildCurrentGroup();

    // Reset transient model
    positions.clear();
    normals.clear();
    coords.clear();
}
```

Notes:

* Some OBJ files _start_ with a group declaration, so the `start` method ignores the case where no vertex data has been added to the current group.

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
    Optional<ByteSizedBufferable> index();
}
```

Next a specialised implementation is introduced for a mesh with an index:

```java
public class IndexedMesh extends DefaultMesh {
    private final List<Integer> index = new ArrayList<>();
    
    @Override
    public final int count() {
        return index.size();
    }

    public IndexedMesh add(int index) {
        if((index < 0) || (index >= super.count())) throw new IndexOutOfBoundsException();
        this.index.add(index);
        return this;
    }
}
```

The index buffer is generated in a similar fashion to the vertex buffer:

```java
private class IndexBuffer implements ByteSizedBufferable {
    @Override
    public int length() {
        return index.size() * Integer.BYTES;
    }

    @Override
    public void buffer(ByteBuffer bb) {
        if(bb.isDirect()) {
            for(int n : index) {
                bb.putInt(n);
            }
        }
        else {
            int[] indices = index.stream().mapToInt(Integer::intValue).toArray();
            bb.asIntBuffer().put(indices);
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
        this.type = notNull(type);
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

Similarly for buffers that are used as descriptor set resources (such as uniform buffers):

```java
public class ResourceBuffer extends VulkanBuffer implements DescriptorResource {
    private final VkDescriptorType type;
    private final long offset;

    public ResourceBuffer(VulkanBuffer buffer, VkDescriptorType type, long offset) {
        super(buffer);
        this.type = notNull(type);
        this.offset = zeroOrMore(offset);
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

### Model Persistence

Although the OBJ loader and indexed mesh are relatively efficient, the process of loading the model is now quite slow.  We could attempt to optimise the code but this is usually very time-consuming and often results in complexity, leading to bugs and code that is difficult to maintain.

Instead we note that as things stand the following steps in the loading process are repeated _every_ time we run the demo:

1. Load and parse the OBJ model.

2. De-duplication of the vertex data.

3. Transformation to NIO buffers.

Ideally the above steps would only be performed _once_ since we are only really interested in the resultant vertex and index bufferable objects.  Therefore a custom persistence mechanism is introduced to write the final model to the file-system _once_ as an off-line activity which can then be loaded with minimal overhead.

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
private static void writeBuffer(ByteSizedBufferable obj, DataOutputStream out) throws IOException {
    // Output length
    int len = obj.length();
    out.writeInt(len);

    // Stop if empty buffer
    if(len == 0) {
        return;
    }

    // Write buffer
    ByteBuffer bb = ByteBuffer.allocate(len).order(ByteOrder.nativeOrder());
    obj.buffer(bb);
    out.write(bb.array());
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
ByteSizedBufferable vertices = loadBuffer(in);
ByteSizedBufferable index = loadBuffer(in);
```

And finally the mesh is instantiated:

```java
return new AbstractMesh(primitive, new CompoundLayout(layout)) {
    @Override
    public int count() {
        return count;
    }

    @Override
    public ByteSizedBufferable vertices() {
        return vertices;
    }

    @Override
    public Optional<ByteSizedBufferable> index() {
        return Optional.of(index);
    }
};
```

The `loadBuffer` helper is the inverse of `writeBuffer` above (with an additional check for an empty buffer):

```java
private static ByteSizedBufferable loadBuffer(DataInputStream in) throws IOException {
    // Read buffer size
    int len = in.readInt();
    
    // Check for empty buffer
    if(len == 0) {
        return null;
    }

    // Load bytes
    byte[] bytes = new byte[len];
    in.readFully(bytes);

    // Convert to buffer
    return ByteSizedBufferable.of(bytes);
}
```

There is still a fair amount of type conversions to/from byte-arrays in this code but the buffered model can now be loaded in a matter of milliseconds.  Result.

### Loader Support

Over the course of this project we have implemented various loaders that share a common pattern but with differing data types:

| implementation        | input type        | resource type     |
| --------------        | ----------        | -------------     |
| Shader.Loader         | InputStream       | Shader            |
| ImageData.Loader      | BufferedImage     | ImageData         |
| ObjectModelLoader     | Reader            | Stream<Model>     |
| ModelLoader           | DataInputStream   | BufferedModel     |

Currently the demo applications are required to implement fiddly I/O code to open the various resources and handle cleanup and checked exceptions.  Ideally we would like to abstract this common pattern by referring to resources by _name_ and encapsulate the nasty code.

We first implement the following abstraction for a general resource loader:

```java
public interface ResourceLoader<T, R> {
    /**
     * Maps an input stream to the intermediate data type.
     * @param in Input stream
     * @return Intermediate data type
     * @throws IOException if the input data cannot be loaded
     */
    T map(InputStream in) throws IOException;

    /**
     * Constructs the resultant resource from the given data.
     * @param data Input data
     * @return Loaded resource
     * @throws IOException if the resource cannot be loaded
     */
    R load(T data) throws IOException;
}
```

Where _R_ is the type of the resource and _T_ is some arbitrary intermediate data-type.

The purpose of this abstraction is probably best illustrated by an example for the new model loader:

```java
public class ModelLoader implements ResourceLoader<DataInputStream, BufferedModel> {
    @Override
    public DataInputStream map(InputStream in) throws IOException {
        return new DataInputStream(in);
    }
    
    @Override
    public BufferedModel load(DataInputStream in) throws IOException {
        ...
    }
}
```

The `map` method transforms an input-stream to the intermediate data stream, the `load` method itself is unchanged (except for the `@Override`).

Next we introduce a _data source_ which abstracts access to a resource:

```java
public interface DataSource {
    /**
     * Opens an input stream for the given resource.
     * @param name Resource name
     * @return Input stream
     * @throws IOException if the resource cannot be opened
     */
    InputStream input(String name) throws IOException;

    /**
     * Opens an output stream to the given resource.
     * @param name Resource name
     * @return Output stream
     * @throws IOException if the resource cannot be opened
     */
    OutputStream output(String name) throws IOException;
}
```

We provide an implementation for the file-system:

```java
public class FileDataSource implements DataSource {
    private final Path root;

    public FileDataSource(Path root) {
        if(!Files.exists(root)) throw new IllegalArgumentException(...);
        this.root = notNull(root);
    }

    @Override
    public InputStream input(String name) throws IOException {
        return Files.newInputStream(root.resolve(name));
    }

    @Override
    public OutputStream output(String name) throws IOException {
        return Files.newOutputStream(root.resolve(name));
    }
}
```

And another for classpath resources (such as the shaders):

```java
public class ClasspathDataSource implements DataSource {
    @Override
    public InputStream input(String name) throws IOException {
        InputStream in = DataSource.class.getResourceAsStream(path);
        if(in == null) throw new FileNotFoundException(...);
        return in;
    }

    @Override
    public OutputStream output(String name) throws IOException {
        throw new UnsupportedOperationException();
    }
}
```

The final piece of the jigsaw is the following adapter that composes a resource loader and a data source:

```java
public class ResourceLoaderAdapter<T, R> {
    private final DataSource src;
    private final ResourceLoader<T, R> loader;

    public R load(String name) {
        try(InputStream in = src.input(name)) {
            T data = loader.map(in);
            return loader.load(data);
        }
        catch(IOException e) {
            throw new RuntimeException(...);
        }
    }
}
```

This class encapsulates the process of opening and loading the resource and nicely centralises the checked exceptions.

Notes:

* The implementation assumes that all resources will be loaded from an underlying input stream (which seems safe enough).

* This framework allows loaders to remain relatively simple (and testable) and factors out the name mapping and exception handling logic.

* Existing resource loaders are refactored accordingly.

In the demo applications we can now configure common data sources (which also avoids duplication of the data directory):

```java
@SpringBootApplication
public class RotatingCubeDemo {
    @Bean
    public static DataSource classpath() {
        return new ClasspathDataSource();
    }

    @Bean
    public static DataSource data() {
        return new FileDataSource(...);
    }
}
```

Resources can now be loaded much more concisely:

```java
public static Model model(DataSource data) {
    var loader = new ResourceLoaderAdapter<>(data, new ModelLoader());
    return loader.load("chalet.model");
}
```

---

## Summary

In this chapter we implemented:

- A loader for an OBJ model.

- Index buffers.

- Model persistence.

- An improved loader abstraction.

