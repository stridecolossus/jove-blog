### Buttons

The final type of event (for now) is a _button_ that represents keyboard keys, mouse buttons, joystick hats, etc:

```java
public interface Button extends Event {
    /**
     * Button name delimiter.
     */
    String DELIMITER = "-";

    /**
     * @return Button identifier
     */
    String id();

    /**
     * @return Button name
     */
    String name();

    /**
     * @return Button action
     */
    Object action();
    
    /**
     * Resolves this button to the given action.
     * @param action Action
     * @return Resolved button
     */
    Button resolve(int action);
}
```

Which has the following skeleton implementation:

```java
abstract class AbstractButton implements Button {
    protected final String id;

    @Override
    public final Object type() {
        return this;
    }
}
```

Notes:

* A button also has an arbitrary _action_ object which is used below.

* A button is also its own event type since it has no additional arguments (unlike the other types of event).

* The purpose of the `resolve` method is explained later.

A simple button is represented by the following concrete implementation:

```java
public class DefaultButton extends AbstractButton {
    private final Action action;

    public DefaultButton(String id) {
        this(id, Action.PRESS);
    }

    protected DefaultButton(String id, Action action) {
        super(id);
        this.action = notNull(action);
    }

    @Override
    public String name() {
        return Button.name(id, action);
    }

    @Override
    public Action action() {
        return action;
    }

    @Override
    public DefaultButton resolve(int action) {
        return new DefaultButton(id, Action.map(action));
    }
}
```

Where _action_ is a simple enumeration (based on the GLFW action codes):

```java
public enum Action {
    RELEASE,
    PRESS,
    REPEAT;

    private static final Action[] ACTIONS = Action.values();

    public static Action map(int action) {
        return ACTIONS[action];
    }
}
```

The _name_ of the button is built using the following helper to construct a hyphen-delimited string representation of the button:

```java
public static String name(Object... tokens) {
    return Arrays
        .stream(tokens)
        .map(String::valueOf)
        .collect(joining(DELIMITER));
}
```

Note that the overridden `action` accessor returns the `Action` enumeration constant rather than `Object`.

Finally this class is extended for buttons with a keyboard modifiers mask:

```java
public class ModifiedButton extends DefaultButton {
    private final int mods;

    public ModifiedButton(String id) {
        this(id, Action.PRESS, 0);
    }

    protected ModifiedButton(String id, Action action, int mods) {
        super(id, action);
        this.mods = mods;
    }

    @Override
    public String name() {
        if(mods == 0) {
            return super.name();
        }
        else {
            Set<Modifier> set = modifiers();
            String str = Button.name(set.toArray());
            return Button.name(super.name(), str);
        }
    }
}
```

Where the modifiers is another enumeration (again using the GLFW values):

```java
public enum Modifier implements IntEnum {
    SHIFT(0x0001),
    CONTROL(0x0002),
    ALT(0x0004),
    SUPER(0x0008),
    CAPS_LOCK(0x0010),
    NUM_LOCK(0x0020)
}
```

We also provide a convenience `resolve` variant for modified buttons:

```java
@Override
public ModifiedButton resolve(int action) {
    return resolve(action, mods);
}

/**
 * Helper - Resolves this button for the given action and keyboard modifiers mask.
 * @param action        Action code
 * @param mods          Keyboard modifiers mask
 * @return Resolved button
 */
public ModifiedButton resolve(int action, int mods) {
    return new ModifiedButton(id, Action.map(action), mods);
}
```

The event source implementation for the mouse device first generates the list of buttons:

```java
private class MouseButton extends DesktopSource<MouseButtonListener> {
    private final List<Button> buttons = IntStream
        .rangeClosed(1, MouseInfo.getNumberOfButtons())
        .mapToObj(id -> Button.name("Mouse", id))
        .map(ModifiedButton::new)
        .toList();
}
```

Surprisingly GLFW does not seem to provide any means of determining the number of mouse buttons supported by the hardware, for the moment we use an AWT method.  However this means that we need to override the default Spring application behaviour which creates a _headless_ application by default (otherwise the AWT method throws an exception):

```java
new SpringApplicationBuilder(ModelDemo.class)
    .headless(false)
    .run(args);
```

The mouse buttons event source looks up a button by index and uses the `resolve` method to apply the action and keyboard modifiers:

```java
private class MouseButton extends DesktopSource<MouseButtonListener> {
    private final List<ModifiedButton> buttons = ...

    @Override
    protected MouseButtonListener listener(Consumer<Event> handler) {
        return (ptr, index, action, mods) -> {
            ModifiedButton button = buttons.get(index);
            Button event = button.resolve(action, mods);
            handler.accept(event);
        };
    }

    @Override
    protected BiConsumer<Window, MouseButtonListener> method(DesktopLibrary lib) {
        return lib::glfwSetMouseButtonCallback;
    }
}
```

The GLFW library is extended accordingly:

```java
interface MouseButtonListener extends Callback {
    /**
     * Notifies a mouse button event.
     * @param window    window
     * @param button    Button index 0..n
     * @param action    Button action
     * @param mods      Modifiers
     */
    void button(Pointer window, int button, int action, int mods);
}

/**
 * Registers a mouse button listener.
 * @param window        Window
 * @param listener      Mouse button listener
 */
void glfwSetMouseButtonCallback(Window window, MouseButtonListener listener);
```

### Keyboard

The final device we will implement in this chapter is the GLFW keyboard:

```java
public class KeyboardDevice extends DesktopDevice {
    private final KeyboardSource keyboard = new KeyboardSource();

    @Override
    public Set<Source> sources() {
        return Set.of(keyboard);
    }
}
```

The keyboard source caches button definitions:

```java
private class KeyboardSource extends DesktopSource<KeyListener> {
    @Override
    protected KeyListener listener(Consumer<Event> handler) {
        return (ptr, key, scancode, action, mods) -> {
            ModifiedButton base = keys.computeIfAbsent(key, this::key);
            Button button = base.resolve(action, mods);
            handler.accept(button);
        };
    }
    
    @Override
    protected BiConsumer<Window, KeyListener> method(DesktopLibrary lib) {
        return lib::glfwSetKeyCallback;
    }
}
```

Where new key definitions are created by the following helper:

```java
private ModifiedButton key(int code) {
    return new ModifiedButton(table.name(code));
}
```

The `table` maps GLFW key codes to key names specified by a resource file (loader not shown) which is a simple text file illustrated in the following fragment:

```
SPACE              32
APOSTROPHE         39
COMMA              44
```

Notes:

* GLFW also provides a _scancode_ argument and the `glfwGetKeyName` API method but this functionality only seems to support a subset of the expected keys.

* The key table is hidden but we provide a factory method to lookup keys by name.

* We add the lazily-instantiated keyboard device to the GLFW window.

Finally the GLFW library is extended to support the keyboard source:

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

/**
 * Registers a key listener.
 * @param window        Window
 * @param listener      Key listener
 */
void glfwSetKeyCallback(Window window, KeyListener listener);
```

---

## Action Bindings

### Overview

Using this framework we could now bind event sources to an action handler, however there are still several problems:

* Each type of event is a separate class so application code would be forced to cast events to access the sub-class properties.

* Ideally we would prefer to bind _parameterized_ events rather than having to implement switching logic (especially for the keyboard).

* Additionally we would also like to be able to bind directly to a method reference with an appropriate signature instead of hand-crafting adapters or messy lambdas.

We _could_ refactor the event class to contain the properties for __all__ cases but this is pretty ugly (even if there are only a handful).  Alternatively we could implement some sort of double-dispatch to transform a base-class event to its sub-type but that also feels overkill in this case.

Instead we will introduce an _action bindings_ class which is responsible for mapping events to handlers (or methods) and encapsulates any switching logic and type casting.

### Bindings

The action bindings is essentially a bi-directional mapping of events to/from handlers:

```java
public class ActionBindings implements Consumer<Event> {
    private final Map<Object, Consumer<Event>> bindings = new HashMap<>();
    private final Map<Consumer<? extends Event>, Set<Object>> handlers = new HashMap<>();
}
```

Note that the bindings class is itself an event handler.

The event `type` discriminator is used as the _key_ for the bindings mapping:

```java
@Override
public void accept(Event e) {
    Consumer<Event> handler = bindings.get(e.type());
    if(handler != null) {
        handler.accept(e);
    }
}
```

We add accessors to retrieve the handler for a given type of event:

```java
public Optional<Consumer<? extends Event>> binding(Object type) {
    return Optional.ofNullable(bindings.get(type));
}
```

And to retrieve the reverse mapping of event types for a given handler:

```java
public Stream<Object> bindings(Consumer<? extends Event> handler) {
    return handlers.get(handler).stream();
}
```

We also add support (not shown) to remove bindings by handler or event type and to clear all bindings.

### Binding Support

The following generic method binds an arbitrary event type to/from an event handler:

```java
private <T extends Event> void bindLocal(Object type, Consumer<? extends T> handler) {
    // Lookup or create reverse mapping
    Set<Object> types = handlers.computeIfAbsent(handler, ignored -> new HashSet<>());

    // Add binding
    @SuppressWarnings("unchecked")
    Consumer<Event> consumer = (Consumer<Event>) handler;
    bindings.put(type, consumer);
    types.add(type);
}
```

Note that this method is type-safe at compile-time but down-casts the handler to the base-class event.

This helper is used to bind an arbitrary event _source_ to a handler:

```java
public <T extends Event> void bind(Source<T> src, Consumer<T> handler) {
    bindLocal(src, handler);
    src.bind(this);
}
```

Note that we also automatically bind the event source to the bindings.

Next we implement convenience methods to bind specific types of event to methods with the appropriate signature.

Generally button events will either be bound to a simple `void` method without parameters:

```java
public void bind(Button button, Runnable handler) {
    Consumer<Button> adapter = ignored -> handler.run();
    bindLocal(button.type(), adapter);
}
```

Or a handler that toggles some property:

```java
public void bind(Button button, Button.ToggleHandler handler) {
    Consumer<Button> adapter = event -> handler.handle(event.action() == Action.PRESS);
    bindLocal(button.type(), adapter);
}
```

Where a _toggle handler_ is defined as follows in the `Button` class:

```java
@FunctionalInterface
interface ToggleHandler {
    void handle(boolean pressed);
}
```

For position events we define the following functional interface:

```java
public class PositionEvent {
    @FunctionalInterface
    public interface Handler {
        void handle(float x, float y);
    }
}
```

Which is used in a second bind variant for position events:

```java
public void bind(Source<PositionEvent> src, PositionEvent.Handler handler) {
    Consumer<PositionEvent> adapter = pos -> handler.handle(pos.x(), pos.y());
    bind(src, adapter);
}
```

Finally we define a similar bind variant and handler abstraction for an axis event:

```java
interface Axis {
    @FunctionalInterface
    public interface Handler {
        void handle(float value);
    }
}
```

### Modified Buttons

The final piece of functionality in the bindings class is support for _button templates_ to implement event switching logic:

1. Find the exact matching binding for a button event and its keyboard modifiers.

2. Or find the binding that matches just the button (ignoring the modifiers).

3. Otherwise ignore the event.

This logic handles the cases for an event where the modifier mask is irrelevant (i.e. a _default_ button) or where bindings are present for both a default button __and__ a _modified button_ binding.

Note that we assume that applications would not require matching by button action (i.e. the switching logic only applies to the keyboard modifiers) or would simply bind the entire keyboard device, e.g. for a text editor.

To support button template the button interface is first modified with a match test:

```java
public interface Button extends Event {
    /**
     * Matches the given button against this template.
     * @param button Button
     * @return Whether matches this template
     */
    boolean matches(Button button);
}
```

The skeleton implementation matches the button identifier:

```java
public boolean matches(Button button) {
    return id.equals(button.id());
}
```

And the default button also matches the action:

```java
public boolean matches(Button button) {
    return super.matches(button) && action.equals(button.action());
}
```

Finally the modified button implementation matches the keyboard modifiers mask:

```java
public boolean matches(Button button) {
    return
            super.matches(button) &&
            (button instanceof ModifiedButton that) &&
            MathsUtil.isMask(that.mods, this.mods);
}
```

Note that this test matches the sub-set of the modifiers in the template.  For example, a button event with `SHIFT` and `CONTROL` modifiers is matched by a template with just `SHIFT`.

In the bindings class the bind variants for buttons delegate to the following helper which applies the match test:

```java
private void bindButton(Button button, Consumer<Button> handler) {
    final Consumer<Button> wrapper = event -> {
        if(button.matches(event)) {
            handler.accept(event);
        }
    };
    bindLocal(button.type(), wrapper);
}
```

Finally the `accept` method is modified to hand off to the second use-case for modified buttons:

```java
public void accept(Event e) {
    final Consumer<Event> handler = bindings.get(e.type());
    if(handler == null) {
        if(e instanceof ModifiedButton mod) {
            accept(mod);
        }
    }
    else {
        handler.accept(e);
    }
}
```

The new method invokes the _default_ button binding (if present) by essentially stripping the modifier mask:

```java
private void accept(ModifiedButton button) {
    final Button def = new DefaultButton(button.id(), button.action());
    final Consumer<Event> handler = bindings.get(def.type());
    if(handler == null) {
        return;
    }
    handler.accept(def);
}
```

The new event handling framework and the action bindings class should satisfy the requirements we identified at the start of the chapter:

* The GLFW specifics are now largely abstracted away (and theoretically could be replaced by an alternative implementation).

* The event-handling logic is separated from the application code, e.g. binding the ESCAPE key to an arbitrary stop method.

* Binding events to handlers or methods is relatively concise and does not require any casting or switching logic.

There are still further use-cases that will be implemented in later chapters:

* The remaining event types, e.g. window enter/leave.

* Devices using GLFW query methods such as a joystick.

* Multi-position buttons such as joystick hats and gamepad controllers.

* A persistence mechanism for the action bindings.

---

## Camera Controller

### Default Controller

The camera is a simple model class, to implement richer functionality a _camera controller_ is introduced that can be bound to input actions.

A basic _free-look_ controller is first implemented that rotates the scene about the cameras position:

```java
public class DefaultCameraController {
    private final Camera cam;
    private final Dimensions dim;
    
    public void update(float x, float y) {
        ...
    }
}
```

The `update` method accepts an X-Y coordinate relative to the specified viewport dimensions, i.e. the mouse pointer location.

The view direction of the free-look camera is determined as follows:

1. Map the coordinates to yaw-pitch angles.

2. Calculate a point on the unit-sphere given these angles.

3. Point the camera in the direction of the calculated point.

To transform the coordinates to yaw-pitch angles the _interpolator_ class is introduced:

```java
@FunctionalInterface
public interface Interpolator {
    /**
     * Applies this interpolator to the given value.
     * @param value Value to be interpolated (assumes normalized)
     * @return Interpolated value
     */
    float interpolate(float t);
    
    static Interpolator linear(float start, float end) {
        float range = end - start;
        return t -> lerp(start, range, t);
    }

    static float lerp(float start, float range, float value) {
        return start + value * range;
    }
}
```

Two interpolator instances are added to the controller to transform in both directions:

```java
private final Interpolator horizontal = Interpolator.linear(0, TWO_PI);
private final Interpolator vertical = Interpolator.linear(-HALF_PI, HALF_PI);
```

Notes:

* The ranges of the interpolators are dependant on the algorithm to calculate the point on the unit-sphere (see below).

* The interpolator ranges in future may need to be mutable, i.e. currently we assume the ranges are linearly interpolated across the entire viewport.

* The interpolator class will be expanded with additional functionality in subsequent chapters.

### Unit Sphere

To determine the camera direction a new helper class is implemented to calculate a point on the unit-sphere given yaw-pitch angles:

```java
public final class Sphere {
    /**
     * Calculates the vector on the unit-sphere for the given rotation angles (in radians).
     * @param theta     Horizontal angle (or <i>yaw</i>) in the range zero to {@link MathsUtil#TWO_PI}
     * @param phi       Vertical angle (or <i>pitch</i>) in the range +/- {@link MathsUtil#HALF_PI}
     * @return Unit-sphere vector
     */
    public static Vector vector(float theta, float phi) {
        ...
    }
}
```

The implementation of the factory method is based on the standard algorithm for calculating a point on a sphere:

```java
float cos = cos(phi);
float x = cos(theta) * cos;
float y = sin(theta) * cos;
float z = sin(phi);
```

However the _coordinate space_ of the generated points is not aligned with the Vulkan system:

1. A horizontal (theta) angle of zero points in the direction of the X axis whereas we generally require the default to be the negative Z axis.

2. The Y and Z axes are transposed, i.e. Z is _up_ in the default implementation of the algorithm.

We assume that applications will always want to the resultant points in the default Vulkan space used elsewhere in JOVE, therefore we modify the default algorithm.

To make the camera point in the negative Z direction the _theta_ is fiddled with a 90 degree clockwise 'rotation':

```java
float angle = theta - MathsUtil.HALF_PI;
```

The axes are transposed by a swizzle of the resultant coordinates:

```java
return new Vector(x, z, y);
```

Finally the camera is pointed at the calculated point on the sphere:

```java
public void update(float x, float y) {
    float yaw = horizontal.interpolate(x / dim.width());
    float pitch = vertical.interpolate(y / dim.height());
    Vector vec = sphere.vector(yaw, pitch);
    cam.direction(vec);
}
```

### Orbital Controller

An _orbital_ (or arcball) camera controller rotates the view position _about_ a target point-of-interest.

The default controller is sub-classed to include a target position and _radius_ which is the distance from the camera:

```java
public class OrbitalCameraController extends DefaultCameraController {
    private Point target = Point.ORIGIN;
    private float radius = 1;

    public OrbitalCameraController(Camera cam, Dimensions dim) {
        super(cam, dim);
        cam.look(target);
    }
}
```

The algorithm to calculate the position of the camera is the same as the default controller except for the final step, which is therefore factored out to a local helper which can then be over-ridden:

```java
public void update(float x, float y) {
    ...
    Vector vec = sphere.vector(yaw, pitch);
    update(vec);
}

protected void update(Vector vec) {
    cam.direction(vec);
}
```

In the orbital implementation the camera is moved to the calculated point on the sphere and then pointed at the target:

```java
@Override
protected void update(Vector vec) {
    Point pos = new Point(vec).scale(radius).add(target);
    cam.move(pos);
    cam.direction(vec);
}
```

The orbital controller provides mutators (not shown) to set the radius and target, and to clamp the radius to a min/max range (to prevent the camera being moved to the target position).

The orbital camera can also be _zoomed_ towards or away from the target:

```java
public void zoom(float inc) {
    radius = MathsUtil.clamp(radius - inc * scale, min, max);
    Vector pos = cam.direction().multiply(radius);
    cam.move(target.add(pos));
}
```

And a `zoom` overload variant is added for an axis event.

Note that the camera controllers will still be subject to gimbal locking at the extreme edges of the viewport (see the previous chapter).

### Integration

To integrate the new event framework and camera controller the following is added to the camera configuration class:

```java
public class CameraConfiguration {
    private final Matrix projection;
    private final Camera cam = new Camera();
    private final OrbitalCameraController controller;

    public CameraConfiguration(Swapchain swapchain) {
        projection = Projection.DEFAULT.matrix(0.1f, 100, swapchain.extents());
        controller = new OrbitalCameraController(cam, swapchain.extents());
        controller.radius(3);
        controller.scale(0.25f);
    }
}
```

Next we add a new component to instantiate the action bindings:

```java
@Bean
public ActionBindings bindings(Window window, RenderLoop loop) {
    ActionBindings bindings = new ActionBindings();
    ...
    bindings.init();
    return bindings;
}
```

The ESCAPE key is bound to the `stop` method of the render loop:

```java
KeyboardDevice keyboard = window.keyboard();
bindings.bind(keyboard.key("ESCAPE"), loop::stop);
keyboard.bind(bindings);
```

And the mouse pointer and wheel are bound to the relevant methods on the controller:

```java
MouseDevice mouse = window.mouse();
bindings.bind(mouse.pointer(), controller::update);
bindings.bind(mouse.wheel(), controller::zoom);
```

Finally the matrix bean is modified to apply the local rotation to the chalet model and to build the final projection matrix:

```java
@Bean
public FrameListener matrix(ResourceBuffer uniform) {
    Matrix x = Rotation.matrix(Axis.X, MathsUtil.toRadians(90));
    Matrix y = Rotation.matrix(Axis.Y, MathsUtil.toRadians(-120));
    Matrix model = y.multiply(x);

    return (start, end) -> {
        Matrix matrix = projection.multiply(cam.matrix()).multiply(model);
        uniform.load(matrix);
    };
}
```

We should now be able to use the mouse to look around the chalet model and zoom in using the scroll wheel.  Nice.
