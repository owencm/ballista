# Ballista Share Interface

**Date**: 2016-04-29

This document is a rough spec (i.e., *not* a formal web standard draft) of the
Ballista share API. The API is split into two parts: requester and handler. A
user agent can choose to support both sides of the API, or just one (filling in
the other side with built-in request generators or handlers). The handler side
requires the user agent to support both [service
workers](https://www.w3.org/TR/service-workers/) and [web app
manifests](https://www.w3.org/TR/appmanifest/); the requester side has no such
prerequisites.

Examples of using the share API for sharing can be seen in the
[explainer document](explainer.md).

## Requester API

### share

The `navigator.share` function (available from both foreground pages and
workers) is the main method of the interface:

```WebIDL
partial interface Navigator {
  Promise<void> share(object data);
};

partial interface WorkerNavigator {
  Promise<void> share(object data);
};
```

The `share` method takes one argument: the object that will be delivered to the
handler. It contains the data being shared between applications.

The `data` object may have any of (and should have at least one of) the
following optional fields:

* `title` (string): The title of the document being shared. May be ignored by
  the handler.
* `text` (string): Arbitrary text that forms the body of the message being
  shared.
* `url` (string): A URL or URI referring to a resource being shared.

We may later expand this to allow image data or file blobs.

`share` always shows some form of UI, to give the user a choice of application
and get their approval to invoke and send data to a potentially native
application (which carries a security risk). UX mocks are shown
[here](user_flow.md).

`share`'s promise is resolved if the user chooses a target application,
and that application accepts the data without error. The promise may be rejected
in the following cases (it is possible to distinguish between these four failure
modes, but again, not learn the identity of the chosen application):

* The share was invalid (e.g., inappropriate fields in the `data` parameter).
* There were no apps available to handle sharing.
* The user cancelled the picker dialog instead of picking an app.
* The data could not be delivered to the target app (e.g., service worker could
  not start, had no event handler, or the chosen native app could not be
  launched), or the target app explicitly rejected the share event.

### canShare

`navigator` also provides a method for determining whether there are any
applications that can handle sharing:

```WebIDL
partial interface Navigator {
  boolean canShare();
};
```

Returns `true` if there are one or more applications that could handle a share
event (i.e., if `share` was called, would any applications be presented to the
user?). May give false positives, but not false negatives (on some systems, it
may not be possible to determine in advance whether any native applications
support sharing, in which case `canShare` should return `true`; `false` means
that `share` will definitely fail). This can be used by websites to hide or
disable the sharing UI, to avoid presenting a button that just fails when users
press it.

**TODO(mgiuca)**: This may have to be asynchronous, so that the implementation
can query the file system without blocking.

### Built-in and native app handlers (web-to-native)

The user agent may choose to provide handlers that do not correspond to
registered web applications. When the user selects these "fake" handlers, the
user agent itself performs the duties of the handler. This can include:

* Providing a built-in service (such as "copy to clipboard").
* Forwarding the event to the native app picking system (e.g., [Android
  intents](http://developer.android.com/training/sharing/send.html), [iOS share
  sheets](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIActivityViewController_Class/index.html),
  [Windows Share contracts](https://msdn.microsoft.com/en-us/windows/uwp/app-to-app/share-data)).
* Forwarding the event directy on to a native system application.

In any case, the user agent is responsible for marshalling data to/from the
required formats and generally ensuring that the built-in or native handler
behaves like a web handler. For example, on Android, when a web requester is
used to send a system intent to a native application, the user agent may create
an [Intent](http://developer.android.com/reference/android/content/Intent.html)
object with `ACTION_SEND`, setting the `EXTRA_SUBJECT` to `title`. Since Android
intents do not have a URL field, `EXTRA_TEXT` would be set to `text` and `url`
concatenated together.

**Note**: An implementation may, in fact, choose to omit the web handler API
altogether, and just present built-in handlers.

## Handler API

Handlers **must** have a registered [service
worker](https://www.w3.org/TR/service-workers/) and a [web app
manifest](https://www.w3.org/TR/appmanifest/).

### App manifest

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

### Event handlers

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

### System-generated shares (native-to-web)

Share events do not need to come from web requesters. The user agent may
trigger a share from some external stimulus, such as the user choosing a web
app as the target of a system intent. As in the web-to-native case, the user
agent is responsible for simulating the requester side of the connection and
marshalling data into the correct format.

For example, the user agent may register web handlers into the operating
system's native application pickers. When the user picks a web handler, the user
agent creates a `data` object and invokes the web handler, as if it had been
triggered by a web requester.
