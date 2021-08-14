# Flitter: A Flutter Internals Walkthrough
Ever since I started using Flutter I fell in love with it. Compared to Android development, Flutter is a breeze of refreshing air. It's a simple yet a powerful solution for UI, and all the magic that happens inside of it to achieve this is so wonderful and elegant, that it intrigued me. At the core, Flutter follows a simple algorithm to layout, but it's strikingly efficient. All this in addition to simplicity of use makes Flutter seem like black magic, and demystifying that magic is my goal in this post.

We will be building a simplified Flutter clone. Of course, it won't have all the features of Flutter, in particular we will focus on the Flutter UI layouting and painting. This will be a a multipart post, so let's get started!

This part is an introduction to some of the basic building blocks of Flutter. It covers what Widgets, Elements, RenderObjects, and Layers are, and how the framework and engine communicate with each other.

## The Flutter Framework and Engine
In addition to all the tools that come with Flutter for dev experience, the Flutter toolkit consists of two main components, the Flutter framework and the Flutter engine.

The Flutter engine is the coordinator, the wheel that runs the mill. For now, all we care about in the engine is that it communicates with the framework through the Window object. This object as we will see in a moment, has a collection of functions that the engine calls to communicate with the framework, and a bunch of other functions that the framework uses to communicate with the engine.

The Flutter framework is where the UI fun takes place, it's where the layout and painting is performed from Widget all the way down to drawing commands that it hands to the engine for rendering (more details later). This will be our main focus.

## Everything is a Widget.. and an Element.. and sometimes a RenderObject
If you have ever programmed with Flutter, you know that as a user of the framework, you mostly deal with Widgets only. In that sense, Widgets are the interface of the framework, they are what the user of the framework uses to communicate with it. But internally Widgets are only part of the story.

Many refer to the hierarchial structure of Widgets defined by the user as the Widgets tree, but this isn't true. For one, the Widget class doesn't define a tree node interface. Here is the definition of the Widget class

```dart
@immutable
abstract class Widget {
  const Widget({this.key});

  final Key key;

  Element createElement();
}
```

That's it! The Widget interface itself doesn't define any type of relationships between Widgets and thus it doesn't form a tree. There is no children and no parent Widget. Widgets are also lightweight and immutable!

***
If you're wondering where the build method is, it's part of the StatelessWidget, and State interfaces. You may be tempted to think that the build method defines a children-parent relationship that's required for trees, but take a look at this. Can you get to the parent from the child?

```dart
final parent = MyStatelessWidget();
final child = parent.build();
```
***

So if Widgets don't form a tree, what are they?

Flutter uses a declarative UI. You describe what you want the UI to look like and Flutter handles the changing the current UI to look like your description. This in contrast to the imperative style, where you handle changing the UI from one state to another. This means immutability is in the core of Flutter approach. But immutable is slow, right?

Flutter aims to render at 60 frames per second, that's every frame should be rendered within 16ms.

Now imagine if Flutter used a naive approach where Widgets do actually form a tree. To display this tree on the screen, it would go to each Widget lay it out, paint it and render it to the screen, on each 16ms long frame. That's, on the next frame, it will discard any work it has done  on the previous frame, because Widgets are immutable thus if we want to change the UI we need to create new Widgets and do that process over again. Take into account that for most frames the pixels on the screen don't change at all, or just part of these pixels do.

It becomes very clear that it's a lost cause to try and maintain a 60fps rendering with that approach. We can't use immutable Widgets to carry all the process. We need a mutable tree for efficiency.

>> TODO insert using-widgets-tree image <<

Part of the way Flutter framework works is that it builds trees described by the Widgets. Generally, trees are very useful data structures for UI layout and painting algorithms, but Flutter uses trees to achieve its declarative UI in a performant manner too.

Instead of discarding all the work of layouting and painting done on the previous frame, internally Flutter maintains several trees that are mutable, we can change their children, their parent, and their state.

The first tree that gets created is the Elements tree. These Elements serve as a bridge between the Widgets description of the UI and another tree that's responsible of layout, painting and hit testing called the RenderObjects tree. Keep in mind these trees and their nodes are mutable unlike our hypothetical Widgets tree, and most of their nodes remain the same from frame to frame.

***
The Elements tree can, and most of the time is, bigger than the RenderObjects tree. In other words, not every Element corresponds to a RenderObject, but the opposite isn't true. This is because some Elements simply serve the purpose of composing other Elements. This will be more clear later.
***

***
Even though not all Elements are associated with a RenderObject, one can walk the Elements subtree below an Element looking for a RenderObjectElement. This is what `context.findRenderObject` does.
***

We still need the immutable widgets for the declarative UI. A Widget describes the configuration of an Element. Each Widget tells the framework what Element it corresponds to. That's what createElement is there for in the previous code snippet. The created element holds a reference to the widget that describes its configuration.

```dart
// the definition of the stateless widget class
class StatelessWidget extends Widget {
  const StatelessWidget({Key key}) : super(key: key);

 // we pass a reference to this widget instance
  Element createElement() => StatelessElement(this);

  // used by the stateless element instance to rebuild its child, its declared
  // here because it must be implemented by the user, otherwise the user will
  // have to extend the Element class.
  // it returns the **child** element's widget.
  Widget build(BuildContext context);
}
```

We won't get into the Element class definition now. We just need to know that it holds a reference to the Widget that created it. Its important to stress the fact that the widget itself doesn't have a reference to the Element.

The Widget reference held by an Element is subject to change. When we need to update a part of the UI, we can ask the framework to *rebuild* the parent Element of that part. Rebuilding the Element may result in updating its children's Widgets configurations by creating new Widgets since Widgets are immutable, and rebuilding those children, and updating the RenderObject if there's any associated with it. The result is that all the relevant parts are updated without needing to recreate the Elements tree or the RenderObjects tree.

>> TODO insert elements tree update image <<

***
It's not always the case that rebuilding an Element will simply update the children's widget configurations. Sometimes, recreating part of the Elements tree, and RenderObjects tree by association, is inevitable.
***

***
Rebuilding a StatelessElement, the Element created by a StatelessWidget, calls build on its Widget which returns a new Widget. This new Widget is used to update the StatelessElement's child, and is assigned to the child as its referenced widget.
***

The trees that the framework create and maintains are the Elements tree, the RenderObjects tree, the Layers tree, and the Semantics tree. Except the semantics tree, we'll cover each tree in the rest of this article.

## Bindings, Trees and the Widgets Layer
Even though the framework and engine communicate through the window object, most of the time the framework doesn't interact directly with it, but instead uses bindings. Bindings are the glue between the flutter engine and the framework layers. The reason is that the window object is part of the engine, and hence the framework needs to bind callbacks to the engine's window object.

Bindings also provide a nice abstraction for common operations. For example the SchedularBinding provides the `ensureVisualUpdates` method which schedules a frame by calling a method on the window object, but it only does that if no frame is scheduled so we don't have to worry about scheduling a frame while one is already scheduled.

Bindings are also the glue between the Flutter engine and the trees. Bindings own the root of these trees but handle them by delegating work to processes owners. For example when the engine requests that the framework draws a frame, the WidgetsBinding delegates building the Elements and RenderObjects tree to the BuildOwner, which owns the build process, then the RendererBinding delegates the layout and painting of the RenderObjects tree to the PipelineOwner, which owns the rendering pipeline process.

***
Building a tree doesn't strictly mean creating the tree, it could also be updating parts of it.
***

Lastly from the last example, we can see that the engine is responsible of calling bindings on relevant events.

In the flutter framework, there are several bindings mixins, namely WidgetsBinding, RendererBinding, SemanticsBinding, PaintingBinding, ServicesBinding, SchedulerBinding, and GestureBinding.

## Elements and State
Since Widgets are immutable and are recreated on each rebuild they can't hold state, because state in its nature is mutable, making the natural place to store state is within the Element.

In Flutter, we have StatefulWidgets, these are Widgets that describe the configuration of StatefulElements. In addition to the `createElement` method in Widget, they have a `createState` method. The StatefulElement is responsible of creating and managing the State object.

State objects have a setState method within them that, when called, causes the Element to rebuild, which calls the build method in the State object and update the child Element's Widget with the one returned by the build method.

## The BuildContext
So we have established that Widgets don't form a tree, Elements tree is the tree that rebuilds and updates internally, and the Elements tree serves as a bridge between the Widgets description of the UI and the RenderObjects tree.

So each Element is a node in the Elements tree and this node holds a reference to its widget configuration, the RenderObject it corresponds to if any, and the State it holds for StatefulElements. This makes the Element a valuable construct for inspecting the Elements tree and reach to their Widget, RenderObject, or State.

That's why build methods accept a BuildContext parameter. Can you guess? Yes a BuildContext is just an Element. Element class implements the BuildContext interface which defines many useful method like `findAncestorStateOfExactType<T>` which walks the Element tree searching for a StatefulElement that holds a State object of type T and returns it.

## RenderObjects
Elements alone can't do much when we want to paint our UI. Because Flutter uses the Elements tree for everything, they can't be directly used for layout and painting, since some Elements don't correspond to a part of the UI. That's why we need the RenderObjects tree. The Elements that has a corresponding RenderObjects for them are subclasses of the RenderObjectElement class.

RenderObjects tree facilitates layouting, painting and hit testing the UI. This allows for doing these processes more efficiently, since we don't need to walk the Elements tree, that parts of it aren't RenderObjectElements.

Essentially a RenderObject is a component of the UI that knows how to layout and paint itself. It could be a padding that sizes itself depending on its child, and paints itself by painting its child offsetted by the padding amount, or it could be a decorated box with a specific size, that paints itself by drawing a colored box.

When a RenderObjectElement updates its RenderObject, the RenderObject can tell the framework that it needs to re-layout or repaint itself. Most of the time, when a RenderObject changes its layout other RenderObjects in the tree must change their layout too. For example, in a Column where extra space is spread evenly between children, if one child changes its size the Column needs to adjust the dividend space between children. Children too sometimes need to adjust their sizes when the parent layout changes.

***
This also applies to painting, but the reason is not as obvious as layout. It involves another tree called the Layers tree.
***

## Painting and Layers trees
Imagine yourself painting a picture on a canvas. When you start the canvas is clean, it's completely blank. You then draw a portrait of a person and a background behind him. Now you're a bit unhappy with the result. You would like to move the person to the left side of the picture instead of the center, but the canvas can't be used again, and as such you will have to redraw the whole picture, even though you don't really need to redraw the person since you're just moving them to the side, nor do you need to repaint the background as it's not really changing (apart from the section that was obscured by the person).

This is tedious, you think to yourself, I might decide that the left side isn't the best place after all, what can I do so I don't have to repaint the background?

The solution as it turns out is actually simple, you buy two canvases. On one you draw the background with all the details, and on the second you draw the person. Now you can place the person on any place you want without redrawing any part of the picture.

You've just invented the Layers tree, where each canvas is a type of a container Layer that can be offsetted called an offset Layer. An offset Layer, contains other Layers at an offset. Certain RenderObjects called repaint boundaries when asked to repaint can use their existing offset Layer and its subtree instead of having to create them, and only change the offset the contained Layers are rendered at.

***
Same as with the Elements and RenderObjects trees, there isn't a one-to-one correspondence between RenderObjects and Layers. But unlike Elements-RenderObjects relationship, a RenderObject is always associated either with a Layer that was created for it, or with the Layer of the first ancestor that has one. In other words, multiple RenderObjects can draw on the same Layer.

In theory every single RenderObject could paint itself into a separate Layer, but this could be quite wasteful in terms of memory.
***

Why would a Layer have a subtree of Layers? Well, we only covered one of the uses of Layers. Some operations, like applying a backdrop filter, are far more efficient to be done on the GPU than on the CPU, and Layers wrap those operations. Another use of Layers is specifying regions for platform views to paint into. Layers also allow for transformations like rotation, scaling and translation, to take place in the GPU. These type of things require a separate Layer, and thus a tree of Layers where container Layers contain multiple leaf Layers, or other container Layer.

## Aside on Layers and GPU-accelerated rendering
A Canvas is a software construct, an interface for programmers to use. It provides commands to draw stuff, like drawing a line, drawing an image, drawing arcs, circles and other shapes, but it just records these commands, and after finishing it produces a Picture which just captures these recorded commands.

Layers are software constructs too that specify a certain visual element like a platform view, or a Picture, and they can be containers for other Layers and can apply some effects on their children.

Textures can be thought of as bitmap images stored in the GPU memory (VRAM). These textures can be mapped into a mesh geometry. Textures can be cheaply mapped to different positions and transformations by applying them to a really simple rectangular mesh, and hence they are very useful for 2D graphics, as we can create Layers and upload them to the GPU memory as textures. If the content of a Layer doesn't change we can reuse the texture counterpart already uploaded.

## The Big Picture (Summary)
We're all familiar with the runApp function. You provide it a Widget and it kick starts the rendering process. When runApp receives a Widget it wraps it in another Widget that represents the root of the widget tree.

It then builds the Element tree by recursively calling createElement on each Widget and mounting the resulting element, which calls createElement on the widget provided to it through build methods or an argument in the its Widget configuration.

***
Mounting an Element is just the process of adding an Element to the tree.
***

At the same time, when mounting a RenderObjectElement, the corresponding RenderObject is created and is inserted into it's RenderObject parent as a child. This continues until all the Element tree and RenderObject tree is constructed. Layers come at a later step when painting takes place and is more complex as when and how Layers are created, we will visit the Layer tree at a later part. 