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
  - timeline-trigger-behavior
  - timeline-trigger-timeline
  - timeline-trigger-range-start
  - timeline-trigger-range-end
  - timeline-trigger-exit-range-start
  - timeline-trigger-exit-range-end
- event-trigger
  - event-trigger-behavior
  - event-trigger-type

## Longhand Descriptions

### `animation-trigger`
`animation-trigger` specifies a dashed ident which is matched against `timeline-trigger-name`s and
`event-trigger-name`s.

### `timeline-trigger-name`
`timeline-trigger-name` specifies a dash ident that names a trigger. This trigger is associated with any `animation`
associated with an `animation-trigger` of the same name. For example:

```CSS
.section {
  timeline-trigger-name: --section-trigger;
}

.animatable {
  animation: ...;
  animation-trigger: --section-trigger;
}
```

### `timeline-trigger-behavior`
`timeline-trigger-behavior` specifies how a trigger behaves during repeated
occurrences of its trigger conditions. Valid values are:

- "once": The trigger plays the animation once and never again.
- "repeat": The trigger plays the animation each time its entry condition is met
and resets the animation each time its exit condition is met.
- "alternate": The trigger plays the animation forward each time its entry
condition is met and plays the animation in reverse each time its exit condition
is met.
- "state": The trigger plays the animation forward each time its entry
condition is met and pauses the animation each time its exit condition is met.

### `timeline-trigger-timeline`
This takes the same values as `animation-timeline` and specifies the
[`AnimationTimeline`](https://drafts.csswg.org/css-animations-2/#animation-timeline)
used to evaluate whether the trigger's conditions have been met.

### `timeline-trigger-{exit-}range-{start, end}`
`timeline-trigger-range-start` and `timeline-trigger-range-end` define a
scroll-based trigger's "entry" range.

`timeline-trigger-exit-range-start` and `timeline-trigger-exit-range-end` define
a scroll-based trigger's "exit" range.

These 4 properties take the same values as
[`animation-range-{start, end}`](https://drafts.csswg.org/scroll-animations/#animation-range).

#### Why have an "exit range" distinct from the "entry range"?

A scroll-based trigger's entry condition is met when the corresponding scroll
container's scroll offset is within the specified entry range. However, a
scroll-based trigger's exit condition is met when said scroll container's
scroll offset is *outside* the exit range.

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

### `event-trigger-behavior`
Mostly similar to `timeline-trigger-behavior`.

### `event-trigger-type`
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
#viewtarget1, #viewtarget2 {
  timeline-trigger: --trigger view() repeat contain 0% contain 100%;
}

#animatable {
  animation: ...;
  animation-trigger: --trigger;
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
#clicktarget1, #clicktarget2 {
  event-trigger: --trigger event(click) repeat;
}

#animatable {
  animation: ...;
  animation-trigger: --trigger;
}
```

and in HTML:

```HTML
<div id="viewtarget1"></div>
<div>Lots of other content.</div>
<div id="viewtarget2"></div>

<div id="animatable"></div>
```

When either `viewtarget1` or `viewtarget2` is clicked the animation on `animatable`
is played.

## Why not just a single animation-trigger shorthand?
An AnimationTrigger might be scroll-based or event-based. Some of the CSS
properties required for scroll-based triggers are not relevent for event-based
triggers. Similarly, some properties for event-based triggers are not relevant
for scroll-based triggers. As such, it seems the natural and clean thing to do to
create separate CSS property "namespaces" for these properties. It would also work
well with hypothetical future types of triggers.

## Why event- & timeline- behaviors and not just animation-trigger-behavior?
This allows space-separated declarations of multiple triggers on an animation.
It seems unlikely that authors will want different behaviors for triggers that
are otherwise identical. But if they do, they can still do so with, for example:
```CSS
section {
  timeline-trigger: --t1 once ..., --t2 alternate <same as previous>;
}

.animatable {
  animation: ...;
  animation-trigger: --t1 --t2;
}
```
