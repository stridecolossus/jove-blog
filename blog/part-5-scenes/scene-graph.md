---
title: Scene Graph
---

---

# Contents

- [Overview](#overview)
- [Scene Graph](#scene-graph)
- [Summary](#summary)

---

# Overview

## Problem

Until this point the demos have generally consisted of a single mesh, pipeline, texture, etc.  However this is already proving to be difficult to maintain and certainly will not scale to more complex or dynamic applications.  Additionally the various Vulkan components are currently declared statically and are largely immutable.  A more developer-friendly approach is required to support richer and more dynamic scenes.

## Analysis

It is probably easier to think about these requirements in terms of an example rather than the abstract:  Consider a space game where the player is in combat against a variety of enemy spaceships, each with individual models and visuals, at different locations, with different  directions and orientations.  Enemy spaceships that have been destroyed are removed from the scene and perhaps new enemies periodically spawn into combat.

From this example the following conclusions are apparent:

* The developer of this fictitious game would probably prefer to organise these dynamic elements _spatially_ rather than (say) an arbitrary list or a map indexed by texture.

* The elements of the scene are easier to manage when expressed in domain terms rather than the underlying vertex buffers, pipelines, etc.

* Transformations and visual properties are likely to be shared across multiple elements of the scene.

* The rendering sequence is essentially implicit and could be generated rather than hard-coded.

* Similarly the developer could also be relieved from having to programatically specify shader parameters such as matrices, colours, etc.

* However a spatial organisation is unlikely to be optimal from a performance perspective, since rendering state changes (pipelines, descriptor sets, etc) are expensive and should therefore be minimised.

## Scene Graph

A common solution to this organisational complexity is a _scene graph_ which is a data structure that arranges the elements of the scene _spatially_ as a hierarchic tree.  Each _node_ in the graph composes objects to be rendered along with their location, orientation and visual characteristics.  The rendering properties of each node are specified by a _material_ that abstracts over the pipeline, textures, etc. required to render each object.  The various properties of a node can also be implicitly inherited from its ancestor(s).

This provides a more convenient and logical data structure to organise scene elements and rendering properties, particularly for applications where the scene is dynamic.  Additionally scenes are expressed in domain terms rather than the underlying rendering technology.

The render sequence is then be generated from the scene graph utilising something along the lines of the _visitor_ pattern rather than being hard coded.  A similar mechanism is used to construct the shader parameters (uniform buffers, and later push constants) used during rendering.

>  Push constants are an alternative to uniform buffers for passing data to the rendering pipeline, this functionality will be introduced in a subsequent chapter.

Note that this approach also inherently lends itself to supporting _composite_ objects.  To continue the example, an individual spaceship could be a compound object comprising multiple meshes, textures, transforms, etc. which is essentially just another scene graph.

## Requirements

A _node_ will compose the following:

* The object(s) that comprise that part of the scene, i.e. a mesh.

* A _local transform_ specifying the position and orientation of the node in relation to its parent(s), i.e. a matrix.

* Optional material properties specifying rendering and shader parameters (which can also be inherited).

* An optional _bounding volume_ used for scene culling and picking (addressed in later chapters).

* A list of children that inherit the properties of that node.

Rendering the scene will require the following processes:

* A visitor mechanism that calculates the _world matrix_ of each node by combining its _local transform_ with that of its parent.

* A mechanism to _flatten_ the scene graph to a list of renderable objects, grouped by material properties in order of cost.

* A _renderer_ component that generates the render sequence from this flattened list of objects.

* A builder that generates the shader parameters (uniform buffers and/or push constants) for each frame.

Note that it is not an explicit requirement that this framework completely abstracts over the underlying Vulkan components, however the natural separation of concerns allows the scene graph functionality to be tested in relative isolation.

also not a requirement for a framework that manages the Vulkan components
out of scope
application is still responsible for the allocation, configuration and deletion of resources
abstraction that uses them

---

# Scene Graph

---

# Summary

summary
