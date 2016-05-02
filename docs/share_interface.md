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
interface Actions {
  Promise<Action> performAction((ActionOptions or DOMString) options,
                                ActionPayload payload);
};

dictionary ActionOptions {
  DOMString verb;
  boolean? bidirectional;
  DOMString? type;
};

dictionary ActionPayload {
  DOMString? title;
  DOMString? text;
  DOMString? url;
};

dictionary Action {
  long id;
};
```

If `options` is a `DOMString`, the string is the verb (it is equivalent to
`{'verb': options}`).
