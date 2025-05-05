## `AnimationTriggers` (as external to animations)

We can accomplish this by introducing explicit `Animation.enableTrigger()` and `Animation.disableTrigger()` APIs and 
thinking of Triggers as invoking the existing [`play()`](https://drafts.csswg.org/web-animations-1/#play-an-animation),
[`pause()`](https://drafts.csswg.org/web-animations-1/#pausing-an-animation-section) and
[`reverse()`](https://drafts.csswg.org/web-animations-1/#reverse-an-animation) APIs (repeat triggers will have 
to introduce a “reset” concept).

AnimationTriggers have 4 states: `inactive`, `idle`, `normal` and `inverse`:

- `inactive`: the trigger does nothing, i.e. it cannot respond to its trigger conditions being met.
- `idle`: the trigger is not inactive but its trigger conditions have not been met.
- `normal`: the trigger is not inactive and the last of the trigger’s conditions to have been met is the trigger condition.
- `inverse`: the trigger is not inactive and the last of the trigger’s conditions to have been met is the exit condition.

`Animation.enableTrigger()` does the following:
- If the animation’s trigger is null, throw.
- If its trigger's state is `inactive`:
    - Set its trigger’s state to `idle`.
    - If the animation’s [`playState`](https://drafts.csswg.org/web-animations-1/#dom-animation-playstate) is `idle` 
    and its [`animation-fill-mode`](https://drafts.csswg.org/css-animations/#animation-fill-mode) is `both | backwards`:
        - `pause()` the animation.

`Animation.disableTrigger()` does the following:
- If the animation’s trigger is null, throw.
- If the animations's trigger’s state is not `inactive`:
    - Set the animation's trigger’s state to `inactive`.

Note: So that authors have flexibility around assigning and updating an animation's trigger via script,
the `Animation` interface's `trigger` field is not readonly. This means that multiple animations may be assigned the same trigger.
However, we would not want the enabling or disabling of one animation's trigger to interfere with that of another animation.
And so the enabling or disabling of a trigger is done on a per-Animation basis and the trigger state is tracked on a per-Animation 
basis. This is why we have `Animation.enableTrigger` rather than `AnimationTrigger.enable`. Subsequent references to the state of
a trigger refer to state that is particular to only the relevant animation.

### [Web Animations API](https://drafts.csswg.org/web-animations/) (WAAPI) Animations

#### Default AnimationTrigger & Existing Animations

A WAAPI `Animation`, by default, gets the default `AnimationTrigger` (whose timeline is the `document.timeline`). This 
trigger is initially in the `inactive` state. As the trigger is in the `inactive` state it cannot `play()` the animation, so the animation 
remains in the `idle` `playState` and has no visual effect, consistent with pre-AnimationTrigger WAAPI animations (aka existing 
WAAPI animations). Calling `enableTrigger` on the animation (which can't happen for existing animations) will transition the trigger 
from the `inactive` state to the `idle` state whereupon it will transition to the `normal` state and call `play()` on
the animation. This automaitc transition to the `normal` state occurs because the `document.timeline` only produces values of time 
and currently, the only valid conditions which `AnimationTrigger` works with are scroll position-based. So a non-`inactive` trigger 
with a `document.timeline` immediately `play()`s and does not function any further.

#### A WAAPI Animation with an alternate AnimationTrigger:

An alternate `AnimationTrigger` can be constructed and assigned to an Animations's `trigger` field. This alternate trigger is 
initially in the `inactive` state and cannot `play()` the animation. Then, an author can call
`Animation.enableTrigger()`, which moves animation's trigger from the `inactive` state to the `idle` state. When its trigger 
condition is met, the trigger moves to the 
`normal` state and calls `play()`. When its exit condition is met, it moves to the `inverse` state and calls `reverse()`.
At any point in time, a developer can call `play()` or `pause()` on this animation and those would function just as before. A 
developer can also call `cancel()` on this animation, and that would call `Animation.disableTrigger()` which puts the trigger back 
into the `inactive` state.

An example of this put together in code:

```js
function setupEntryExitAnimation(section) {
    const animation = new Animation(new KeyframeEffect(section, [
        { transform: "translateX(-50px)", opacity: 0.5 },
        { transform: "translateX(0px)", opacity: 1 },
    ], { duration: 500, fill: "both" }));
    // At this point, this animation won't have any effect on the target element.

    const entryExitTrigger = new AnimationTrigger({
        type: "alternate",
        timeline: new ViewTimeline({ subject: section, axis: "y" }),
        rangeStart: "entry 50%",
        rangeEnd: "exit 50%",
    });
    animation.trigger = entryExitTrigger;
    // At this point, the animation still has no effect on the target element.

    animation.enableTrigger();
    // Because the animation has fillMode "both", at the next opportunity, the user agent will cause
    // the first keyframe, { "translateX(-50px)", opacity: 0.5 }, to be in effect on the target element.
    // The first keyframe will stay in effect until the trigger's condition is met causing the animation
    // to advance and subsequent keyframes to be shown.
}

function setupSections() {
    const sections = document.querySelectorAll(".section");
    sections.forEach((s) => { setupEntryExitAnimation(s); });
    ...
}

function main () {
    setupSections();
    ...
}

```

Note that since `reverse()` changes the [`playBackRate`](https://drafts.csswg.org/web-animations-1/#dom-animation-playbackrate) of 
the animation, the developer calling `play()` directly could result in the 
animation being played in the reverse direction. This seems like the correct behavior. Further, through 
[`updatePlaybackRate`](https://drafts.csswg.org/web-animations-1/#dom-animation-updateplaybackrate) a developer could get the exact 
behavior they want. Of course, they'd need to be aware that triggers function this way, but it 
seems reasonable that if a developer has set up and enabled a trigger so that their animation plays in the reverse direction 
under certain conditions, if they directly invoke `play()` while those conditions are still true, their expectation is that the 
animation should play in the reverse direction.

### CSS Animations

#### Default AnimationTrigger & Existing CSS Animations

When an author declares the [`animation`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation) property on an element but not the `animation-trigger` property, that `Animation` gets the default trigger. This trigger is initially in the `inactive` 
state. However, just as `play()` is currently automatically called for CSS animations but not for WAAPI animations, `Animation.enableTrigger` is called on the CSS animation, causing the animation to immediately `play()`, consistent with existing CSS 
animations.

#### CSS Animation with an alternate AnimationTrigger

To create a CSS animation with an alternate `AnimationTrigger`, an author will declare the `animation-trigger` property with
a `view()` or `scroll()` timeline, along with the `animation` property. As with the default case, `enableTrigger` is immediately 
called on this animation, moving the trigger from the `inactive` state to the `idle` state. Now in the `idle` state, the trigger 
will respond to its trigger condition being met by transitioning to the `normal` state and calling `play()` on the animation.
If or when its exit condition is met, the trigger will transition to the `inverse` state and call
`reverse()` causing the animation to play in the reverse direction.

Here is sample CSS doing so:
```css
#target {
    ...
    animation: fade-in linear .5s both;
    animation-trigger: view() alternate contain 0% contain 100%;
}
```

At any point, the developer can call `play()` or `pause()` and those will work just like before.
Similar to WAAPI animations, an author can call `cancel()` on the animation, causing the animation's visual effect to be removed
as well as calling `Animation.disableTrigger()`.

### Repeat triggers & "resets"
A repeat-type trigger resets it animation when its exit condition is met. To reset its animation, a repeat trigger will function
by [setting the `currentTime`](https://drafts.csswg.org/web-animations-1/#setting-the-timeline) of the animation to the
[before-active-boundary-time](https://drafts.csswg.org/web-animations-1/#before-active-boundary-time) of the 
animation’s effect (if the animation plays forwards) or the [active-after-boundary-time](https://drafts.csswg.org/web-animations-1/#active-after-boundary-time) (if the animation plays backwards) and then calling `pause()`. This leaves even an `animation-fill-mode: none` animation having visual 
effect at the point of the reset. This proposal assumes this is the right thing to do but this is an issue worth having a discussion about. If it is not the right thing to do, we can probably come up with a way to cancel the visual effect of an animation when resetting.

One caveat is that if the animation had finished before the reset was triggered, the 
animation's effect would be moving from the [after phase](https://drafts.csswg.org/web-animations-1/#animation-effect-after-phase) to the [active phase](https://drafts.csswg.org/web-animations-1/#animation-effect-active-phase) (or [before phase](https://drafts.csswg.org/web-animations-1/#animation-effect-before-phase), whichever we call it), which, by current
[spec](https://www.w3.org/TR/css-animations-2/#event-dispatch), warrants an `animationstart` event - this is probably not desirable. 
We can address this by introducing an `eventDispatchEnabled` bit/flag for `Animation` (not web observable) which when true, allows 
events related to that animation to be dispatched and does not allow it otherwise. The trigger can temporarily set this bit to false 
while performing this reset.

### `animation-play-state`
AnimationTriggers should respect the `animation-play-state` of CSS animations (if not nullified by WAAPI APIs) and always leave the 
animation paused if the `animation-play-state` is paused.
As an example using an alternate-type trigger, on meeting the trigger condition while the `animation-play-state` is `paused`, the 
trigger will effectively perform: `animation.play(); animation.pause();`
On meeting the exit condition, an alternate-type trigger will effectively perform `animation.reverse(); animation.pause();` while 
repeat-type triggers will effectively perform `animation.currentTime = 0; animation.pause()`.

### `Animation.playState` & `Animation.currentTime`
Giving `enableTrigger` the ability to call `pause()` will mean that `enableTrigger` will transition an animation from `idle` to 
`paused` playState, depending on its `animation-fill-mode`. This doesn't seem problematic. In fact, it seems to be
a better alternative to having an animation which is in `idle` playState while maintaining visual effect.
Similarly, `enableTrigger` will change the `currentTime` of an animation from `null` to zero. These seem quite resonable
as they better reflect that the animation currently has a visual effect due to its `animation-fill-mode`.

### once-type `AnimationTriggers`
`AnimationTriggers` of type `once` are to play their animation only once. Under this proposal, "play their animation only once" is 
scoped to the trigger's "active lifetime". I.e. while a trigger is not "inactive" it will play its animation only once. If it is 
disabled (i.e. transitioned to "inactive") and then re-enabled, it may once again play its animation once.

### `animation.ready`
This proposal does not modify the way [`animation.ready`](https://drafts.csswg.org/web-animations-1/#dom-animation-ready) works. Of 
course, by invoking the `play()`, `pause()` and `reverse()`,
`AnimationTrigger` will cause an animation to create, update and resolve its ready promise but since it will be doing so strictly 
through these established methods, `AnimationTrigger` creates no new question surrounding the management, meaning or semantics of
animation.ready`.
