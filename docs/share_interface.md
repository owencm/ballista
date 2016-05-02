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
