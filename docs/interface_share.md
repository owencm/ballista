# Ballista Share Interface

**Date**: 2016-05-13

This document is a rough spec (i.e., *not* a formal web standard draft) of the
Ballista share API. The basic share API just sends share requests (with no
capability for a website to receive share requests; only native system services
can receive). For a follow-up plan to have websites receive share requests from
the system, or other websites, see the [Share Target
API](interface_share_target.md).

Examples of using the share API for sharing can be seen in the
[explainer document](explainer.md).

## share

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

## canShare

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

## Built-in and native app handlers (web-to-native)

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
