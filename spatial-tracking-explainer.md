# WebXR Device API - Spatial Tracking
This document is a subsection of the main WebXR Device API explainer document which can be found [here](explainer.md).  The main explainer contains all the information you could possibly want to know about setting up a WebXR session, the render loop, and more.  In contrast, this document covers the technology and APIs for tracking users' movement for a stable, comfortable, and predictable experience that works on the widest range of XR hardware.

## Introduction
A big differentiating aspect of XR, as opposed to standard 3D rendering, is that users control the view of the experience via their body motion.  To make this possible, XR hardware needs to be capable of tracking the user's motion in 3D space.  Within the XR ecosystem there is a wide range of hardware form factors and capabilities which have historically only been available to developers through device-specific SDKs and app platforms. WebXR changes that by allowing developers to deliver experiences without going through an app store.  The Web gives developers broader reach, but the consequence is that developers no longer have predictability about the capability of the hardware their experiences will be running on.

## Frames of Reference
The WebXR Device API makes developers think upfront about the mobility needs of the experience they are building (rather than underlying tracking technology) by requring them to explicitly request an appropriate `XRFrameOfReference`.  The `XRFrameOfReference` object acts as a substrate for the XR experience being built by establishing guarantees about supported motion and providing a coordinate system in which developers can retrieve `XRDevicePose` and view matrices.  

There are three types of frames of reference: `bounded`, `unbounded`, and `stationary`.  A bounded experience is one in which the user will move around their physical environment to fully interact, but will not need to travel beyond a fixed boundary defined by the XR hardware.  An unbounded experience is one in which a user is able to freely move around their physical environment and travel significant distances.  A stationary experience is one which does not require the user to move around in space, and includes "seated" or "standing" experiences.

The critical aspect of this approach is that the User Agent (or underlying platform) is responsible for providing consistently behaved lower-capability `XRFrameOfReference` objects even when running on a higher-capability tracking system.  That said, not all experiences will work on all XR hardware and not all XR hardware will support all experiences (see Appendix A: XRFrameOfReference Availability).  In the spirit of [progressive enhanncement](https://developer.mozilla.org/en-US/docs/Glossary/Progressive_Enhancement), it is strongly recommended that developers select the least capable `XRFrameOfReference` that suffices for the experience they are building.  Requesting a more capable frame of reference will artificially restrict the set of XR devices their experience will otherwise be viewable from.

### Bounded Frame of Reference
A _bounded_ experience is one in which a user moves around their physical environment to fully interact, but will not need to travel beyond a pre-established boundary.  A _bounded_ experience is similar to a _unbounded_ experience in that both rely on XR hardware capable of tracking a users' locomotion.  However, _bounded_ experiences are explicitly focused on nearby content which allows them to target XR hardware that requires a preconfigured play area as well as XR hardware able to track location freely.

The origin of this type will be initialized at a position on the floor for which a boundary can be provided to the app, defining an empty region where it is safe for the user to move around. The y value will be 0 at floor level, while the exact x, z, and orientation values will be initialized based on the conventions of the underlying platform for room-scale experiences. Platforms where the user defines a fixed room-scale origin and boundary may initialize the remaining values to match the room-scale origin. Users with fixed-origin systems are familiar with this behavior, however developers may choose to be extra resilient to this situation by building UI to guide users back to the origin if they are too far away. Platforms that generally allow for unbounded movement may display UI to the user during the asynchronous request, asking them to define or confirm such a floor-level boundary near the user's current location.

```js
let xrSession = null;
let xrFrameOfReference = null;

function onSessionStarted(session) {
  xrSession = session;
  xrSession.requestFrameOfReference({ type:'bounded' })
  .then((frameOfReference) => {
    xrFrameOfReference = frameOfReference;
  })
  .then(setupWebGLLayer)
  .then(() => {
    xrSession.requestAnimationFrame(onDrawFrame);
  });
}
```

The `XRBoundedFrameOfReference` also reports geometry within which the application should try to ensure that all content the user needs to interact with can be reached. This polygonal boundary represents a loop of points at the edges of the safe space. The points are given in a clockwise order as viewed from above, looking towards the negative end of the Y axis. The shape it describes is not guaranteed to be convex. The values reported are relative to the frame of reference origin, and must have a `y` value of `0` and a `w` value of `1`.

```js
// Demonstrated here using a fictional 3D library to simplify the example code.
function createBoundsMesh() {
  boundsMesh.clear();
  
  // Visualize the bounds geometry as 2 meter high quads
  let pointCount = xrFrameOfReference.boundsGeometry.length;
  for (let i = 0; i < pointCount - 1; ++i) {
    let pointA = xrFrameOfReference.boundsGeometry[i];
    let pointB = xrFrameOfReference.boundsGeometry[i+1];
    boundsMesh.addQuad(
        pointA.x, 0, pointA.z, // Quad Corner 1
        pointB.x, 2.0, pointB.z) // Quad Corner 2
  }
  // Close the loop
  let pointA = xrFrameOfReference.boundsGeometry[pointCount-1];
  let pointB = xrFrameOfReference.boundsGeometry[0];
  boundsMesh.addQuad(
        pointA.x, 0, pointA.z, // Quad Corner 1
        pointB.x, 2.0, pointB.z) // Quad Corner 2
  }
}
```

### Unbounded Frame of Reference
A _unbounded_ experience is one in which the user is able to freely move around their physical environment. An example of this sort of experience would be a campus tour or a whole-house renovation preview. These experiences explicitly require that the user be unbounded in their ability to walk around, and the unbounded frame of reference will adjust its origin as needed to maintain optimal stability for the user, even if the user walks many meters from the origin. In doing so, the origin may drift from its original physical location.

> **Note regarding XRCoordinateSystem**
>
> When building an _unbounded_ experience, developers may additionally desire to "lock" content to a specific physical location to prevent it from drifting as users travel beyond a few meters; this functionality is known as _anchors_.  In addition to _anchors_, today's XR platforms offer other environment-related features such as _spatial meshes_, _point clouds_, _markers_, and more. While none of these features are yet part of the WebXR Device API, there are proposed designs under active development.
>
> The common property of these additional features and `XRFrameOfReference`, is that they all act as _spatial roots_ that are independently tracked by the underlying tracking systems. The concept of a _spatial roots_ is represented in the WebXR Device API as an `XRCoordinateSystem`.  Each instance of an `XRCoordinateSystem` does not have a fixed relationship with any other.  Instead, on a frame-by-frame basis, the tracking systems must attempt to locate them and compute their relative locations.  

```js
let xrSession = null;
let xrFrameOfReference = null;

function onSessionStarted(session) {
  xrSession = session;
  xrSession.requestFrameOfReference({ type:'unbounded' })
  .then((frameOfReference) => {
    xrFrameOfReference = frameOfReference;
  })
  .then(setupWebGLLayer)
  .then(() => {
    xrSession.requestAnimationFrame(onDrawFrame);
  });
}
```

As developers start building experiences with the potential to span kilometers, they will also need to deal with the fact that floating point precision issues will be introduced.  When this is close to occurring, developers should create a new `XRUnboundedFrameOfReference`,and use the `coordinateSystem.getTransformTo()` method to reparent any nearby virtual object to the new frame of reference.  However, practically speaking, large _unbounded_ experiences will often segment app/game assets and logic into discrete map segments and, depending on map segment size, this may also be an opportune time to create a new `XRUnboundedFrameOfReference`.

### Stationary Frame of Reference
A _stationary_ experience is one which does not require the user to move around in space.  This includes several categories of experiences that developers are commonly building today.  "Standing" experiences can be created by passing the `floor-level` subtype.  "Seated" experiences can be created by passing the `eye-level` subtype.  Orientation-only experiences such as 360 photo/video viewers can be created by passing the `position-disabled` subtype.

It is important to note that `XRDevicePose` objects retrieved using the `floor-level` and `eye-level` subtypes may include position information as well as rotation information.  For example, hardware which does not support 6DOF tracking (ex: GearVR) may still use neck-modeling to improve user comfort. Similarly, a user may lean side-to-side on a device with 6DOF tracking (ex: HTC Vive).  It is important for user comfort that developers do not attempt to remove position data from these matrices and instead use the `position-disabled` subtype.  The result is that `floor-level` and `eye-level` experiences should be resilient to position changes despite not being dependent on receiving them.  

#### Floor-level Subtype

The origin of this subtype will be initialized at a position on the floor where it is safe for the user to engage in standing-scale experiences, with a `y` value of `0` at floor level. The exact `x`, `z`, and orientation values will be initialized based on the conventions of the underlying platform for standing-scale experiences. Some platforms may initialize these values to the user's exact position/orientation at the time of creation. Other platforms may place this standing-scale origin at the user's chosen floor-level origin for bounded experiences. It is also worth noting that some XR hardware will be unable to determine the actual floor level and will instead use an emulated floor.

```js
let xrSession = null;
let xrFrameOfReference = null;

function onSessionStarted(session) {
  xrSession = session;
  xrSession.requestFrameOfReference({ type:'stationary', subtype:'floor-level' })
  .then((frameOfReference) => {
    xrFrameOfReference = frameOfReference;
  })
  .then(setupWebGLLayer)
  .then(() => {
    xrSession.requestAnimationFrame(onDrawFrame);
  });
}
```

#### Eye-level Subtype

The origin of this subtype will be initialized at a position near the user's head at the time of creation. The exact `x`, `y`, `z`, and orientation values will be initialized based on the conventions of the underlying platform for stationary eye-level experiences. Some platforms may initialize these values to the user's exact position/orientation at the time of creation. Other platforms that allow users to reset a common eye-level origin shared across multiple apps may use that origin instead.

```js
let xrSession = null;
let xrFrameOfReference = null;

function onSessionStarted(session) {
  xrSession = session;
  xrSession.requestFrameOfReference({ type:'stationary' , subtype:'eye-level' })
  .then((frameOfReference) => {
    xrFrameOfReference = frameOfReference;
  })
  .then(setupWebGLLayer)
  .then(() => {
    xrSession.requestAnimationFrame(onDrawFrame);
  });
}
```

#### Position-disabled Subtype

The origin of this subtype will be initialized at a position near the user's head at the time of creation.  `XRDevicePose` objects retrieved with this subtype will have varying orientation values but will always report `x`, `y`, `z` values to be `0`.

```js
let xrSession = null;
let xrFrameOfReference = null;

function onSessionStarted(session) {
  xrSession = session;
  xrSession.requestFrameOfReference({ type:'stationary', subtype:'position-disabled' })
  .then((frameOfReference) => {
    xrFrameOfReference = frameOfReference;
  })
  .then(setupWebGLLayer)
  .then(() => {
    xrSession.requestAnimationFrame(onDrawFrame);
  });
}
```

## Practical Usage Guidelines

### Inline Sessions
Inline sessions, by definition, do not require a user gesture or user permission to create, and as a result there must be strong limitations on the pose data that can be reported for privacy and security reasons.  Requests for an `XRBoundedFrameOfReference` or an `XRUnboundedFrameOfReference` will always be rejected on inline sessions.  Requests for `XRStationaryFrameOfReference` may succeed, but may also be rejected if the UA is unable provide any tracking information such as for an inline session on a desktop PC or a 2D browser window in a headset.

Additionally, some inline experiences explicitly want tracking information disabled, such as a furniture viewer that will use pointer data to rotate the furniture.  Under both of these situations, there is no `XRFrameOfReference` needed.  Passing `null` as the `XRFrameOfReference` parameter to `getDevicePose()` or `getInputPose()` will return pose data whose values are based on the identity matrix.  Developers are then free to multiply in whatever transform they choose.

### Ensuring hardware compatibility
Immersive sessions will always be able to provide a `XRStationaryFrameOfReference`, but may not support other `XRFrameOfReference` types due to hardware limitations.  Developers are strongly encouraged to follow the spirit of [progressive enhanncement](https://developer.mozilla.org/en-US/docs/Glossary/Progressive_Enhancement) and provide a reasonable fallback behavior if their desired `XRBoundedFrameOfReference` or `XRUnboundedFrameOfReference` is unavailable.  In many cases it will be adequate for this fallback to behave similarly to an inline preview experience.

```js
let xrSession = null;
let xrFrameOfReference = null;

function onSessionStarted(session) {
  xrSession = session;
  xrSession.requestFrameOfReference(
    { type:'unbounded' }, 
    { type:'stationary', subtype:'eye-level' }
  )
  .then((frameOfReference) => {
    xrFrameOfReference = frameOfReference;
    if (xrFrameOfReference instanceof XRUnboundedFrameOfReference) {
      // Set up unbounded experience
    } else if (xrFrameOfReference instanceof XRStationaryFrameOfReference) {
      // Build an appropriate fallback experience if needed; perhaps similar to the inline version
    }
  })
  .then(setupWebGLLayer)
  .then(() => {
    xrSession.requestAnimationFrame(onDrawFrame);
  });
}
```

While many sites will be able to provide this fallback, for some sites this will not be possible.  Under these circumstances, it is instead preferable for session creation to reject rather than spin up immersive display/tracking systems only to immediately exit the session.

```js
function beginImmersiveSession() {
  xrDevice.requestSession({ immersive: true,  requiredFrameOfReferenceType:'unbounded' })
      .then(onSessionStarted)
      .catch(err => {
        // Error will indicate required frame of reference type unavailable
      });
}
```

### Floor Alignment Compatibility
Some XR hardware with inside-out tracking has users establish "known spaces" that can be used to easily provide `XRBoundedFrameOfReference` and the `floor-level` subtype of `XRStationaryFrameOfReference`.  On inside-out XR hardware which does not intrinsically provide these known spaces, the User Agent may respond to the promise request by presenting UI for selecting a floor-aligned origin.  If the User Agent chooses not to do this, it will reject the request for floor aligned `XRFrameOfReference` objects with an error indicating the reason.  In response, a polyfill can be used to provide the same behavior by using another `XRFrameOfReference` type to draw UI for selecting a floor origin.  If anchors are available to the polyfill, it could then place an anchor at the user's selected floor point.

Other XR hardware that is orientation-only tracking, may provide an emulated value for the floor offset.  On these devices, it is recommended that the User Agent or underlying platform provide a setting for users to customize this value.

### Transitioning between XRFrameOfReferences
It is expected that developers will often choose to preview `immersive` experiences with a similar experience `inline`.  In this situation, users often expect to see the scene from the same perspective when they make the transition from `inline` to `immersive`. In most cases, developers can accomplish this by computing the transform between the user and the virtual scene's origin in the previous `XRSession`.  When the new `XRSession` is initialized, the same transform can be applied to the virtual scene's origin in the new `XRFrameOfReference`.

### Handling a Tracking System Reset
Many XR systems have a mechanism for allowing the user to reset which direction is "forward" or re-center the scene's origin at their current location. For security and comfort reasons the WebXR Device API has no mechanism to trigger a pose reset programmatically, but it can still be useful to know when it happens. Pages may want to take advantage of the visual discontinuity to reposition the user or other elements in the scene into a more natural position for the new orientation. Pages may also want to use the opportunity to clear or reset any additional transforms that have been applied if no longer needed.

The exact behavior of a reset is platform specific, but in most cases it's expected that the origin and/or forward direction of the `XRStationaryFrameOfReference` will shift to be aligned to the users current physical location and orientation. On `XRBoundedFrameOfReference`, it is also expected that the `boundsGeometry` may have changed.

A page can be notified when a pose reset happens by listening for the 'onreset' event from the `XRStationaryFrameOfReference` or `XRBoundedFrameOfReference`.  The `XRUnboundedFrameOfReference` will not receive this event.  The event must fire prior to any poses being delivered with the new origin/direction, and all poses queried following the event must be relative to the reset origin/direction. 

```js
xrFrameOfReference.addEventListener('reset', xrFrameOfReferenceEvent => {
  // For an app that allows artificial Yaw rotation, this would be a perfect
  // time to reset that.
  resetYawTransform();

  // For an app using the XRBoundedFrameOfReference, this would be a perfect time to
  // reset the bounds and potentially re-layout content intended to be reachable within
  // the bounds
  createBoundsMesh();
});
```
## Appendix A : Miscellaneous

### Tracking Systems Overview
In the context of XR, the term _tracking system_ refers to the technology by which an XR device is able to determine a user's motion in 3D space.  There is a wide variance in the capability of tracking systems.

**Orientation-only** tracking systems typically use accelerometers to determine the yaw, pitch, and roll of a user's head.  This is often paired with a technique known as _neck-modeling_ that adds simulated position changes based on an estimation of the orientation changes originating from a point aligned with a simulated neck position.

**Outside-in** tracking systems involve setting up external sensors (i.e. sensors not built into the HMD) to locate a user in 3D space.  These sensors form a bounded area in which the user can reasonably expect be tracked.

**Inside-out** tracking systems typically use cameras and computer vision technology to locate a user in 3D space.  This same technique is also used to "lock" virtual content at specific physical locations.

### Frame of Reference Examples

| Type                           | Subtype             | Example                                       |
| -------------------------------| ------------------- | --------------------------------------------- |
| `XRStationaryFrameOfReference` | `position-disabled` | 360 photo/video viewer                        |
| `XRStationaryFrameOfReference` | `eye-level`         | Racing game or space simulator                |
| `XRStationaryFrameOfReference` | `floor-level`       | Action game where you duck and dodge in place |
| `XRBoundedFrameOfReference`    |                     | Room redecorator                              |
| `XRUnboundedFrameOfReference`  |                     | Walking tour                                  |

### XRFrameOfReference Availability

**Guaranteed** The UA will always be able to provide this frame of reference 

**Hardware-dependent** The UA will only be able to supply this frame of reference if running on XR hardware that supports it

**Rejected** The UA will never provide this frame of reference

&ast; Or polyfillable on inside-out tracking systems with anchor support.  See the section on "Floor Alignment Compatibility".

| Type                           | Subtype             | Inline             | Immersive |
| -------------------------------| ------------------- | ------------------ | ---------- |
| `XRStationaryFrameOfReference` | `position-disabled` | Hardware-dependent | Guaranteed |
| `XRStationaryFrameOfReference` | `eye-level`         | Hardware-dependent | Guaranteed |
| `XRStationaryFrameOfReference` | `floor-level`       | Hardware-dependent | Guaranteed* |
| `XRBoundedFrameOfReference`    |                     | Rejected           | Hardware-dependent* |
| `XRUnboundedFrameOfReference`  |                     | Rejected           | Hardware-dependent |

## Appendix B: Proposed partial IDL
This is a partial IDL and is considered additive to the core IDL found in the main [explainer](explainer.md).

```webidl
//
// Session
//

partial dictionary XRSessionCreationOptions {
  XRFrameOfReferenceType requiredFrameOfReferenceType;
};

partial interface XRSession {
  Promise<XRFrameOfReference> requestFrameOfReference(XRFrameOfReferenceOptions options, XRFrameOfReferenceOptions... fallbackOptions);
};

//
// Frames and Poses
//

partial interface XRFrame {
  // Also listed in the main explainer.md
  XRDevicePose? getDevicePose(optional XRFrameOfReference frameOfReference);
  XRInputPose? getInputPose(XRInputSource inputSource, optional XRFrameOfReference frameOfReference);
};

[SecureContext, Exposed=Window]
interface XRInputPose {
  readonly attribute boolean emulatedPosition;
  readonly attribute XRRay targetRay;
  readonly attribute Float32Array? gripMatrix;
};

//
// Coordinate System
//

[SecureContext, Exposed=Window] interface XRCoordinateSystem {
  Float32Array? getTransformTo(XRCoordinateSystem other);
};

//
// Frame of Reference
//

enum XRFrameOfReferenceType {
  "stationary",
  "bounded",
  "unbounded"
};

dictionary XRFrameOfReferenceOptions {
  required XRFrameOfReferenceType type;
};

[SecureContext, Exposed=Window] interface XRFrameOfReference : EventTarget {
  readonly attribute XRCoordinateSystem coordinateSystem;
};

//
// Stationary Frame of Reference
//

enum XRStationaryFrameOfReferenceSubtype {
  "eye-level",
  "floor-level",
  "position-disabled"
}

dictionary XRStationaryFrameOfReferenceOptions : XRFrameOfReferenceOptions {
  required XRStationaryFrameOfReferenceSubtype subtype;
};

[SecureContext, Exposed=Window]
interface XRStationaryFrameOfReference : XRFrameOfReference {
  readonly attribute XRStationaryFrameOfReferenceSubtype subtype;
  
  attribute EventHandler onreset;
};

//
// Bounded Frame of Reference
//

[SecureContext, Exposed=Window]
interface XRBoundedFrameOfReference : XRFrameOfReference {
  readonly attribute FrozenArray<DOMPointReadOnly> boundsGeometry;

  attribute EventHandler onreset;
};

//
// Unbounded Frame of Reference
//

[SecureContext, Exposed=Window] 
interface XRUnboundedFrameOfReference : XRFrameOfReference {
};

//
// Events
//

[SecureContext, Exposed=Window,
 Constructor(DOMString type, XRFrameOfReferenceEventInit eventInitDict)]
interface XRFrameOfReferenceEvent : Event {
  readonly attribute XRFrameOfReference frameOfReference;
};

dictionary XRFrameOfReferenceEventInit : EventInit {
  required XRFrameOfReference frameOfReference;
};
```