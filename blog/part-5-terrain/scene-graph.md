---
title: Scene Graph
---

---

## Contents

- [Overview](#overview)
- [Scene Graph](#scene-graph)
- 

---

## Overview

---

## Scene Graph

### Analysis

Up until now our demo applications have generally consisted of a handful of models with hand-crafted render sequences, this approach will obviously not scale to more complex scenarios for a variety of reasons:

* The scene structure is hard-coded.

* Most real-world applications will require scenes to be mutable.

* The elements of each object are hard-coded:
    - The render sequence.
    - Population of the modelview matrix.
    - Vulkan state changes (i.e. pipelines and descriptor sets).

Therefore a new framework is required to construct general scenes that supports the following assumptions:

* Users will generally prefer to think in terms of _objects_ in a scene rather than the various Vulkan domain types (VBO, draw command, etc).

* The visual properties of an object are best characterised by a _material_ abstracting over implementation details such as pipelines, samplers, etc.

* Objects in the scene are arranged spatially and/or logically depending on the application and _inherit_ transformations applied to their group.

* A mechanism is required to support _frustum culling_ and _picking_ of the objects in the scene.

A _scene graph_ is a common abstraction for 3D applications that supports these requirements.  However there are additional factors arising from how Vulkan and the GPU operate:

* The implementation should minimise expensive state changes (i.e. pipelines, descriptor sets) to optimise performance.

* The transformation matrix for a given model can be implemented either as a uniform buffer or a push constant.

* Many applications require a further top-level _render queue_ grouping to separate the following cases:
    - Arbitrarily ordered _opaque_ geometry, often utilising the depth buffer.
    - Translucent geometry rendered __after__ the opaque scene in order of _reverse_ distance from the camera.
    - Background models (e.g. skybox) usually rendered last.

### Design

Based on the above the scene graph will be comprised of a hierarchical tree of _nodes_ with the following properties:

* A _local transform_ that composes the _world matrix_ of the node with that of its ancestors.

* A _material_ specifying the rendering properties (pipeline, descriptor set) and the _render queue_ of each node.

* An optional _bounding volume_ used for culling and picking (which can also support collision tests).

The transform and material properties _can_ be inherited (or overridden) by an ancestor.  For example a node could have an explicit local transformation or could inherit that of its parent (the local transform is the identity matrix in the latter case).  Bounding volumes are slightly more complicated and are deferred until later in the chapter.

The local transform also contains a _matrix strategy_ that specifies the mechanism for passing the various matrices to the shader (projection, view/camera and model).  For example, an aggregated modelview matrix stored as a push constant with a separate projection matrix contained in a static uniform buffer.  The strategy also provides an extension point for alternative approaches later in development.

Initially there will be two node implementations:

* A _model node_ representing an object in the scene, containing a _mesh_ which aggregates the VBO, optional index buffer and draw command (derived from the model).

* A _group node_ comprising a list of children which can inherit its properties and those of its ancestors.

To optimise state changes the _model nodes_ in a scene graph are grouped as follows:

1. Render queue.

2. Pipeline.

3. Descriptor set (textures and uniform buffers).

4. Push constant updates.

Initially we envisaged a Java streams-based solution for this logic, which seemed logical given the code is dealing with collections.  However the dynamic nature of a scene graph means groups would be evaluated on _every_ frame which feels wasteful, particularly for scenes with low volatility.  Therefore the scene graph is comprised of __two__ data structures: the node hierarchy and a complimentary set of _render groups_ that index the model nodes.  The appropriate groups for a node are determined _once_ when it is added to the scene or a relevant property (e.g. the material) is modified.

The various properties of a node are inherited from its ancestors (in particular the world matrix), requiring a sub-tree of the scene to be _updated_ when the properties of a node are mutated.  This implies a _visitor_ operation performed on a node and its descendants to update the scene graph as required.  To minimise the amount of processing, invoking an update is a responsibility of the application rather than an on-demand approach.

Finally the following Vulkan specific components will be required to render the scene:

* A _renderer_ that generates the command sequence for a model node (usually on each frame unless the scene is completely static).

* Matrix strategy implementations to support the basic cases outlined above.

Note that this design is largely Vulkan agnostic for the goals are code simplicity and testability by separation of concerns.  The intention is not to support a requirement for multiple rendering back-ends, although this 

TODO - VK enumerates the state change types: pipeline, DS/texture, push constants? how would node know which property applies to which type?
Q - introduce 'Texture' here?

---

## Secondary Command Buffers


```java
class PrimaryBuffer extends Buffer {
    public PrimaryBuffer begin(VkCommandBufferUsage... flags) {
        super.begin(null, flags);
        return this;
    }

    public PrimaryBuffer add(List<SecondaryBuffer> buffers) {
        validate(State.RECORDING);
        execute(buffers);
        return this;
    }
}
```

```java
protected final void execute(List<SecondaryBuffer> buffers) {
    // Validate secondary buffers can be executed
    for(var e : buffers) {
        e.validate(State.EXECUTABLE);
    }

    // Record secondary buffers
    VulkanLibrary lib = this.pool().device().library();
    Pointer array = NativeObject.array(buffers);
    lib.vkCmdExecuteCommands(this, buffers.size(), array);
}
```

```java
class SecondaryBuffer extends Buffer {
    public SecondaryBuffer begin(Handle pass) {
        // Check can be recorded
        validate(State.INITIAL);

        // Init inheritance
        var info = new VkCommandBufferInheritanceInfo();
        info.renderPass = notNull(pass);
        info.subpass = 0; // TODO - subpass index, query stuff

        // Begin recording
        super.begin(info, VkCommandBufferUsage.RENDER_PASS_CONTINUE);

        return this;
    }
}
```

allocation changed

sequence

integration

---

## Summary

TODO



still really need some sort of VK implementation of a model comprising the header (layout, count, primitive) and VBO and optional index, e.g. Vulkan Model?  Should all be renamed Mesh anyway?
- model / builder generates bounds, optional
- volume exposes bounds, can then generate overall volume as max bounds, not compound but russian doll
- distance point to vector to support cone, cylinder volumes. Also plane?

