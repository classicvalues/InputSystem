# Input events

* [Types of events](#types-of-events)
    * [State events](#state-events)
    * [Device events](#device-events)
    * [Text events](#text-events)
* [Working with events](#working-with-events)
    * [Listening to events](#listening-to-events)
    * [Reading state events](#reading-state-events)
    * [Creating events](#creating-events)
    * [Capturing events](#capturing-events)
* [Merging of events](#merging-of-events)

The Input System is event-driven. All input is delivered as events, and you can generate custom input by injecting events. You can also observe all source input by listening in on the events flowing through the system.

>__Note__: Events are an advanced, mostly internal feature of the Input System. Knowledge of the event system is mostly useful if you want to support custom Devices, or change the behavior of existing Devices.

Input events are a low-level mechanism. Usually, you don't need to deal with events if all you want to do is receive input for your app. Events are stored in unmanaged memory buffers and not converted to C# heap objects. The Input System provides wrapper APIs, but unsafe code is required for more involved event manipulations.

Note that there are no routing mechanism. The runtime delivers events straight to the Input System, which then incorporates them directly into the Device state.

Input events are represented by the [`InputEvent`](../api/UnityEngine.InputSystem.LowLevel.InputEvent.html) struct. Each event has a set of common properties:

|Property|Description|
|--------|-----------|
|[`type`](../api/UnityEngine.InputSystem.LowLevel.InputEvent.html#UnityEngine_InputSystem_LowLevel_InputEvent_type)|[`FourCC`](../api/UnityEngine.InputSystem.Utilities.FourCC.html) code that indicates what type of event it is.|
|[`eventId`](../api/UnityEngine.InputSystem.LowLevel.InputEvent.html#UnityEngine_InputSystem_LowLevel_InputEvent_eventId)|Unique numeric ID of the event.|
|[`time`](../api/UnityEngine.InputSystem.LowLevel.InputEvent.html#UnityEngine_InputSystem_LowLevel_InputEvent_time)|Timestamp of when the event was generated. This is on the same timeline as [`Time.realtimeSinceStartup`](https://docs.unity3d.com/ScriptReference/Time-realtimeSinceStartup.html).|
|[`deviceId`](../api/UnityEngine.InputSystem.LowLevel.InputEvent.html#UnityEngine_InputSystem_LowLevel_InputEvent_deviceId)|ID of the Device that the event targets.|
|[`sizeInBytes`](../api/UnityEngine.InputSystem.LowLevel.InputEvent.html#UnityEngine_InputSystem_LowLevel_InputEvent_sizeInBytes)|Total size of the event in bytes.|

You can observe the events received for a specific input device in the [input debugger](Debugging.md#debugging-devices).

## Types of events

### State events

A state event contains the input state for a Device. The Input System uses these events to feed new input to Devices.

There are two types of state events:

* [`StateEvent`](../api/UnityEngine.InputSystem.LowLevel.StateEvent.html) (`'STAT'`)
* [`DeltaStateEvent`](../api/UnityEngine.InputSystem.LowLevel.StateEvent.html) (`'DLTA'`)

[`StateEvent`](../api/UnityEngine.InputSystem.LowLevel.StateEvent.html) contains a full snapshot of the entire state of a Device in the format specific to that Device. The [`stateFormat`](../api/UnityEngine.InputSystem.LowLevel.StateEvent.html#UnityEngine_InputSystem_LowLevel_StateEvent_stateFormat) field identifies the type of the data in the event. You can access the raw data using the [`state`](../api/UnityEngine.InputSystem.LowLevel.StateEvent.html#UnityEngine_InputSystem_LowLevel_StateEvent_state) pointer and [`stateSizeInBytes`](../api/UnityEngine.InputSystem.LowLevel.StateEvent.html#UnityEngine_InputSystem_LowLevel_StateEvent_stateSizeInBytes).

A [`DeltaStateEvent`](../api/UnityEngine.InputSystem.LowLevel.DeltaStateEvent.html) is like a [`StateEvent`](../api/UnityEngine.InputSystem.LowLevel.StateEvent.html), but only contains a partial snapshot of the state of a Device. The Input System usually sends this for Devices that require a large state record, to reduce the amount of memory it needs to update if only some of the Controls change their state. To access the raw data, you can use the [`deltaState`](../api/UnityEngine.InputSystem.LowLevel.DeltaStateEvent.html#UnityEngine_InputSystem_LowLevel_DeltaStateEvent_deltaState) pointer and [`deltaStateSizeInBytes`](../api/UnityEngine.InputSystem.LowLevel.DeltaStateEvent.html#UnityEngine_InputSystem_LowLevel_DeltaStateEvent_deltaStateSizeInBytes). The Input System should apply the data to the Device's state at the offset defined by [`stateOffset`](../api/UnityEngine.InputSystem.LowLevel.DeltaStateEvent.html#UnityEngine_InputSystem_LowLevel_DeltaStateEvent_stateOffset).

### Device events

Device events indicate a change that is relevant to a Device as a whole. If you're interested in these events, it is usually more convenient to subscribe to the higher-level [`InputSystem.onDeviceChange`](../api/UnityEngine.InputSystem.InputSystem.html#UnityEngine_InputSystem_InputSystem_onDeviceChange) event rather then processing [`InputEvents`](../api/UnityEngine.InputSystem.LowLevel.InputEvent.html) yourself.

There are three types of Device events:

* [`DeviceRemoveEvent`](../api/UnityEngine.InputSystem.LowLevel.DeviceRemoveEvent.html) (`'DREM'`)
* [`DeviceConfigurationEvent`](../api/UnityEngine.InputSystem.LowLevel.DeviceConfigurationEvent.html) (`'DCFG'`)
* [`DeviceResetEvent`](../api/UnityEngine.InputSystem.LowLevel.DeviceResetEvent.html) (`'DRST'`)

`DeviceRemovedEvent` indicates that a Device has been removed or disconnected. To query the device that has been removed, you can use the common [`deviceId`](../api/UnityEngine.InputSystem.LowLevel.InputEvent.html#UnityEngine_InputSystem_LowLevel_InputEvent_deviceId) field. This event doesn't have any additional data.

`DeviceConfigurationEvent` indicates that the configuration of a Device has changed. The meaning of this is Device-specific. This might signal, for example, that the layout used by the keyboard has changed or that, on a console, a gamepad has changed which player ID(s) it is assigned to. You can query the changed device from the common [`deviceId`](../api/UnityEngine.InputSystem.LowLevel.InputEvent.html#UnityEngine_InputSystem_LowLevel_InputEvent_deviceId) field. This event doesn't have any additional data.

`DeviceResetEvent` indicates that a device should get reset. This will trigger [`InputSystem.ResetDevice`](../api/UnityEngine.InputSystem.InputSystem.html#UnityEngine_InputSystem_InputSystem_ResetDevice_UnityEngine_InputSystem_InputDevice_System_Boolean_) to be called on the Device.

### Text events

[Keyboard](Keyboard.md) devices send these events to handle text input. If you're interested in these events, it's usually more convenient to subscribe to the higher-level [callbacks on the Keyboard class](Keyboard.md#text-input) rather than processing [`InputEvents`](../api/UnityEngine.InputSystem.LowLevel.InputEvent.html) yourself.

There are two types of text events:

* [`TextEvent`](../api/UnityEngine.InputSystem.LowLevel.TextEvent.html) (`'TEXT'`)
* [`IMECompositionEvent`](../api/UnityEngine.InputSystem.LowLevel.IMECompositionEvent.html) (`'IMES'`)

## Working with events

### Listening to events

If you want to do any monitoring or processing on incoming events yourself, subscribe to the [`InputSystem.onEvent`](../api/UnityEngine.InputSystem.InputSystem.html#UnityEngine_InputSystem_InputSystem_onEvent) callback.

```CSharp
InputSystem.onEvent +=
   (eventPtr, device) =>
   {
       Debug.Log($"Received event for {device}");
   };
```

An [`IObservable`](https://docs.microsoft.com/en-us/dotnet/api/system.iobservable-1) interface is provided to more conveniently process events.

```CSharp
// Wait for first button press on a gamepad.
InputSystem.onEvent
    .ForDevice<Gamepad>()
    .Where(e => e.HasButtonPress())
    .CallOnce(ctrl => Debug.Log($"Button {ctrl} pressed"));
```

To enumerate the controls that have value changes in an event, you can use [`InputControlExtensions.EnumerateChangedControls`](../api/UnityEngine.InputSystem.InputControlExtensions.html#UnityEngine_InputSystem_InputControlExtensions_EnumerateChangedControls_UnityEngine_InputSystem_LowLevel_InputEventPtr_UnityEngine_InputSystem_InputDevice_System_Single_).

```CSharp
InputSystem.onEvent
    .Call(eventPtr =>
    {
        foreach (var control in eventPtr.EnumerateChangedControls())
            Debug.Log($"Control {control} changed value to {control.ReadValueFromEventAsObject(eventPtr)}");
    };
```

This is significantly more efficient than manually iterating over [`InputDevice.allControls`](../api/UnityEngine.InputSystem.InputDevice.html#UnityEngine_InputSystem_InputDevice_allControls) and reading out the value of each control from the event.

### Reading state events

State events contain raw memory snapshots for Devices. As such, interpreting the data in the event requires knowledge about where and how individual state is stored for a given Device.

The easiest way to access state contained in a state event is to rely on the Device that the state is meant for. You can ask any Control to read its value from a given event rather than from its own internally stored state.

For example, the following code demonstrates how to read a value for [`Gamepad.leftStick`](../api/UnityEngine.InputSystem.Gamepad.html#UnityEngine_InputSystem_Gamepad_leftStick) from a state event targeted at a [`Gamepad`](../api/UnityEngine.InputSystem.Gamepad.html).

```CSharp
InputSystem.onEvent +=
    (eventPtr, device) =>
    {
        // Ignore anything that isn't a state event.
        if (!eventPtr.IsA<StateEvent>() && !eventPtr.IsA<DeltaStateEvent>())
            return;

        var gamepad = device as Gamepad;
        if (gamepad == null)
        {
            // Event isn't for a gamepad or device ID is no longer valid.
            return;
        }

        var leftStickValue = gamepad.leftStick.ReadValueFromEvent(eventPtr);
    };
```

### Creating events

Anyone can create and queue new input events against any existing Device. Queueing an input event is thread-safe, which means that event generation can happen in background threads.

>__Note__: Unity allocates limited memory to events that come from background threads. If background threads produce too many events, queueing an event from a thread blocks the thread until the main thread flushes out the background event queue.

Note that queuing an event doesn't immediately consume the event. Event processing happens on the next update (depending on [`InputSettings.updateMode`](Settings.md#update-mode), it is triggered either manually via [`InputSystem.Update`](../api/UnityEngine.InputSystem.InputSystem.html#UnityEngine_InputSystem_InputSystem_Update), or automatically as part of the Player loop).

#### Sending state events

The easiest way to create a state event is directly from the Device.

```CSharp
// `StateEvent.From` creates a temporary buffer in unmanaged memory that holds
// a state event large enough for the given device and contains a memory
// copy of the device's current state.
InputEventPtr eventPtr;
using (StateEvent.From(myDevice, out eventPtr))
{
    ((AxisControl) myDevice["myControl"]).WriteValueIntoEvent(0.5f, eventPtr);
    InputSystem.QueueEvent(eventPtr);
}
```

Alternatively, you can send events for individual Controls.

```CSharp
// Send event to update leftStick on the gamepad.
InputSystem.QueueDeltaStateEvent(Gamepad.current.leftStick,
    new Vector2(0.123f, 0.234f);
```

Note that delta state events only work for Controls that are both byte-aligned and a multiple of 8 bits in size in memory. You can't send a delta state event for a button Control that is stored as a single bit, for example.

### Capturing Events

>NOTE: To download a sample project which contains a reusable MonoBehaviour called `InputRecorder`, which can capture and replay input from arbitrary devices, open the Package Manager, select the Input System Package, and choose the sample project "Input Recorder" to download.

You can use the [`InputEventTrace`](../api/UnityEngine.InputSystem.LowLevel.InputEventTrace.html) class to record input events for later processing:

```CSharp
var trace = new InputEventTrace(); // Can also give device ID to only
                                   // trace events for a specific device.

trace.Enable();

//... run stuff

var current = new InputEventPtr();
while (trace.GetNextEvent(ref current))
{
    Debug.Log("Got some event: " + current);
}

// Also supports IEnumerable.
foreach (var eventPtr in trace)
    Debug.Log("Got some event: " + eventPtr);

// Trace consumes unmanaged resources. Make sure to dispose.
trace.Dispose();
```

Dispose event traces after use, so that they do not leak memory on the unmanaged (C++) memory heap.

You can also write event traces out to files/streams, load them back in, and replay recorded streams.

```CSharp
// Set up a trace with such that it automatically grows in size as needed.
var trace = new InputEventTrace(growBuffer: true);
trace.Enable();

// ... capture some input ...

// Write trace to file.
trace.WriteTo("mytrace.inputtrace.");

// Load trace from same file.
var loadedTrace = InputEventTrace.LoadFrom("mytrace.inputtrace");
```

You can replay captured traces directly from [`InputEventTrace`](../api/UnityEngine.InputSystem.LowLevel.InputEventTrace.html) instances using the [`Replay`](../api/UnityEngine.InputSystem.LowLevel.InputEventTrace.html#UnityEngine_InputSystem_LowLevel_InputEventTrace_Replay_) method.

```CSharp
// The Replay method returns a ReplayController that can be used to
// configure and control playback.
var controller = trace.Replay();

// For example, to not replay the events as is but rather create new devices and send
// the events to them, call WithAllDevicesMappedToNewInstances.
controller.WithAllDevicessMappedToNewInstances();

// Replay all frames one by one.
controller.PlayAllFramesOnyByOne();

// Replay events in a way that tries to simulate original event timing.
controller.PlayAllEventsAccordingToTimestamps();
```

## Merging of events

Input system uses event mering to reduce amount of events required to be processed.
This greatly improves performance when working with high refresh rate devices like 8000 Hz mice, touchscreens and others.

For example let's take a stream of 7 mouse events coming in the same update:

```

Mouse       Mouse       Mouse       Mouse       Mouse       Mouse       Mouse
Event no1   Event no2   Event no3   Event no4   Event no5   Event no6   Event no7
Time 1      Time 2      Time 3      Time 4      Time 5      Time 6      Time 7
Pos(10,20)  Pos(12,21)  Pos(13,23)  Pos(14,24)  Pos(16,25)  Pos(17,27)  Pos(18,28)
Delta(1,1)  Delta(2,1)  Delta(1,2)  Delta(1,1)  Delta(2,1)  Delta(1,2)  Delta(1,1)
BtnLeft(0)  BtnLeft(0)  BtnLeft(0)  BtnLeft(1)  BtnLeft(1)  BtnLeft(1)  BtnLeft(1)
```

To reduce workload we can skip events that are not encoding button state changes:

```
                        Mouse       Mouse                               Mouse
                        Time 3      Time 4                              Time 7
                        Event no3   Event no4                           Event no7
                        Pos(13,23)  Pos(14,24)                          Pos(18,28)
                        Delta(3,3)  Delta(1,1)                          Delta(4,4)
                        BtnLeft(0)  BtnLeft(1)                          BtnLeft(1)
```

In that case we combine no1, no2, no3 together into no3 and accumulate the delta,
then we keep no4 because it stores the transition from button unpressed to button pressed,
and it's important to keep the exact timestamp of such transition.
Later we combine no5, no6, no7 together into no7 because it is the last event in the update.

Currently this approach is implemented for:
- `FastMouse`, combines events unless `buttons` or `clickCount` differ in `MouseState`.
- `Touchscreen`, combines events unless `touchId`, `phaseId` or `flags` differ in `TouchState`.

You can disable merging of events by:
```
InputSystem.settings.disableRedundantEventsMerging = true;
```
