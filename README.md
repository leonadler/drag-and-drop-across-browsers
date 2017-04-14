# HTML5 drag and drop API across browsers

This is a writeup to describe the differences between browser implementations of the HTML5 drag & drop API.


## Browser versions tested

* Google Chrome 57
* Mozilla Firefox 52
* Microsoft Internet Explorer 11
* Microsoft Edge 38
* Apple Safari 10.0.3

Last updated: 2017-04-14


## About the test case

Browser testing is done using a single self-contained HTML file [in this repository](drag-and-drop-test.html).
The page basically consists of this html structure:

```html
<body>
    <div class="droptarget"></div>
</body>
```

Files are then dragged into or out of the page and the `<div>`.


## Common grounds

All tested browsers follow the same event pattern when a file is dragged into or out of the page:

 1. User drags into the page:

    ```
    DragEvent { type: "dragenter", target: <body>, relatedTarget: null }
    ```

 2. User drags across the page:

    ```
    DragEvent { type: "dragover", target: <body>, relatedTarget: null }
    ```
    _(repeated every time the mouse cursor is moved)_

 3. User drags out of the page:

    ```
    DragEvent { type: "dragenter", target: <body>, relatedTarget: null }
    ```

 4. User drops on the page:

    ```
    DragEvent { type: "drop", target: <body>, relatedTarget: null }
    ```

 5. User drags from the page into an element:

    ```
    DragEvent { type: "dragenter", target: <div class="droptarget">, relatedTarget: <body> }
    DragEvent { type: "dragleave", target: <body>, relatedTarget: <div class="droptarget"> }
    ```

    Exception: Edge & Safari do _not_ provide the `relatedTarget`, it is always `null`.

6. User drags from an element to the body element:

    ```
    DragEvent { type: "dragenter", target: <body>, relatedTarget: <div class="droptarget"> }
    DragEvent { type: "dragleave", target: <div class="droptarget">, relatedTarget: <body> }
    ```

    Exception: Edge & Safari do _not_ provide the `relatedTarget`, it is always `null`.


## Detecting if a drag & drop is happening anywhere on the page

All browsers except for Edge and Safari (yes, even Internet Explorer) provide a meaningful
`relatedTarget` on `dragenter` and `dragleave` events.
This would make the _"enter page"_ / _"leave page"_ logic very simple:

```javascript
var draggingInPage = false;

body.addEventListener('dragenter', function () {
    draggingInPage = true;
});
body.addEventListener('dragleave', function (event) {
    draggingInPage = !event.relatedTarget;
});
body.addEventListener('drop', function () {
    draggingInPage = false;
});
```

This does not work in Edge, however - counting "number of events received" is necessary:

```javascript
var eventCounter = 0;
var draggingInPage = false;

body.addEventListener('dragenter', function () {
    eventCounter += 1;
    draggingInPage = true;
});
body.addEventListener('dragleave', function () {
    eventCounter -= 1;
    draggingInPage = eventCounter > 0;
});
body.addEventListener('drop', function () {
    eventCounter = 0;
    draggingInPage = false;
});
```


## Detecting if the user is dragging files or text

To detecting dragging text vs dragging files, different properties of the events `dataTransfer` need to be accessed in different Browsers.

In Chrome, Firefox and Safari, the `DataTransfer` object at `event.dataTransfer` has a `types` property which is a string array that will contain `"Files"` when files are dragged:

```javascript
var draggingFiles = false;
function onDragEnter(event) {
    draggingFiles = event.dataTransfer.types.indexOf('Files') >= 0;
}
```

Microsoft Internet Explorer and Edge will provide a `DOMStringList` instead of an array,
therefore using `Array` methods like `includes` or `indexOf` is not possible.
For `DOMStringList`, using the `contains` method yields the same result:

```javascript
var draggingFiles = false;
function onDragEnter(event) {
    draggingFiles = event.dataTransfer.types.contains('Files');
}
```


### DataTransfer.files

All browsers have a `event.dataTransfer.files` property, but it is only filled for `drop` events
and has a `length` of 0 during `dragenter`/`dragover`/`dragleave` (or is `null` altogether).


### jQuery and drag & drop events

When jQuery is used for adding the event handler, the `DataTransfer` object might not be carried over
to the jQuery-created event, but is still accessible via the browser event stored as `originalEvent`:

```javascript
function dragEventHandler(event) {
    var dataTransfer = event.dataTransfer || (event.originalEvent && event.originalEvent.dataTransfer);
    // ... other code ...
}
$('body').on('dragenter dragover dragleave', dragEventHandler);
```


### Using `instanceof` for events

Unlike all other tested browsers, Safari uses `MouseEvent` instead of `DragEvent`.

```javascript
function handleAllEvents(event) {
    if (event instanceof DragEvent) {
        handleDragEvent(event); // not called in Safari
    } else {
        // ...
    }
}
```

Since this is inconsistent for other events as well, using `instanceof` over `event.type` is generally a bad idea and _not_ recommended.


## Detecting the mime types of the dragged files

To prevent dropping of unexpected files during the drag _(e.g. text files when images are expected)_
or to show a message like "Drop here to upload 2 images" vs "Drop here to upload 2 text files",
detecting the mime types or at least the amount of dragged files would be useful.

While all browsers provide a list of files and their mime types once the drag & drop operation
is over and the `drop` event is dispatched, not all browsers support detecting the mime types
during `dragenter`/`dragover`/`dragleave`.

Chrome and Edge provide the mime types of the dragged files under `dataTransfer.items`.
Firefox does provide an array with a `length` corresponding to the number of dragged files,
but sets the mime types to `"application/x-moz-file"` before the file is actually dropped.

In the example below, Chrome and Edge would set `draggedFileTypes` to
`["text/plain", "application/javascript"]`, Firefox would set it to
`["application/x-moz-file", "application/x-moz-file"]`.

```javascript
var draggedFileTypes = null;
function handleDragEnter(event) {
    draggedFileTypes = [];
    for (let i = 0; i < dataTransfer.items.length) {
        if (dataTransfer.items[i].kind === 'file') {
            draggedFileTypes.push(dataTransfer.items[i].type || 'unknown');
        }
    }
}
```


Older versions of Firefox do not provide the mime types during drag events, but provide a `mozItemCount`
property on the `DataTransfer` object to determine _how many_ files are dragged on the page:

```javascript
var amountOfDraggedFiles = 0;
function handleDragEnter(event) {
    amountOfDraggedFiles = event.dataTransfer.mozItemCount;   // only works in Firefox
}
```


## Drag & drop for folders

Google Chrome allows drag and drop for folders since Chrome 21, Firefox and Edge added support later.

On all supported browsers, `DataTransferItem` has an additional `webkitGetAsEntry()` method _(also on
browsers which are not based on Webkit)_ which should be used to distinguish file drag operations from
folder drag operations.
`webkitGetAsEntry()` is currently provided by Chrome, Firefox, Edge and Safari.

The `File` objects provided under `event.files` is mostly indistinguishable from a regular file,
but the `type` property is set to `""`, which unfortunately is also the fallback mime type
for files without an extension. None of the tested browsers detects the mime type from file contents.

In Chrome and Edge, checking the mime type in `event.items` against `""` could be used during
`dragenter`/`dragover`/`dragleave` events to detect folders only if you can assume that your users
do not upload files without an extension _(see note about default mime type above)_.
Firefox reports `"application/x-moz-file"` for files and folders alike.

The `size` of the `File` object is set to a non-consistent value and should not be used
for detecting folders - `4096` in Chrome, `0` in Edge and Firefox, an arbitrary value
in Safari _(68 for an empty folder, 238 for a 918KB folder)_.

Internet Explorer and Safari do not support drag & drop for folders, but do not prevent it.
On `dragenter`/`dragover`/`dragleave`, they contain a `"Files"` entry in `event.dataTransfer.items`.
In Safari, `"Files"` is still present on `drop` events for folders, with a mime type of `""`.
Internet Explorer reports `"Files"` during drag events, but not on `drop`, which can be used
to detect folders.


```javascript
function handleDragEvent(event) {
    var isFolder = false;
    if (event.items) {
        for (var i = 0; i < event.items.length; i++) {
            var item = event.items[i];
            if (item.webkitGetAsEntry) {
                var entry = item.webkitGetAsEntry();
                isFolder = entry ? entry.isDirectory : false;
            }
        }
    }

    // isFolder is set correctly during all events in Chrome & Edge and during drop in Firefox
}
```


## Differences by browser

Differences of file drag & drop in the tested browsers, compared to Google Chrome as a sane baseline.


### Mozilla Firefox

- `event.dataTransfer.effectAllowed` has an initial value of `"uninitialized"` (Chrome: `"all"`)
- `event.dataTransfer.dropEffect` has an initial value of `"move"` (Chrome: `"none"`)
- `lastModifiedDate` of `File` objects is logged as deprecated in favor of `lastModified`
- `event.dataTransfer.types` contains both `"Files"` and `"application/x-moz-file"`
- `event.dataTransfer.items[n].type` is always set to `"application/x-moz-file"`, even on `drop`
- `webkitGetAsEntry()` returns `FileSystemFileEntry`/`FileSystemDirectoryEntry` instances  
  Use `isFile`/`isDirectory` instead of `instanceof` to check the type (Chrome: `FileEntry`/`DirectoryEntry`)


### Apple Safari

- All drag / drop events are `MouseEvent` instances in Safari (Chrome: `DragEvent`)
- `event.dataTransfer.items` is not available in Safari  
  Mime types or number of dragged files can therefore not be determined in drag events, only on `drop`
- `webkitGetAsEntry()` is not available in Safari
- `dragenter` and `dragleave` events always have their `relatedTarget` set to `null`
- `lastModifiedDate` is not available on `File` objects, but `lastModified` is


### Microsoft Edge

- `event.dataTransfer.types` is a `DOMStringList`, not a string array
- `event.dataTransfer.dropEffect` has an initial value of `"copy"` (Chrome: `"none"`)
- `dragenter` and `dragleave` events always have their `relatedTarget` set to `null`
- `lastModified` is not available on `File` objects, but `lastModifiedDate` is
- `webkitGetAsEntry()` returns `WebKitFileEntry`/`WebKitDirectoryEntry` instances  
  Use `isFile`/`isDirectory` instead of `instanceof` to check the type (Chrome: `FileEntry`/`DirectoryEntry`)


### Microsoft Internet Explorer

- `event.dataTransfer.items` is not available in Safari  
  Mime types or number of dragged files can therefore not be determined in drag events, only on `drop`
- `webkitGetAsEntry()` is not available in Internet Explorer
- `event.dataTransfer.types` is a `DOMStringList`, not a string array
- Drag events _for folders_ contain `"Files"` in `event.dataTransfer.types` but `drop` does not
- `lastModified` is not available on `File` objects, but `lastModifiedDate` is


## Changes to this document

If you find any incorrect or outdated information, [I would appreciate a pull request to this repository](https://github.com/leonadler/drag-and-drop-across-browsers/fork).


## Authors

[Leon Adler](https://github.com/leonadler) - Test case & this document  
[Bernhard Riegler](https://github.com/bernhardriegler) - Testing on macOS

## License

[CC0/Public Domain](https://creativecommons.org/publicdomain/zero/1.0/)
