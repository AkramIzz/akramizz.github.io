Bindings are the glue between the flutter engine and the framework layers,
the engine is responsible of calling bindings on relevent events.

rendering a frame begins with the SchedulerBinding, when the window is ready for a new frame it calls SchedulerBinding. A frame is rendered by the following:

1. engine calls handleBeginFrame on SchedulerBinding is called. All active animations tick at this point.
2. microtasks are run (ticker and controller futures that have completed)
3. engine calls handleDrawFrame on SchedulerBinding, that calls WidgetsBinding.drawFrame Which invokes the build/layout/paint pipeline to run.
4. post frame callbacks are invoked.

The BuildOwner and PipelineOwner:
WidgetsBinding holds a reference to a BuildOwner, this object manages the widgets tree, by keeping track of the widgets that need rebuilding, and the inactive elements of each build (drawFrame) and providing the operations for rebuilding and finalizing the tree.
RendererBinding holds a reference to a PipelineOwner, this object manages the renderObjects by keeping track of the renderObjects that needs to be visited for layouting, and/or painting. It also provides an interface for driving the rendering pipeline, which include relayouting, repainting, and semantics updating for the renderObjects.

The rendering pipeline
the pipeline build and updates trees that at the end result a scene which is rendered to the screen. These trees are:

- The widgets tree (not really kept in memory)
- The elements tree
- The renderObjects tree
- The layer tree (the one that's converted into a scene)
- The semantics node tree (not part of the scene generating trees)

These trees are built and updated in step 3 above. Expanding on step 3, the steps of the rendering pipeline are:

1. build/update the widgets tree -> elements tree
2. build/update the renderObjects tree
3. layout the renderObjects tree
4. paint the renderObjects tree -> layer tree
5. send frame to engine
6. build/update the semantics tree
7. send semantics to engine

The Elements Tree and The RenderObjects Tree:
the main trees are the elements and renderObjects trees.
The elements tree is built from the widgets tree returned by the build method of each widget. the elements tree holds the widget that created it, the state of the widget, and it creates its renderObject.
The renderObjects tree is the tree used for layouting, painting, and hit testing.

for each widget there's an element, and for some elements that need to draw to the screen there's a render object. A widget acts as a configuration for an element.

The BuildContext instance passed in widget's build method is the parent element, it's used as a handle to a location in the elements tree

The initialization of the framework begins when runApp is called. This results in the following:

1. Bindings are initialized. These are GestureBinding, ServicesBinding, SchedulerBinding, PaintingBinding, SemanticsBinding, RendererBinding, and WidgetsBinding.
2. A frame is scheduled to run as soon as possible. When the frame is run, handleBeginFrame and then handleDrawFrame are called.
3. handleBeginFrame calls all the registered transient callbacks. These mainly are the Ticker and AnimationControllers callbacks.
4. handleDrawFrame calls all the registered persistent callback. These include the rendering pipeline drivers callbacks, the WidgetsBinding.drawFrame and RendererBinding.drawFrame, finally it calls the post frame callbacks
5. scheduleFrame is called, which is similar to step 2 except that the frame is scheduled to run when the engine deems suitable. Again when the frame is run handleBeginFrame and handleDrawFrame are called.

# State, Dirty Elements and Frame scheduling
- setState mark the element dirty
- this adds the element to the buildOwner's dirty list and calls onBuildScheduled callback. It's initialized by the WidgitsBinding to call _handleBuildScheduled
- _handleBuildScheduled in turn schedules a frame.