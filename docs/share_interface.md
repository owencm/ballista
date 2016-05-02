# Ballista Share Interface

**Date**: 2016-04-29

This document is a rough spec (i.e., *not* a formal web standard draft) of the
Ballista one-directional API. To avoid complexity, we avoid discussing the
bidirectional parts of the Ballista project here.

Examples of using the one-directional API for sharing can be seen in the
[explainer.md](explainer document).

## Requester API

First, we add the `navigator.actions` interface (available from both foreground
pages and workers), which is where our API surface lives.

```WebIDL
partial interface Navigator {
  readonly attribute Actions actions;
};

partial interface WorkerNavigator {
  readonly attribute Actions actions;
};
```

### performAction

The `navigator.actions` interface provides the `performAction` method which
initiates a request.

```WebIDL
partial interface Actions {
  Promise<Action> performAction((ActionOptions or DOMString) options,
                                ActionData data);
};

dictionary ActionOptions {
  DOMString verb;
};

dictionary ActionData {
  DOMString? title;
  DOMString? text;
  DOMString? url;
};

dictionary Action {
  long id;
};
```

The `performAction` method takes two arguments:
* `options` provides metadata about the action which is used by the user agent
  to decide how to behave (what apps to show, how the UI is presented, whether
  to expect a response, etc). Currently just has `verb` (the only non-optional
  field) but we will expand this with more options in the future. If this is a
  string `x`, that is short-hand for `{verb: x}`.
* `data` is the object that will be delivered to the handler. It contains the
  data being shared between applications. Its fields depend on the verb (shown
  above are some of the fields for `share`, but more may be allowed, such as
  image data).

`performAction` always shows some form of UI, to give the user a choice of
application and get their approval to invoke and send data to a potentially
native application (which carries a security risk). UX mocks are shown
[here](user_flow.md).

`performAction` returns (asynchronously, via a promise) an `Action` object. In
general, `id` will be an integer that uniquely identifies an ongoing action
session. In the one-way use case, `id` will be `null`, indicating that the
session has already ended. It is not possible for the requester to learn the
identity of the chosen application.

`performAction`'s promise may be rejected in the following cases (it is possible
to distinguish between these four failure modes, but again, not learn the
identity of the chosen application):

* The action was invalid (e.g., an unknown verb or inappropriate fields for the
  given verb).
* There were no apps available to handle that specific action.
* The user cancelled the action instead of picking an app.
* The data could not be delivered to the target app (e.g., no service worker was
  registered, or the chosen native app could not be launched), or the target app
  explicitly rejected the action.

### canPerformAction

We also provide a method for determining whether there are any applications that
can handle a particular action:

```WebIDL
partial interface Actions {
  boolean canPerformAction((ActionOptions or DOMString) options);
};
```

Returns `true` if there are one or more applications that could handle the
action described by `options` (i.e., if `options` was passed to `performAction`,
would any applications be presented to the user?). May give false positives, but
not false negatives (on some systems, it may not be possible to determine in
advance whether any native applications support the action, in which case
`canPerformAction` should return `true`; `false` means that `performAction` will
definitely fail). This can be used by websites to hide or disable the sharing
UI, to avoid presenting a button that just fails when users press it.

**TODO(mgiuca)**: This may have to be asynchronous, so that the implementation
can query the file system without blocking.

**For consideration**: `canPerformAction` may present a fingerprinting issue, by
providing several bits of entropy about the identity of the device. (For
example, an attacker may passively run `canPerformAction` on all available verbs
with many different options sets, and the set of resulting bits will in some way
reflect the set of web and native applications registered/installed on the
device.) I suspect this entropy is minimal as most devices of a given operating
system would have a similar set of capabilities, but it may allow identification
of, e.g., users with native editors of obscure MIME types. Further analysis is
warranted.

### Verbs

The *verb* is a string that determines what handlers are available (handlers
explicitly register for certain verbs), as well as what fields are expected in
the `options` and `data` objects, and how the interaction will take place (e.g.,
whether it will be one-way or bidirectional). Each verb is its own
mini-protocol.

To avoid a) proliferation of overly specialized verbs, and b) mismatched
expectations about what a particular verb means, we will limit the set of verbs
to those defined in the standard. We will leave the set of verbs open for new
additions in the future, but not allow individual handlers or requesters to
invent their own verbs ad-hoc. User agents are expected to reject actions with
unexpected verbs, and enforce that the correct options and data are supplied for
the given verb.

In this document, we define only a single verb, `"share"`, but we expect several
other verbs to be defined in the initial standard.

## Handler API

Handlers **must** have a registered [service
worker](https://www.w3.org/TR/service-workers/) and a [web app
manifest](https://www.w3.org/TR/appmanifest/).

### App manifest

The first thing a handler needs to do is declare its action handling
capabilities in the app manifest:

```WebIDL
partial dictionary Manifest {
  ManifestAction[]? actions;
};

dictionary ManifestAction {
  DOMString verb;
};
```

The `"actions"` member of the manifest contains a list of objects, one per verb
the application can handle. For the share verb, there is no additional metadata
required, so the following is sufficient:

```JSON
"actions": [
  {
    "verb": "share"
  }
]
```

For more complex verbs, other metadata may be provided, such as which MIME types
will be accepted.

The declarative nature of the manifest allows web applications to be indexed by
their action handling capabilities, allowing search services that, say, find all
web applications that handle "share" actions.

Handlers declaring actions in their manifest will **not** be automatically
registered; the user must explicitly authorize the registration. How this takes
place is still under consideration (see [User
Flow](user_flow.md#registering-a-website-as-a-handler-on-mobile), but will
ultimately be at the discretion of the user agent (the user may be automatically
prompted, or may have to explicitly request registration).

**For consideration**: We may wish to provide a method for websites to
explicitly request to prompt the user for handler registration. This would
*only* allow sites to register for verbs they have declared in the manifest. For
now, we have omitted such a method from the design to keep control in the hands
of user agents. It is easier to add such a method later than remove it.

### Event handlers

When the user picks a registered web app as the target of an action, the
handler's service worker starts up (if it is not already running), and an
`"action"` event is fired at the service worker's global scope (`self`).

```WebIDL
partial interface ServiceWorkerGlobalScope {
  attribute EventHandler onaction;
};

interface ActionEvent : ExtendableEvent {
  readonly attribute long id;
  readonly attribute ActionOptions options;
  readonly attribute ActionData data;

  void reject(DOMException error);
};
```

The `onaction` handler (with corresponding event type `"action"`) takes an
`ActionEvent`. The `id` is a unique integer corresponding to the incoming
action. The `options` and `data` fields are clones of the `options` and `data`
parameters passed to the `performAction` method by the requester.

If the `reject` method is called, the action fails and the requester's promise
is rejected. This must be called within the lifetime of the event (either in the
event handler, or in the promise passed to the event's
[`waitUntil`](https://www.w3.org/TR/service-workers/#wait-until-method) method).
The action also fails if the promise passed to `waitUntil` is rejected. Once the
event completes without failing, the action automatically succeeds, and the
requester's promise is resolved.

For one-way actions (like `"share"`), the end of the event's lifetime marks the
end of the action, and there is no further communication in either direction.

The handler-side API is defined entirely within the service worker. If the
handler needs to provide UI (which should be the common case), the service
worker must create a foreground page and send the appropriate data between the
worker and foreground page, out of band. The action event handler is [allowed to
show a
popup](https://html.spec.whatwg.org/multipage/browsers.html#allowed-to-show-a-popup)
for the purpose of the
[`clients.openWindow`](https://www.w3.org/TR/service-workers/#clients-openwindow-method)
method.

### IDs

Both the requester and handler APIs deal with integer IDs for uniquely
identifying actions. In both cases, these IDs provide a handle on potentially
long-running actions that can outlive both the foreground page and the service
worker context. We use integer IDs rather than Action objects so that they may
be serialized into a local database for retrieval after the service worker has
been stopped and restarted.

The ID values may be chosen arbitrarily by the user agent (but a simple
incrementing counter should suffice). The values in the requester and handler do
not correspond (i.e., if a requester and handler are engaged in an ongoing
action, they may each be using a different number to refer to that action). The
values are expected to be unique for all time, for a given worker.
