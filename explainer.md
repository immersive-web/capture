# XRCapture Explainer

## Introduction

This work aims to provide a low friction API to initiate a screenshot or begin a recording of an XR Session without giving the page immediate access to the file. This is similar to the native capabilities built into many runtimes to allow this capture. It also permits AR sessions to create a capture which includes the user’s real environment without needing access to the camera feed. Finally such captures can incorporate other features (e.g. DOM Overlay, Layers) that may be difficult or impossible for a page to capture on its own.


## Use Cases

Some example use cases include, but are not limited to:



*   Recording a WebXR AR session from a shopping site to show a friend/family member/housemate how an item might fit in a particular room.
*   Recording a visual glitch for a bug report
*   Recording a 3D Game to share/post on social media later


## Non-Goals

Though an integration with the [Web Share API](https://w3c.github.io/web-share/) is proposed, this API does not attempt to provide immediate access to such a recording to the capturing site. When presented with the UX for the Web Share API the user is presented with a list of native apps to choose to share with, which the page cannot influence.


## Proposed API

The feature will be gated with the “xr-capture” feature descriptor, this can be used to guarantee or determine if captures are supported by the granted session. Note that UAs may still choose to seek user consent before initiating a capture, even during a session. Photos and Videos should be saved to the corresponding well-known photo or video location rather than the downloads folder where appropriate.


```
// This can be expanded in the future to cover 360 or stereoscopic or other
// XR-specific capture types. Note that to support this feature, a UA need only
// support "image" or "video"
enum XRCaptureType {
  image,
  video
};

interface XRCapture {}

// Note that no "start" method is provided, because an XRVideoCapture should
// be started immediately, and it should not be possible to "restart" it.
// UA's MAY limit the max length of this recording by terminating it at any time.
interface XRVideoCapture : XRCapture, EventTarget {
  // Requests for the recording to stop. A new recording will not begin until
  // this one has been fully stopped. (Indicated by the "onstop" event).
  // TODO: Would the API shape be cleaner if this returned a promise, rather than
  // causing the other promises which require recording to be stopped to wait?
  void stopRecording();

  // Whether or not the capture is actively recording or has been stopped.
  readonly attribute boolean recording;

  // Triggered when the VideoCapture is terminated, whether by the system or by
  // stopRecording.
  attribute EventHandler onstop;
}

interface XRVideoCaptureStopEvent : Event {
  constructor(DOMString type, XRVideoCaptureStopEventInit eventInitDict);
  // The VideoCapture which generated this event.
  // TODO: Determine if this is necessary. With the ability to just request
  // stopRecording synchronously it is possible for race conditions to occur where
  // the previous recording has been discarded. However, it does seem unlikely
  // if only one recording is allowed to be active at the time.
  [SameObject] readonly attribute XRVideoCapture capture;
}

interface XRVideoCaptureStopEventInit : EventInit {
  required XRVideoCapture videoCapture;
}

partial interface XRSession {
  // Note: Requires user gesture, may require additional user consent, only one
  // capture is allowed to be active at a time.
  promise<XRCapture> requestCapture(XRCaptureType captureType);
  promise<boolean> supportsCaptureType(XRCaptureType captureType);
}
```


The `onstop` attribute is an [Event handler IDL attribute](https://html.spec.whatwg.org/multipage/webappapis.html#event-handler-idl-attributes) for the `XRVideoCaptureStop` event type.


### Limiting Recording Length

If a UA wishes to limit the length of a recording, it MAY do so. When doing so, the recording should be immediately terminated and should preserve all data from the start of the recording until this termination. This follows the principle of least surprise, where a user can be notified that a recording has been terminated; but not that they have lost any initial parts of the recording. Some scenarios that a UA may use to consider stopping a recording, include but are not limited to:



*   The tab becoming backgrounded
*   The XR Session being terminated
*   Device storage constraints
*   Length restrictions designed to prevent abuse

_Non-Normative: This recording limit should be chosen such that a typical user will not hit it, excepting out-of-memory scenarios. This means that the limit should be on the order of minutes rather than seconds._


### Web Share Integration

If the WebShare API is implemented; the following integration should also be implemented. Note that the presence of this entry-point may also depend on if the platform has a native Share API.


```
partial interface XRCapture {
  // Invokes the web share API with the passed in ShareData, appending the file
  // that this object represents.
  // TODO: It seems that this integration provides an ergonomics benefit to
  // developers, because we can guarantee the inclusion of the recording, but is it
  // a better pattern to omit it?
  promise<undefined> share(optional ShareData data = {})
}
```



### Example Usage


```
let recording = null;

// Called when the user presses the "Record" button
async function startRecording() {
   recording = await xrSession.requestCapture("video");
}

// Called when the user presses the "Stop" button.
async function stopRecording() {
  recording.stopRecording();

  recording.share();
}

async function takeScreenshot() {
  let screenshot = await xrSession.requestCapture("image");
  screenshot.share();
}
```



### Example User Flow

The proposed user flow of the above sample usage would be:

1. User presses a “Record Experience” button on the page.
2. User is prompted to allow the site to save a recording
    1. Note that this is done to mitigate malicious sites simply continually starting recordings and filling up the user’s disk by making numerous recordings, though user agents are encouraged to end recordings after a set amount of time.
3. Recording Begins.
    2. Though some form of notification may be shown to indicate that a recording is in progress, User Agents are encouraged to differentiate this recording from a capture occurring via getViewportMedia or getDisplayMedia. For example, different “Capturing” UI  should be shown than those APIs, perhaps excluding an overlay border, to hint to the user that the site does not have immediate access to the recording.
4. The user presses a “Stop Recording” button on the page.
5. At this time, the site may wish to initiate a share, and the [WebShare](https://w3c.github.io/web-share/) [User Flow](https://github.com/w3c/web-share/blob/main/docs/explainer.md#user-flow) would then begin.


## Alternative API Shapes Considered


### Alternative Share Mechanism

The XRCapture object currently exposes a “share” method; as an alternative to this, it was considered to modify the “files” member of the ShareData dictionary to also accept an “OpaqueFileHandle” type: `sequence&lt;File or OpaqueFileHandle> files`. XRCapture would then inherit from this opaque file handle. Ultimately this approach was not pursued in favor of keeping this specification more self contained, and providing a “forwarding” endpoint to the WebShare API.

An additional alternative mechanism would be to not provide a share mechanism at all. We could rely on the proposed [file system access API extension ](https://github.com/WICG/file-system-access/blob/main/SuggestedNameAndDir.md)to open to a recommended location (and specify the [known location](https://github.com/WICG/file-system-access/blob/main/SuggestedNameAndDir.md#specifying-a-well-known-starting-directory) for saving the XRCapture).  If we wish to integrate even more tightly with the file system access API, we could change the “share” method to an [“id”](https://github.com/WICG/file-system-access/blob/main/SuggestedNameAndDir.md#distinguishing-the-purpose-of-different-file-picker-invocations) attribute. This would be opaque to the page, but allow it to open a picker to the location of the saved image/video. However, it’s not clear how actively this proposal is being pursued, and thus we would prefer not to rely on it; but it is ultimately a good fallback if we opt to not pursue the XRCapture.share method.


### getDisplayMedia

The current security requirements of implementing [getDisplayMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia) require the user to have multiple surfaces to pick from, which may not be possible on all platforms. Even on platforms where this is possible, it can introduce a lot of friction to the user flow, especially when compared to other mobile applications (e.g. SceneViewer, the main native competitor to WebXR on Android) which can directly access screen recording APIs to save content to disk. And somFor many of the use cases that XRCapture is targeting, the site may neither need nor want access to the recording or camera image after the user has taken it (except to facilitate easier sharing via the WebShare API so that someone could send the recording to a friend/family member/housemate). It is this immediate access to the capture that makes getDisplayMedia require so much friction before initiating a capture. Thus, by removing the immediate access, we can similarly remove some of the friction. Furthermore, the XRCapture APIs allows the site to save the recording directly to the users disk, which while possible with a stream from getDisplayMedia requires additional hurdles that a site must jump through if the end goal is just to give the user a local recording.  On Desktop, the fact that the headset is a standalone display means that many of the more traditional screen capture mechanisms do not even show the headset as a viable capture target.


### getViewportMedia

There are two main problems precluding the use of [getViewportMedia](https://github.com/w3c/mediacapture-screen-share/issues/155). The first is that, similar to getDisplayMedia, the site gets immediate access to the capture of the tab. Though the UI requirements for this API are less friction (and closer to what is desired for XRCapture), this comes at the cost of extra restrictions on the content of the page for the API to even run, meaning that existing AR experiences would have to be restructured to enable this capture. These extra restrictions are still in development and will require time to make sure they are right; but in some instances, since the goal of the site is to simply facilitate a recording for personal use/sharing (rather than the site wanting or needing access), we can open the XRCapture API up more broadly by further removing some of getViewportMedia’s more stringent requirements, while still leaving the lower friction UX. The second issue is the fact that the “viewport” that should be captured is often ambiguous for WebXR. Particularly for Desktops running WebXR on a separate HMD, both the “traditional” content of the page and the WebXR content are visible, and either could be considered the “viewport.”


### WebXR Raw Camera Access feature

One of the primary use cases for this API is creating recordings of AR Sessions, and WebXR will be rolling out a feature to grant sites direct access to the [camera image](https://immersive-web.github.io/raw-camera-access/). A site could use this to composite the camera stream and it’s own content that it already knows about to create a recording similar to what the XRCapture API would provide. However, this would only satisfy one potential use case, and even that incompletely, as content rendered by other WebXR features such as [DOM Overlays](https://immersive-web.github.io/dom-overlays/), or XR Layers would either not be captured or would be infeasible to capture with such an approach. Further, the XRCapture API enables the site to make a recording without the site needing to request (or even receiving full access to the camera feed (e.g. unrestricted camera access after user approval). It’s worth noting that while the site \*does\* control all of the visuals for VR Sessions, there may be distortions or other items taken into account when compositing that the site cannot easily replicate.


### General Purpose Recording API

The main issue with a more general purpose recording API (e.g. something that like getDisplayMedia or getViewportMedia would live on navigator.media), is that while it is a bit more privacy preserving, most content has the general ability to directly incorporate getDisplayMedia or getViewportMedia with the MediaRecorder APIs thus reducing the need for an API like this. Indeed, similar to getDisplayMedia or getViewportMedia such an API still potentially falls a little short in the cases of actually selecting the XR Runtime, which often won’t show up in a standard (OS-level) Desktop/Window picker, nor is the content typically available to be captured via basic tab capture.
