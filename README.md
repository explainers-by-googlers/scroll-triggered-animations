# Explainer for the Scroll-Triggered animations

<!--**Instructions for the explainer author: Search for "todo" in this repository and update all the
instances as appropriate. For the instances in `index.bs`, update the repository name, but you can
leave the rest until you start the specification. Then delete the TODOs and this block of text.**
-->
This proposal is a design sketch by the Blink Interactions team to describe the problem below and
solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

<!--
TODO: Fill in the whole explainer template below using https://tag.w3.org/explainers/ as a
reference. Look for [brackets].
-->
## Proponents

- Blink Interactions Team

## Participate
- https://github.com/explainers-by-googlers/scroll-triggered-animations/issues
- https://github.com/w3c/csswg-drafts/issues/8942

## Table of Contents 
<!-- [if the explainer is longer than one printed page] -->

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
<!-- - [Non-goals](#non-goals) -->
<!-- - [User research](#user-research) -->
- [Use cases](#use-cases)
  - [Section Slide-In](#section-slide-in)
  - [Animating Gallery](#animating-gallery)
- [Proposed Solution](#proposed-solution)
  - [Example](#example)
  <!--
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
    - [Use case 1](#use-case-1-1)
    - [Use case 2](#use-case-2-1)
  -->
- [Detailed design discussion](#detailed-design-discussion)
  - [Single Point vs Range Boundaries](#single-point-vs-range-boundaries)
  - [Enter Range vs Exit Range](#enter-range-vs-exit-range)
  - [CSS Syntax: Event Placement](#css-syntax-event-placement)
  - [AnimationTrigger Behavior Model](#animation-trigger-behavior-model)
  <!-- - [[Tricky design choice 2]](#tricky-design-choice-2)-->
- [Considered alternatives](#considered-alternatives)
  - [Modifying animation-play-state](#modifying-animation-play-state)
  - [Viewenter / Viewleave Events](#viewenter--viewleave-events)
  <!-- - [[Alternative 2]](#alternative-2) -->
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

<!--
[The "executive summary" or "abstract".
Explain in a few sentences what the goals of the project are,
and a brief overview of how the solution works.
This should be no more than 1-2 paragraphs.]
-->

A common user interaction design pattern among web pages is to make the playback of an animation
tied to the scroll position of a scrollable element. This often involves playing, pausing, reversing
or resetting the animation based on whether an element within the scrollable element enters or exits
the scrollport (the viewport of the scrollable element) during scrolling.

To accomplish this today, authors have to rely on some form of JavaScript, e.g. scroll event
listeners or IntersectionObservers, to detect the condition under which they would like to take action
on their animation. This means that the logic to control their animations' playback lives in the
same thread where the rest of their application logic (and other user agent work) lives, the
"main thread." This arrangement makes scroll-position-controlled animations susceptible to being
delayed by unrelated main thread work.

<!-- 
For example, an author might have an animation to horizontally slide into view a section of a page
that should now be visible based on the current vertical scroll position. These types of effects
are usually achieved using JavaScript to both monitor the scroll position of interest and to trigger
an animation in response. However for many of such use cases, all the information needed to create
such effects can be made available declaratively. A declarative means of setting up such animations
offers web authors a more convenient and performant option than common patterns of
observing and evaluating scroll offsets via JavaScript. -->

This proposal allows scroll-position-controlled animations to be set up declaratively.
A declarative setup allows web authors create seamless scroll-triggered animation experiences by
giving the user agent information that lets it offload the control of such animations to a dedicated
thread rather than running on the main thread.


## Goals

<!--[What is the **end-user need** which this project aims to address? Make this section short, and
elaborate in the Use cases section.]-->
This project aims to enable seamless scroll-position-controlled web animations.

<!-- ## Non-goals
Although the primary use of this project is expected to be via declarative CSS, it includes
JavaScript interfaces which help to maintain parity between the
[Web Animation API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Animations_API) and [CSS
Animations](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_animations/Using_CSS_animations).

[If there are "adjacent" goals which may appear to be in scope but aren't,
enumerate them here. This section may be fleshed out as your design progresses and you encounter necessary technical and other trade-offs.]
The API should provide authors convenient methods for creating scroll-position-controlled web
animations. -->

<!-- ## User research

[If any user research has been conducted to inform your design choices,
discuss the process and findings. User research should be more common than it is.]
N/A -->

## Use cases

<!--[Describe in detail what problems end-users are facing, which this project is trying to solve. A
common mistake in this section is to take a web developer's or server operator's perspective, which
makes reviewers worry that the proposal will violate [RFC 8890, The Internet is for End
Users](https://www.rfc-editor.org/rfc/rfc8890).]
-->

The declarative API put forward by this proposal aims to reduce the possibility of lag between a user's
scrolling action and an animation an author has constructed to be tied to that action. The following
are use cases that would typically be affected by such lag.

### Section Slide-In
A common example of scroll-position-controlled animations animates the entry and the exit of
elements within the scrollport of a scrollable element. When the element, e.g. a section of a page, is
scrolled into view, it is introduced to the user via a smooth translation and/or opacity animation
and when being scrolled out of view the animation is played in reverse. If the animation is delayed
by other application work, the user is subjected to a visual delay, resulting in a degraded web
experience.

### Animating Gallery
Authors also often tie the visiblity of one element to an animation on a separate element, i.e. an
element not within the same scrollport. For example, a page could have two halves: scrolling content
on its right half and a gallery of images on its left. The images in the gallery are connected to
sections within the scrolling content such that when the section comes into view the appropriate
image in the gallery is animated into view. As with the previous example, lag between the scrolling
and the animation makes for a bad web experience.

<!-- In your initial explainer, you shouldn't be attached or appear attached to any of the potential
solutions you describe below this. -->

## Proposed Solution

<!--
[For each related element of the proposed solution - be it an additional JS method, a new object, a new element, a new concept etc., create a section which briefly describes it.]
// Provide example code - not IDL - demonstrating the design of the feature.

// If this API can be used on its own to address a user need,
// link it back to one of the scenarios in the goals section.

// If you need to show how to get the feature set up
// (initialized, or using permissions, etc.), include that too.
-->

This proposal introduces an `animation-trigger` CSS property which, in coordination with the existing
`animation` property, allows authors declaratively specify playback control of their animations.

This proposal also introduces a CSS property `timeline-trigger` which defines the conditions under
which "enter" and "exit" are considered to have occured.

`timeline-trigger` builds on the existing concepts behind the
[`animation-timeline`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-timeline) and
[`animation-range`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-range) properties which provide extensive syntax for expressing scroll-related
information.

`timeline-trigger` is a shorthand for the following CSS properties (also introduced by this
proposal):
- `timeline-trigger-name`
- `timeline-trigger-source`
- `timeline-trigger-range-start`
- `timeline-trigger-range-end`
- `timeline-trigger-exit-range-start`
- `timeline-trigger-exit-range-end`

`timeline-trigger-name` names the trigger, allowing it to be referred to by `animation-trigger`.

`timeline-trigger-source` specifies the
[AnimationTimeline](https://developer.mozilla.org/en-US/docs/Web/API/AnimationTimeline) within which
the trigger will evaluate whether its trigger or exit conditions have been met.

`timeline-trigger-range-*` specify the boundaries of the `timeline-trigger-source` that define the
trigger's "enter" and "exit" conditions.

### Example 

Here is an example of HTML & CSS that could be used to implement the Section Slide-In example
described above.

CSS:
```css
@keyframes fadein {
  from { transform: translateX(-50px); opacity: 0 }
  to { transform: translateX(0px); opacity: 1 }
}

#section {
  animation: fadein 0.5s both;
  timeline-trigger: --trigger view() contain 0% contain 100%;
  animation-trigger: --trigger play-forwards play-backwards;
}
```

HTML:
```HTML
<div id="scroll-container">
  <div>Lots of other content.</div>
  <div id="section"></div>
  <div>Lots of other content.</div>
</div>
```

The above example sets up an animation that will slide `#section` from left (outside) of the screen
into the screen as soon as its scroll container is scrolls so that `#section` is fully (vertically)
contained within `scroll-container`'s scrollport.

It accomplishes this by specifying `view()` as the timeline of the trigger. `view()` sets up a
[ViewTimeline](https://developer.mozilla.org/en-US/docs/Web/API/ViewTimeline) within which
"contain 0% contain 100%" marks the boundaries of scrolling that make `#section` fully
visible. This creates a scenario in which, once `#section`'s non-animated position is fully in view
(vertically speaking) it is introduced into the scroll port and becomes visible to the user via a
smooth animation.


To accomplish a similar effect with a scroll event listener, an author would need to write script
similar to the following:

```JS
function evaluate_in_viewport(element, scroll_container) {
  /* Logic evaluating whether element is within scroll_container's scrollport. */
}

const last_evaluated = false;

function setup_animation_trigger(element) {
  const scroll_container = document.getElementById("scroll-container");

  const animation = new Animation(new KeyframeEffect(element,
    [
      { transform: "translateX(-50px)", opacity: 0 },
      { transform: "translateX(0px)", opacity: 1 },
    ],
    {
      duration: 500,
    }
  ));

  scroll_container.addEventListener("scroll", () => {
    const in_viewport = evaluate_in_viewport(element, scroll_container);

    if (in_viewport != last_evaluated) {
      const playback_rate = Math.abs(animation.playbackRate);
      if (in_viewport) {
        animation.playbackRate = playback_rate;
      } else {
        animation.playbackRate = -playback_rate;
      }
      animation.play();
      last_evaluated = in_viewport;
    }
  });
}

document.onload = () => {
  /* Other application logic */
  /*            ...          */

  const section = document.getElementById("section");
  setup_animation_trigger(section);
}

```

Thier reliance on script means their logic for detecting the trigger condition could be delayed by
other main thread work.
<!--
[Where necessary, provide links to longer explanations of the relevant pre-existing concepts and API.
If there is no suitable external documentation, you might like to provide supplementary information as an appendix in this document, and provide an internal link where appropriate.]

[If this is already specced, link to the relevant section of the spec.]

[If spec work is in progress, link to the PR or draft of the spec.]

[If you have more potential solutions in mind, add ## Potential Solution 2, 3, etc. sections.]

### How this solution would solve the use cases

[If there are a suite of interacting APIs, show how they work together to solve the use cases described.]

#### Use case 1

[Description of the end-user scenario]

```js
// Sample code demonstrating how to use these APIs to address that scenario.
```

#### Use case 2

[etc.]
-->

## Detailed design discussion
<!--
### [Tricky design choice #1]

[Talk through the tradeoffs in coming to the specific design point you want to make.]

```js
// Illustrated with example code.
```

[This may be an open question,
in which case you should link to any active discussion threads.]
-->
### Single Point vs Range Boundaries

Use cases of scroll-triggered animations typically expect the animating element to be in view when
their animation is playing. Scrolling may happen so as to skip over the animating element entirely,
e.g. if the page is loaded with a URL hash fragment such that the page instantaneously scrolls past the
animation's target. In this intstance the author might not want the animation to play. As such,
specifying the triggering portion of the timeline as a range rather than a single point gives the
author the flexibility to opt into either behavior. A single point boundary can be accomplished by
having the trigger range extend to the extent of the scroll range, effectively making the range
unskippable.

### Enter Range vs Exit Range?

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

### CSS Syntax: Event Placement
In [CSSWG Issue 12652](https://github.com/w3c/csswg-drafts/issues/12336), the CSS Working group
considered the question of whether `animation-trigger` should specify "enter" and "exit" (or UIEvents
in the case of `event-trigger`) along with the animation playback actions, i.e "play", "pause",
"reset", etc.

Under this syntax, the Slide-In example from earlier would be specified similarly, exept for the
`animation-trigger` declaration:
```css
#section {
  animation: fadein 0.5s both;
  timeline-trigger: --trigger view() contain 0% contain 100%;
  animation-trigger: --trigger enter play-forwards exit play-backwards;
}
```

The working group decided that a cleaner API would be to declare the events at the
source and the actions at the target, as is reflected in this proposal ("enter" and "exit" are
implicitly defined by `timeline-trigger`).

### Animation Trigger Behavior Model
In [CSSWG Issue 12119](https://github.com/w3c/csswg-drafts/issues/12119) the CSS working group
considered 2 higher level models for the relationship between triggers and `Animations`.

In the "arming" model, the trigger is a mechanism internal to how an animation works. This model
makes triggers an integral part of animations, i.e. an animation cannot play unless its trigger
condition has been met. This model redefines the methods of the Animation API. For example, `play()`
is defined as "arming" the trigger, putting it in a state where, when its condition is met, causes
the animation to make progress.

In the "controller" model, the trigger is an external actor that can invoke the methods of the
Animation interface, i.e. `play()`, `pause()`, etc, on an animation when its trigger condition is
met.

The working group decided that the controller model better matches developers' expectations. For
instance, a developer would expect that invoking `play()` would cause an animation to play
regardless of whether its trigger condition has been met.

<!--
### [Tricky design choice 2]

[etc.]
-->
## Considered alternatives

<!--
[This should include as many alternatives as you can,
from high level architectural decisions down to alternative naming choices.]
-->
### Modifying `animation-play-state`

<!--
[Describe an alternative which was considered,
and why you decided against it.]
-->
Some thought was given to modifying
[`animation-play-state`](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-play-state), an
existing CSS property, to incorporate the new capabilities. However `animation-play-state` serves a
different purpose altogether. It determines whether an animation is "active" or not whereas a
trigger is controlling when and whether an active animation is playing forwards or backwards, or is
to be reset or paused.

### Viewenter / Viewleave Events

In [this CSSWG Github Issue](https://github.com/w3c/csswg-drafts/issues/12336), some consideration was
given to leveraging a related proposal,
[Declarative Interactions](https://szager-chromium.github.io/declarative-interactions/) which also
builds on the `animation-trigger` property. The idea was to create `viewenter()` and `viewleave()`
events which could be declared by the `event-trigger` property. Roughly speaking `viewenter()` and
`viewleave()`would capture `timeline-trigger`'s "enter" and "exit" concepts, respectively. This
approach was decided against
as [event-triggers](https://drafts.csswg.org/css-animations-2/#event-triggers) operate on a
fundamentally different model that doesn't capture some of the details of `timeline-triggers`.
To summarize, `timeline-trigger`s operate based on concepts of "enter" and "exit" which are not
defined independently of each other whereas the events of an `event-trigger` are (some subset of)
independently defined UIEvents.

<!--
### [Alternative 2]

[etc.]
-->
## Stakeholder Feedback / Opposition

<!--
[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]
-->
N/A at the moment. Will add when vendor positions requested.

## References & acknowledgements

<!--
[Your design will change and be informed by many people; acknowledge them in an ongoing way! It helps build community and, as we only get by through the contributions of many, is only fair.]

[Unless you have a specific reason not to, these should be in alphabetical order.]
-->
Many thanks for valuable feedback and advice from:

- @ydaniv
- @fantasai
- @tabatkins
- @flackr
