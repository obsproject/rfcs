# Summary

API similar to browser panels API to expose the internal implementation of an OBS frontend editable canvas for re-use by developers.

# Motivation

This should enable consistency in plugins which create additional canvases in OBS, and prevent OBS code from being duplicated in this type of plugins.

# Drawbacks

The drawbacks are an additional API to maintain.

# Additional Information

## API interface

```c++
class QOBSEditableCanvasView: public QWidget {
public:
    // When currentScene == nullptr, no scene editing facilities are present
    //
    // The class will add a reference to the currentScene which is passed in, and
    // release previously held reference when switching to a different currentScene.
    //
    virtual void setOBSRenderCallbackAndEditableScene(render_callback_t* renderCallback, obs_scene_t* currentScene = nullptr) = 0;
    
    // Release references to canvas and scene.
    virtual void clearOBSRenderCallbackAndEditableScene() = 0;
};

QOBSEditableCanvasView* obs_frontend_create_editable_canvas_view(render_callback_t* renderCallback, obs_scene_t* currentScene = nullptr);
```

## External render callback

We need an external render callback in order to support rendering different OBS objects to the canvas view surface (sources, canvases, views, etc.)

## Usage

When a component wishing to display an editable canvas initializes, it will call `obs_frontend_create_editable_canvas_view`
to obtain an instance pointer of `QOBSEditableCanvasView`.

From that point on, it will interact with the QWidget-based class as with any other QWidget, with the addition of specialized
methods for specifying current canvas & editable scene.

## Changes

This will require changes to `OBSProjector` (and maybe `OBSQTDisplay`) to support additional methods, and calling an external render callback.
