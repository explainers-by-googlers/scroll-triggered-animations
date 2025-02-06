# Explainer for the scroll-triggered animations API

<!--**Instructions for the explainer author: Search for "todo" in this repository and update all the
instances as appropriate. For the instances in `index.bs`, update the repository name, but you can
leave the rest until you start the specification. Then delete the TODOs and this block of text.**
-->
This proposal is an early design sketch by Blink Interactions team to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

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
- [Non-goals](#non-goals)
- [User research](#user-research)
- [Use cases](#use-cases)
  - [Section Slide-In](#section-slide-in)
  - [Animating Sidebar](#animating-sidebar)
- [Proposed Solution: animation-trigger](#proposed-solution-animation-trigger)
  <!--
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
    - [Use case 1](#use-case-1-1)
    - [Use case 2](#use-case-2-1)
  -->
- [Detailed design discussion](#detailed-design-discussion)
  - [Single Point vs Range Boundaries](#single-point-vs-range-boundaries)
  <!-- - [[Tricky design choice 2]](#tricky-design-choice-2)-->
- [Considered alternatives](#considered-alternatives)
  - [Modifying animation-play-state](#modifying-animation-play-state)
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

Animations triggered by scrolling are common among pages on the web.
For example, an author might do an animation to horizontally slide into view a section of a page
that should now be visible based on the current vertical scroll position. These types of effects
are usually achieved using JavaScript to both monitor the scroll position of interest and to trigger
an animation in response. However for many of such use cases, all the information needed to create
such effects can be made available declaratively. A declarative means of setting up such animations
offers web authors a more convenient, reliable and performant option than common patterns of
observing and evaluating scroll offsets via JavaScript.

Although the primary use of this project is expected to be via declarative CSS, it includes
JavaScript interfaces which help to maintain parity between the
[Web Animation API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Animations_API) and [CSS
Animations](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_animations/Using_CSS_animations).

## Goals

<!--[What is the **end-user need** which this project aims to address? Make this section short, and
elaborate in the Use cases section.]-->
This project creates a more seamless experience of scroll-position-controlled web animations.

## Non-goals

<!--[If there are "adjacent" goals which may appear to be in scope but aren't,
enumerate them here. This section may be fleshed out as your design progresses and you encounter necessary technical and other trade-offs.]-->
More convenient methods for creating scroll-position-controlled web animations.

## User research

<!--[If any user research has been conducted to inform your design choices,
discuss the process and findings. User research should be more common than it is.]-->
N/A

## Use cases

<!--[Describe in detail what problems end-users are facing, which this project is trying to solve. A
common mistake in this section is to take a web developer's or server operator's perspective, which
makes reviewers worry that the proposal will violate [RFC 8890, The Internet is for End
Users](https://www.rfc-editor.org/rfc/rfc8890).]
-->

### Section Slide-In
When a section of a page is scrolled into view, it is introduced to the user
via a smooth animation and when being scrolled out of view, it is slid out of view by a
smooth reverse animation.

### Animating Sidebar
Take the example of a page divided into scrolling content on its right half and a gallery of images
on its left.
A different picture slides into the gallery on the left as you scroll
content on the right side.

<!-- In your initial explainer, you shouldn't be attached or appear attached to any of the potential
solutions you describe below this. -->

## Proposed Solution: animation-trigger

<!--
[For each related element of the proposed solution - be it an additional JS method, a new object, a new element, a new concept etc., create a section which briefly describes it.]
// Provide example code - not IDL - demonstrating the design of the feature.

// If this API can be used on its own to address a user need,
// link it back to one of the scenarios in the goals section.

// If you need to show how to get the feature set up
// (initialized, or using permissions, etc.), include that too.
-->
This project introduces a CSS property `animation-trigger` which defines a "trigger condition" and
an "exit condition" for its associated animation.

`animation-trigger` is a shorthand for the following CSS properties (also introduced by this
project):
- `animation-trigger-type`
- `animation-trigger-timeline`
- `animation-trigger-range-start`
- `animation-trigger-range-end`
- `animation-trigger-exit-range-start`
- `animation-trigger-exit-range-end`

`animation-trigger-type` dictates how the trigger actions the playback of an animation.
It accepts the following keywords:
- `once`, which indicates that the animation should be played only the first time its trigger
    condition is met,
- `repeat`, which indicates that the animation should be played forward each time its trigger
    condition is met and reset when its exit condition is met,
- `alternate`, which indicates that the animation should be played forward each time its trigger
    condition is met and played backwards each time its exit condition is met, and
- `state`, which indicates that the animation should be played or resumed when its trigger
    condition is met and paused each time its exit condition is met.

`animation-trigger-timeline` specifies the [AnimationTimeline](https://developer.mozilla.org/en-US/docs/Web/API/AnimationTimeline) within which the trigger will evaluate whether its trigger or exit conditions have been met.

An [AnimationTimeline](https://developer.mozilla.org/en-US/docs/Web/API/AnimationTimeline) might
be time-based ([DocumentTimeline](https://developer.mozilla.org/en-US/docs/Web/API/DocumentTimeline))
or scroll-based [ScrollTimeline](https://developer.mozilla.org/en-US/docs/Web/API/ScrollTimeline).
Although this project is aimed at scroll-triggered animations, it is worth mentioning that the use
of AnimationTimeline as the underlying source for a trigger leaves open the posibility of
time-triggered animations in the future and allows scroll-triggered animations to integrate will
with the rest of the animations ecosystem.

`animation-trigger-range-start` and `animation-trigger-range-end` define the range of the trigger's
timeline which when entered constitute the "trigger condition" being met.
`animation-trigger-exit-range-start` and `animation-trigger-exit-range-end` define the range of the trigger's
timeline which when exited constitute the "exit condition" being met.

Here is an example of how `animation-trigger` might be declared:
```css
@keyframe slidein {
  from { transform: translateX(-100vw) }
  to { transform: translateX(0vw) }
}

.target {
  animation: slidein linear 1s forwards;
  animation-trigger: view() repeat contain 0% contain 100% cover 0% cover 100%;
  transform: translateX(-100vw);
}

.scroller {
  overflow-x: hidden;
  overflow-y: scroll;
}
```

The above example setups up an animation that will slide `.target` from left (outside) of the screen
into the screen as soon as its scroll container is scrolls (vertically) so that  `.target` is fully
contained within its scroll container's scrollport.

It accomplishes this by specifying `view()` as the timeline of the trigger. `view()` sets up a
[ViewTimeline](https://developer.mozilla.org/en-US/docs/Web/API/ViewTimeline) within which
"contain 0% contain 100%" marks the boundaries of scrolling within which `.target` is fully visible
and "cover 0% cover 100%" marks the boundaries of scrolling within which `.target` is at least
partially visible. This creates a scenario in which, once `.target` is fully in view (vertically
speaking) it is introduced into the scroll port and becomes visible to the user via a smooth
animation.

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

Use cases of scroll-triggered animations to typically be in view when their animation is playing.
If scrolling happens so as to skip over the animations target entirely they might not want the
animation to play. As such, specifying the triggering portion of the timeline as a range rather
than a single point allows the author to avoid having their animation played when, for example,
their page is loaded with a URL hash fragment such that the page immediately scrolls past the
animation's target.

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
Some thought was given to modifying `animation-play-state`, an existing CSS property, to incorporate
the new capabilities. However `animation-play-state` serves a different purpose altogether.
It determines whether the an "active" animation is playing or paused whereas the trigger is
controlling where the animation should be considered active or not. More to add...

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

- Yehonatan Daniv (TODO: add GitHub username)
