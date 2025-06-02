# `AnimationTrigger`

This is a proposal describing how `AnimationTrigger`s should work based on discussion in CSS Working
Group issue 
[#12119](https://github.com/w3c/csswg-drafts/issues/12119) .

## Background
[AnimationTriggers](https://drafts.csswg.org/web-animations-2/#triggers) are a way to automatically invoke
[playing](https://drafts.csswg.org/web-animations-1/#play-an-animation),
[pausing](https://drafts.csswg.org/web-animations-1/#pause-an-animation),
[reversing](https://drafts.csswg.org/web-animations-1/#reverse-an-animation) or resetting an
animation when specified conditions are met.

Note: this document makes a distinction between the animation APIs
[play()](https://drafts.csswg.org/web-animations-1/#dom-animation-play),
[pause()](https://drafts.csswg.org/web-animations-1/#dom-animation-pause), etc and the
corresponding spec algorithms ([play](https://drafts.csswg.org/web-animations-1/#play-an-animation),
[pause](https://drafts.csswg.org/web-animations-1/#pause-an-animation), etc) they run.

### state
Each AnimationTrigger is associated with a state property which can be ‘idle’, ‘primary’ or
‘inverse’. **_state_** is initially ‘idle.’ 

The transitions of state are restricted to the following:
- `idle` to `primary`
- `primary` to `inverse`
- `inverse` to `primary`

Note: `state` is not a web-exposed field.

The conditions specified on the trigger determine what **_state_** the trigger is in.

### Type
The trigger “trips” when it transitions from one **_state_** to another. The action the trigger
takes depends on the new **_state_** of the trigger and the type of the trigger:

- `once` triggers play their animations when they trip into the primary **_state_**.
They do nothing in response to tripping into the inverse **_state_**.

- `repeat` triggers play their animations when they trip into the primary **_state_** and reset
their animations in response to tripping into the inverse **_state_**.

- `alternate` triggers play their animations when they trip into the primary **_state_** and reverse
their animations in response to tripping into the inverse **_state_**.

- `state` triggers play their animations when they trip into the primary **_state_** and pause their
animations in response to tripping into the inverse **_state_**.

## Summary

In this proposal, a trigger is associated with an animation when it is "attached" to that animation.
When attaching a trigger to an animation, if the trigger has previously tripped, it will immediately
perform the action corresponding to its current **_state_** on the animation.

A trigger may be associated with more than one animation. As such, each trigger maintains an
**_animationList_**, a set containing all animations associated with the trigger.

We introduce [`addAnimation` and `removeAnimation`](#method-to-attach-trigger) methods which authors can use to attach and detach
triggers to/from animations.

## Details

### The Default Trigger
When creating an `AnimationTrigger`, if no options are provided, an instance of the default trigger is
returned. The default trigger has the document.timeline as its timeline and is considered to always
be in a tripped (primary) state.

### Endpoint Inclusivity
When an animation has a trigger which has not been tripped, we pause the animation in a way that
reflects that we are waiting for the animation’s trigger to trip. We reflect this state by setting
the animation’s [endpoint-inlcusive](https://github.com/w3c/csswg-drafts/pull/11881) flag to false
when attaching a trigger. The effect this has on an animation is to put the animation in the
[before phase](https://drafts.csswg.org/web-animations-1/#animation-effect-before-phase) if it’s a
forward-playing animation or the
[after phase](https://drafts.csswg.org/web-animations-1/#animation-effect-after-phase) if it is a
backwards-playing animation. This allows a CSS animation to visually reflect its
`animation-fill-mode` while awaiting trigger events.

Note: The [endpoint-inlcusive](https://github.com/w3c/csswg-drafts/pull/11881) flag is yet to be
formalized/specified. Since it is not a web-exposed feature, user agents can simply mimic its
behavior for AnimationTriggers until the flag doesn’t exist.

### Method to Attach a Trigger
We introduce an `AnimationTrigger.addAnimation(Animation)` method that works as follows:
1. If the animation being added is already in the trigger’s **_animationList_**, return.
Add the animation to the trigger’s **_animationList_**.
2. [Pause](https://drafts.csswg.org/web-animations-1/#pause-an-animation) the animation.
3. If the trigger’s state is `idle`, set the animation’s
[endpoint-inclusive](http://endpoint-inlcusive/) flag to false.
4. If the animation is a CSS animation whose `animation-play-state` is `paused`, return.
5. If the trigger’s state is `primary`, take the action that corresponds to the trigger’s `primary`
state.
6. Else if the trigger’s state is `inverse`, take the action that corresponds to the trigger’s
`inverse` state.

We also introduce a `AnimationTrigger.removeAnimation(Animation)` method which removes an animation
from a trigger’s **_animationsList_**.

### CSS Animations
CSS animations get a trigger corresponding to the
[`animation-trigger`](https://drafts.csswg.org/css-animations-2/#animation-trigger) property.
When `animation-trigger` isn't specified, the corresponding trigger is the
[default trigger](#the-default-trigger).

To delay [playing](https://drafts.csswg.org/web-animations-1/#play-an-animation) a CSS animation
until its trigger condition (as specified by the
`animation-trigger` CSS property) is met, we modify the manner in which CSS animations are
initiated. Instead of automatically playing a CSS animation, we automatically
[attach](#method-to-attach-a-trigger) it to a trigger.

Since existing CSS animations have the default trigger (as they do not have `animation-trigger`
specified), their triggers trip immediately and they are immediately
[played](https://drafts.csswg.org/web-animations-1/#play-an-animation), consistent with
the pre-existing behavior of CSS animations.

For example, the following CSS continues to function like before:

```css
@keyframes fade-in {
 from {
   opacity: 0.5;
   transform: translateX(-50px);
   background-color: yellow;
 }
 to {
   background-color: yellow;
 }
}
#target {
 ...
 /* This animation plays immediately, just like before AnimationTrigger existed. */
 animation: fade-in linear .5s both;
}
```

By specifying `animation-trigger`, an author can [attach](#method-to-attach-a-trigger) a non-default
trigger to their CSS animation. The animation’s trigger will
[play](https://drafts.csswg.org/web-animations-1/#play-an-animation) the animation when its trigger
condition is met.

Here's an example of a CSS animation that will be played when the matching element is fully scrolled
into view:

```css
@keyframes fade-in {
 from {
   opacity: 0.5;
   transform: translateX(-50px);
   background-color: yellow;
 }
 to {
   background-color: yellow;
 }
}
#target {
 ...
 /* This animation remains paused at its first keyframe, until #target is within */ 
 /* the viewport.                                                                */
 animation: fade-in linear .5s both;
 animation-trigger: view() alternate contain 0% contain 100%;
}
```

#### animation-play-state

A trigger will not take action on an animation while its `animation-play-state` is `paused`
(provided no WAAPI has rendered `animation-play-state` void).

### [Web Animations API](https://drafts.csswg.org/web-animations/) (WAAPI) Animations

#### Explicitly Constructed Animations (i.e. new Animation(...))

Explicitly constructed animations do not automatically get triggers. An author can attach a trigger
to their explicitly constructed WAAPI animation by invoking the [method to attach a trigger](#method-to-attach-a-trigger). When
this method is invoked on an animation with a specified timeline and specified trigger conditions,
the trigger will play the animation when it trips into its `primary` **_state_**.

A WAAPI equivalent of the scroll-triggered CSS example above is:

```css
 var effect = new KeyframeEffect(elem, { opacity: [0, 1] },
                                 { duration: 2000, fill: 'both' });

 var animation = new Animation(effect, elem.ownerDocument.timeline);
 // The effect is not active yet.

 var trigger = new AnimationTrigger({
        type: "alternate",
        timeline: new ViewTimeline({ subject: elem, axis: "y" }),
        rangeStart: "contain 0%",
        rangeEnd: "contain 0%"});
 // The effect is still not active.

 trigger.addAnimation(animation);
 // Now, because the has been attached to the trigger, the effect is 
 // paused in the before phase and the fill mode is reflected in the     
 // visual state.
```

#### Implicitly Constructed Animations (i.e Element.animate)

We can add an optional `AnimationTrigger` parameter to `Element.animate` which will, similar to a
CSS animation, be automatically attached to the created Animation.

#### play(), pause(), cancel(), reverse(), finish()

These APIs function as before, but also:
- Remove the animation from any `AnimationTrigger.animationList` where it is present.

The same applies to
[setting the currentTime](https://drafts.csswg.org/web-animations-1/#setting-the-current-time-of-an-animation)
and
[setting the startTime](https://drafts.csswg.org/web-animations-1/#setting-the-start-time-of-an-animation)
of an animation.

### Repeat triggers & "resets"
A repeat-type trigger resets it animation when its exit condition is met. To reset its animation, a
repeat trigger will function as follows:
- If  the animation plays forwards:
[set the `currentTime`](https://drafts.csswg.org/web-animations-1/#setting-the-timeline) of the
animation to zero.
- If the animation plays backwards:
set the currentTime of the animation to the
[animation's effect end](https://drafts.csswg.org/web-animations-1/#associated-effect-end) 
- [Pause](https://drafts.csswg.org/web-animations-1/#pause-an-animation) the animation.
- Set the animation's `endpoint-inclusive` flag to false.

### Resolving the effects of multiple triggers on a single animation.

`AnimationTriggers` get the chance to take action on their animations when their timelines are
[updated](https://drafts.csswg.org/web-animations-2/#animation-frame-loop). `AnimationTriggers` will
be updated in the same order in which they are created. This means the effect of multiple triggers
operating on the same animation will be the result of performing the actions of each
trigger in the order in which the triggers where created.
