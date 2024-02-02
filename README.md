# DataTransfer.noFailureEffect API

## Author:

* Cheng Zhao (czha@microsoft.com)

## Introduction

Many web apps have evolved to support multiple tabs and windows inside the app. Using the PWA version of VS Code as example, open https://vscode.dev and drag the "Welcome" tab out of the window, you will see a new window created with the "Welcome" tab moved into it.

(Depending on your browser, you may have to enable popup windows to be able to do this.)

https://github.com/zcbenz/dnd-drag-hint/assets/639601/4714f537-a6ed-44ac-9946-70eebf9d7bf4

### The Implementation

The tech behind this is quite simple: make the tab draggable, ensure the data transfer would not end with up a drag operation, and create a new window on the end of drag.

```html
<span id="drag" draggable="true">Drag Me</span>
<script>
  drag.ondragstrat = (event) => event.dataTransfer.effectAllowed = 'none'
  drag.ondragend = () => window.open('', '_blank', 'toolbar=no')
</script>
```

### The Problem

Since we don't want the tab dragging end up with an actual file operation on the drop target (for example creating an link file on user's desktop), the data transfer associated with the dragging does not carry any data nor has any effect.

As the result the dragging is essentially cancelled on mouse release, and the operating system may choose to display some visual effects. For example in the screencast below which does tab dragging on macOS, you can find out that after the dragging is done (user releases mouse), the tab image bounded back to the original window first before opening the new window, which is the default animation on macOS for failed draggings.

https://github.com/zcbenz/dnd-drag-hint/assets/639601/8a451620-3305-4194-bd45-216094d93c1c

This would give user a false impression that the dragging was failed but it acutally did succeed.

## Goals

* Allow web apps to provide native-feel tab dragging experiences.

## Additional Background

The operating systems do provide low-level APIs which allow developers to the visual details of the draggings. In the case of macOS, the failure animation can be easily disabled with the [animatesToStartingPositionsOnCancelOrFail](https://developer.apple.com/documentation/appkit/nsdraggingsession/1531277-animatestostartingpositionsoncan) API.

Firefox browser, who implements its tabs with HTML DnD, also faced [the same issue](https://bugzilla.mozilla.org/show_bug.cgi?id=1635761) with VS Code, and ended up adding a [private `DataTransfer.mozShowFailAnimation` property](https://hg.mozilla.org/mozilla-central/rev/b2954fa754d2) to solve it.

The desktop app version of VS Code, which runs on a modified Chromium runtime, solved the issue by patching Chromium directly with the patch below.

```patch
diff --git a/content/app_shim_remote_cocoa/web_contents_view_cocoa.mm b/content/app_shim_remote_cocoa/web_contents_view_cocoa.mm
index c050c86271d6d..9aba95b1d1b51 100644
--- a/content/app_shim_remote_cocoa/web_contents_view_cocoa.mm
+++ b/content/app_shim_remote_cocoa/web_contents_view_cocoa.mm
@@ -277,9 +277,10 @@ - (void)startDragWithDropData:(const DropData&)dropData
   _dragOperation = operationMask;
 
   // Run the drag operation.
-  [self beginDraggingSessionWithItems:@[ draggingItem ]
+  auto* drag_session = [self beginDraggingSessionWithItems:@[ draggingItem ]
                                 event:dragEvent
                                source:self];
+  drag_session.animatesToStartingPositionsOnCancelOrFail = NO;
 }
 
 // NSDraggingSource methods
 ```

## Proposal

We propose adding a new property to the `DataTransfer` object, which allows web apps to hint the dragging operation is expected to transfer no data to drop target, and thus no failure animation should be played.

```js
draggable.ondragstrat = (event) => event.dataTransfer.noFailureEffect = true
```

### IDL Changes

```
[
    Exposed=Window
] interface DataTransfer {
  attribute boolean noFailureEffect;
}
```

## Privacy and Security

This feature should have no affects on privacy and security.
