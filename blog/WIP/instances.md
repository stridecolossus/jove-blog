---

## Instanced Drawing

### Instance Index

There are several approaches that can be used to render multiple cube instances.

The simplest uses the built-in `gl_InstanceIndex` variable in the vertex shader to index into a hard-coded array.

First the draw command is modified to render four instances:

```java
Command draw = (lib, buffer) -> lib.vkCmdDraw(buffer, mesh.count(), 4, 0, 0);
```

In the vertex shader each cube instance is arranged as a two-by-two grid:

```glsl
vec2 offset[4] = vec2[](
    vec2(-0.5, -0.5),
    vec2(-0.5, +0.5),
    vec2(+0.5, -0.5),
    vec2(+0.5, +0.5)
);
```

And the resultant vertex position is fiddled for each instance by indexing into the array:

```glsl
void main() {
    gl_Position = matrix * vec4(inPosition, 1);
    gl_Position += vec4(offset[gl_InstanceIndex], 0, 0);
    outTexCoord = inTexCoord;
}
```

Obviously this is a very quick-and-dirty hack just to test instanced rendering (especially since the offsets are applied _after_ perspective projection).

It may be worth increasing the rotation period (or disabling the animation altogether) to verify the results:

![Cube Grid](cube.grid.png)

### Instanced Vertex Attributes

A second approach is to use _per-instance_ vertex attributes to specify the offsets.

First the array is moved to a new method in the vertex buffer configuration class:

```java
@Bean
static VertexBuffer offsets(LogicalDevice dev, Allocator allocator, Command.Pool graphics) {
    Vector[] offsets = {
        new Vector(-0.5f, -0.5f, 0),
        new Vector(-0.5f, +0.5f, 0),
        new Vector(+0.5f, -0.5f, 0),
        new Vector(+0.5f, +0.5f, 0),
    };
    
    ...
}
```

Which is wrapped as a bufferable object:

```java
ByteSizedBufferable data = new ByteSizedBufferable() {
    @Override
    public int length() {
        return offsets.length * Vector.LAYOUT.stride();
    }

    @Override
    public void buffer(ByteBuffer bb) {
        for(Vector v : offsets) {
            v.buffer(bb);
        }
    }
};
```

And a second VBO is created using the same staged approach as the cube and bound to the pipeline.

In the pipeline configuration for the vertex input stage a second binding is added for the new VBO:

```java
.input()
    .add(cube.layout())
    .binding()
        .index(1)
        .rate(VkVertexInputRate.INSTANCE)
        .stride(Vector.LAYOUT.stride())
        .attribute()
            .location(2)
            .format(FormatBuilder.format(Vector.LAYOUT))
            .build()
        .build()
    .build()
```

Where the `rate` for the offsets vertex attribute is configured as `INSTANCE`, i.e. the data is iterated per-instance rather than per-vertex.

In the vertex shader the new vertex attribute is configured as follows:

```glsl
layout(location=2) in vec3 offset;
```

Note that the `location` of each vertex attribute must be unique within the shader and each VBO, however the source of each attribute is opaque to the shader.
i.e. There is no notion or need for a `binding` parameter in the `layout` specification for a vertex attribute.

Finally the position of each vertex is fiddled by the per-instance offset rather than using the `gl_InstanceIndex` variable index:

```glsl
gl_Position += vec4(offset, 0);
```

The results should be the same as the previous hard-coded shader version.

Note that there is no `VkFormat` for a matrix type (the largest format is four 32-bit floating-point values) so we cannot represent matrices as vertex attributes directly, at least not without the complexity of compression and the possible loss of precision.

### Instanced Data

The final approach uses a separate model matrix for each instance, which is the general approach that will be used later when we address scene graphs.

In the vertex shader the projection and view matrices are separated and an _array_ of model matrices is introduced:

```glsl
layout(set=0, binding=1) uniform Matrices {
    mat4 projection;
    mat4 view;
    mat4[4] model;
};

void main() {
    gl_Position = projection * view * model[gl_InstanceIndex] * vec4(inPosition, 1);
    ...
}
```

The uniform buffer is resized to accommodate six matrices (projection, view and four instances) which are written on each frame:

```java
ByteBuffer bb = uniform.buffer();
projection.buffer(bb);
view.buffer(bb);
```

With a model matrix for each instance comprising the rotation and an offset translation:

```java
for(int n = 0; n < instances; ++n) {
    Matrix trans = Matrix.translation(offset[n]);
    Matrix model = trans.multiply(rot.matrix());
    model.buffer(bb);
}
bb.rewind();
```

Notes:

* The (maximum) size of the model matrix array __must__ be specified in the shader.

* The rotation of each cube instance is applied first and _then_ the translation offset.

* The second VBO and per-instance vertex attribute were removed.

* The model matrices use the same uniform buffer as the projection and view but could have been stored in a different uniform or a storage buffer (which will be used in a later chapter).

* GLSL version 460 supports the `gl_BaseInstance` which is the index of the first instance passed to the draw command.

An alternative approach could be to pass the offset vectors in the uniform buffer and construct a translation matrix in the shader:

```glsl
layout(set=0, binding=1) uniform Matrices {
    mat4 projection;
    mat4 view;
    mat4 model;
    vec3[4] offsets;
};

void main() {
    mat4 trans = mat4(
        vec4(1, 0, 0, 0),
        vec4(0, 1, 0, 0),
        vec4(0, 0, 1, 0),
        vec4(offsets[gl_InstanceIndex], 1)
    );

    gl_Position = projection * view * trans * model * vec4(inPosition, 1);
    ...
}
```

---
