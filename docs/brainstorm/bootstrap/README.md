# GO UI

## Motivation

[Go Programming Language](https://golang.org) is modern, efficient,
provides a great standard library, however lacks native Graphical User
Interface module.

Attempts to integrate existing frameworks, such as
[Gtk](https://www.gtk.org) and [Qt](https://www.qt.io) are not that
great, since those bring a big pile of extra components, many to solve
C/C++ issues such as `GObject`, `QList`, etc. They're also based on a
main loop, which feels awkward from Go's point of view. Last but not
least, they were designed decades ago and bring too much legacy
elements into play.

The [React](http://reactjs.org) Web User Interface framework brought
back simplicity, yet allowing rich user interfaces. It's declarative
and consists of returning a full DOM (Document Object Model) for a
given UI Component. The developer is not demanded to implement error
prone code such as keeping elements life cycle, instead just return
the whole tree and the React Core will calculate the difference,
deleting, creating new and modifying properties as needed. It's very
close to Programming 101, where students just print out the whole
screen over and over again.

Following the web success,
[React-Native](https://facebook.github.io/react-native/) was born to
deal with native UI elements (as opposed to HTML elements rendered
inside the browser). Particularly designed to allow faster Mobile Apps
(Tablets and Smartphones) by using native elements and rendering
system, it runs the JavaScript on a thread, while the UI runs in
another, thus delays in the application logic do not impact scrolls
and other animations. Share concepts with its web sibling, however
offers just a small set of base components exist: around 30.

On top of React, many new frameworks were created, such as
[Redux](https://redux.js.org) to manage application state and
[Redux-Saga](https://redux-saga.js.org) to make it easier to manage
application side effects.

The **motivation** of this project is to implement a native Graphical
User Interface module for Go, leveraging ideas from React-Native,
Redux and Redux-Saga:

 - Implemented as much as possible in Go;
 - Avoid JSX-like _transpile_, use Go `struct` declaration instead;
 - Avoid `propTypes`, use Go `struct` properties and reflect;
 - Reduced set of base UI elements;
 - Declarative, with render returning full DOM tree;
 - Redux-like state container (ie: [godux](https://github.com/luisvinicius167/godux));
 - Redux-Saga-like asynchronous handling (Database, Networking...);
 - DOM calculation threads (one per window);
 - Rendering (painting) thread/process (one per window -- whenever
   allowed by backend);
 - Extensible to native widgets (ie: `DatePickerIOS` and `DatePickerAndroid`).


## Architecture

```
  .-------------.   Component
  | Application |-> render()
  `-------------'      |
        ^              v
        |         .--------.               .---------.
     onPress()    |   DOM  | id, op, props | Native  |
        ^---------| Differ |-------------->|    UI   |
                  | Thread |<------------- | Process |
                  `--------' id, ev, props `---------'
```

The `Application` thread is what the developer sees and interacts
with. Once it creates a window (unlike Phones and other single-window
applications, this should be designed to allow multiple windows per
application), it will then start the `DOM Differ Thread`. This thread
receives the result of `render()`, will walk the whole tree and
generate the low level commands to be executed by the rendering thread
or process (`Native UI Process`).

The `DOM Differ Thread` will keep the "mounted" state and know what
must be updated in the `Native UI`, serializing those as
messages. Alongside with visual updates, it will also update which
events each component must handle, such as listening for mouse or
keyboard events.

Ideally the "port" layer to Operating System's primitives should be
isolated to the rendering thread or process (`Native UI Process`).
It's going to receive the operation (creating, deletion or
update), the object identifier and the new properties, taking care of
the translation to actual Operating System primitives. This thread
also captures the Operating System events selected by the DOM differ,
once they happen it will report back to the DOM differ, that will
dispatch to Application.

The communication with the rendering thread or process should be
minimum and restricted, this would allow that to be isolated to
another process such as done by WebKit2 and Blink browser rendering
systems. In the future that would allow to restart UI process and
replay the state whenever needed, another benefit is to detect
Application died and present that information to users. The linkage to
Native UI libraries will be restricted to this process, thus not
polluting the application process. Special care must be taken
regarding larger data, such as image pixels, which should be
transferred using `shm` (Shared Memory).


## Common Native UI

In order to speed up development of new ports and run in constrained
environments, it would be nice to implement a common Native UI that
manages simple objects such as rectangles, images and text.

This implementation would render efficiently, picking the best
implementation, calculating visual changes and occlusion. Examples:

 - favor blit over blend depending on opacity and alpha: blend
   requires "read", then multiply, then "write"; blit just requires
   writes;

 - recolour for rectangles results in blit or blend with the new
   color, not an intermediate image generation and recolour;

 - analyze image on load, detecting alpha and using blit over blend;

 - occlusion: do not render under opaque regions. This requires to
   split one paint into up to 4 elements based on each opaque
   region. Needs care on small regions, that may result into too much
   rendering commands of 1x1 areas -- needs a threshold;

 - architecture optimized operations (ie: use pixman library or
   implement SSE/NEON... directly);

 - OpenGL-ES implementation;


Image handling must also be optimized:

 - asynchronous loading;

 - caching (filename, mtime... and load size), with option for
   cross-process shared cache (ie: toolkit images, such as buttons,
   icons...)

 - load size, allows faster jpeg decoding at 1/N sizes (ie: 1/2,
   1/4...) based on macro blocks;

 - scale cache: allows reuse of scaled images.

 - scale cache heuristics: account scales per image, passively create
   the scale cache entry for next requests.


Event processing:

 - dynamically selected (meaningful) events;

 - bubbling and relationship between events and other objects are not
   covered.

 - special care for window resize events (? - to avoid old X11
   "expose" event and delay to repaint its contents, resulting in
   screen garbage)

With these in place, Linux bindings for Wayland and fbdev should
derive easily, allowing its usage in constrained embedded systems.

> **NOTE:** These match what the [EFL](http://www.enlightenment.org)
> provides. That set of libraries are known to be fast and the reason
> is the aforementioned optimizations done in Evas, the canvas
> subsystem.
