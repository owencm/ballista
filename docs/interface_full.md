# Ballista Full Interface

**Date**: 2016-05-04

This document is a rough spec (i.e., *not* a formal web standard draft) of the
full Ballista API. To avoid complexity, we have split this into two parts: the
[Share-only portion of the API](interface_share.md) and this document, which
describes the additional features to support the full bidirectional API. This
document assumes you have read the Share API.

Examples of using the full bidirectional API for editing documents can be seen
in the [explainer document](explainer.md).

## Requester API

### performAction

The `performAction` method does not change, but the `options` and `data`
arguments, as well as the return value, have additional fields.

```WebIDL
dictionary ActionOptions {
  DOMString verb;
  boolean? bidirectional;
  DOMString? type;
};

partial dictionary ActionData {
  Blob? file;
};

dictionary Action {
  long? id;
};
```

This adds the following new capabilities suitable for the edit use case:
* The requester can request a *bidirectional* action (expecting a response).
* The data can include a file or blob.

If `options.bidirectional` is `true` (only available for certain verbs), the
resulting `Action` object will have an integer `id` that uniquely identifies an
ongoing action (facilitating updates from the handler). If `bidirectional` is
`false` or omitted, `id` will be `null`, indicating that the session has already
ended.

A bidirectional action *must* be sent from a service worker (not a foreground
page), to ensure there is a way to receive responses (see [Design
Notes](design_notes.md#resilience-in-the-face-of-death)).

The remaining fields are described in the verb-specific documentation.

### Event handlers

When a handler sends an update on a bidirectional action, an `"update"` event is
fired at the `navigator.actions` object in the requester's service worker.

```WebIDL
partial interface Actions {
  attribute EventHandler onupdate;
};
Actions implements EventTarget;

interface UpdateEvent : Event {
  readonly attribute long id;
  readonly attribute ActionData data;
  readonly attribute boolean done;
};
```

The `onupdate` handler (with corresponding event type `"update"`) takes an
`UpdateEvent`. The `id` matches the `id` returned in the result of a previous
`performAction`; this can match any action from this service worker's history
(not necessarily the current session) that has not been completed by an update
with `done == true`.

The `data` field has the same format as the `data` argument to `performAction`
and represents an updated copy of the data that was sent when initiating this
action.

If `done` is `true`, the action is terminated and no further update events will
be delivered with the given `id` (the requester can clean up any local state
associated with that action).

## Handler API

### App manifest

The `actions` objects in the manifest are extended with additional fields:

```WebIDL
dictionary ManifestAction {
  DOMString verb;
  boolean? bidirectional;
  DOMString[]? types;
};
```

If `bidirectional` is true, the handler can receive bidirectional actions. It
defaults to false.

The `types` member is an optional list of MIME types that the handler supports.
If present and non-empty, the handler will only be available in a picking dialog
if the request's `options.type` matches one of the types in the handler's
manifest. Supports glob syntax (`"*"` matches any string; e.g., `"text/*"`
matches any MIME type starting with "text/"). If `types` is omitted, defaults to
`["*"]` (matching all types).

### Event handlers

We add the `id` attribute to `HandleEvent` objects:

```WebIDL
interface HandleEvent : ExtendableEvent {
  readonly attribute long id;
  readonly attribute ActionOptions options;
  readonly attribute ActionData data;

  void reject(DOMException error);
};
```

The `id` is a unique integer corresponding to the incoming action, which must be
retained by the handler for sending updates. If the incoming action is one-way,
`id` will be `null`.

### update and close

The new response methods are only available from service workers.

```WebIDL
partial interface Actions {
  void update(long id, ActionData data);
  void close(long id, ActionData data);
};
```

The `update` and `close` methods must be called with an `id` that was previously
received in a `handle` event. The `data` argument should contain an updated copy
of the `data` that was received in the `handle` event. Both methods send a
response back to the requester containing a clone of `data`.

The only difference is that `close` also terminates the action. The requester's
`update` event will have `done` set to true, and it will be an error to send
further updates using that `id`.

### IDs

Both the requester and handler APIs deal with integer IDs for uniquely
identifying bidirectional actions. In both cases, these IDs provide a handle on
potentially long-running actions that can outlive both the foreground page and
the service worker context. We use integer IDs rather than Action objects so
that they may be serialized into a local database for retrieval after the
service worker has been stopped and restarted.

The ID values may be chosen arbitrarily by the user agent (but a simple
incrementing counter should suffice). The values in the requester and handler do
not correspond (i.e., if a requester and handler are engaged in an ongoing
action, they may each be using a different number to refer to that action). The
values are expected to be unique for all time, for a given worker.

## The "edit" verb

When an action is sent using the `"edit"` verb, the following extra rules
apply:

* It must be a bidirectional action (`options.bidirectional == true`).
* The `data` object must have a `file` field. This *should* be a `File` object
  with an appropriate name (but *may* be a `Blob` object without a name).
* The `options.type` field must be equal to `data.file.type` (the MIME type of
  the blob).

Note the redundancy of the MIME type being present in both `options` and `data`.
This is deliberate: the user agent should not need to access `data` in order to
filter the handlers list or decide how to behave (e.g., so that
`canPerformAction` can operate without a `data` payload). Therefore, the MIME
type is duplicated into the `options` object.
