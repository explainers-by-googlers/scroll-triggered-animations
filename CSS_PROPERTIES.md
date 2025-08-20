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
  - timeline-trigger-type
  - timeline-trigger-actions
  - timeline-trigger-entry-range-start-block*
  - timeline-trigger-entry-range-end-block*
  - timeline-trigger-entry-range-start-inline*
  - timeline-trigger-entry-range-end-inline*
  - timeline-trigger-exit-range-start-block*
  - timeline-trigger-exit-range-end-block*
  - timeline-trigger-exit-range-start-inline*
  - timeline-trigger-exit-range-end-inline*
- event-trigger
  - event-trigger-name
  - event-trigger-actions

\* To make these less verbose, we could delete "range" from the property names.

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
  animation-trigger: trigger(--view-section-trigger, enter play);
}
```

### `timeline-trigger-actions`
`timeline-trigger-actions` names and emits "actions" emitted by a trigger. For example, A
timeline-based trigger's native actions are `enter` and `exit` of its specified ranges. An
event-based trigger's native actions are a subset of [UI events](https://w3c.github.io/uievents/#ref-for-uievent).
Triggers emit these actions using (dashed ident) names.

#### Why name the emitted actions?
Conceptually, the actions that are associated with a trigger are more associated with the trigger
than with the animations on which the triggers act. This is perhaps best demonstrated by the manner
in which the same objective would be accomplished in JavaScript:

```JS
let source = document.getElementById("source");
let target = document.getElementById("target");

let keyframes = new KeyframeEffect(target,
  [
    { transform: "translateX(0)" },
    { transform: "translateX(100px)" },
    { transform: "translateX(0)" },
  ]);

let animation = new Animation(keyframes);

target.onclick = () => {
  animation.play();
};
```
As such, it makes sense to have `event-trigger` and `timeline-trigger` specify their
associated actions. However, while it makes sense to define these actions close to the source,
baking animation-specific behavior (response to these actions) into triggers could be missing an
opportunity to define triggers as a concept that can be used outside the context of
animations. So, we place the behaviors on the target, i.e. `animation-trigger`. This means future
CSS properties can receive the same actions signals from triggers and perform other types of
non-animation-related work. Naming the actions creates a layer of indirection that means the
receiver of these signals need not be coupled with the underlying/native actions driving the
trigger. We think this layer of abstraction is a better design choice because it decouples the
receiver of the actions signal from the trigger's internals.

### `timeline-trigger-type`
The describes the type of [AnimationTimeline](https://drafts.csswg.org/web-animations-1/#the-animationtimeline-interface)
underlying the trigger.

Valid values are:
- `view`, meaning [`ViewTimeline`](https://drafts.csswg.org/scroll-animations/#view-timelines)s &
- `scroll`, meaning [`ScrollTimeline`](https://drafts.csswg.org/scroll-animations/#scroll-timelines)s

In the near possible future, `pointer` could be another type.

### `timeline-trigger-{exit, entry}-range-{start, end}-{block, inline}`
`timeline-trigger-entry-range-start-block` and `timeline-trigger-entry-range-end-block` define the
block axis component of a timeline-based trigger's "entry" range.
`timeline-trigger-entry-range-start-inline` and `timeline-trigger-entry-range-end-inline` similarly
define the "entry" range's inline component.

`timeline-trigger-exit-range-start-block` and `timeline-trigger-exit-range-end-block` define the
block axis component of a timeline-based trigger's "exit" range.
`timeline-trigger-exit-range-start-inline` and `timeline-trigger-exit-range-end-inline` similarly
define the "exit" range's inline component.

These 8 properties take the same values as
[`animation-range-{start, end}`](https://drafts.csswg.org/scroll-animations/#animation-range).

#### Why have an "exit range" distinct from the "entry range"?

A timeline-based trigger's entry condition is met when the timeline's "[current time](https://drafts.csswg.org/web-animations-1/#dom-animationtimeline-currenttime)" value
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

#### Why have *-block and *-inline variants?

Timelines are inherently one-dimensional (1D) and this single dimensionality suffices for the vast
majority of scroll-based use cases. However, this might not be the case for other types
(non-scroll-based) types of timelines, e.g. pointer-timelines where the concepts of "entry" and
"exit" are almost always better thought of in two dimensions.

### `event-trigger-name`
Mostly similar to `timeline-trigger-name`.

### `event-trigger-actions`
Similar to `timeline-trigger-actions` except the native actions are [UI events](https://w3c.github.io/uievents/#ref-for-uievent).

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
  timeline-trigger: --trigger1 view contain;
}

#viewtarget2 {
  timeline-trigger: --trigger2 view contain;
}

#animatable {
  animation: ...;
  animation-trigger: trigger(--trigger1, enter play-forwards, exit play-backwards)
                     trigger(--trigger2, enter play-forwards, exit play-backwards);
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
  event-trigger: --trigger1;
}
#clicktarget2 {
  event-trigger: --trigger2;
}

#animatable {
  animation: ...;
  animation-trigger: trigger(--trigger1, click play) trigger(--trigger2, click play);
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
  event-trigger-name: --click-trigger;
}

@keyframes glow { 0%: {..} 50%: {..} 100%: {..} }

.target {
  animation: glow linear 1s;
  animation-trigger: trigger(--click-trigger, click play);
}
```

“Play when clicked, paused when double-clicked”
```CSS
.source {
  event-trigger-name: --click-trigger;
}

@keyframes glow {
  0%: {}
  50%: {}
  100%: {}
}

.target {
  animation: glow linear 1s;
  animation-trigger: trigger(--click-trigger, click play, dbl-click pause);
}
```

### `timeline-trigger` Examples

“Play on entry, reverse on exit”

```CSS
.source {
  timeline-trigger-name: --view-trigger;
  timeline-trigger-type: view;
  timeline-trigger-entry-range-block: contain;
}

.target {
  animation-trigger: trigger(--view-trigger, enter play-forwards, exit play-backwards);
}
```

“Play on entry, reset on exit”

```CSS
.source {
  timeline-trigger-name: --view-trigger;
  timeline-trigger-type: view;
  timeline-trigger-entry-range-block: contain;
  timeline-trigger-exit-range-block: cover;
}

.target {
  animation-trigger: trigger(--view-trigger, enter play-forwards, exit reset);
}
```
