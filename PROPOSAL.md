## `AnimationTriggers` as a mechanism entirely external to animations.

We might be able to accomplish this by introducing explicit `Animation.enableTrigger()`, `Animation.disableTrigger()` APIs and thinking of Triggers as invoking the existing `play()`, `pause()` and `reverse()` APIs (repeat triggers will have to introduce a “reset” concept).

I’ll attempt to describe complete AnimationTrigger stories.

AnimationTriggers have 4 states: `inactive`, `idle`, `normal`, `inverse`:

- `inactive`: the trigger does nothing, i.e. it cannot respond to its trigger conditions being met.
- `idle`: the trigger is “active” but its trigger conditions have not been met.
- `normal`: the last of the trigger’s conditions to have been met is the trigger condition.
- `inverse`: state, the last of the trigger’s conditions to have been met is the exit condition.

`Animation.enableTrigger()` does the following:
- If the animation’s trigger is null, throw.
- If the trigger is in inactive:
    - sets the trigger’s state to `idle`.
    - If the animation’s playState is `idle` and the animation’s fill-mode is `both | backwards`:
        - `pause()` the animation.

`Animation.disableTrigger()` does the following:
- If the trigger’s state is not `inactive`:
    - Set the trigger’s state to `inactive`.

### WAAPI Animations
A WAAPI animation with an alternate trigger can perhaps work this way:
When a WAAPI animation is created, it gets a (alternate-type) trigger which is in the `inactive` state. Its trigger can then be enabled via `Animation.enableTrigger()`, which moves it from the `inactive` state to the `idle` state. When its trigger condition is met, it moves to the `normal` state and calls `play()`. When its exit condition is met, it moves to the `inverse` state and calls `reverse()`.
At any point in time, a developer can call `play()` or `pause()` on this animation and those would function just as before. A developer can also call `cancel()` on this animation, and that would call `Animation.disableTrigger()` which puts the trigger back into the `inactive` state.

Note that since `reverse()` changes the `playBackRate` of the animation, the developer calling `play()` could result in the animation being played in the reverse direction.

### CSS Animations
A CSS animation with an alternate trigger can perhaps work this way:
When a CSS animation is created, it gets a (alternate-type) trigger which is in the `idle` state (`Animation.enableTrigger` is immediately called on it). When its trigger condition is met it will call play and transition to normal. When its exit condition is met it will call `reverse()` and transition to `inverse`. At any point in time the developer can call `play()` or `pause()` and those should work just like before. `Cancel()` will call `Animation.disableTrigger()`.

### Repeat triggers & "resets"
Repeat triggers can perhaps function by setting the `currentTime` of the animation to the before-active-boundary time of the animation’s effect and then calling pause(). I considered that this leaves even an `animation-fill-mode: none` having visual effect at the point of the reset but I think that’s actually good. If not, we can probably come up with a way to cancel the visual effect of an animation when resetting. One caveat here is that if the animation had finished before the reset was triggered, the animation could be moving from the after phase to the active phase, which, by current [spec](https://www.w3.org/TR/css-animations-2/#event-dispatch), warrants an `animationstart` event - this is probably not desirable. We'd need a mechanism to silently change phases, but as phases are an effects concept, I'm wary of a potential layering issue.

### `animation-play-state`
AnimationTriggers should respect the `animation-play-state` of CSS animations (if not nullified by WAAPI APIs) and always leave the animation paused if the `animation-play-state` is paused.
As an example using an alternate-type trigger, on meeting the trigger condition while the `animation-play-state` is `paused`, the trigger will effectively perform: `animation.play(); animation.pause();`
On meeting the exit condition, an alternate-type trigger will effectively perform `animation.reverse(); animation.pause();` while repeat-type triggers will effectively perform `animation.currentTime = 0; animation.pause()`.
