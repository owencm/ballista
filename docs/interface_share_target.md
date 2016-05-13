# Ballista Share Target Interface

**Date**: 2016-05-13

This document is a rough spec (i.e., *not* a formal web standard draft) of the
Ballista share target API. This API allows websites to register to receive
shared content from either the [Ballista Share API](interface_share.md), or
system events (e.g., shares from native apps).

This API requires the user agent to support both [service
workers](https://www.w3.org/TR/service-workers/) and [web app
manifests](https://www.w3.org/TR/appmanifest/).

Examples of using the share target API for sharing can be seen in the
[explainer document](explainer.md).

Handlers **must** have a registered [service
worker](https://www.w3.org/TR/service-workers/) and a [web app
manifest](https://www.w3.org/TR/appmanifest/).

## App manifest

The first thing a handler needs to do is declare its share handling capabilities
in the app manifest:

```WebIDL
partial dictionary Manifest {
  boolean supports_share;
};
```

The `"supports_share"` member of the manifest, if `true`, indicates that the app
can receive share events from requesters, or the system. The declarative nature
of the manifest allows search services to index and present web applications
that handle shares.

Handlers declaring `supports_share` in their manifest will **not** be
automatically registered; the user must explicitly authorize the registration.
How this takes place is still under consideration (see [User
Flow](user_flow.md#registering-a-website-as-a-handler-on-mobile), but will
ultimately be at the discretion of the user agent (the user may be automatically
prompted, or may have to explicitly request registration).

**For consideration**: We may wish to provide a method for websites to
explicitly request to prompt the user for handler registration. There would
still be a requirement to declare `supports_share` in the manifest. For now, we
have omitted such a method from the design to keep control in the hands of user
agents. It is easier to add such a method later than remove it.

## Event handlers

When the user picks a registered web app as the target of a share, the
handler's service worker starts up (if it is not already running), and a
`"share"` event is fired at the global object.

```WebIDL
partial interface WorkerGlobalScope {
  attribute EventHandler onshare;
};

interface ShareEvent : ExtendableEvent {
  readonly attribute object data;

  void reject(DOMException error);
};
```

The `onshare` handler (with corresponding event type `"share"`) takes a
`ShareEvent`. The `data` field is a clone of the `data` parameter passed to the
`share` method by the requester.

How the handler deals with the data object is at the handler's discretion, and
will generally depend on the type of app. Here are some suggestions:

* An email client might draft a new email, using `title` as the subject of an
  email, with `text` and `url` concatenated together as the body.
* A social networking app might draft a new post, ignoring `title`, using `text`
  as the body of the message and adding `url` as a link. If `text` is missing,
  it might use `url` in the body as well. If `url` is missing, it might scan
  `text` looking for a URL and add that as a link.
* A text messaging app might draft a new message, ignoring `title` and using
  `text` and `url` concatenated together. It might truncate the text or replace
  `url` with a short link to fit into the message size.

If the `reject` method is called, the share fails and the requester's promise is
rejected. This must be called within the lifetime of the event (either in the
event handler, or in the promise passed to the event's
[`waitUntil`](https://www.w3.org/TR/service-workers/#wait-until-method) method).
The share also fails if the promise passed to `waitUntil` is rejected. Once the
event completes without failing, the share automatically succeeds, and the
requester's promise is resolved. The end of the event's lifetime marks the end
of the share, and there is no further communication in either direction.

The handler-side API is defined entirely within the service worker. If the
handler needs to provide UI (which should be the common case), the service
worker must create a foreground page and send the appropriate data between the
worker and foreground page, out of band. The `share` event handler is [allowed
to show a
popup](https://html.spec.whatwg.org/multipage/browsers.html#allowed-to-show-a-popup),
which means it can call the
[`clients.openWindow`](https://www.w3.org/TR/service-workers/#clients-openwindow-method)
method.

## System-generated shares (native-to-web)

Share events do not need to come from web requesters. The user agent may
trigger a share from some external stimulus, such as the user choosing a web
app as the target of a system intent. As in the web-to-native case, the user
agent is responsible for simulating the requester side of the connection and
marshalling data into the correct format.

For example, the user agent may register web handlers into the operating
system's native application pickers. When the user picks a web handler, the user
agent creates a `data` object and invokes the web handler, as if it had been
triggered by a web requester.
