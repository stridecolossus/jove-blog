---
title: The Render Loop and Synchronisation
---

---

## Contents

- [Overview](#overview)
- [Refactoring](#refactoring)
- [Synchronisation](#synchronisation)
- [Rotation and Animation](#rotation-and-animation)

---

## Overview

Before we wrap up the rotating cube demo there are several outstanding issues with the crude render loop implemented in the previous chapter:

* The rendering code is completely single-threaded.

* The rendering and presentation tasks are 'synchronised' by blocking on the work queues.

* The GLFW event queue is not being polled, meaning the window cannot be moved or closed.

* There is no mechanism to terminate the application other than the dodgy timer or force-quitting the process.

* The existing render loop is cumbersome and mixes unrelated activities (rendering, animation, event polling).

In this chapter these issues are addressed by the introduction of the following:

* New JOVE components to separate the various activities into reusable, coherent components that can also be unit-tested in isolation.

* Synchronisation to fully utilise the multi-threaded nature of the Vulkan pipeline.

* A GLFW keyboard handler to gracefully exit the application.

* An animation framework.

---

## Refactoring

### Render Loop

We start with a simple utility class that will track the elapsed duration of a rendered frame:

```java
public class Frame {
    private Instant start = Instant.now();
    private Instant end;

    public Duration elapsed() {
        return Duration.between(start, end);
    }

    public void stop() {
        if(end != null) throw new IllegalStateException();
        end = Instant.now();
    }
}
```

The timer also defines a listener for frame completion events:

```java
@FunctionalInterface
public interface Listener {
    /**
     * Notifies a completed frame.
     * @param frame Completed frame
     */
    void update(Frame frame);
}
```

The following implementation provides a basic FPS counter that can be used to monitor the application:

```java
public static class Counter implements Listener {
    private Instant next = Instant.EPOCH;
    private int count;

    public int fps() {
        return count;
    }

    @Override
    public void update(Frame frame) {
        Instant now = frame.time();
        if(now.isAfter(next)) {
            count = 1;
            next = now.plusMillis(MILLISECONDS_PER_SECOND);
        }
        else {
            ++count;
        }
    }
}
```

Next the render loop is factored to a new component that repeatedly schedules an arbitrary task:

```java
public class RenderLoop {
    private final Set<Listener> listeners = new HashSet<>();
    private int rate;
    private Future<?> future;
    private Consumer<Exception> handler = System.err::println;

    public void start(Runnable task) {
        if(isRunning()) throw new IllegalStateException();
        ...
    }
}
```

In the `start` method the given task is wrapped by a frame timer:

```java
Runnable wrapper = () -> {
    try {
        Frame timer = new Frame();
        task.run();
        timer.stop();
        update(timer);
    }
    catch(Exception e) {
        handler.accept(e);
    }
};
```

Where `update` notifies the attached listeners on completion of the task.

The task is then scheduled to run at a configured frame-rate using the executor framework:

```java
long period = TimeUnit.SECONDS.toMillis(1) / rate;
ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
future = executor.scheduleAtFixedRate(wrapper, 0, period, TimeUnit.MILLISECONDS);
```

Finally the loop can be interrupted:

```java
public void stop() {
    if(!isRunning()) throw new IllegalStateException();
    future.cancel(true);
    future = null;
}
```

This new component should allow applications to configure a target frame-rate without having to implement fiddly timing or thread yielding logic.

Notes:

* This implementation assumes that only __one__ task is being executed at any one time, i.e. the executor uses a single thread pool.

* A task that takes longer than the specified period simple backs up the next task, i.e. tasks will not overlap or run concurrently.

* This component is Vulkan agnostic.

* Exceptions thrown by a task are intercepted and handed off to the configured `handler` consumer.

### Render Task

Next a new task is implemented that composes the various collaborating Vulkan components used during rendering of a frame:

```java
public class VulkanRenderTask {
    private final List<FrameBuffer> buffers;
    private final FrameComposer composer;
    private final Swapchain swapchain;
}
```

The acquire-render-present code is factored out to the `render` method:

```java
public void render() {
    // Acquire next frame
    int index = swapchain.acquire(...);
    FrameBuffer fb = buffers.get(index);

    // Compose render task
    Command.Buffer render = composer.compose(fb);
    Command.Pool pool = render.pool();
    pool.waitIdle();

    // Present rendered frame
    swapchain.present(pool.queue(), index, null);
}
```

The frame composer is responsible for building the rendering command sequence:

```java
public class FrameComposer {
    private final Pool pool;
    private VkCommandBufferUsage[] flags = {VkCommandBufferUsage.ONE_TIME_SUBMIT};
    private VkSubpassContents contents = VkSubpassContents.INLINE;
}
```

And the existing code is moved to the `compose` method:

```java
public Buffer compose(FrameBuffer frame) {
    // Allocate a command buffer
    Buffer buffer = pool.allocate();

    // Start recording
    buffer.begin(flags);

    // Create a render pass for the given frame buffer
    Command begin = frame.begin(contents);
    buffer.add(begin);

    // Record render pass
    ...

    // Finish recording
    buffer.add(FrameBuffer.END);
    buffer.end();

    return buffer;
}
```

Finally a new convenience wrapper for a group of frame buffers is introduced:

```java
public static class Group implements TransientObject {
    public Group(RenderPass pass, Swapchain swapchain) {
        var extents = new Rectangle(swapchain.extents());
        this.buffers = swapchain
            .attachments()
            .stream()
            .map(view -> create(pass, extents, List.of(view)))
            .toList();
    }

    @Override
    public void destroy() {
        for(var b : buffers) {
            b.destroy();
        }
    }
}
```

This should also resolve the Vulkan warnings about orphaned frame buffers when the application shuts down.

### Integration

The new framework is used to break up the existing demo into simpler components.
First the presentation configuration is modified to create the frame buffer group:

```java
@Bean
static FrameBuffer.Group frames(Swapchain swapchain, RenderPass pass) {
    return new FrameBuffer.Group(pass, swapchain);
}
```

The frame composer:

```java
@Bean
static FrameComposer composer(@Qualifier("graphics") Command.Pool pool) {
    return new FrameComposer(pool);
}
```

The render task:

```java
@Bean
VulkanRenderTask render(FrameBuffer.Group group, FrameComposer composer, Swapchain swapchain) {
    return new VulkanRenderTask(group.buffers(), composer, swapchain);
}
```

And finally the render loop:

```java
@Bean
public RenderLoop loop(VulkanRenderTask task, Collection<Frame.Listener> listeners) {
    var loop = new RenderLoop();
    loop.rate(60);
    loop.start(task::render);
    for(var listener : listeners) {
        loop.add(listener);
    }
    return loop;
}
```

Note that Spring auto-magically generates the set of frame `listeners` from those registered with the container.

The last temporary bit of refactoring work is to move the animation logic into a frame listener:

```java
@Bean
public static FrameListener animation(Matrix projection, Matrix view, ResourceBuffer uniform) {
    long period = 2500;
    ByteBuffer bb = uniform.buffer();
    return () -> {
        // Build rotation matrix
        long time = System.currentTimeMillis();
        float angle = (time % period) * MathsUtil.TWO_PI / period;
        Matrix h = Rotation.matrix(Vector.Y, angle);
        Matrix v = Rotation.matrix(Vector.X, MathsUtil.toRadians(30));
        Matrix model = h.multiply(v);

        // Update matrix
        Matrix matrix = projection.multiply(view).multiply(model);
        matrix.buffer(bb);
        bb.rewind();
    };
}
```

### Graceful Exit

To gracefully exit the application a GLFW keyboard listener is added to the main configuration class:

```java
public class RotatingCubeDemo {
    @Bean
    KeyListener exit(Window window) {
        KeyListener listener = (ptr, key, scancode, action, mods) -> System.exit(0);
        window.desktop().library().glfwSetKeyCallback(window.handle(), listener);
        return listener;
    }
}
```

The keyboard listener is a GLFW callback defined as follows:

```java
interface KeyListener extends Callback {
    /**
     * Notifies a key event.
     * @param window            Window
     * @param key               Key index
     * @param scancode          Key scan code
     * @param action            Key action
     * @param mods              Modifiers
     */
    void key(Pointer window, int key, int scancode, int action, int mods);
}
```

A complication of particular importance is that GLFW event processing __must__ be performed on the main application thread, unfortunately it cannot be implemented as a separate thread or using the executor framework.  Therefore the polling logic has to remain as an activity on the main thread:

```java
@Bean
static CommandLineRunner runner(Desktop desktop) {
    return args -> {
        while(true) {
            desktop.poll();
        }
    };
}
```

See the [GLFW thread documentation](https://www.glfw.org/docs/latest/intro.html#thread_safety) for more details.

> Apparently GLFW version 4 will deprecate callbacks in favour of query methods which would remove this problem.

Finally a cleanup method is added to the main class to ensure the Vulkan device is shutdown correctly.

```java
@PreDestroy
void destroy() {
    dev.waitIdle();
}
```

### Application Configuration

Since the demo is moving towards a proper double-buffered swapchain and render loop, now is a good point to externalise some of the application configuration to simplify testing and data modifications.

A new `ConfigurationProperties` component is added to the demo:

```java
@Configuration
@ConfigurationProperties
public class ApplicationConfiguration {
    private String title;
    private int frameCount;
    private int frameRate;
    private Colour col;
    private long period;

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    ...

    public void setBackground(float[] col) {
        this.col = Colour.of(col);
    }
}
```

The properties are configured in the `application.properties` file:

```java
title: Rotating Cube Demo
frameCount: 2
frameRate: 60
background: 0.3, 0.3, 0.3
period: 2500
```

Notes:

* The properties class __must__ be a simple POJO (with old-school getters and setters) to be auto-magically populated by the Spring framework.

* The `background` property (the clear colour for for the swapchain views) is a comma-separated list which Spring injects as an array.

The demo application is refactored by injecting the properties into the relevant beans replacing any hard-coded values and injected `@Value` parameters used previously.

For example the swapchain can now be refactored as follows:

```java
class PresentationConfiguration {
    @Autowired private LogicalDevice dev;
    @Autowired private ApplicationConfiguration cfg;

    @Bean
    public Swapchain swapchain(Surface surface) {
        return new Swapchain.Builder(dev, surface)
            .count(cfg.getFrameCount())
            .clear(cfg.getBackground())
            .build();
    }
}
```

---

## Synchronisation

### Semaphores

So far we have avoided synchronisation by simply blocking the work queues after rendering and presentation of a frame.  However Vulkan is designed to be multi-threaded from the ground up, in particular the following methods are asynchronous operations:

* Acquiring the next swapchain image

* Submitting a render task to the work queue

* Presentation of a rendered frame

All of these methods return immediately with the actual work queued for execution in the background.

The Vulkan API provides several synchronisation mechanisms that can be used by the application, a _semaphore_ is the simplest of these and is used to synchronise operations within or across work queues.

The class itself is trivial since semaphores do not have any public functionality:

```java
public class Semaphore extends VulkanObject {
    private Semaphore(Pointer handle, DeviceContext dev) {
        super(handle, dev);
    }

    @Override
    protected Destructor<Semaphore> destructor(VulkanLibrary lib) {
        return lib::vkDestroySemaphore;
    }
}
```

A semaphore is created using a factory method:

```java
public static Semaphore create(DeviceContext dev) {
    VkSemaphoreCreateInfo info = new VkSemaphoreCreateInfo();
    VulkanLibrary lib = dev.library();
    PointerByReference handle = dev.factory().pointer();
    VulkanLibrary.check(lib.vkCreateSemaphore(dev, info, null, handle));
    return new Semaphore(handle.getValue(), dev);
}
```

Two semaphores are created in the constructor of the render task to signal the following conditions:

1. An acquired swapchain image is `available` for rendering.

2. A frame has been rendered and is `ready` for presentation.

The _available_ semaphore is passed to the acquire method:

```java
int index = swapchain.acquire(available, null);
```

And _ready_ is used when presenting the frame:

```java
swapchain.present(queue, index, ready);
```

The `present` method is modified to populate the relevant member of the descriptor:

```java
public void present(WorkQueue queue, int index, Semaphore semaphore) {
    ...
    info.waitSemaphoreCount = semaphores.size();
    info.pWaitSemaphores = NativeObject.array(List.of(semaphore));
}
```

Finally the semaphores are released when the task is destroyed:

```java
public void destroy() {
    available.destroy();
    ready.destroy();
}
```

### Work Submission

To integrate the semaphores the work class is extended by adding two new members:

```java
public class Work {
    private final Map<Semaphore, Set<VkPipelineStage>> wait = new LinkedHashMap<>();
    private final Set<Semaphore> signal = new HashSet<>();
}
```

Each entry in the `wait` table consists of a semaphore that must be signalled before the work can be performed and the stages(s) of the pipeline to wait on.
The `signal` member is the set of semaphores to be signalled when the work has completed.

The builder is modified to configure the semaphores:

```java
public Builder wait(Semaphore semaphore, Collection<VkPipelineStage> stages) {
    wait.put(semaphore, Set.copyOf(stages));
    return this;
}

public Builder signal(Semaphore semaphore) {
    signal.add(semaphore);
    return this;
}
```

And the populate method of the work class is updated to include the signals:

```java
info.signalSemaphoreCount = signal.size();
info.pSignalSemaphores = NativeObject.array(signal);
```

Population of the wait semaphores is slightly more complicated because the two components are separate fields, rather than an array of some child structure.  Therefore the table is a linked map to ensure that both fields are iterated in the same order.

First the array of semaphores is populated:

```java
info.waitSemaphoreCount = wait.size();
info.pWaitSemaphores = NativeObject.array(wait.keySet());
```

And then the list of pipeline stages for each semaphore:

```java
int[] stages = wait.values().stream().map(BitMask::new).mapToInt(BitMask::bits).toArray();
info.pWaitDstStageMask = new PointerToIntArray(stages);
```

Note that although `pWaitDstStageMask` implies this is a bitfield it is in fact a pointer to an integer array.

Here the `PointerToIntArray` helper class is introduced to transform the array of pipeline stages to a contiguous memory block:

```java
public class PointerToIntArray extends Memory {
    public PointerToIntArray(int[] array) {
        super(Integer.BYTES * array.length);
        for(int n = 0; n < array.length; ++n) {
            setInt(n * Integer.BYTES, array[n]);
        }
    }
}
```

Additional helpers are also implemented for arrays of floating-point values and pointers.  Surprisingly JNA does not provide these helpers (or where it does they are hidden for some reason).

The step to submit the render task is factored out to a local helper method and integrated with the semaphores:

```java
protected void submit(Buffer buffer) {
    new Work.Builder(buffer.pool())
        .add(buffer)
        .wait(available, VkPipelineStage.COLOR_ATTACHMENT_OUTPUT)
        .signal(ready)
        .build()
        .submit();
}
```

This should resolve the validation errors due to the semaphores never being signalled.

### Fence

However if one were to remove the `waitIdle` calls in the existing render loop the validation layer will again flood with errors, the command buffers are incorrectly being used concurrently for multiple frames.  Additionally the application is continually queueing up rendering work without checking whether it actually completes, which can be seen if one watches the memory usage.

To resolve both of these issues the second synchronisation mechanism is introduced which synchronises Vulkan and application code:

```java
public class Fence extends VulkanObject {
    @Override
    protected Destructor<Fence> destructor(VulkanLibrary lib) {
        return lib::vkDestroyFence;
    }
}
```

Again fences are created via a factory:

```java
public static Fence create(DeviceContext dev, VkFenceCreateFlag... flags) {
    // Init descriptor
    var info = new VkFenceCreateInfo();
    info.flags = new BitMask<>(flags);

    // Create fence
    VulkanLibrary lib = dev.library();
    PointerByReference handle = dev.factory().pointer();
    check(lib.vkCreateFence(dev, info, null, handle));

    // Create domain object
    return new Fence(handle.getValue(), dev);
}
```

Fences are signalled in the same manner as semaphores but can also be explicitly waited on by the application:

```java
public static void wait(DeviceContext dev, Collection<Fence> fences, boolean all, long timeout) {
    Pointer array = NativeObject.array(fences);
    VulkanLibrary lib = dev.library();
    check(lib.vkWaitForFences(dev, fences.size(), array, all, timeout));
}
```

Where _all_ specifies whether to wait for any or all of the supplied fences and _timeout_ is expressed in nanoseconds.

A signalled fence can also be reset:

```java
public static void reset(DeviceContext dev, Collection<Fence> fences) {
    Pointer array = NativeObject.array(fences);
    VulkanLibrary lib = dev.library();
    check(lib.vkResetFences(dev, fences.size(), array));
}
```

Convenience over-loads are also implemented for these methods:

```java
public void reset() {
    reset(device(), Set.of(this));
}

public void waitReady() {
    wait(device(), Set.of(this), true, Long.MAX_VALUE);
}
```

The state of the fence can also be programatically queried:

```java
public boolean signalled() {
    DeviceContext dev = this.device();
    VulkanLibrary lib = dev.library();
    VkResult result = lib.vkGetFenceStatus(dev, this);
    return switch(result) {
        case SUCCESS -> true;
        case NOT_READY -> false;
        default -> throw new VulkanException(result);
    };
}
```

A fence is created in the constructor of the render task:

```java
public class VulkanRenderTask {
    private final Fence fence;

    public VulkanRenderTask(DeviceContext dev) {
        this.fence = Fence.create(dev, VkFenceCreateFlag.SIGNALED);
    }
}
```

And the existing blocking code is replaced with a `waitReady` call on the fence to wait until the frame has been rendered.

Note that the swapchain could acquire frame buffers out of order, or even return the same buffer for consecutive frames.
Therefore a further blocking call is introduced at the _start_ of the render process to ensure that the _previous_ frame for a given buffer has been completed.
For this reason the fence is initialised to the `SIGNALED` state.

The updated `render` method now looks like this:

```java
public void render(RenderSequence seq) {
    // Wait for previous frame to complete
    fence.waitReady();
    fence.reset();

    // Acquire next frame buffer
    int index = swapchain.acquire(available, null);
    FrameBuffer fb = buffers.get(index);

    // Render frame
    Command.Buffer render = composer.compose(fb);
    submit(render);

    // Wait until frame has been completed
    fence.waitReady();

    // Present rendered frame
    WorkQueue queue = buffer.pool().queue();
    swapchain.present(queue, index, ready);
}
```

The demo should now run without validation errors (for the render loop anyway), however there are still further improvements that can be implemented.

### Subpass Dependencies

The final synchronisation mechanism is a _subpass dependency_ which specifies memory and execution dependencies between the stages of a render pass.

A subpass dependency is configured via two new mutable inner classes:

```java
public class Dependency {
    private Subpass dependency;
    private final Set<VkDependencyFlag> flags = new HashSet<>();
    private final Properties src = new Properties();
    private final Properties dest = new Properties();
}
```

Where _source_ and _destination_ specify the properties of the dependency:

```java
public class Properties {
    private final Set<VkPipelineStage> stages = new HashSet<>();
    private final Set<VkAccess> access = new HashSet<>();
}
```

Note that the _destination_ of the dependency is implicitly the enclosing subpass.

The subpass domain type is modified to include the list of dependencies:

```java
public class Subpass {
    private final List<Dependency> dependencies = new ArrayList<>();

    public Dependency dependency() {
        var dependency = new Dependency();
        dependencies.add(dependency);
        return dependency;
    }
}
```

The source and destination subpass instances are referenced by _index_ in the same manner as the attachment references.
A hidden mutable `index` is added to the subpass class which is initialised in the `create` method of the render pass:

```java
int index = 0;
for(Subpass subpass : subpasses) {
    subpass.init(index++);
}
```

Next the subpass dependencies are added to the render pass:

```java
List<Dependency> dependencies = subpasses.stream().flatMap(Subpass::dependencies).toList();
info.dependencyCount = dependencies.size();
info.pDependencies = StructureCollector.pointer(dependencies, new VkSubpassDependency(), Dependency::populate);
```

Which are populated as follows:

```java
void populate(VkSubpassDependency info) {
    info.srcSubpass = index;
    info.dstSubpass = Subpass.this.index;
    info.srcStageMask = new BitMask<>(src.stages);
    info.srcAccessMask = new BitMask<>(src.access);
    info.dstStageMask = new BitMask<>(dest.stages);
    info.dstAccessMask = new BitMask<>(dest.access);
}
```

A subpass can also be dependant on the implicit subpass before or after the render pass:

```java
public Dependency external() {
    this.dependency = VK_SUBPASS_EXTERNAL;
    return this;
}
```

Where `VK_SUBPASS_EXTERNAL` is a synthetic subpass with a special case index (copied from the Vulkan header file):

```java
public class Dependency {
    private static final Subpass VK_SUBPASS_EXTERNAL = new Subpass();

    static {
        VK_SUBPASS_EXTERNAL.index = ~0;
    }
}
```

In the demo a dependency can now be configured on the implicit `external` subpass:

```java
Subpass subpass = new Subpass()
    .colour(attachment)
    .dependency()
        .external()
        .source()
            .stage(VkPipelineStage.COLOR_ATTACHMENT_OUTPUT)
            .build()
        .destination()
            .stage(VkPipelineStage.COLOR_ATTACHMENT_OUTPUT)
            .access(VkAccess.COLOR_ATTACHMENT_WRITE)
            .build()
        .build();
```

The _destination_ clause specifies that the subpass should wait for the colour attachment to be ready for writing, i.e. when the swapchain has finished using the image.

This should allow Vulkan to more efficiently use the multi-threaded nature of the pipeline.

### Frames In-Flight

The new render loop is still not fully utilising the pipeline since the rendering code is still essentially single-threaded, whereas Vulkan is designed to allow completed pipeline stages to be used to render the next frame in parallel.  Multiple _in flight_ frames are introduced to take advantage of this feature.

First the existing render task is 'inverted' by factoring out the acquire and presentation logic into a separate object that tracks the state of an in-flight frame:

```java
public class VulkanFrame implements TransientObject {
    private final Semaphore available, ready;
    private final Fence fence;
}
```

The acquire step is factored out into a separate method:

```java
public int acquire(Swapchain swapchain) {
    // Wait for completion of the previous frame
    fence.waitReady();

    // Retrieve frame buffer index
    int index = swapchain.acquire(available, null);

    // Ensure still waiting if the swapchain has been invalidated
    fence.reset();

    return index;
}
```

Note that the fence is only reset _after_ the acquire step which may fail with a `SwapchainInvalidated` exception, preventing a potential deadlock scenario.

The presentation step is factored out similarly:

```java
public void present(Command.Buffer render, int index, Swapchain swapchain) {
    // Submit render task
    submit(render);

    // Wait for render to be completed
    fence.waitReady();

    // Present completed frame
    WorkQueue queue = render.pool().queue();
    swapchain.present(queue, index, ready);
}
```

The render task now comprises an _array_ of in-flight frames:

```java
public class VulkanRenderTask implements TransientObject {
    private final VulkanFrame[] frames;
    private int next;

    @Override
    public void destroy() {
        for(var frame : frames) {
            frame.destroy();
        }
    }
}
```

And the render logic is modified to cycle through the frames on each iteration:

```java
public void render() {
    // Select next frame
    VulkanFrame frame = frames[next];

    // Acquire next frame buffer
    int index = frame.acquire(swapchain);
    FrameBuffer fb = buffers.get(index);

    // Compose render task
    Command.Buffer render = composer.compose(next, fb);

    // Present rendered frame
    frame.present(render);

    // Move to next frame
    if(++next >= frames.length) {
        next = 0;
    }
}
```

Multiple in-flight frames can now be executed in parallel, the introduced synchronisation better utilises the pipeline, and the overall rendering work is bounded.

Notes:

* The number of in-flight frames does not necessarily have to be the same as the number of swapchains images (though in practice this is generally the case).

* The render task is now a transient object and releases the synchronisation primitives for each frame on destruction.

---

## Rotation and Animation

### Normals

The rotation animation in the demo has been factored out to a frame listener component, however the logic itself is still hard-coded and mixes the animation logic, calculation of the various matrices and management of the uniform buffer.  Therefore we introduce an animation framework that separates these activities and provides reusable components for future demo applications.

First the vector class is modified to support normals (or unit vectors) which will be used by the new framework.

Many geometric operations assume that a vector has been _normalized_ to _unit length_ with possibly undefined results if the assumption is invalid.  This responsibility is left to the application which can use the following method to normalize a vector as required:

```java
public Vector normalize() {
    float len = magnitude();
    if(MathsUtil.isEqual(1, len)) {
        return this;
    }
    else {
        float f = MathsUtil.inverseRoot(len);
        return multiply(f);
    }
}
```

Where `multiply` scales a vector by a given value:

```java
public Vector multiply(float f) {
    return new Vector(x * f, y * f, z * f);
}
```

A vector has a _magnitude_ (or length) which is calculated using the _Pythagorean_ theorem as the square-root of the _hypotenuse_ of the vector.  Although square-root operations are generally delegated to the hardware and are therefore less expensive than in the past, we prefer to avoid having to perform roots where possible.  Additionally many algorithms work irrespective of whether the distance is squared or not.

Therefore the `magnitude` is expressed as the __squared__ length of the vector (which is highlighted in the documentation):

```java
/**
 * @return Magnitude (or length) <b>squared</b> of this vector
 */
public float magnitude() {
    return x * x + y * y + z * z;
}
```

Note that the vector class is immutable and all 'mutator' methods create a new instance.

### Rotations

Next a new abstraction is introduced for a general transformation implemented by a matrix:

```java
@FunctionalInterface
public interface Transform {
    /**
     * @return Transformation matrix
     */
    Matrix matrix();
}
```

With the following intermediate specialisation for rotation transforms (a marker interface for the time being):

```java
public interface Rotation extends Transform {
}
```

Note that a matrix is itself a transform:

```java
public Matrix {
    @Override
    public final Matrix matrix() {
        return this;
    }
}
```

The simplest rotation implementation is an axis-angle which specifies a counter-clockwise rotation about a given normal:

```java
public class AxisAngle implements Rotation {
    private final Vector axis;
    private final float angle;
    
    @Override
    public Matrix matrix() {
        ...
    }
}
```

### Quaternions

The existing matrix code generates a rotation matrix for the cardinal axes, however the new framework needs to support rotations about an arbitrary axis (which is currently explicitly disallowed).  Implementing code to generate a rotation matrix about an arbitrary axis is relatively straight-forward (if somewhat messy) but a better alternative is to introduce _quaternions_ which also offer additional functionality that will be required in later chapters.

A _quaternion_ is a more compact and efficient representation of a rotation often used when multiple rotations are frequently composed (e.g. skeletal animation), but is generally less intuitive to use and comprehend:

```java
public final class Quaternion implements Rotation {
    public final float w, x, y, z;

    /**
     * @return Magnitude <b>squared</b> of this quaternion
     */
    public float magnitude() {
        return w * w + x * x + y * y + z * z;
    }

    public Matrix matrix() {
        ...
    }
}
```

A quaternion can be constructed from an axis-angle using the following factory:

```java
public static Quaternion of(AxisAngle rot) {
    float half = rot.angle() * MathsUtil.HALF;
    Vector vec = rot.axis().multiply(MathsUtil.sin(half));
    return new Quaternion(MathsUtil.cos(half), vec.x, vec.y, vec.z);
}
```

And converted back to an axis-angle in the inverse operation:

```java
@Override
public AxisAngle toAxisAngle() {
    float scale = MathsUtil.inverseRoot(1 - w * w);
    float angle = 2 * MathsUtil.acos(w);
    Vector axis = new Vector(x, y, z).multiply(scale);
    return new AxisAngle(axis, angle);
}
```

The rotation matrix for an axis-angle rotation can now be constructed from a quaternion:

```java
public Matrix matrix() {
    return Quaternion.of(this).matrix();
}
```

### Player

The final enhancement to the existing code is support for playable media and animations.

First the following new abstraction defines a media resource or animation that can be played:

```java
public interface Playable {
    enum State {
        STOP,
        PLAY,
        PAUSE
    }

    default boolean isPlaying() {
        return state() == State.PLAY;
    }

    State state();

    void apply(State state);
}
```

Which is partially implemented by the following skeleton:

```java
public abstract class AbstractPlayable implements Playable {
    private State state = State.STOP;

    @Override
    public void apply(State state) {
        this.state = notNull(state);
    }
}
```

A playable resource is managed by the _player_ controller:

```java
public class Player extends AbstractPlayable {
    @FunctionalInterface
    public interface Listener {
        void update(Player player);
    }

    private Playable playable;
    private final Collection<Listener> listeners = new HashSet<>();
}
```

Note that a player is itself a `Playable` and delegates state changes to the underlying playable resource:

```java
public void apply(State state) {
    super.apply(state);
    playable.apply(state);
    update();
}
```

Where `update` notifies interested observers of any state changes:

```java
private void update() {
    for(Listener listener : listeners) {
        listener.update(this);
    }
}
```

A playable resource may stop playing in the background, for example:

* OpenAL audio generally runs on a separate thread and stops playing at the end of a non-looping clip.

* A non-repeating animation terminates when the animation duration expires.

This case is handled by testing the underlying playable when querying the player state:

```java
public State state() {
    State state = super.state();
    if((state == State.PLAY) && !playable.isPlaying()) {
        super.apply(State.STOP);
        update();
        return State.STOP;
    }
    return state;
}
```

### Animation

Next the `Animator` is introduced which interpolates an _animation_ position over a given duration:

```java
public class Animator extends AbstractPlayable implements Frame.Listener {
    @FunctionalInterface
    public interface Animation {
        void update(float pos);
    }

    private final Animation animation;
    private final long duration;
    private long time;
    private float speed = 1;
    private boolean repeat = true;

    @Override
    public void update(Frame frame) {
        ...
    }
}
```

The animator is a playable object, therefore the `update` method ignores frame events if the animation is not running:

```java
public void update() {
    if(!isPlaying()) {
        return;
    }
    ...
}
```

Next the _time position_ of the animation is calculated:

```java
time += frame.elapsed().toMillis() * speed;
```

At the end of the duration the `time` is quantised or the animation stops if it is not repeating:

```java
if(time > duration) {
    if(repeat) {
        time = time % duration;
    }
    else {
        time = duration;
        apply(Playable.State.STOP);
    }
}
```

And the updated position is then applied to the animation:

```java
animation.update(time / (float) duration);
```

The final piece of the jigsaw is a mutable rotation implementation that can also be animated.

```java
public class MutableRotation implements Rotation {
    private AxisAngle rot;

    @Override
    public Matrix matrix() {
        return rot.matrix();
    }

    @Override
    public AxisAngle toAxisAngle() {
        return rot;
    }
}
```

This class provides various mutators (not shown) to set the axis and rotation angle which are used by the following animation adapter:

```java
public Animation animation() {
    return pos -> {
        float angle = pos * Trigonometric.TWO_PI;
        set(angle);
    };
}
```

### Integration

In the cube demo the existing hand-crafted matrix code is replaced by a rotation animation:

```java
@Bean
static MutableRotation rotation() {
    Vector axis = new Vector(MathsUtil.HALF, 1, 0).normalize();
    return new MutableRotation(axis);
}
```

The animation and timing logic is replaced by an animator:

```java
@Bean
static Animator animator(MutableRotation rot, ApplicationConfiguration cfg) {
    return new Animator(rot.animation(), cfg.getPeriod());
}
```

Which is controlled by a player:

```java
@Bean
public static Player player(Animator animator) {
    Player player = new Player(animator);
    player.apply(Playable.State.PLAY);
    return player;
}
```

And finally the code to update the uniform buffer is refactored accordingly (for the non-instanced version):

```java
public static Frame.Listener update(ResourceBuffer uniform, Matrix projection, Matrix view, MutableRotation rot) {
    ByteBuffer bb = uniform.buffer();
    return frame -> {
        Matrix model = rot.rotation().matrix();
        Matrix matrix = projection.multiply(view).multiply(model);
        matrix.buffer(bb);
        bb.rewind();
    };
}
```

---

## Summary

In this chapter the render loop was improved by:

- Factoring out the various aspects of the application and render loops into separate, reusable components.

- Integration of the GLFW window event queue and a means of gracefully terminating the demo.

- Implementation of Vulkan synchronisation to safely utilise the multi-threaded rendering pipeline.

- New geometry code to support unit-vectors, rotations and quaternions.

- The addition of a new framework for playable media and animations.

This might appear a lot of work for little benefit, however we now a basis that can be more easily extended in future chapters, in particular when input devices and scene graphs are introduced in later chapters.
