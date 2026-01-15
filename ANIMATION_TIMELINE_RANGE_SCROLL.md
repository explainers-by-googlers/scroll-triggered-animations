# Explainer for ViewTimeline Range "scroll"

## Summary
A [ViewTimeline](https://www.w3.org/TR/scroll-animations-1/#view-timelines) represents the portion
of a scroll container's scroll range within which an element, the ViewTimeline's subject, is
visible.

Authors can refer to sub-portions of a ViewTimeline using named ranges, e.g. "entry", "exit",
"contain" and "cover", and tie the progress of an animation directly to the progress of the scroll
container through that range.

However, all the [ranges](https://www.w3.org/TR/scroll-animations-1/#view-timelines-ranges)
available to authors are limited to the portion of the scroll container within which the
ViewTimeline's subject is visible. This means that in scenarios where an
author would like to construct, for example, a range which is bounded on one end by the scroll
container's extremity and on the other by a point at which the subject is visible, the author has to
compute the scroll offset (relative to the default range of the ViewTimeline) that corresponds to the
scroll-extremity-based boundary.

### Examples

#### Progress Indicator

An author might want to build a progress indicator that not only indicates progress when the section
is within view but also indicates how far from the scroll port the section is when the subject is
out of view. An author might use such a UI with a sidebar table of contents (TOC) with each TOC item
displaying some indication of how far the corrersponding section is above or below the scroll port.

With the current set of named ranges, the author would have to do some JavaScript
calculations and then update the elements' style as in the following example:

CSS:
```CSS
@keyframes expand {
  from { width: 0% }
  to { width: 100% }
}
.scroller {
  overflow-y: scroll;
  height: 200px;
  width: 200px;
  border: solid 1px;
}
.subject {
  height: 100px;
  width: 100px;
  background-color: pink;
  view-timeline: --timeline;
}
.pad {
  height: 400px;
  width: 200px;
}
#indicators {
  height: 5px;
  width: 200px;
  position: relative;
}
.pre {
  background-color: skyblue;
}
.in {
  background-color: lemonchiffon;
}
.post {
  background-color: orange;
}
.indicator {
  height: 5px;
  animation: expand both;
  animation-timeline: --timeline;
  position: absolute;
}
.wrapper {
  timeline-scope: --timeline;
}
```
HTML:
```HTML
<div class=wrapper>
    <div id=indicators>
      <div class="pre indicator" id=pre_indicator></div>
      <div class="in indicator" id=in_indicator></div>
      <div class="post indicator" id=post_indicator></div>
    </div>
    <div id=scroller class=scroller>
      <div class=pad></div>
      <div id=subject class=subject></div>
      <div class=pad></div>
    </div>
</div>
```
JS:
```JS
function ComputeStartToEntryRange(scroller, subject) {
    const offset = (subject.offsetTop - scroller.offsetTop) - scroller.offsetHeight;
    return `cover -${offset}px entry 0%`;
}

function ComputeEntryToExitRange(scroller, subject) {
  return `entry exit`;
}

function ComputeExitToEndRange(scroller, subject) {
  const offset = (subject.offsetTop - scroller.offsetTop) + subject.offsetHeight;
  return `exit 100% cover ${offset}px`;
}

const pre_indicator = document.getElementById("pre_indicator");
const start_to_entry_range = ComputeStartToEntryRange(scroller, subject);
pre_indicator.style.animationRange = `${start_to_entry_range}`;

const in_indicator = document.getElementById("in_indicator");
const entry_to_exit_range = ComputeEntryToExitRange(scroller, subject);
in_indicator.style.animationRange = `${entry_to_exit_range}`;

const post_indicator = document.getElementById("post_indicator");
const exit_to_end_range = ComputeExitToEndRange(scroller, subject);
post_indicator.style.animationRange = `${exit_to_end_range}`;
```

Here is an image of the above example:

![image](https://services.google.com/fh/files/misc/scroll_pre_post_indicator.gif)

The blue bar indicates how far the pink square (the subject) is above the scroll port, the yellow
bar indicates how far along the visible range the subject is and the orange bar indicates how far
below the scroll port the subject is.

The author would also need to set up their application so that the page knows to recompute the
boundaries should the layout of the page change.

#### One-Sided Scroll-Triggered Animations

[TimelineTriggers](https://drafts.csswg.org/css-animations-2/#timeline-triggers) operate on the
entry and the exit of a specified range on a specified timeline. In the typical scenario, an author
would configure an animation to play forward when the range is entered and play backwards (or perhaps be
reset) when the range is exited. This makes both boundaries of the specified range trigger points. In some
cases, an author might want a single trigger point. To accomplish this, the author effectively has
to set one of the boundaries to some form of infinity, so that that boundary cannot act as a trigger
point.

With the current set of ranges, an author would have to rely on similar calculations as
`ComputeStartToEntryRange` and `ComputeExitToEndRange` above to make a range boundary act as
(+/-)infinity. Or, they could use a very large value they believe will effectively act as infinity,
e.g. `timeline-trigger-range: entry 50% cover 10000%`.

Here is an example:

CSS:
```CSS
@keyframes slide {
  from { left: -50px; opacity: 0; }
  to { left: 0px; opacity: 1; }
}

.scroller {
  overflow-y: scroll;
  height: 200px;
  width: 200px;
  border: solid 1px;
  position: relative;
}

.source {
  animation: slide .3s both;
  animation-trigger: --trigger play-forwards play-backwards;
  timeline-trigger: --trigger view() entry 50% cover 10000%;
  height: 100px;
  width: 99%;
  background-color: pink;
  position: relative;
}

.pad {
  height: 400px;
  width: 50px;
}
```

HTML:
```HTML
<div class=scroller>
  <div class=pad></div>
  <div id=source class=source></div>
  <div class=pad></div>
</div>
```

where the subject's being at the bottom of the scroll container triggers the playing forwards or
backwards but its being at the top does not.

## Proposed Solution
This explainer describes a "scroll" range which gives authors the ability to refer to the entire
scroll range of the scroll container underlying a ViewTimeline.

### Examples Revisited

#### Progress indicator
To achieve the progress indicator described above, the author would be able to set the
`animation-range` of the animating elements as follows:

```CSS
.pre {
  background-color: skyblue;
  animation-range: scroll entry;
}
.in {
  background-color: lemonchiffon;
  animation-range: entry exit;
}
.post {
  background-color: orange;
  animation-range: exit scroll;
}
```

and remove the JavaScript computations.

#### One-Sided Scroll-Triggered Animation

For the one-sided scroll-triggered animation, they can modify `timeline-trigger` by replacing
`cover 10000%` with `scroll 100%`.

```CSS
.source {
  animation: slide .3s both;
  animation-trigger: --trigger play-forwards play-backwards;
  timeline-trigger: --trigger view() entry 50% scroll 100%;
  height: 100px;
  width: 99%;
  background-color: pink;
  position: relative;
  border: solid 1px;
}
```
