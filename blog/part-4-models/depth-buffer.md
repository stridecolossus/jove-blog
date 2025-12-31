---
title: Depth Buffers
---

---

# Contents

- [Overview](#overview)
- [Camera](#camera)
- [Depth Buffer](#depth-buffer)
- [Summary](#summary)

---

# Overview

In this chapter we render the chalet model constructed previously and resolve various visual problems that arise.  This will involve the use of the _depth buffer_ and requires some refactoring of the existing rendering process.

Firstly, since the demo is now dealing with a model with a specific orientation, the current view-transform matrix will be encapsulated into a _camera_ model allowing the application to more easily configure the view.  This will be extended in the next chapter to allow the camera to be dynamically controlled by the keyboard and mouse.

---

# Camera

## Normals Redux

Currently the application code is responsible for ensuring that vectors have been normalised as appropriate.  This either requires the overhead of checking whether a vector has already been normalised (albeit a relatively trivial test) or relying on documentation hints, neither of which are rigorous or explicit.

A better approach is to _enforce_ this requirement at compile-time, therefore the `Vector` class is extended by the introduction of specialisations for normals and the cardinal axes.

A _normal_ is a unit-vector:

```java
public class Normal extends Vector {
    public static final Layout LAYOUT = new Layout(SIZE, Type.NORMALIZED, true, Float.BYTES);

    public Normal(Vector vec) {
        super(normalize(vec));
    }
    
    public final float magnitude() {
        return 1;
    }
    
    public Normal invert() {
        return new Normal(super.invert());
    }
    
    public final Normal normalize() {
        return this;
    }
}
```

This new type is further specialised for the cardinal axes:

```java
public final class Axis extends Normal {
    public static final Axis
        X = new Axis(0),
        Y = new Axis(1),
        Z = new Axis(2);

    private final int index;
}
```

The vector of each axis is initialised in the constructor:

```java
private Axis(int ordinal) {
    var array = new float[Vector.SIZE];
    array[ordinal] = 1;

    Vector vector = new Vector(array);
    super(vector);
    this.ordinal = ordinal;
    this.invert = new Normal(vector.invert());
}
```

And finally the code to construct a rotation matrix about one of the cardinal axes is moved from the matrix class:

```java
public Matrix rotation(float angle) {
    var matrix = new Matrix.Builder().identity();
    float sin = MathsUtility.sin(angle);
    float cos = MathsUtility.cos(angle);
    switch(index) {
        case 0 -> matrix.set(...);
        ...
    }
    return matrix.build();
}
```

The intent of code using the new types is now more expressive and type-safe, with less reliance on documented assumptions or defensive checks.

## Camera Model

The camera is a model class representing the position and orientation of the viewer:

```java
public class Camera {
    private Point position = Point.ORIGIN;
    private Normal direction = Axis.Z;
    private Normal up = Axis.Y;
}
```

Note that under the hood the camera direction is the inverse of the view direction, i.e. the camera points _out_ of the screen whereas the view is obviously _into_ the screen.

Mutators are provided to reposition the camera:

```java
public void move(Point position) {
    this.position = position;
}

public void move(Vector vector) {
    position = position.add(vector);
}

public void move(float distance) {
    move(direction.multiply(distance));
}
```

Next the following transient members are added to the camera class to support view transform:

```java
public class Camera {
    ...
    private Normal right;
    private Normal y;
    private Matrix matrix;
}
```

Where:

* `right` is the local horizontal axis of the camera (also used to strafe the camera).

* and `y` is the vertical axis (or the _actual_ up direction).

The camera can be pointed in a given direction:

```java
public Camera direction(Normal direction) {
    this.direction = direction;
    update();
    return this;
}
```

The `update` method recalculates the cameras local axes:

```java
private void update() {
    right = new Normal(up.cross(direction));
    y = new Normal(direction.cross(right));
    matrix = null;
}
```

Where the _cross product_ yields the vector perpendicular to two other vectors using the right-hand rule:

```java
public Vector cross(Vector vec) {
    float x = this.y * vec.z - this.z * vec.y;
    float y = this.z * vec.x - this.x * vec.z;
    float z = this.x * vec.y - this.y * vec.x;
    return new Vector(x, y, z);
}
```

The following convenience method points the camera at a given target:

```java
public void look(Point target) {
    if(pos.equals(target)) {
        throw new IllegalArgumentException(...);
    }
    Vector look = Vector.between(target, position);
    direction(new Normal(look));
}
```

Where `between` is a new factory method on the vector class:

```java
public static Vector between(Point start, Point end) {
    float dx = end.x - start.x;
    float dy = end.y - start.y;
    float dz = end.z - start.z;
    return new Vector(dx, dy, dz);
}
```

Note that this camera model is subject to _gimbal locking_ if the direction becomes the same as the _up_ axis.

The view transform `matrix` is cleared in the various mutator methods (not shown) when any of the camera properties are modified and is generated on demand:

```java
public Matrix matrix() {
    if(matrix == null) {
        build();
    }
    return matrix;
}
```

The matrix is constructed from the translation and rotation components as before:

```java
Vector translation = new Vector(
    right.dot(position),
    y.dot(position),
    direction.dot(position)
);

matrix = new Matrix.Builder()
    .row(0, right)
    .row(1, y)
    .row(2, direction)
    .column(3, translation.invert())
    .set(3, 3, 1)
    .build();
```

The camera matrix is the _inverse_ of the view transformation and is also ordered _column major_ as expected by Vulkan.

However a matrix comprised solely of rotational and translation components (like this one) is also the _transpose_ of the rotation combined with the translation.  Therefore the camera axes (the rotation component) are arranged as _rows_ and the translation component (the camera position) is populated without requiring an intermediate matrix multiplication.

## Draw Command

The draw command is different for an indexed model, therefore we take the opportunity to implement a proper abstraction:

```java
public record DrawCommand(...) implements Command {
    /**
     * Constructor.
     * @param vertexCount           Number of vertices
     * @param instanceCount         Number of instances
     * @param firstVertex           First vertex
     * @param firstInstance         First instance
     * @param firstIndex            Optional starting index
     * @param library               Drawing library
     */
    public DrawCommand {
        ...
    }
}
```

The new class selects the appropriate command variant:

```java
public void execute(Buffer buffer) {
    if(firstIndex == null) {
        library.vkCmdDraw(buffer, vertexCount, instanceCount, firstVertex, firstInstance);
    }
    else {
        library.vkCmdDrawIndexed(buffer, vertexCount, instanceCount, firstIndex, firstVertex, firstInstance);
    }
}
```

A companion builder is also implemented along with convenience factory methods for common use-cases:

```java
static DrawCommand draw(int count) {
    return new Builder().count(count).build();
}

static DrawCommand indexed(int count) {
    return new Builder().indexed().count(count).build();
}
```

Finally a further helper is implemented to create a draw command for a given mesh:

```java
public static DrawCommand of(Mesh mesh, LogicalDevice device) {
    var draw = new Builder();
    draw.vertexCount(mesh.count());

    if(mesh.index().isPresent()) {
        draw.indexed();
    }

    return draw.build(device);
}
```

The hard-coded draw command can now be replaced in the render sequence:

```java
Command draw = DrawCommand.of(mesh);
```

## Integration #1

A new `ModelDemo` project is started based on the previous rotating cube demo.

The existing view-transform code is replaced with a camera:

```java
class CameraSetup {
    @Bean
    static Camera camera() {
        return new Camera();
    }
}
```

The previous VBO configuration is replaced with a new class that loads the persisted mesh:

```java
@Configuration
class Model {
    @Bean
    static Mesh load() throws IOException {
        Path path = Path.of("chalet.model");
        var loader = new MeshLoader();
        ...
        return loader.load(path);
    }
}
```

To assist debugging the `load` method regenerates the model if the file is not present:

```java
if(!path.toFile().exists()) {
    Mesh mesh = create();
    loader.write(mesh, path);
}
```

Where `create` loads and builds the OBJ model:

```java
private static Mesh create() throws IOException {
    var path = Path.of("chalet.obj");
    var loader = new ObjectModelLoader();
    try(var in = Files.newBufferedReader(path)) {
        return loader.load(in).getFirst();
    }
}
```

The VBO and index buffers are created for the mesh:

```java
@Bean
static VertexBuffer vbo(Mesh mesh) {
    VulkanBuffer buffer = buffer(mesh.vertices(), VkBufferUsage.VERTEX_BUFFER);
    return new VertexBuffer(buffer);
}

@Bean
static IndexBuffer index(Mesh mesh) {
    VulkanBuffer buffer = buffer(mesh.index().get(), VkBufferUsage.INDEX_BUFFER);
    return new IndexBuffer(buffer, VkIndexType.UINT32);
}
```

Which both delegate to the following helper:

```java
private VulkanBuffer buffer(Allocator allocator, MeshData data, VkBufferUsage usage) {
    // Create staging buffer
    VulkanBuffer staging = VulkanBuffer.staging(allocator, data);

    // Init buffer memory properties
    var properties = new MemoryProperties.Builder<VkBufferUsage>()
        .usage(VkBufferUsage.TRANSFER_DST)
        .usage(usage)
        .required(VkMemoryProperty.DEVICE_LOCAL)
        .build();

    // Create buffer
    var buffer = VulkanBuffer.create(allocator, staging.length(), properties);

    // Copy staging to buffer
    staging.copy(buffer).submit(graphics);

    // Release staging
    staging.destroy();

    return buffer;
}
```

And finally the index is bound in the render configuration:

```java
@Bean("index.bind")
static Command index(IndexBuffer index) {
    return index.bind(0);
}
```

The chalet model is orientated with the viewer looking down from above, therefore a _model_ transformation is added:

```java
@Bean
static Matrix modelmatrix() {
    Matrix tilt = new AxisAngle(Axis.X, toRadians(-90)).matrix();
    Matrix rotation = new AxisAngle(Axis.Y, toRadians(120)).matrix();
    Matrix down = Matrix.translation(new Vector(0, -0.25f, 0));
    return down.multiply(rotation.multiply(tilt));
}
```

Where:

* The _tilt_ orientates the model so the view is from the side.

* The _rotation_ spins the model vertically such that the camera is facing the corner of the chalet with the door.

* And _down_ essentially moves the camera slightly above the 'ground' level.

The model matrix is combined with the camera and projection matrices and written to the shader:

```java
@Bean
static Resource uniform(VulkanBuffer uniformBuffer, Matrix projection, Camera camera, Matrix modelmatrix) {
    var matrix = projection.multiply(camera.matrix()).multiply(modelmatrix);
    var uniform = new ResourceBuffer(VkDescriptorType.UNIFORM_BUFFER, 0L, buffer);
    matrix.buffer(uniformBuffer.buffer());
    return uniform;
}
```

When we run the demo the results are a bit of a mess:

![Broken Chalet Model](mess.png)

There are a couple of issues here but the most obvious is that the texture appears to be upside down, the grass is obviously on the roof and vice-versa.

## Texture Coordinate Invert

The upside-down texture is due to the fact that OBJ texture coordinates (and OpenGL) assume an origin at the bottom-left corner of the image whereas Vulkan uses the top-left corner.

The texture coordinates _could_ be fiddled by one of the following options:

* Flip the vertical texture component in the vertex shader (nasty).

* Flip the image once using an image editing package (just adds more manual work).

* Invert the image programmatically at load-time (makes loading slower).

None of these actually resolve the root problem, a better solution is to perform the flip _once_ when the OBJ model is constructed off-line.

The following adapter flips the vertical texture coordinate:

```java
private static Coordinate2D flip(float[] array) {
    assert array.length == 2;
    return new Coordinate2D(array[0], -array[1]);
}
```

And the parser in the OBJ loader is updated accordingly:

```java
add("vt", new VertexComponentParser<>(2, ObjectModelLoader::flip, model.coordinates()));
```

We assume that this will apply to all OBJ models, it can always be made an optional feature if that assumption turns out to be incorrect.

After regenerating the model it now looks to be textured correctly, in particular the signs on the front of the chalet are the right way round (so the model is not being rendered inside-out for example):

![Less Broken Chalet Model](mess2.png)

---

# Depth Buffer

## Overview

The second problem is that fragments are being rendered arbitrarily overlapping.  This is solved by enabling the _depth test_ which ensures that obscured fragments are either overwritten by nearer geometry or discarded altogether, based on distance from the camera.

The depth test is configured by a new pipeline stage:

```java
public class DepthStencilStage extends AbstractPipelineBuilder<VkPipelineDepthStencilStateCreateInfo> {
    private final VkPipelineDepthStencilStateCreateInfo info = new VkPipelineDepthStencilStateCreateInfo();

    public DepthStencilStageBuilder() {
        enable(false);
        write(true);
        compare(VkCompareOp.LESS);
    }
}
```

The depth test uses the _depth buffer_ which is a special attachment that records the distance of each rendered fragment, and discards subsequent fragments that are obscured.

Since we are now dealing with multiple types of attachments the existing framework needs to be revised to support both.  Additionally in the previous demos the clear value for the colour attachments was hard-coded, with the addition of the depth buffer this functionality now needs to be properly implemented.

## Attachments

The following are required to support both colour and depth attachments:

* An attachment description.

* The image format.

* The image views for that attachment: either the swapchain images for the colour attachment or the _single_ depth-stencil attachment.

* An optional clear value with properties specific to that type of attachment.

Slightly surprisingly there is no equivalent Vulkan object for an 'attachment' as such, instead these properties are spread around the collaborating objects (the render pass, framebuffers, etc) as required.  For our purposes it seems logical to aggregate the above into a single, coherent type:

```java
public interface Attachment {
    /**
     * Types of attachment.
     */
    enum AttachmentType {
        COLOUR,
        DEPTH,
        RESOLVE
    }

    /**
     * @return Type of attachment
     */
    AttachmentType type();

    /**
     * @return Image format
     */
    VkFormat format();

    /**
     * @return Attachment description
     */
    AttachmentDescription description();

    /**
     * Retrieves an image view of this attachment by index.
     * @param index Frame index
     * @return Attachment view for the given index
     */
    View view(int index);
    
    /**
     * @return Clear value for this attachment
     */
    ClearValue clear();
}
```

Notes:

* The attachment _type_ may turn out to be superfluous since there will be multiple attachment sub-classes anyway.

* Clear values are covered shortly.

A skeleton implementation is added as a convenience base-class for attachments:

```java
public abstract class AbstractAttachment extends AbstractTransientObject implements Attachment {
    private final AttachmentType type;
    private final AttachmentDescription description;
    private List<View> views;
    private ClearValue clear;
}
```

Note that the colour attachment has a separate image for each frame, whereas a __single__ depth-stencil image is shared across all frames:

```java
public View view(int index) {
    return switch(type) {
        case COLOUR -> views.get(index);
        default -> views.getFirst();
    };
}
```

The image views are destroyed when the attachment is released:

```java
protected void release() {
    for(View view : views) {
        view.destroy();
    }
    views = null;
}
```

The colour attachment can now be implemented as follows:

```java
public class ColourAttachment extends AbstractAttachment {
    private final Swapchain swapchain;

    public ColourAttachment(AttachmentDescription description, Swapchain swapchain) {
        super(AttachmentType.COLOUR, description);
        this.swapchain = swapchain;
        clear(Colour.BLACK);
    }

    public VkFormat format() {
        return swapchain.format();
    }
}
```

The views for the colour attachment are created from the swapchain images:

```java
protected List<View> views(LogicalDevice device, Dimensions extents) {
    return swapchain
        .attachments()
        .stream()
        .map(image -> View.of(device, image))
        .toList();
}
```

Finally a helper is added to the new attachment class to create a reference with default properties:

```java
public Attachment.Reference reference() {
    return new Attachment.Reference(this, VkImageLayout.COLOR_ATTACHMENT_OPTIMAL);
}
```

## Depth Attachment

A second attachment implementation can now be added for the depth buffer:

```java
public class DepthStencilAttachment extends AbstractAttachment {
    private final VkFormat format;
    private final Allocator allocator;

    public DepthStencilAttachment(VkFormat format, AttachmentDescription description, Allocator allocator) {
        super(AttachmentType.DEPTH, description);
        this.format = format;
        this.allocator = allocator;
        super.clear(DepthClearValue.DEFAULT);
    }

    public Attachment.Reference reference() {
        return new Attachment.Reference(this, VkImageLayout.DEPTH_STENCIL_ATTACHMENT_OPTIMAL);
    }

    public List<View> views(LogicalDevice device, Dimensions extents) {
        ...
    }
}
```

The depth-stencil attachment is also responsible for creating and managing the depth buffer image in addition to the attachment view, unlike the colour attachment where the images are created by Vulkan when the swapchain is instantiated.

First an image descriptor specifies the properties of the depth buffer:

```java
var descriptor = new Image.Descriptor.Builder()
    .aspect(VkImageAspectFlags.DEPTH)
    .format(format)
    .extents(extents)
    .build();
```

The buffers memory is obviously device local:

```java
var properties = new MemoryProperties.Builder<VkImageUsageFlags>()
    .usage(VkImageUsageFlags.DEPTH_STENCIL_ATTACHMENT)
    .required(VkMemoryPropertyFlags.DEVICE_LOCAL)
    .build();
```

The image can now be instantiated:

```java
Image image = new DefaultImage.Builder()
    .descriptor(descriptor)
    .properties(properties)
    .tiling(VkImageTiling.OPTIMAL)
    .build(allocator);
```

And wrapped by a single image view:

```java
View view = new View.Builder()
    .release()
    .build(device, image);

return List.of(view);
```

## Clear Values

A new type is introduced for the various clear values:

```java
public sealed interface ClearValue {
    record None() implements ClearValue {
    }

    record ColourClearValue(Colour colour) implements ClearValue {
    }

    record DepthClearValue(Percentile depth) implements ClearValue {
    }
}
```

The `None` implementation is the default, unspecified value.  This is required since a clear value descriptor must be provided for every attachment in the render pass whether it is cleared or not.

A setter for the clear value colour is added to the colour attachment class:

```java
public void clear(Colour colour) {
    super.clear(new ColourClearValue(colour));
}
```

And similarly for the depth-stencil:

```java
public void clear(Percentile depth) {
    super.clear(new DepthClearValue(depth));
}
```

The clear descriptor for a given attachment can now be populated accordingly:

```java
static VkClearValue populate(ClearValue clear) {
    var descriptor = new VkClearValue();

    switch(clear) {
        case None _ -> {
            // Unspecified
        }

        case ColourClearValue(Colour colour) -> {
            descriptor.color = new VkClearColorValue();
            descriptor.color.float32 = colour.toArray();
        }

        case DepthClearValue(Percentile depth) -> {
            descriptor.depthStencil = new VkClearDepthStencilValue();
            descriptor.depthStencil.depth = depth.value();
            descriptor.depthStencil.stencil = 0;
        }
    }

    return descriptor;
}
```

Finally the `begin` method of the frame buffer is updated to populate the array of clear values at the start of the render pass:

```java
// Enumerate clear values
List<ClearValue> clear = pass
    .attachments()
    .stream()
    .map(Attachment::clear)
    .toList();

// Init clear values
info.clearValueCount = clear.size();
info.pClearValues = StructureCollector.pointer(clear, new VkClearValue(), ClearValue::populate);
```

Introducing clear values should have been easy, however there was a nasty surprise when the depth-stencil was added to the demo, with JNA throwing the infamous `Invalid memory access` error.

After some considerable time we eventually realised that `VkClearValue` and `VkClearColorValue` are in fact __unions__ and not structures.  Vulkan is expecting __either__ a colour array __or__ a floating-point depth, whereas currently the code sends both in all cases (probably resulting in some sort of array length or alignment violation).

> Presumably the temporary hard-coded bodge only worked by luck because the code generator treated unions as a plain structures, and the properties for a colour attachment happen to be the first field in each object.

As far as we can tell this is the __only__ instance of the use of unions in the whole Vulkan API!

Thankfully JNA supports unions out-of-the-box using the `setType` method to 'select' the relevant properties:

```java
case ColourClearValue(Colour colour) -> {
    descriptor.color = new VkClearColorValue();
    descriptor.setType("color");
    descriptor.color.setType("float32");
    descriptor.color.float32 = colour.toArray();
}
```

And similarly for the depth-stencil:

```java
case DepthClearValue(Percentile depth) -> {
    descriptor.setType("depthStencil");
    ...
}
```

## Integration #2

To use the depth test in the demo a new depth-stencil attachment is added to the configuration:

```java
@Bean
@Order(2)
static DepthStencilAttachment depth(Allocator allocator) {
    return new DepthStencilAttachment(VkFormat.D32_SFLOAT, AttachmentDescription.depth(), allocator);
}
```

Which uses the following convenience factory method to configure the common properties of the depth-stencil:

```java
public static AttachmentDescription depth() {
    return new Builder()
        .operation(new LoadStore(VkAttachmentLoadOp.CLEAR, VkAttachmentStoreOp.DONT_CARE))
        .finalLayout(VkImageLayout.DEPTH_STENCIL_ATTACHMENT_OPTIMAL)
        .build();
}
```

Notes:

* The format of the depth buffer is temporarily hard-coded to `D32_SFLOAT` which is generally available on all Vulkan implementations.

* The _store_ operation is left as the `DONT_CARE` default.

* Similarly the _old layout_ property is `UNDEFINED` since the previous contents are not relevant for the depth-stencil.

The new attachment is then added to the render pass:

```java
Subpass subpass = new Subpass()
    .add(colour.reference())
    .add(depth.reference(VkImageLayout.DEPTH_STENCIL_ATTACHMENT_OPTIMAL))
    ...
```

And the view of the depth buffer image is also passed to the `create` method of the framebuffers along with the colour attachments.

Finally the clear values for the demo are configured:

```java
colour.clear(new ColourClearValue(...));
depth.clear(new DepthClearValue(1));
```

Notes:

* The depth buffer image does not need to be programatically transitioned as Vulkan automatically manages the layout during the render pass.

* The same depth buffer can safely be used in each frame since only a single subpass will be be rendering at any one time.

With the depth buffer enabled we should finally be able to see the chalet model:

![Chalet Model](chalet.png)

Ta-da!

## Format Filter

A final enhancement is to provide support for selection of the image format of the depth-stencil rather than hard-coding.

The following utility tests a given image format against those supported by the hardware:

```java
public class FormatFilter implements Predicate<VkFormat >{
    private final Function<VkFormat, VkFormatProperties> provider;
    private final boolean optimal;
    private final EnumMask<VkFormatFeatureFlags> features;
    private final Map<VkFormat, VkFormatProperties> cache = new HashMap<>();

    public boolean test(VkFormat format) {
        VkFormatProperties properties = cache.computeIfAbsent(format, provider);
        EnumMask<VkFormatFeatureFlags> supported = optimal ? properties.optimalTilingFeatures : properties.linearTilingFeatures;
        return supported.contains(features);
    }
}
```

Where:

* The `provider` looks up the properties of a given image format (see below).

* The `optimal` flag selects _optimal_ or _linear_ tiling features.

* And `features` configures the format features to be included in the test.

This is used by the following helper in the depth-stencil attachment class to select the best available image format:

```java
public static VkFormat format(Function<VkFormat, VkFormatProperties> provider, List<VkFormat> formats) {
    var filter = new FormatFilter(provider, true, Set.of(VkFormatFeatureFlags.DEPTH_STENCIL_ATTACHMENT));
    var selector = new PrioritySelector<>(filter, PrioritySelector.first());
    return selector.select(formats);
}
```

The `PrioritySelector` is another utility (covered in a later chapter) that applies the filter to a list of candidates ordered by preference, falling back to the _first_ in the list if none match.

Finally the commonly used depth-stencil formats are defined by the following convenience constant:

```java
public class DepthStencilAttachment extends AbstractAttachment {
    /**
     * Commonly supported image formats for the depth-stencil attachment.
     */
    public static final List<VkFormat> IMAGE_FORMATS = List.of(D32_SFLOAT, D32_SFLOAT_S8_UINT, D24_UNORM_S8_UINT);
}
```

The format of the depth buffer image can now be determined programatically using the new helper rather than hard-coding:

```java
static DepthStencilAttachment depth(PhysicalDevice device, Allocator allocator) {
    VkFormat format = DepthStencilAttachment.format(device::properties, DepthStencilAttachment.IMAGE_FORMATS);
    return new DepthStencilAttachment(format, AttachmentDescription.depth(), allocator);
}
```

Nice.

---

# Summary

This chapter introduced the _depth test_ by implementation of the following:

- A camera model.

- Vertex normals and the cardinal axes.

- A builder for draw commands.

- Texture coordinate inversion for OBJ models.

- The depth-stencil pipeline stage.

- A refactored attachment class to support both colour and depth-stencil attachments.

- A mechanism to configure attachment clear values.

