---
title: Skybox
---

---

## Contents

- [Overview](#overview)
- [Skybox](#skybox)
- [KTX](#ktx-images)
- [Device Features](#device-features)

---

## Overview

In this chapter we will add a _skybox_ to the demo implemented as a cube centred on the camera.

The image framework will also be enhanced to support multiple layers and MIP levels with a new and more efficient image loader.

Finally we also introduce new functionality to support optional device features.

---

## Skybox

### Pipeline Configuration

We start with a new configuration class for the skybox which requires new shaders and a custom pipeline configuration:

```java
@Configuration
public class SkyBoxConfiguration {
    @Bean
    public Pipeline skyboxPipeline(...) {
        return new Pipeline.Builder()
            ...
            .depth()
                .enable(true)
                .write(false)
                .build()
            .rasterizer()
                .cull(VkCullMode.FRONT)
                .build()
            .build(dev);
    }
}
```

Notes:

* The depth test is enabled but does not update the depth buffer (specified by the `write` property) which ensures the skybox is not rendered over existing geometry.

* Front faces are culled (see below) since the camera is _inside_ the skybox.

* The new pipeline shares the existing pipeline layout.

The builder for the _rasterizer_ pipeline stage is essentially a wrapper for the underlying descriptor:

```java
public class RasterizerStageBuilder extends AbstractPipelineBuilder<VkPipelineRasterizationStateCreateInfo> {
    private final VkPipelineRasterizationStateCreateInfo info = new VkPipelineRasterizationStateCreateInfo();

    public RasterizerStageBuilder cull(VkCullMode cullMode) {
        info.cullMode = notNull(cullMode);
        return this;
    }

    ...

    @Override
    VkPipelineRasterizationStateCreateInfo get() {
        return info;
    }
}
```

The rasterizer properties are initialised in the constructor:

```java
public RasterizerStageBuilder() {
    depthClamp(false);
    discard(false);
    polygon(VkPolygonMode.FILL);
    cull(VkCullMode.BACK);
    winding(VkFrontFace.COUNTER_CLOCKWISE);
    lineWidth(1);
}
```

Next a cube model is created for the skybox:

```java
@Bean
public static Mesh skybox() {
    return new CubeBuilder(List.of(Point.LAYOUT)).build();
}
```

Note that we only use the vertex positions in the skybox shader (see below).
because?
TODO

Finally a cubemap sampler is created which clamps the texture coordinates:

```java
@Bean
public Sampler cubeSampler() {
    return new Sampler.Builder(dev)
        .mode(AddressMode.EDGE.mode())
        .build();
}
```

### Shaders

The camera is at the centre of the skybox and therefore does not need to be moved, but is rotated according to the current view.

The vertex shader for the skybox uses the `mat3` operator to extract the rotation component of the view transform:

```glsl
void main() {
    vec3 pos = mat3(view) * inPosition;
}
```

The projection transformation sets the last two components to be the same value such that the resultant vertex lies on the far clipping plane, i.e. the skybox is always drawn _behind_ the rest of the geometry:

```glsl
gl_Position = (projection * vec4(pos, 0.0)).xyzz;
```

Finally the texture coordinate is simply set to the incoming vertex position (hence the reason for not including texture coordinates in the cube model).

The full vertex shader is as follows:

```glsl
#version 450

layout(set=0, binding=1) uniform Matrices {
    mat4 projection;
    mat4 view;
    mat4 model;
};

layout(location=0) in vec3 inPosition;

layout(location=0) out vec3 outCoords;

void main() {
    vec3 pos = mat3(view) * inPosition;
    gl_Position = (projection * vec4(pos, 0.0)).xyzz;
    outCoords = inPosition;
}
```

Notes:

* The `model` matrix is not used in the skybox shader.

* The skybox uses the same render pass and pipeline layout as the chalet model.

The fragment shader is the same as the previous demo except for the `samplerCube` declaration:

```glsl
#version 450

layout(set=0, binding=0) uniform samplerCube cubemap;

layout(location=0) in vec3 inCoords;
layout(location=0) out vec4 outColour;

void main() {
    outColour = texture(cubemap, inCoords);
}
```

### Loader

Initially a separate image is loaded for each face of the cubemap texture.

First a texture is configured with an array layer for each face:

```java
Descriptor descriptor = new Descriptor.Builder()
    .type(VkImageType.TWO_D)
    .aspect(VkImageAspect.COLOR)
    .extents(...)
    .format(format)
    .arrayLayers(6)
    .build();
```

The image for the texture is configured as a cubemap:

```java
Image texture = new DefaultImage.Builder()
    .descriptor(descriptor)
    .properties(props)
    .cubemap()
    .build(dev, allocator);
```

Which uses a new method on the builder:

```java
public Builder cubemap() {
    return flag(VkImageCreateFlag.CUBE_COMPATIBLE);
}
```

Next the image for each face is loaded and copied to a staging buffer:

```java
var loader = new ResourceLoaderAdapter<>(src, new NativeImageLoader());
String[] filenames = {"posx", "negx", ...};
for(int n = 0; n < 6; ++n) {
    ImageData image = loader.load(filenames[n] + ".jpg");
    VulkanBuffer staging = VulkanBuffer.staging(dev, allocator, image.data());
    ...
}
```

And a separate copy operation is performed:

```java
new ImageCopyCommand.Builder()
    .buffer(staging)
    .image(texture)
    .layout(VkImageLayout.TRANSFER_DST_OPTIMAL)
    .subresource(res)
    .build()
    .submit(graphics);
```

Where the sub-resource specifies the array layer for each face of the cubemap:

```java
SubResource res = new SubResource.Builder(descriptor)
    .baseArrayLayer(n)
    .build();
```

Finally a view is created for the cubemap texture:

```java
SubResource subresource = new SubResource.Builder(descriptor)
    .layerCount(6)
    .build();

return new View.Builder(texture)
    .type(VkImageViewType.CUBE)
    .subresource(subresource)
    .mapping(image)
    .build();
```

### Integration

Integration of the skybox requires:

* Creation of a separate vertex buffer and draw command for the skybox model.

* The addition of a second group of descriptor sets for the new pipeline.

* Doubling the descriptor pool size.

The final change is the addition of the following commands to the render sequence:

```java
skyboxPipeline.bind()
set.bind(skyboxPipeline.layout())
skyboxVertexBuffer.bindVertexBuffer()
DrawCommand.of(skybox)
```

Note that the skybox is drawn _after_ the rest of the geometry reducing the number of fragments being rendered, i.e. the cubemap is only rendered to 'empty' parts of the scene.

If all goes well we should now see the skybox rendered behind the chalet model:

TODO

---

## KTX Images

### Overview

The process of loading the cubemap texture is quite slow since the native image loader has considerable overhead and each cubemap texture is loaded and copied separately.

Ideally we would prefer a solution that encompasses:

* Image data that more closely matches the target Vulkan format with minimal (or ideally zero) transformation.

* Compound images allowing more efficient bulk transfer operations.

* Multiple MIP levels.

* And eventually compressed image formats.

To satisfy all these requirements we will use the [KTX2](https://www.khronos.org/ktx/) image format which is pretty much the standard for Vulkan texture images.
Texture images, cubemaps, MIP pyramids, etc. can then be prepared offline minimising the work that needs to be performed at runtime.

The following changes are required:

* A new loader for a KTX image.

* Modifications to the image class to support multiple layers and MIP levels.

* The addition of regions to the copy command to support multiple layers and/or MIP levels.

First the general image abstraction is modified to support multiple layers and MIP levels:

```java
public class ImageData {
    /**
     * @return Image depth
     */
    public int depth() {
        return 1;
    }

    public record Level(int offset, int length) { ... }

    /**
     * @return MIP levels
     */
    public List<Level> levels() {
        return List.of(new Level(0, data.length));
    }

    /**
     * @return Number of array layers
     */
    public int layers() {
        return 1;
    }

    /**
     * @return Vulkan format hint
     */
    public int format() {
        return 0;
    }
}
```

The `format` accessor is a _hint_ for the Vulkan image format, note however that this class is a general image abstraction.

### Loose Ends

A KTX image is a binary format with _little endian_ byte ordering whereas Java is big-endian by default.  The entire file _could_ be loaded into an NIO byte buffer with little endian ordering (which has a similar API to the data stream) but we would prefer to stick with I/O streams for consistency with the existing loaders.  Additionally we anticipate that we will need to support other little endian file formats in the future.

The matter is further complicated when one considers the weird implementation of `DataInputStream`.  This class implements `DataInput` but there is no way to provide a custom implementation so the abstraction is completely pointless.  Additionally __all__ the methods are `final` (though not the class itself oddly enough) so it is essentially closed  for extension.  Therefore we are forced to completely re-implement a whole data stream class rather than building on what is already available - great design!

We start with a custom data stream wrapper:

```java
public class LittleEndianDataInputStream extends InputStream implements DataInput {
    private final DataInputStream in;

    public LittleEndianDataInputStream(InputStream in) {
        this.in = new DataInputStream(in);
    }
}
```

Most of the methods defined in `DataInput` simply delegate to the underlying stream, overridden implementations are added for the methods that need to support little endian data types, for example:

```java
public class LittleEndianDataInputStream extends InputStream implements DataInput {
    private static final int MASK = 0xff;

    private final byte[] buffer = new byte[8];

    public int readInt() throws IOException {
        in.readFully(buffer, 0, Integer.BYTES);
        return
            (buffer[3])        << 24 |
            (buffer[2] & MASK) << 16 |
            (buffer[1] & MASK) <<  8 |
            (buffer[0] & MASK);
    }
}
```

### KTX Loader

The new stream class is used in the implementation for the KTX loader:

```java
public class VulkanImageLoader implements ResourceLoader<DataInput, ImageData> {
    @Override
    public DataInput map(InputStream in) throws IOException {
        return new LittleEndianDataInputStream(in);
    }

    @Override
    public ImageData load(DataInput in) throws IOException {
        ...
    }
}
```

The [KTX2 file format](https://www.khronos.org/registry/KTX/specs/2.0/ktxspec_v2.html) consists of the following sections:

1. A header token.

2. Image details (dimensions, number of layers, etc).

3. A MIP level index.

4. A Data Format Descriptor (DFD)

5. Additional information represented as a set of key-value pairs.

6. Compression data.

7. Image data ordered by MIP level and layer.

Notes:

* The DFD, key-values and compression data are optional.

* For the moment we will assume image data is uncompressed and gloss over this section of the loader.

* The KTX file format has a large number of byte offsets/indices into the file itself which are largely ignored.

* The loader is constrained to support the latest KTX2 version of the file format which is designed around Vulkan (the SDK provides tools to transform older files if necessary).

The header token is a fixed length byte array which is first used to validate the file format:

```java
// Load header
byte[] header = new byte[12];
in.readFully(header);

// Validate header
String str = new String(header);
if(!str.contains("KTX 20")) throw new IOException();
```

Next the image details are loaded (comments illustrate the values for the chalet image):

```java
// Load image format
int format = in.readInt();                              // 43 = R8G8B8A8_SRGB
int typeSize = in.readInt();                            // Size of the data type in bytes (1)

// Load image extents
Dimensions size = new Dimensions(in.readInt(), in.readInt());
int depth = Math.max(1, in.readInt());                  // Image depth (0)

// Load image data size
int layerCount = Math.max(1, in.readInt());             // Assume 1
int faceCount = in.readInt();                           // 1 or 6 for a cubemap
int levelCount = in.readInt();                          // 13 MIP levels

// Load compression scheme
int scheme = in.readInt();                              // 0..3 (0 = uncompressed)
```

Notes:

* Some values can be zero and need to be constrained accordingly, e.g. the image depth.

* The KTX format supports image arrays specified by the _layerCount_ field, the loader constrains this value to one.

* There is some overlap in terminology here: A Vulkan image can have multiple _array layers_ which we are using for the cubemap image, the KTX equivalent is the `faceCount`.  However the KTX format also supports multiple _layers_ which map to one of the array types defined in the `VkImageViewType` enumeration.  We try to use the terms appropriate to the loader and the modified image class in each case.

* Vulkan does not support an array of 3D images.

Next the MIP level index is parsed:

```java
private static List<Level> loadIndex(DataInput in, int count) throws IOException {
    Level[] index = new Level[count];
    for(int n = 0; n < count; ++n) {
        int offset = (int) in.readLong();
        int len = (int) in.readLong();
        in.readLong();
        index[n] = new Level(offset, len);
    }
    ...
}
```

The _offset_ field is an offset into the file itself, which is truncated to the start of the image data:

```java
int offset = index[index.length - 1].offset();
return Arrays
    .stream(index)
    .map(level -> new Level(level.offset() - offset, level.length()))
    .toList();
```

Note that the index is in MIP level order (starting at zero for the largest image) whereas the actual image data is the __reverse__ (smallest MIP image first).

### Data Format Descriptor

The next section is the DFD (Data Format Descriptor) which specifies the structure of the image components.  This section could be simply skipped and the corresponding fields hard-coded in the image, but actually parsing and using this data makes the loader more robust and also exercises our understanding of the file format.

The DFD starts with a header block (again comments added to illustrate the values for the chalet image):

```java
in.readShort();                           // Vendor
short type = in.readShort();              // DFD Type (0)
short ver = in.readShort();               // Version number (2)
short blockSize = in.readShort();         // DFD size, 24 + 16 per sample (bytes)
byte model = in.readByte();               // KHR_DF_MODEL_RGBSDA (1)
byte primaries = in.readByte();           // KHR_DF_PRIMARIES_BT709 (1)
byte transferFunction = in.readByte();    // KHR_DF_TRANSFER_LINEAR (1) or KHR_DF_TRANSFER_SRGB (2)
byte flags = in.readByte();               // KHR_DF_FLAG_ALPHA_STRAIGHT (0) or KHR_DF_FLAG_ALPHA_PREMULTIPLIED (1)
```

The enumeration names are taken from the KTX documentation.

The header is followed by the texel dimensions and byte planes of the image which are ignored (for the moment anyway):

```java
// Skip texel dimensions
byte[] array = new byte[4];
in.readFully(array);

// Skip planes
in.readFully(array);                            // 0-3
in.readFully(array);                            // 4-7
```

Finally the _samples_ section specifies the structure of each component of the image pixels:

```java
int num = (blockSize - 24) /  16;
char[] components = new char[num];
for(int n = 0; n < num; ++n) {
    // Load sample information
    in.readShort();                             // Bit offset
    byte len = in.readByte();                   // Bit length (7)
    components[n] = channel(in.readByte());     // Channel

    // Skip sample position and lower/upper bounds
    in.readInt();
    in.readInt();
    in.readInt();
}
```

Where the `channel` helper method maps a channel byte to the corresponding character:

```java
private static char channel(byte channel) {
    return switch(channel & 0x0f) {
        case 0 -> 'R';
        case 1 -> 'G';
        case 2 -> 'B';
        case 15 -> 'A';
        default -> throw new IllegalArgumentException();
    };
}
```

The resultant array of _components_ is used to determine the pixel layout:

```java
Layout layout = new Layout(components.length(), Layout.Type.NORMALIZED, false, 1);
```

### Key-Values

A KTX image can also contain an arbitrary number of _key-value_ pairs that provide supplementary information about the image.

Again this could be skipped to position the stream at the start of the image data, but parsing the key-values verifies our understanding of the file format.

Each key-value is essentially a byte array:

```java
static List<byte[]> loadKeyValues(DataInput in, int size) throws IOException {
    List<byte[]> entries = new ArrayList<>();
    int count = 0;
    while(true) {
        // Load key-value length
        int len = in.readInt();

        // Load key-value entry
        byte[] entry = new byte[len];
        in.readFully(entry);
        entries.add(entry);
        
        ...
    }
    
    return entries;
}
```

Native binary formats are often required to be byte-aligned, each key-value is followed by a number of padding bytes:

```java
int padding = padding(len);
in.skipBytes(padding);
```

Which is calculated by the following helper:

```java
private static int padding(int len) {
    return 3 - ((len + 3) % 4);
}
```

A slight annoyance is that this section does not simply provide the number of key-values, instead the loop bound has to be inferred from the size of the data itself:

```java
// Calculate position
count += len + padding + 4;
assert count <= size;

// Stop at end of key-values
if(count >= size) {
    break;
}
```

### Image Data

Next the uncompressed image data is loaded as a single byte array:

```java
int len = index.stream().mapToInt(Level::length).sum();
byte[] data = new byte[len];
in.readFully(data, 0, len);
```

Notes:

* The data is ordered in __reverse__ MIP level order (smallest image first).

* A cubemap image is ordered by MIP level and __then__ face.

Finally the resultant image domain object is instantiated:

```java
return new ImageData(size, components, layout, data) {
    @Override
    public int depth() {
        return depth;
    }

    @Override
    public int format() {
        return format;
    }

    @Override
    public List<Level> levels() {
        return index;
    }

    @Override
    public int layers() {
        return faceCount;
    }
};
```

### Copy Region

To support the cubemap image the transfer command is extended to support multiple layers and MIP levels such that the image can be transferred in one operation.

First the `region` method of the builder is modified to iterate through the MIP levels and image layers:

```java
public Builder region(ImageData image) {
    Descriptor descriptor = this.image.descriptor();
    int count = descriptor.layerCount();
    Level[] levels = image.levels().toArray(Level[]::new);
    for(int level = 0; level < levels.length; ++level) {
        for(int layer = 0; layer < count; ++layer) {
            ...
        }
    }
    return this;
}
```

Within the loop the extents of each mipmap level are calculated:

```java
Extents extents = descriptor.extents().mip(level);
```

Which delegates to the following new helper method on the image extents class:

```java
public Extents mip(int level) {
    if(level == 0) {
        return this;
    }

    int w = mip(size.width(), level);
    int h = mip(size.height(), level);
    return new Extents(new Dimensions(w, h), depth);
}

private static int mip(int value, int level) {
    return Math.max(1, value >> level);
}
```

A sub-resource is configured for each MIP level and layer:

```java
SubResource res = new SubResource.Builder(descriptor)
    .baseArrayLayer(layer)
    .mipLevel(level)
    .build();
```

Next the offset into the image data is calculated:

```java
int offset = levels[level].offset(layer, count);
```

Which delegates to a new helper on the `Level` class:

```java
public int offset(int layer, int count) {
    return offset + layer * (length / count);
}
```

Finally a copy region is constructed and added to the command:

```java
CopyRegion region = new CopyRegion.Builder()
    .offset(offset)
    .subresource(res)
    .extents(extents)
    .build();
```

### Integration

Although we should now have all the functionality required to support multiple MIP levels and cubemap images we start with the texture for the chalet model with a single level.

The following command (from the SDK) is used to generate the KTX file:

```
toktx --t2 --target_type RGBA chalet.ktx2 chalet.jpg
```

Where:

* The `t2` argument specifies a KTX version 2 file format.

* The `target_type` specifies an RGBA image (the tool handily injects the alpha channel for us).

The texture configuration class is modified to use the new loader and resource:

```java
var loader = new ResourceLoaderAdapter<>(data, new VulkanImageLoader());
ImageData image = loader.load("chalet.ktx2");
```

A new helper is added to determine the image format using the hint if available or delegating to the previous implementation using the image layout:

```java
public class FormatBuilder {
    public static VkFormat format(ImageData image) {
        int format = image.format();
        if(format == VkFormat.UNDEFINED.value()) {
            return format(image.layout());
        }
        else {
            return IntEnum.mapping(VkFormat.class).map(image.format());
        }
    }
}
```

Finally the copy command uses the new helper method to construct the regions:

```java
// Create staging buffer
VulkanBuffer staging = VulkanBuffer.staging(dev, allocator, image.data());

// Copy staging to image
new ImageCopyCommand.Builder()
    .image(texture)
    .buffer(staging)
    .layout(VkImageLayout.TRANSFER_DST_OPTIMAL)
    .region(image)
    .build()
    .submit(transfer);
```

A crude timing comparison shows that the KTX image loader is almost an order of magnitude faster than using the previous native image.  However note that the KTX file is currently uncompressed and is considerably larger than the JPEG equivalent.  In most cases an application would generally prefer to compromise on larger file sizes to reduce loading times.

Next the following command generates a MIP pyramid for the image (which takes a few seconds):

```
toktx --t2 --target_type RGBA --genmipmap chalet.ktx2 chalet.jpg
```

The quality of the texture should now be considerably improved when zooming out.

Notes:

* The loading time for the whole mipmap image is approximately a 100% overhead (which makes sense).

* The default configuration of the texture sampler should automatically support the mipmap image.

* One can test that the MIP levels are being used by setting the `maxLod` property of the sampler (which essentially switches of the mipmap).

Note that we could generate the MIP pyramid at runtime (which the tutorial does) but in reality mipmap images will always be generated using an offline tool, producing better results and avoiding the runtime loading penalty.  Additionally the process of generating mipmap images at runtime is quite convoluted so there in no point spending the time and effort.

TODO...

For the skybox we replace the previous images with an existing KTX cubemap texture from the excellent 
[Vulkan samples](https://github.com/SaschaWillems/Vulkan/blob/master/data/README.md)

```
ktx2ktx2 cubemap_vulkan.ktx
```

...TODO

Replacing the code that loaded each cube face separately with a compound KTX cubemap image significantly reduces the loading time for the skybox (again almost an order of magnitude improvement).

---

## Device Features

### Overview

With support for mipmap images in place now is a good point to enable anisotropy to further improve the texture quality.  However anisotropy is an optional Vulkan feature that needs to be specifically enabled when creating the logical device.

The features supported by the hardware are specified by the `VkPhysicalDeviceFeatures` structure, which somewhat surprisingly is comprised of over fifty boolean fields as opposed to (say) a number of bit-field enumerations.  We make the assumption that applications would prefer to specify requirements by feature _names_ (possibly loaded from a configuration file) rather than forcing the developer to programatically locate and manipulate structure fields.

Therefore we introduce a new abstraction to specify the features _supported_ by the hardware (wrapping the structure) and the features _required_ for a given application (feature names).

### Framework

First the following interface (and skeleton implementation) define a set of device features for both cases:

```java
public interface DeviceFeatures {
    /**
     * @return Feature names
     */
    Set<String> features();

    /**
     * @return Descriptor for this set of features
     */
    VkPhysicalDeviceFeatures descriptor();

    /**
     * Tests whether this set contains the given features.
     * @param features Required features
     * @return Whether this set contains the given features
     */
    boolean contains(DeviceFeatures features);
}
```

The _required_ device features can be created from a set of feature _names_ using the following factory:

```java
static DeviceFeatures of(Collection<String> required) {
    return new AbstractDeviceFeatures() {
        @Override
        public Set<String> features() {
            return Set.copyOf(required);
        }

        @Override
        public VkPhysicalDeviceFeatures descriptor() {
            ...
        }

        @Override
        public boolean contains(DeviceFeatures features) {
            return required.containsAll(features.features());
        }
    };
}
```

A Vulkan descriptor for the device features can be constructed from the names as follows:

```java
public VkPhysicalDeviceFeatures descriptor() {
    // Skip if empty
    if(required.isEmpty()) {
        return null;
    }

    // Build descriptor
    var struct = new VkPhysicalDeviceFeatures();
    required.forEach(field -> struct.writeField(field, true));

    return struct;
}
```

Note the use of the the JNA `writeField` to populate a structure field via reflection.

The _supported_ device features is essentially a wrapper for the underlying structure:

```java
static DeviceFeatures of(VkPhysicalDeviceFeatures features) {
    // Init structure
    features.write();

    // Create wrapper
    return new AbstractDeviceFeatures() {
        @Override
        public Set<String> features() {
            ...
        }

        @Override
        public VkPhysicalDeviceFeatures descriptor() {
            return features.copy();
        }

        @Override
        public boolean contains(DeviceFeatures required) {
            return required.features().stream().allMatch(this::isEnabled);
        }

        private boolean isEnabled(String field) {
            return features.readField(field) == true;
        }
    };
}
```

The JNA `getFieldOrder` method is used to reflect the names of the enabled features:

```java
public Set<String> features() {
    return features
        .getFieldOrder()
        .stream()
        .filter(this::isEnabled)
        .collect(toSet());
}
```

### Integration

The new wrapper implementation is used to retrieve the _supported_ features for a physical device:

```java
public class PhysicalDevice implements NativeObject {
    private final Supplier<DeviceFeatures> features = new LazySupplier<>(this::loadFeatures);

    public DeviceFeatures features() {
        return features.get();
    }

    private DeviceFeatures loadFeatures() {
        var struct = new VkPhysicalDeviceFeatures();
        instance.library().vkGetPhysicalDeviceFeatures(this, struct);
        return DeviceFeatures.of(struct);
    }
}
```

Next we implement a convenience factory to create a filter for physical devices that match a set of _required_ features:

```java
public static Predicate<PhysicalDevice> predicate(DeviceFeatures features) {
    return dev -> dev.features().contains(features);
}
```

And finally in the builder for the logical device the relevant field is populated in the create descriptor:

```java
public LogicalDevice build() {
    info.pEnabledFeatures = required.descriptor();
    ...
}
```

This framework should allow an application to query the features _supported_ by the hardware and to specify the _required_ feature set.  Well behaved applications could adapt their functionality to the supported features.  For example the demo could avoid using sampler anisotropy if that feature is not supported (not likely, but illustrates the point).

In the device configuration class we first specify the _required_ features for the demo:

```java
static DeviceFeatures required(ApplicationConfiguration cfg) {
    return DeviceFeatures.of(cfg.getFeatures());
}
```

The required features are declared as a comma-delimited list in the properties file, for example:

```java
features: samplerAnisotropy, geometryShader
anisotropy: 8
```

The required features are then used to select the appropriate physical device:

```java
public PhysicalDevice physical(Instance instance, DeviceFeatures required) {
    return PhysicalDevice
        .devices(instance)
        .filter(graphics)
        .filter(presentation)
        .filter(PhysicalDevice.predicate(required))
        .findAny()
        .orElseThrow();
}
```

And are also used when constructing the logical device to enable the selected features:

```java
public LogicalDevice device(PhysicalDevice dev, DeviceFeatures required) {
    return new LogicalDevice.Builder(dev)
        .extension(VulkanLibrary.EXTENSION_SWAP_CHAIN)
        .layer(ValidationLayer.STANDARD_VALIDATION)
        .queue(graphics.family())
        .queue(presentation.family())
        .features(required)
        .build();
}
```

Finally we can now enable anisotropy in the sampler for the chalet model:

```java
public Sampler sampler(ApplicationConfiguration cfg) {
    return new Sampler.Builder()
        .anisotropy(cfg.getAnisotropy())
        .build(dev);
}
```

The anisotropy is set to 8 texel samples which is generally accepted as a good trade-off between quality and performance.

---

## Summary

In this chapter we added a skybox which involved implementation of the following:

* The rasterizer pipeline stage.

* Extension of the image class to support array layers and MIP levels.

* Implementation of the KTX loader (for uncompressed images).

* The addition of regions to the image copy command.

* Support for device features.

