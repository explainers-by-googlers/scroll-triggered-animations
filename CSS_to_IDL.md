# Supporting text for CSSWG issue [#12652](https://github.com/w3c/csswg-drafts/issues/12652)

In [#12336](https://github.com/w3c/csswg-drafts/issues/12336), we resolved to use separate namespaces for different types of animation triggers, i.e. `event-trigger-*` and `timeline-trigger-*`. We still need to work out what properties/longhands exist within the namespaces.

The conversation continued in [#12611](https://github.com/w3c/csswg-drafts/issues/12611) where the [most recent suggestion](https://github.com/w3c/csswg-drafts/issues/12611#issuecomment-3206927852) was something like this:

```
.source {
  event-trigger-name: --foo;
  event-trigger-actions: --bar click press("k"), --baz press("Enter");
  /* the --foo event trigger source defined two named actions, --bar and --baz. */
}
.anim {
  animation: my-anim;
  animation-trigger: trigger(--foo, --bar play, --baz pause, press("Escape") reset);
  /* can refer to multiple trigger actions, either by "action name" or directly */
}
```

However, after trying to work through the implications of this syntax for JavaScript/IDL, I've realized it might obscure the paired nature of `timeline-trigger`'s "enter" and "exit" concepts by treating them individually, i.e. as if they are no different from discrete `event-trigger` events like `click`, `dblclick`, etc.
In particular, I've thought about what the equivalent CSS & script might look like for `timeline-trigger`.

In CSS:

```
.source {
  timeline-trigger-name: --view-trigger;
  timeline-trigger-source: view();
  timeline-trigger-entry-range-block: contain;
  timeline-trigger-actions: --enter enter;
}

.target {
  animation-trigger: trigger(--view-trigger, --enter play-forwards, exit play-backwards);
}
```

in JS:

```
let animation = new Animation(...);

let trigger = new TimelineTrigger();

trigger.addAnimation(animation, [
  { action: "enter", behavior: "play-forwards"} ,
  { action: "exit", behavior: "play-backwards"}
]);

```

The last line demonstrates that we would be conceptually treating "enter" and "exit" as unrelated things, so I think we'd be providing authors a somewhat misleading API. As [pointed out](https://github.com/w3c/csswg-drafts/issues/12611#issuecomment-3206934900) by @fantasai, having the target specify events/actions it receives might be problematic/error-prone.

Two options we have for addressing this:

(brief summary)
1. separate the `trigger()` function accepted by `animation-trigger` into `event()` and `timeline()` functions which have different signatures, or
2. put the actions/behaviors within the `event-trigger-*` and `timeline-trigger-*` namespaces.

## More details & examples

### Option 1

Option 1 builds on the following IDL model:

```
interface AnimationTriggerAttachmentOptions {
  Animation animation;
}
interface EventTriggerAttachmentOptions : AnimationTriggerAttachmentOptions {
  DOMString behavior; // e.g. "play", "play-pause", etc.
}
interface TimelineTriggerAttachmentOptions : AnimationTriggerAttachmentOptions {
  DOMString entryBehavior; // e.g. "play", "play-pause", etc.
  DOMString exitBehavior; // e.g. "play", "play-pause", etc.
}

interface AnimationTrigger {
  undefined addAnimation(AnimationTriggerAttachmentOptions options);
  undefined removeAnimation(Animation animation);
  sequence<Animation> getAnimations();
}
dictionary EventTriggerBehavior {
  DOMString rawBehavior;
}
interface EventTrigger : AnimationTrigger {
  // keeps track of what behavior to exhibit when a specified event occurs, e.g.:
  // {
  //    anim1 : { .rawBehavior = "play-alternate" },
  //    anim2 : { .rawBehavior = "play-pause" },
  // }
  record<Animation, EventTriggerBehavior> animationConfigurations;
}
dictionary TimelineTriggerBehavior {
  DOMString entryBehavior?;
  DOMString exitBehavior?;
}
interface TimelineTrigger : AnimationTrigger {
  record<Animation, TimelineTriggerBehavior> animationConfigurations;
}
```

#### Option 1 Grammar:

We'd then have the following CSS grammar:

```
<single-event-trigger>: <dashed-ident> [<event-type>]# 
event-trigger: <single-event-trigger> [, <single-event-trigger>]#

<single-timeline-trigger>: <dashed-ident> [<timeline-type>] [<entry-range-block>] [<entry-range-inline>] [<exit-range-block>] [<exit-range-inline>];
timeline-trigger: <single-timeline-trigger> [, <single-timeline-trigger>]#

<single-animation-trigger>: event(<dashed ident>, <behavior>) | timeline(<dashed ident>, <entry behavior>, <exit behavior>);
animation-trigger: <single-animation-trigger> [<space> <single-animation-trigger>]# [, <single-animation-trigger> [<space> <single-animation-trigger>]# ]#
```
For example:

```
.event_source {
  event-trigger: --click-touch-trigger click touch, --dblclick-trigger dblclick;
}
.event_target {
  animation-trigger: event(--click-touch-trigger, play) event(--dblclick-trigger, pause);
}
.timeline_source {
  timeline-trigger: --timeline-trigger view contain;
}
.timeline_target {
  animation-trigger: timeline(--timeline-trigger, play-forwards, play-backwards);
}
```
and the way this maps to JavaScript/IDL is as follows:

For `event-trigger`:

- `event-trigger: --click-touch-trigger click touch, --dblclick-trigger dblclick;` maps to:
```
let source = getElementById("source");

let click_touch_trigger = new EventTrigger(source, ["click", "touch"]);
let dblclick_trigger = new EventTrigger(source, ["dblclick"]);
```

- ` animation-trigger: event(--click-trigger, play) event(--dblclick-trigger, pause);` maps to 

```
let animation = new Animation(...);

let play_option = new EventTriggerAttachmentOptions(animation, "play");
let pause_option = new EventTriggerAttachmentOptions(animation, "pause");

click_touch_trigger.addAnimation(play_option);
dblclick_trigger.addAnimation(pause_option);
```

- `timeline-trigger: --timeline-trigger view contain;` maps to 

```
let source = getElementById("source");

let timeline_trigger = new TimelineTrigger(source, "view", "contain");
```

and
 
- `animation-trigger: timeline(--timeline-trigger, play-forwards, play-backwards);` maps to
```
let animation = new Animation();
let alternate_options = new TimelineTriggerAttachmentOptions(/*animation=*/animation,
                                                             /*entryBehavior=*/"play-forwards",
                                                             /*exitBehavior=*/"play-backwards");
timeline_trigger.addAnimation(alternate_options);
```

### Option 2

With option 2, we would have a simpler IDL foundation:

```
interface AnimationTrigger {
  undefined addAnimation(Animation animation);
  undefined removeAnimation(Animation animation);
  sequence<Animation> getAnimations();
}
dictionary EventBehaviorConfiguration {
  DOMString action;
  DOMString behavior;
}
interface EventTrigger : AnimationTrigger{
  constructor(EventTarget source, sequence<EventBehaviorConfiguration> behaviorConfigurations)
  // This maps event type, e.g. "click", "dblclick" to behavior, e.g. "play", "pause", etc. 
  record<DOMString, DOMString> behaviorConfigurations;
}
interface TimelineTrigger : AnimationTrigger {
  DOMString entryBehavior;
  DOMString exitBehavior;
}
```

#### Option 2 Grammar:

We'd then have the following CSS grammar:

```
<single-event-trigger>: <dashed ident> [<event type> [<space> <event type>]# <behavior>]#;
event-trigger: <single-event-trigger> [, <single-event-trigger>]#;

<single-timeline-trigger>: <dashed ident> <timeline type> [<entry-behavior>] [<entry-range-block>] [<entry-range-inline>] [exit-behavior] [<exit-range-block>] [<exit-range-inline>]
timeline-trigger: <single-timeline-trigger> [,<single-timeline-trigger>]#

<single-animation-trigger>: <dashed ident>;
animation-trigger: <single-animation-trigger>
```

For example: 

```
.event_source {
  event-trigger: --event-trigger click touch play dblclick pause;
}
.event_target {
  animation-trigger: --event-trigger;
}
.timeline_source {
  timeline-trigger: --timeline-trigger view play-forwards contain play-backwards;
}
.timeline-target {
  animation-trigger: --timeline-trigger;
}
```

and this maps to JavaScript/IDL as follows:

- `event-trigger: --event-trigger click touch play dblclick pause;` maps to:
```
let source = document.getElementById("source");

let event_trigger = new EventTrigger(source,
                                    [
                                     { action: "click", behavior: "play"},
                                     { action: "touch", behavior: "play"},
                                     { action: "dblclick", behavior: "pause" }
                                     ]);
```

and

- `animation-trigger: --event-trigger;` maps to:

```
let animation =  new Animation();
event_trigger.addAnimation(animation);
```

For `timeline-trigger`:

- `timeline-trigger: --timeline-trigger view contain;` maps to:

```
let source = document.getElementById("source");

let timeline_trigger = new TimelineTrigger(source, "view", "contain");
```

and 

- `animation-trigger: --timeline-trigger;` maps to:

```
let animation = new Animation();

timeline_trigger.addAnimation(animation);
```