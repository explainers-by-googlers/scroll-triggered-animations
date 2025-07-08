# Animation Trigger CSS properties

This is a description of CSS properties that can be used to set up
[AnimationTriggers](https://drafts.csswg.org/web-animations-2/#animationtrigger)
on CSSAnimations.

## Summary
`animation-trigger` is a shorthand which lets authors specify options with which
user agents create
[`AnimationTriggers`](https://drafts.csswg.org/web-animations-2/#animationtrigger)
for CSS animations. This proposal lists the set of longhands which set up an
`AnimationTrigger` and describes how they are associated with CSS animations.

### CSS properties

The set of CSS properties for specifying AnimationTriggers are as follows:

- animation-trigger
- animation-trigger-scope
- timeline-trigger
  - timeline-trigger-name
  - timeline-trigger-source
  - timeline-trigger-entry-action
  - timeline-trigger-entry-range-start
  - timeline-trigger-entry-range-end
  - timeline-trigger-exit-action
  - timeline-trigger-exit-range-start
  - timeline-trigger-exit-range-end
- event-trigger
  - event-trigger-name
  - event-trigger-source
  - event-trigger-action

## Longhand Descriptions

### `animation-trigger`
`animation-trigger` specifies a dashed ident which is matched against `timeline-trigger-name`s and
`event-trigger-name`s.

### `timeline-trigger-name`
`timeline-trigger-name` specifies a dash ident that names a trigger. This trigger is associated with any `animation`
associated with an `animation-trigger` of the same name. For example:

```CSS
.section {
  timeline-trigger-name: --view-section-trigger;
}

.animatable {
  animation: ...;
  animation-trigger: --view-section-trigger;
}
```

### `timeline-trigger-{entry, exit}-action`
`timeline-trigger-entry-action` & `timeline-trigger-exit-action` specify what action a `timeline-trigger`
should take on attached animations on entry and exit, respectively, of the specified ranges. Valid values are:

`play-forwards`, `play-backwards`, `play-alternate`, `pause`, `reset`, `replay`, `play`.

### `timeline-trigger-source`
This takes the same values as `animation-timeline` and specifies the
[`AnimationTimeline`](https://drafts.csswg.org/css-animations-2/#animation-timeline)
used to evaluate whether the trigger's conditions have been met.

### `timeline-trigger-{exit, entry}-range-{start, end}`
`timeline-trigger-entry-range-start` and `timeline-trigger-entry-range-end` define a
timeline-based trigger's "entry" range.

`timeline-trigger-exit-range-start` and `timeline-trigger-exit-range-end` define
a timeline-based trigger's "exit" range.

These 4 properties take the same values as
[`animation-range-{start, end}`](https://drafts.csswg.org/scroll-animations/#animation-range).

#### Why have an "exit range" distinct from the "entry range"?

A timeline-based trigger's entry condition is met when the timeline's "current time" value
is within the range corresponding to the trigger's specified entry range. However, a
timeline-based trigger's exit condition is met when said timeline's current time is *outside* the
exit range.

In many cases, authors will want to have the same values for both ranges, but in
some cases, it is useful to have different values. An example is with a
"repeat" trigger. "repeat" triggers reset their animations in response to their
exit condition being met. An author might not want their animation to play 
until their element is fully within the viewport. However, they might also
not want the reset of the animation to occur until said element is fully out of
view. In other words, they may not want the reset to happen while the element is
still partially in view.

In this case, the boundary for playing the animation and the boundary for
resetting the animation are not the same.

See the pictures below:

![image](https://i.imgur.com/JTaWb6O.gif)

In the image above, the entry boundary is the same as the exit boundary: the
point at which the animating box is fully within view. As soon as the box is no
longer fully in view, the animation gets reset. This might not be desirable as
the author might not want the reset to be visible to the user.

Being able to set a different exit boundary allows the author to create the
situation below:

![image](https://i.imgur.com/SOk58yS.gif)

Here, the reset does not happen until the element is completely out of view.

### `event-trigger-name`
Mostly similar to `timeline-trigger-name`.

### `event-trigger-action`
Mostly similar to `timeline-trigger-action`.

### `event-trigger-source`
This specifies which events trigger the playing, pausing, etc of the associated
animation. Examples are `click`, `touch`, etc.

### `animation-trigger-scope`

Named CSS AnimationTriggers will work analogously to [`anchor-name`](https://drafts.csswg.org/css-anchor-position-1/#propdef-anchor-name) and [`position-anchor`](https://drafts.csswg.org/css-anchor-position-1/#position-anchor):
names are global unless `animation-trigger-scope` limits the visibility of a name to the subtree of
the element declaring the scope.

## Multiple triggers on an animation

By refering to a name, `animation-trigger` attaches its corresponding `animation`
to any trigger with a matching name (within the relevant scope).

The following is an example in which an animation is attached to two scroll-based triggers:

```CSS
#viewtarget1 {
  timeline-trigger: --trigger1 view() contain play-forwards play-backwards;
}

#viewtarget2 {
  timeline-trigger: --trigger2 view() contain play-forwards play-backwards;
}

#animatable {
  animation: ...;
  animation-trigger: --trigger1 --trigger2;
}
```

and in HTML:

```HTML
<div id="scroller">
  <div id="viewtarget1"></div>
  <div>Lots of other content.</div>
  <div id="viewtarget2"></div>
</div>

<div id="animatable"></div>
```
When either `viewtarget1` or `viewtarget2` is scrolled into view, the animation on
`animatable` is played.

Here is a click-based example:

```CSS
#clicktarget1 {
  event-trigger: --trigger1 click play;
}
#clicktarget2 {
  event-trigger: --trigger2 event(click) repeat;
}

#animatable {
  animation: ...;
  animation-trigger: --trigger1 --trigger2;
}
```

and in HTML:

```HTML
<div id="clicktarget1"></div>
<div>Lots of other content.</div>
<div id="clicktarget2"></div>

<div id="animatable"></div>
```

When either `clicktarget1` or `clicktarget2` is clicked the animation on
`animatable` is played.


More Examples:

### `event-trigger` Examples

“Play when clicked”

```CSS
.source {
  event-trigger-name: --play-on-click;
  event-trigger-source: click;
  event-trigger-actions: play;
}
.shorthand {
  event-trigger: --play-on-click click play;
}

@keyframes glow { 0%: {..} 50%: {..} 100%: {..} }

.target {
  animation: glow linear 1s;
  animation-trigger: --play-on-click;
}
```

“Play when clicked, paused when double-clicked”
```CSS
.source {
  event-trigger-name: --play-on-click, --pause-on-double-click;
  event-trigger-source: click, dblclick;
  event-trigger-actions: play, pause;
}
.source-shorthand {
  event-trigger: --play-on-click click play, --pause-on-double-click dbl-click pause;
}

@keyframes glow {
  0%: {}
  50%: {}
  100%: {}
}

.target {
  animation: glow linear 1s;
  animation-trigger: --play-on-click --pause-on-dblclick;
}

```

Alternatively, configuring a single `event-trigger` to respond to multiple events:

```CSS
.source {
  event-trigger-name: --click-ctrl;
  event-trigger-source: click dblclick;
  event-trigger-actions: play pause;
}
.source-shorthand {
  event-trigger: --click-ctrl click play dblclick pause;
}

@keyframes glow {
  0%: {}
  50%: {}
  100%: {}
}

.target {
  animation: glow linear 1s;
  animation-trigger: --click-ctrl;
}
```

### `timeline-trigger` Examples

“Play on entry, reverse on exit”

```CSS
.source {
  timeline-trigger-name: --view-alternate-trigger;
  timeline-trigger-source: view();
  timeline-trigger-entry-range: contain;
  timeline-trigger-entry-action: play-forwards;
  timeline-trigger-exit-action: play-backwards;
 }

.source-shorthand {
  timeline-trigger: --view-alternate-trigger view() contain play-forwards play-backwards;
}

.target {
  animation-trigger: --view-alternate-trigger;
}
```

“Play on entry, reset on exit”

```CSS
.source {
  timeline-trigger-name: --view-repeat-trigger;
  timeline-trigger-source: view();
  timeline-trigger-entry-range: contain;
  timeline-trigger-entry-range: cover;
  timeline-trigger-entry-action: play-forwards;
  timeline-trigger-exit-action: reset;
 }

.source-shorthand {
  timeline-trigger: --view-alternate-trigger view() contain play-forwards play-backwards;
}

.target {
  animation-trigger: --view-alternate-trigger;
}
```
