# Container Timing: contentful paint attribution and the aggregate view

**Author:** Jose Dapena Paz (Igalia) &lt;jdapena@igalia.com&gt;

**Last updated:** 2026-09-17

**Status:** Proposal for discussion.

**Audience:** assumes working knowledge of Container Timing, Element Timing, LCP and ICP. Background on those is not repeated.

**Participate:**

- Spec draft: https://wicg.github.io/container-timing/
- Repository / issues: https://github.com/WICG/container-timing/

## Table of contents

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [What is wrong with the entry today](#what-is-wrong-with-the-entry-today)
- [Motivating use cases](#motivating-use-cases)
- [Proposed API](#proposed-api)
- [Key scenarios](#key-scenarios)
- [Detailed design discussion](#detailed-design-discussion)
- [Semantics that need explicit spec text](#semantics-that-need-explicit-spec-text)
- [Deprecations and migration](#deprecations-and-migration)
- [Considered alternatives](#considered-alternatives)
- [Privacy and security considerations](#privacy-and-security-considerations)
- [Stakeholder feedback](#stakeholder-feedback)
- [Decisions to take](#decisions-to-take)
- [References](#references)

## Introduction

The starting point for this explainer was to align Container Timing's attribution with the shape ICP converged on. ICP attributes contentful paints to a scope by nesting a whole `LargestContentfulPaint` in the attributing entry; Container Timing does something similar with `lastPaintedElement`, an `Element?` that carries no size, no URL, no load time and no render time of its own. Adopting ICP's shape is mostly mechanical, and the paint feed already carries every field the nested object needs.

Working through it surfaced two things that are not mechanical, so this document will also cover those.

#### The entry cannot carry the container's accumulated state

`lastPaintedElement` resets its comparison on each emission, so it describes one update. But a developer also wants the container hero content over its whole life, and that is not derivable from an entry stream you may not have received.

One concrete case today: a contentful paint landing entirely inside the already-painted region queues no entry at all (the container did not grow) while still being able to be the largest thing the container ever painted. [Decision 2](#decisions-to-take) proposes closing that hole at the trigger, which would be a good change on its own, but it does not remove the need for the aggregate: an entry reports what changed at the moment it was queued, and the state has to be reachable later, by code that was not listening then.

The entry buffer is a second case: it is capped, so on a busy page the entries carrying the state are eventually dropped, and code that reads them later sees a truncated history.

#### So the division of labour should be explicit

The timeline carries _updates_ (what changed, when, and what content drove it). Accumulated state is reached by _asking_ ( `Element.getContainerTimingInfo()`) with no observer required and no dependence on buffering.

Every attribute below follows from that split, and it is also the framing under which [Decision 1](#decisions-to-take) (do `container` entries still need a timeline buffer at all?) becomes answerable.

#### Where this sits relative to the origin trial

The Chrome origin trial runs from m148 to m153 and closes around October 2026. This document was first written inside that window, when every change had to be additive, and that constraint is expiring: discussion of this proposal is expected over the coming weeks, and an implementation prepared now would land after the trial has closed.

That changes what is on the table rather than what is proposed. The attribute changes are additive either way and do not depend on the timing. What the closing trial unlocks is the removal of the attributes this document deprecates, and the two decisions that were held back precisely because they are not additive: whether `container` entries keep a timeline buffer ([Decision 1](#decisions-to-take)) and whether an entry should also be queued when the container's largest contentful paint grows ([Decision 2](#decisions-to-take)).

The staging this implies:

- **Landing during the trial**, if any of it does: additive only. `lastPaintedElement` and `firstRenderTime` are deprecated but keep working, and `startTime`, `duration`, `name`, `entryType`, `identifier`, `rootElement`, `intersectionRect` and the emission trigger are untouched.
- **Landing after it closes**: the deprecated attributes can be removed in the same change rather than left to a later one, and Decisions 1 and 2 can be taken on their merits instead of on what the trial can absorb.

Where this document says an attribute is kept for compatibility, read it as compatibility with what trial participants measured, which stops binding once the trial is over. Where a choice also has a design argument, that argument is stated separately and stands on its own.

## Goals

- Align Container Timing's attribution with ICP's: a nested `LargestContentfulPaint` instead of an element reference, with `size` measured the way LCP measures it so the values are comparable across the three APIs.
- Split per-update reporting from accumulated state, and make the accumulated state reachable without an observer and without depending on the entry buffer.
- Avoid gratuitous breakage for origin trial participants: anything not additive is staged for after the trial closes rather than dropped into it.

## Non-goals

- **Changing `startTime` / `duration` anchoring.** ICP anchors `startTime` to the interaction that caused the paint. Out of scope, and for a reason that outlives the trial: under ICP's convention `startTime` would become constant per container root, which reorders `getEntriesByType()` results by container instead of chronologically. The compatibility argument is the weaker of the two and is the one that expires.
- **Adopting LCP's candidate eligibility rules.** Container Timing deliberately reports paints LCP declines to treat as candidates. See [Container Timing's contentful set is its own](#container-timings-contentful-set-is-its-own-and-should-be-stated-as-such).
- **Removing the deprecated attributes, if any part of this lands during the trial.** Once it closes, removal moves back on the table and is the preferred end state.
- **Deciding when recording stops.** Container Timing records for the document's lifetime today and this document does not change that, but the question deserves its own discussion rather than a decision buried in a proposal about attributes. Both sides are real: stopping bounds the memory and per-paint cost described in [What maintaining the aggregate costs](#what-maintaining-the-aggregate-costs), while continuing is what developers already depend on, since containers legitimately keep updating and people use the entries that arrive after input or scroll. One shape worth exploring is to stop under LCP's conditions but let containers added after a stop record until the next one, which scopes recording per interaction the way ICP does rather than stopping once and permanently the way LCP does. Whatever is chosen has to say what becomes of the aggregate: with a stop condition, `largestContentfulPaint` and `size` accumulate over a recording window rather than over the container's whole life, and `getContainerTimingInfo()` should keep answering with a frozen view rather than going null, since a container that has stopped recording is still in the document and still has numbers worth reading.
- **Clearing or resizing the entry buffer.** Moot if [Decision 1](#decisions-to-take) drops the buffer, and a separate question if it stays. A long lived page eventually fills it and loses everything after that, and the platform's only precedent for handling this is Resource Timing, which has both `clearResourceTimings()` and `setResourceTimingBufferSize()`. Element Timing has neither, and Container Timing follows Element Timing today. Whether container entries deserve the Resource Timing treatment is a question about buffer management rather than about attribution, so it is left out. Note the aggregate is unaffected either way, since it is not held in the buffer and cannot age out of it.
- **Exposing a list of everything newly painted**, LoAF-`scripts`-style. A container filling with hundreds of images in one frame would carry hundreds of sub-entries.
- **Unifying with ICP's aggregate shape.** A shared `ContentfulPaintAggregate` mixin is discussed in [Considered alternatives](#considered-alternatives) as future work, not as part of this change.

## What is wrong with the entry today

Three problems, all consequences of putting everything on a per-update entry:

1. **The attribution is a bare `Element?`.** No size, no URL, no load time, no render time. Everything a developer would correlate by hand against Resource Timing, and everything ICP's nested LCP already carries.
2. **There is no cumulative hero content.** `lastPaintedElement` resets its comparison on every emission by design, so each entry names the largest element of one update and nothing names the largest over the container's life. Script cannot recover it from the entry stream either: an entry carries no size for the element it nominates, since `size` is the container's accumulated region, so folding the stream yields a list of elements with no way to order them short of measuring the DOM afterwards. ICP has neither problem, because its nested `LargestContentfulPaint` carries its own size and soft navigation exposes the running largest through an accessor. And without a per-element size measured the way LCP measures it, a container's hero cannot be compared against the page's LCP at all.
3. **Entries are not always there when the state is wanted.** The entry buffer is capped, so a long-lived page loses its early entries, and code that never registered an observer has no interest in the stream at all but may still want the numbers. A contentful paint inside the already-painted region also queues nothing today, because the container did not grow; that particular gap is what [Decision 2](#decisions-to-take) would close.

## Motivating use cases

- **Attributing a slow widget:** which element dominates it and when it rendered, and for an image, which resource backed it and when that resource loaded, without correlating against Resource Timing by hand. Text candidates carry no resource, following LCP's convention of an empty `url` and a zero `loadTime`, so the resource half of the answer exists only where there is a resource to name.
- **Comparing a container against page LCP or against an interaction's ICP:** requires the same `size` definition and a cumulative, not per-update, largest.
- **Reading final numbers as a widget is torn down**, from an SPA that kept no observer alive for the widget's lifetime, querying while the container is still in the document.
- **Content that fills already-painted space:** an image landing over its own placeholder is the widget's real content and today produces no entry and no attribution.

## Proposed API

```webidl
[Exposed=Window, RuntimeEnabled=ContainerTiming]
interface PerformanceContainerTiming : PerformanceEntry {
    readonly attribute DOMString identifier;
    readonly attribute Element? rootElement;

    // Aggregate view of the container, as of this entry. Unchanged, except
    // that `size` is now `unsigned long long` rather than `double`.
    readonly attribute unsigned long long size;
    readonly attribute DOMRectReadOnly intersectionRect;

    // NEW. The largest contentful paint among content newly painted since this
    // container's previous entry. Non-nullable. Supersedes lastPaintedElement.
    readonly attribute LargestContentfulPaint largestNewContentfulPaint;

    // NEW. The container's first contentful paint, named
    // "first-contentful-paint". Constant for a given container root, and the
    // same object on every entry for it. Supersedes firstRenderTime.
    readonly attribute PerformancePaintTiming firstContentfulPaint;

    // DEPRECATED. Removed once the origin trial closes, in this change if it
    // lands after that, in a follow-up if it lands before.
    readonly attribute Element? lastPaintedElement;
    readonly attribute DOMHighResTimeStamp firstRenderTime;

    object toJSON();
};
PerformanceContainerTiming includes PaintTimingMixin;   // this update's paint

// NEW. The aggregate view of a container root, accumulated over its whole
// lifetime rather than describing a single update.
[Exposed=Window, RuntimeEnabled=ContainerTiming]
interface ContainerTimingInfo {
    // The container's first contentful paint, the same object the entry
    // carries. Supersedes firstRenderTime.
    readonly attribute PerformancePaintTiming firstContentfulPaint;

    // The most recent paint that grew the painted region. See Decision 4:
    // the lean is to drop this.
    readonly attribute PerformancePaintTiming lastPaint;

    // The largest contentful paint over the container's lifetime. Monotonic.
    readonly attribute LargestContentfulPaint largestContentfulPaint;

    // Accumulated as of the last presented frame, so these can be ahead of the
    // last emitted entry. They are never behind it.
    readonly attribute unsigned long long size;
    readonly attribute DOMRectReadOnly intersectionRect;
};

partial interface Element {
    // Null unless this is a container root with recorded paints.
    ContainerTimingInfo? getContainerTimingInfo();
};
```

### Which API surface answers what

| To answer...                                                 | Use                                                                                                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| What content grew this container just now?                   | `entry.largestNewContentfulPaint`                                                                                                     |
| How big is the container now, and where?                     | `entry.size` / `entry.intersectionRect` (as of this entry), or `info.size` / `info.intersectionRect` (as of the last presented frame) |
| What is this container's hero content, overall?              | `info.largestContentfulPaint`                                                                                                         |
| When did this container first render content, and with what? | `entry.firstContentfulPaint`, or `info.firstContentfulPaint`                                                                          |
| When did it last grow?                                       | `info.lastPaint`                                                                                                                      |
| Any of the above, with no observer attached                  | `element.getContainerTimingInfo()`                                                                                                    |

...where `info` is `element.getContainerTimingInfo()`.

### Four things to notice

**The nested objects are never queued**, as in ICP and LoAF. Their `entryType` and `name` are the ordinary ones for their interface (`"largest-contentful-paint"`, and `"paint"` named `"first-contentful-paint"`, or `"last-paint"` for `lastPaint` if it survives [Decision 4](#decisions-to-take)) and `toJSON()` behaves normally. No spec currently says an instance may exist unqueued, so that needs naming somewhere. See [Semantics that need explicit spec text](#semantics-that-need-explicit-spec-text).

**`getContainerTimingInfo()` is a live query, and it lives only on `Element`.** It returns the root's state at the moment of the call, which is the point of it. An earlier draft put the same method on the entry as well; that is [rejected](#considered-alternatives), because a `PerformanceEntry` is otherwise frozen at creation and a method on it answering about now invites exactly the wrong reading. It never throws: a non-root element, a retired root, and a root that has painted nothing all return null.

**The nested paint attributes are non-nullable because content is reported once.** An entry exists only because the painted region grew, so there is always newly painted content to nominate; a container has an aggregate view only once something painted in it. "Nothing yet" is `getContainerTimingInfo()` returning null, not null attributes inside a `ContainerTimingInfo`.

**`"last-paint"` is a real ask on Paint Timing**, which enumerates two names and would have a third. It would never show on the timeline, since these objects are never queued, but it is another spec enumeration member, and it is the only one this proposal asks for. Whether `lastPaint` earns that is [Decision 4](#decisions-to-take), where the lean is to drop the attribute.

## Key scenarios

#### The update stream

The stream, which is what the timeline is for:

```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const paint = entry.largestNewContentfulPaint; // never null
    log(
      `${entry.identifier} grew to ${entry.size} pixels; ` +
        `driven by <${paint.element?.localName}> ${paint.size} pixels, ` +
        `url=${paint.url || "(text)"}, loaded=${paint.loadTime}, ` +
        `rendered=${paint.renderTime}`,
    );
  }
}).observe({ type: "container", buffered: true });
```

#### The accumulated state

This way we can implement, using the pull API, cases as this in teardown:

```js
function teardownWidget(widget) {
  const info = widget.getContainerTimingInfo(); // query *before* removing
  if (info) {
    beacon({
      id: widget.getAttribute("containertiming"),
      firstPaint: info.firstContentfulPaint.startTime,
      lastPaint: info.lastPaint.startTime,
      hero: info.largestContentfulPaint.element?.id,
      heroSize: info.largestContentfulPaint.size,
      area: info.size,
    });
  }
  widget.remove();
}
```

The order matters. Once the element is disconnected, by its own removal or an ancestor's, the recording is gone and `getContainerTimingInfo()` returns null. Removing the `containertiming` attribute or changing its identifier ends it in the same way, so a widget that is retagged before being removed has already lost its numbers. See [a container root's history ends when it stops being a connected container root](#a-container-roots-history-ends-when-it-stops-being-a-connected-container-root).

An observer that does not hold the element is not left empty handed. Each entry carries `firstContentfulPaint` and a `largestNewContentfulPaint` with a real `size`, so folding the stream recovers the largest over the paints it saw. What it cannot recover is the state of a root it never observed, which is the trade accepted when the aggregate accessor was taken off the entry.

## Detailed design discussion

### Two maxima, because the entry and the aggregate answer different questions

|             | `entry.largestNewContentfulPaint`     | `info.largestContentfulPaint`                                   |
| ----------- | ------------------------------------- | --------------------------------------------------------------- |
| Window      | Since this container's previous entry | The container root's whole lifetime                             |
| Reset       | On every emission                     | Never                                                           |
| Where       | On the entry                          | On `ContainerTimingInfo`                                        |
| Nullability | Non-nullable                          | Non-nullable (the containing `ContainerTimingInfo` is nullable) |

A Container Timing entry fires on _aggregate area growth_, which is neither of these. Without the cumulative one you cannot identify a container's hero content; without the per-update one you cannot tell what changed. Because both use the same selection rule and differ only in the reset, the lifetime maximum is exactly the maximum of the per-entry ones.

**On naming:** `lastContentfulPaint` is the obvious alternative for the per-update attribute and is not proposed. `last*` says "most recent", and the attribute names the largest, not the most recent. The distinction is the whole point of the change: `lastPaintedElement` named whichever element happened to paint last, which is an artifact of the order paints are processed in rather than an answer to what drove the update, and a name beginning with `last` invites that reading straight back in. Giving both attributes the same name in their two places was also rejected: two identically named attributes with different accumulation windows would make this API hard to read.

### Both maxima are selected and sized by the element's own clipped area

Neither uses "the area this element newly contributed to the container". Both use the element's own clipped painted area: the measure LCP uses for its candidates.

This matters for three reasons:

1. **Paint order stops mattering.** Selecting by contributed area hands the overlap between two elements to whichever happened to paint first, both within a frame and across loads of the same page.
2. **The values become comparable** with each other, and with LCP and ICP entries.
3. **The double-counting caveat retires.** Under contributed-area accounting, two elements overlapping within one frame are each credited the shared area, so credited areas can sum to more than the container's genuinely new area.

A membership test of the form "only elements that actually grew the container may be nominated" is deliberately **not** applied. One rule has to cover both windows, or the lifetime maximum stops being the maximum of the per-entry ones. It is also unnecessary as a guard against stale re-nomination: **content reaches Container Timing only once per thing it paints**, so a repaint, layout shift or scroll cannot re-nominate a large stale element. Only genuinely new content can.

Two consequences to state plainly in spec text:

- `size` on a nominated paint can exceed the area it actually contributed. A 200×200 image overlapping 190×190 of already-painted content reports 40000, not 3900.
- An entry's `largestNewContentfulPaint` can be an element that contributed **zero** new area, and whose `paintTime` therefore predates the entry's own `startTime`.

The second one is interesting enough so it could deserve an explicit example, because the natural reading of "what just changed" is "what added area":

```
frame 1   a 300x200 text placeholder paints at (0,0)
          -> entry: size 60000, largestNewContentfulPaint = the text (60000 pixels)

frame 2   a 200x200 image finishes loading at (0,0), entirely inside the
          region painted in frame 1, and a 60x60 icon paints on empty space
          -> entry: size 63600, largestNewContentfulPaint = the image (40000 pixels)
```

In frame 2 the image adds **no** new area to the container: every pixel of it was already inside the painted region. All 3600 pixels of the container's growth came from the icon, and the icon is the reason an entry was queued at all. The image is still the most significant thing that painted in that interval (it is the widget's actual content arriving over its placeholder) and it is what a developer debugging the widget wants named. Reporting the icon instead, because the icon happened to land on empty space, is exactly the placement issue this rule would improve.

The nested object carries its own `paintTime` / `presentationTime`, so an element whose paint predates the entry stays self-describing: the entry says when the container grew, but the nested object says when that content rendered.

### Why the aggregate is a pull API rather than more entry attributes

(If [decision 1](#decisions-to-take) goes the way it is leaning, this section stops being a workaround argument and becomes the definition of the split.)

The aggregate's whole value is being independent of the entry stream:

- **An entry describes a moment, the aggregate describes the container.** Reading accumulated state off a stream of updates means keeping every update, or trusting that the one you have is the last. Asking is neither.
- **The entry buffer is capped** (150 in Chromium, with the overflow counted as dropped entries). A cumulative largest carried on entries inherits that cap, so it goes stale exactly on the pages that paint the most. Pulled state does not.
- **The reader may never have been listening.** Teardown code, a beacon at unload, a debugging console session: none of them have an observer, and with `buffered: true` they would still only receive what the buffer kept.

There is a fourth hole today, a contentful paint landing entirely inside the already-painted region, which queues no entry because the container did not grow. [Decision 2](#decisions-to-take) would close that one at the trigger. The three above survive it.

None of those three is fixable by adding attributes to the entry, and a pull API has none of them, as long as it is asked while the container is still in the document.

One implementation consequence, worth carrying into any engine: **recording must not be gated on whether an entry can be queued.** Chromium's text path used to consult "can I report to container timing?", which is false once the buffer is full with nobody observing, so text paints stopped reaching the aggregate while image paints kept arriving. With a pull API that becomes web-visible.

### The first contentful paint is an object, and deliberately not an attributed one

`firstRenderTime` is a presentation timestamp and nothing else. It cannot distinguish when the frame was painted from when it was presented, and the two can be tens of milliseconds apart on a loaded machine, which is exactly the situation a developer profiling a slow widget is in. `firstContentfulPaint` is a `PerformancePaintTiming`, which is `PerformanceEntry` plus `PaintTimingMixin`, so it carries that split and can be extended later without adding another attribute next to it.

It carries no element, and that is a decision rather than an omission. A container's first frame usually paints several things at once: a heading, a background image, an icon. They share a presentation time, so "which one was first" has no principled answer and would be settled by whichever the implementation happened to process first. That is precisely the paint-order dependence that [selecting the largest by its own area](#both-maxima-are-selected-and-sized-by-the-elements-own-clipped-area) removed from `lastPaintedElement`, and it should not come back through a different door. "Largest" is a total order on a well-defined quantity; "first" is not, within a frame.

Where the question is genuinely "what appeared first", the entry stream answers it in a well-defined form: the first entry for a container reports `largestNewContentfulPaint`, the largest content of that first update, selected by its own area rather than by processing order.

The name matters for a second reason. `"first-contentful-paint"` is already a `PerformancePaintTiming` name, so nothing is asked of Paint Timing's enumeration, and it describes this better than `"first-paint"` would: the page level first paint counts any pixels at all, including non-contentful ones, while Container Timing only ever records images and text, so every first paint it has is a contentful one. This is still not the page level entry scoped to a container: the page's is a document milestone with no element and no container, and the two share only the name string. The shorter name is also being kept free on purpose, for a container level first paint of any kind, contentful or not, which would need paint system plumbing this proposal does not have. See [Considered alternatives](#considered-alternatives).

`lastPaint` remains the most recent paint that grew the container's painted region, and is the only member asking for a new enumeration name, `"last-paint"`, which is the substance of [Decision 4](#decisions-to-take) where the lean is to drop it. Note the asymmetry it creates while it survives: a fully contained paint can advance `largestContentfulPaint` without advancing `lastPaint`, so the cumulative largest's `renderTime` may be later than `lastPaint`.

### The nested paint attributes are non-nullable

An entry is queued only when the region grew, which means content painted inside the container, which means there is a largest such paint. A container has an aggregate view only once something has painted in it. `ContainerTimingInfo` itself is nullable, and that is where "nothing yet" is expressed.

This holds because Container Timing defines its own contentful set; see below.

### Container Timing's contentful set is its own, and should be stated as such

Reusing the `LargestContentfulPaint` _interface_ will be read as importing LCP's _behaviour_. It does not, and the specification should say positively what the set is rather than leave it to be inferred from the interface: **the content Container Timing records is the content Element Timing records**, because both are driven by the same paint feed. That is a constraint worth writing down rather than an implementation detail, since if the two sets diverged, an image annotated with `elementtiming` inside a marked container could report to one and not the other.

That places Container Timing between the two metrics it is most likely to be compared against.

It is broader than LCP. Paints that LCP declines to treat as candidates, such as viewport-covering images, low-entropy images, and background images that fail LCP's contentful-background test, do reach Container Timing, and it keeps reporting them. This is already true today, it is deliberate, and it is load-bearing: it is part of why the nested attributes can be non-nullable.

It is narrower than first contentful paint. FCP counts content Container Timing does not record at all, a non-blank canvas being the clearest case. Adopting FCP's candidate set is not proposed, and FCP's model is not reusable in any case: FCP is a document level milestone, a single flag for whether anything contentful has painted, which computes nothing per element. Container Timing needs a rectangle per paint to accumulate the region, an area per element to select the largest, and an identity per paint to attribute it. The overlap in the word "contentful" is the only thing the two share, which is exactly why the difference needs stating.

Related, and equally needing explicit spec text: **Container Timing does not stop recording at first input or scroll.** Containers legitimately keep updating for the document's lifetime; that is the point of the API. "Same as LCP" is not a usable shorthand here, because the four consumers all differ: page LCP stops on first input unconditionally and permanently; ICP stops per soft-navigation context, and only once that context has painted; Element Timing never stops. Container Timing should be specified alongside Element Timing.

### Object identity across calls

The nested objects are built when the paint happens, not when the query is made. There is a hard reason for that: `PerformanceEntry`'s identity includes the navigation it belongs to, so an object built on demand during a later soft navigation would carry the wrong one.

Building eagerly has an observable consequence. When a container's cumulative largest has not changed, consecutive `getContainerTimingInfo()` calls hand back the **same** `LargestContentfulPaint` object (likewise `firstContentfulPaint`, which never changes for a container and is the same object the entries carry), so script can `===`-compare across calls to detect "the hero content did not change". The enclosing `ContainerTimingInfo` is a fresh object per call, since `size` and `intersectionRect` move.

This is presented as a consequence worth documenting, not as a contract this proposal asks to have specified. Requiring it normatively would constrain implementations that would rather build the nested objects at query time, for a convenience script can reproduce by comparing `renderTime` and `size`. Whether the spec should guarantee it, merely permit it, or say nothing is [decision 6](#decisions-to-take). What the spec does need to say either way is the related timing point: a nested object keeps the `paintTime` / `presentationTime` of the paint that made it the largest, so those legitimately differ from `lastPaint` and from the enclosing entry's. That is informative, since it tells you when the hero landed, possibly many frames earlier, but only if it is intentional.

### The aggregate can be ahead of the last entry, never behind it

`ContainerTimingInfo.size` and `.intersectionRect` are measured as of the last presented frame rather than as of the last emitted entry, so they keep advancing while emission is throttled.

### A container root's history ends when it stops being a connected container root

What is being accumulated is the history of **a connected container root under a given identifier**, and it ends when any part of that stops holding. Precisely:

| Event                                                             | Effect on the history                                                                                                                          |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `containertiming` attribute removed                               | Ends. `getContainerTimingInfo()` returns null.                                                                                                 |
| `containertiming` set to a **different** identifier               | Ends. A fresh history starts under the new identifier.                                                                                         |
| `containertiming` set to the **same** identifier                  | No effect; the history continues.                                                                                                              |
| Identifier changed away and later **changed back**                | The original history ended at the first change and does not come back. The second change starts a third history, empty until something paints. |
| `containertimingignore` toggled                                   | No effect on the history; it changes propagation to ancestors.                                                                                 |
| Element becomes disconnected, by its own removal or an ancestor's | Ends. The record is dropped and `getContainerTimingInfo()` returns null.                                                                       |
| A disconnected element is inserted again                          | A new history starts, empty until something paints.                                                                                            |
| Element adopted into another document                             | Ends, for the same reason: the recording belongs to the document it happened in.                                                               |

The identifier round-trip is worth spelling out because "cumulative over the container's lifetime" invites the opposite reading. There is no memory of a retired identifier: a container root's history is not resurrectable, and an author who toggles the attribute to pause reporting will find the numbers gone rather than paused.

The rule is written in terms of **connectedness, not removal**, because removal is only one way to stop being connected and not the most common one. A widget wrapper is torn out and every marked root inside it is disconnected without being touched. `isConnected` covers that, along with a root that was never inserted and one sitting in a `DocumentFragment`. It is also shadow-including, which costs nothing while shadow trees remain out of scope for this API, and means the rule needs no revision when that changes: a root inside a shadow tree is connected while its host is, and removing the host ends the recordings inside it along with the host's own.

Dropping the recording at that point is a choice, and the alternative was to keep answering from a detached element for as long as script holds it, which is what the current implementation happens to do, since nothing erases a record on removal. It is not the better rule. The recording belongs to the document where the painting happened, an element outside the tree has no painted region on screen any more, and keeping it alive creates a second, quieter lifetime that has to be specified on top of the attribute one. It also matches what the paint pipeline does underneath: re-inserted content is laid out afresh and reported again, so continuing the old recording would be mixing two of them.

The corollary for authors is the important part, and it is now a requirement rather than advice: **query the final numbers before the container leaves the document**, and treat the attribute as write-once.

Two notes for implementers. This is a behaviour change rather than a description of what ships today, and it needs a removal hook that Container Timing currently does not have. Whether the record is dropped eagerly when the element is disconnected or lazily when it is next asked for is not observable, apart from memory, so either is fine.

### `size` becomes `unsigned long long`

The IDL declared `size` as `double` while the spec draft says `unsigned long long`. This aligns on `unsigned long long`. The value is an integer pixel count in every existing implementation, so no observable value changes; `toJSON()` now emits an integer.

### Content is reported once, and growing does not re-report it

An element that is small at first paint and later grows never produces a larger candidate. Combined with "Container Timing keeps recording forever", this produces a mild surprise: `size` keeps growing while `largestContentfulPaint` can stay pinned to something small. This matches page LCP and is probably acceptable, but it should be called out.

The once-only rule is not absolute, and the two content types differ. Text can be re-armed: an interaction can make an already-painted aggregator eligible again, which is how soft navigation re-measures text. Images cannot, but their record is keyed on the pair (element, image request), so a new load is a new record. Container Timing is excluded from the text re-arm, which is the behaviour we want: a re-armed repaint is withheld from both Element Timing and Container Timing, reaching only LCP and ICP, so the same text cannot be credited twice to one container.

### What maintaining the aggregate costs

The container's painted region is the one part of this API whose cost grows with the page rather than with the number of containers. In Chromium it is a `cc::Region`, an `SkRegion` underneath, so it is stored as bands of rectangles rather than as a bounding box. Every contentful paint inside a container does a containment test and a union against that region, and both scale with how many disjoint rectangles the region has come to need. A container holding a few large blocks stays cheap. A feed or a grid with hundreds of separately painted items builds a region with hundreds of runs, every later paint pays for that, and so does every ancestor container root, since the same rectangle is unioned into each of them in turn.

This proposal makes it worse in one specific way, and it should be said plainly rather than discovered later. Recording used to stop being fed in some situations, because the text path consulted whether an entry could be queued at all. The aggregate has to stay correct for a caller that never observed, so recording is now unconditional. What was an accidental brake on region growth is gone, and nothing replaces it: a long-lived page keeps unioning into the same regions for as long as content keeps painting in them.

Two implementation notes, neither needing spec text:

- The region's area is recomputed by walking every rectangle. That happens on each emission and, with the pull API, on each `getContainerTimingInfo()` call, so script polling in an animation frame pays it every time. Caching the area and updating it when the region changes removes this cost entirely.
- An implementation that wants a ceiling can bound the region's complexity, collapsing to the bounding box past some threshold at the cost of overreporting `size`. That is a quality-of-implementation choice, but the specification has to permit it explicitly or it is not one; see [`size` is allowed to be approximate](#semantics-that-need-explicit-spec-text).

Caching and ceilings are mitigations. The structural question is when recording should stop, which is a non-goal here and is discussed as such above.

## Semantics that need explicit spec text

These are the contract, not afterthoughts. Each is behaviour the design above depends on, currently either unstated or stated only in a spec that consumes it rather than defines it. Several of them are about rules Container Timing _inherits_, which is exactly where "obviously it works like LCP" would produce a wrong implementation.

1. **Container Timing's candidate set is not LCP's**, and the nested attributes' non-nullability depends on that.
2. **Container Timing does not stop recording today**, not at first input and not at first scroll, so as things stand it is specified alongside Element Timing rather than alongside LCP. What needs writing down is that this is a choice and which one it is, because "same as LCP" is not a usable shorthand here: LCP stops at the first input, permanently; ICP stops per soft navigation context; Element Timing never stops. If the discussion under [when recording stops](#non-goals) lands on a stop condition, this is the text that changes, together with the wording that calls the aggregate cumulative over the container's lifetime.
3. **"Lifetime" means the container root's record lifetime**, which ends when the element is disconnected from the document, when the `containertiming` attribute is removed, or when its identifier changes. The disconnection half is the part implementations are most likely to get wrong by omission, since it needs a removal hook rather than an attribute one.
4. **The once-only rule.** Today it is an emergent property of two Paint Timing internal data structures (a persistent "set of elements with rendered text" and a consume-once "images pending rendering" list), and the only prose asserting it as a _guarantee_ lives in the LCP spec, which consumes the rule rather than implementing it. Container Timing now depends on it for its selection rule, so it should be stated somewhere Container Timing can cite.
5. **Nested entries exist without being queued.** `LargestContentfulPaint` and `PerformancePaintTiming` instances reachable only through these attributes are never added to the performance timeline, never delivered to an observer, and never returned by `getEntries()`. ICP and LoAF already rely on this, but neither the LCP nor the Paint Timing spec currently says an instance may exist unqueued, so the construction should be named ("synthetic" or "unqueued" entries) in whichever spec is the right home, and Container Timing should cite it rather than re-describe it.
6. **`size` is allowed to be approximate.** Keeping the painted region exact means paying for every distinct rectangle for as long as the container records, which is the cost described under [What maintaining the aggregate costs](#what-maintaining-the-aggregate-costs). The spec should permit bounded precision rather than leave implementations to choose between conformance and a ceiling: an implementation may simplify the region it tracks, merging rectangles into stripes or collapsing to the bounding box past some complexity, provided the error runs one way. `size` may over-report the painted area, never under-report it, and it stays monotonically non-decreasing. Two things should be required to stay exact, and they are what makes the latitude cheap. `intersectionRect` is the region's bounds, and every simplification worth making adds area inside those bounds, so they need not move and the spec should say they do not. Attribution is untouched, because `largestNewContentfulPaint` and `largestContentfulPaint` are selected and sized by each element's own clipped area rather than by the region, so a coarser region costs accuracy in one number and changes nothing about which element is reported. The spec should not prescribe a simplification or a numeric bound; that is a quality of implementation matter.

   Two consequences to carry with it. A coarser region swallows more paints, so fewer of them look like growth and fewer entries are queued, which is an argument in favour of [Decision 2](#decisions-to-take): with the largest contentful paint as a second trigger, the content that matters still produces an entry even when the region has absorbed it. And tests have to be written accordingly, which is a test suite matter rather than prose: asserting an exact `size` is only meaningful for geometries simple enough that every conforming implementation is exact, and anything richer should assert bounds.

7. **The nested entry's resource fields follow Element Timing and LCP, and should say so.** A `container` entry carries no resource information today; with a nested `LargestContentfulPaint` it carries the image's URL and load time. These are not new to the platform: Element Timing reports the same two values for an annotated image, LCP and ICP nest exactly the same construction, and none of them applies a timing allow gate to them, because Resource Timing already exposes both to the same document for any resource. The URL is the resource entry's `name`, and `responseEnd`, which is what an image's `loadTime` amounts to, is one of the few attributes not gated on the timing allow check. What the specification should state is that these fields have the same treatment as in Element Timing and LCP, so that if the platform ever tightens that treatment, Container Timing tightens with it instead of being left as the loose end.

   What is genuinely different here is the granularity of the opt-in, and it deserves a sentence of its own. Element Timing reports per annotated element. A single `containertiming` attribute turns resource reporting on for every image painting anywhere in that subtree, including images the author never annotated and, in a widget assembled from third party markup, images the author may not control. That is a change in scale rather than in kind, and it is the shape of the attribute rather than a property of the nested entry, but a reviewer will ask about it and the answer should already be written down.

8. **The soft-navigation re-arm.** Soft Navigations' "reset paint tracking" algorithm is an explicit placeholder whose issue text mentions Element Timing and LCP but not Container Timing. Whatever lands should say that a re-armed repaint contributes to neither `largestContentfulPaint` nor `largestNewContentfulPaint`, because otherwise the same text can be credited twice to one container, and the per-update attribute in particular would report repainted content as newly painted.

## Deprecations and migration

Both deprecated attributes stay in the IDL and keep working, annotated so that access increments a use counter and raises a DevTools issue naming the replacement. This mirrors the pattern already used in this feature for the `containertiming-ignore` to `containertimingignore` attribute rename.

How long that state lasts depends on when this lands. The deprecation window exists to protect origin trial participants, and the trial closes around October 2026: if the implementation lands after that, the attributes can simply be removed in the same change, and the `[DeprecateAs]` annotations are worth keeping only for as long as the feature is reachable by pages that were written against the trial. The migration below is written for the deprecation case, since that is the strictly larger amount of work, and it describes exactly what a removal would require anyway.

| Deprecated                 | Replacement                               | Why                                                                                                                              |
| -------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `entry.lastPaintedElement` | `entry.largestNewContentfulPaint.element` | Same element, same exposure filtering, plus size, url, loadTime and renderTime                                                   |
| `entry.firstRenderTime`    | `entry.firstContentfulPaint.startTime`    | Same place on the same object, and the replacement also names what painted first, how big it was and which resource it came from |

Before:

```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    report({
      id: entry.identifier,
      element: entry.lastPaintedElement?.id,
      firstRender: entry.firstRenderTime,
      area: entry.size,
    });
  }
}).observe({ type: "container", buffered: true });
```

After:

```js
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    const paint = entry.largestNewContentfulPaint; // never null
    report({
      id: entry.identifier,
      element: paint.element?.id, // same element lastPaintedElement gave
      firstRender: entry.firstContentfulPaint.startTime,
      area: entry.size,
      // newly available, no extra bookkeeping:
      elementArea: paint.size,
      resource: paint.url,
      resourceLoaded: paint.loadTime,
      firstContent: entry.firstContentfulPaint.element?.id,
    });
  }
}).observe({ type: "container", buffered: true });
```

Both migrations are a rename in place. `lastPaintedElement` becomes `largestNewContentfulPaint.element`, which is derived from it during the deprecation window so the two cannot disagree, and `firstRenderTime` becomes `firstContentfulPaint.startTime`, on the same object the reader already has.

An earlier draft routed the `firstRenderTime` replacement through the aggregate view, which made it a method call, an optional chain and a property walk, and left it returning null for a root that had stopped being a container root, the one case where the old scalar still worked. Putting the first contentful paint on the entry removes all of that. It is also the one piece of aggregate state that can live on a frozen record without contradiction, because it never changes after the container's first paint: everything that moves stays on the pull API.

## Considered alternatives

**Keep `lastPaintedElement` and add scalar attributes next to it** (`lastPaintedSize`, `lastPaintedUrl`, ...). Rejected: it re-derives, attribute by attribute, an interface the platform already has, and the result would not be comparable with LCP or ICP entries without callers knowing the correspondence by hand.

**Put the cumulative largest on the entry as a second attribute.** Rejected: the entry buffer is capped, and an entry only exists at the moments the container happened to change, so an entry attribute cannot answer for a reader that arrived later. Widening the trigger, as in [Decision 2](#decisions-to-take), adds moments but does not make them reachable afterwards.

**Select the largest by newly contributed area** (the "delta credit" rule that shipped first). Rejected after review: it makes the result depend on paint processing order, both within a frame and across loads, and it double-counts overlaps.

**Restrict candidates to the elements that grow the painted region**, so that an entry only ever nominates content which contributed to it. Rejected: one rule has to cover both accumulation windows, or the lifetime largest stops being the maximum of the per-entry ones. Besides, content reaches Container Timing only once, so the guard is not needed.

**An aggregate accessor on the entry**, `PerformanceContainerTiming.getContainerTimingInfo()`, mirroring the one on `Element`. Proposed in an earlier draft and now rejected. A `PerformanceEntry` is frozen at creation, so a method on it that answers about the container's state now reads as a snapshot and is not one, and no amount of prose fixes a shape that has to be explained twice. It was load-bearing for one case, a late observer wanting final numbers for a root that has since been detached and that it holds no reference to, and that case no longer exists: a disconnected root has no recording to hand out, whichever surface is asking. That case is now partly covered, since every entry carries `firstContentfulPaint` and a `largestNewContentfulPaint` with a real `size`, so an observer can fold what it received; what remains uncovered is state from before it was listening, and that is accepted.

**Keeping `firstRenderTime` and adding nothing**, rather than replacing it with `firstContentfulPaint` on the entry. Rejected: a presentation-time-only scalar cannot express the `paintTime` and `presentationTime` split, and it cannot be extended later without another attribute beside it.

**Making `firstContentfulPaint` a `LargestContentfulPaint`**, so that it names the element that appeared first along with its size and resource. Proposed in an earlier draft and rejected: within the frame where a container first paints, several elements typically paint together and share a presentation time, so the element chosen would be whichever the implementation processed first. That is the paint-order dependence this proposal removes elsewhere, and the extra fields are only worth having if the selection is well defined.

**Adopt ICP's `startTime` / `duration` anchoring.** Rejected for compatibility, and there is an independent design argument: under ICP's convention `startTime` would become constant per container root, so every entry for a long-lived container would carry the same timestamp. Unlike `interaction-contentful-paint` and `largest-contentful-paint`, which are observer-only, `container` entries are retrievable through `performance.getEntriesByType()`, which sorts by `startTime`, so the result would be grouped by container root rather than chronological.

**Reuse LCP's `LargestContentfulPaintCalculator` to compute the container's largest.** Rejected: it would drag in LCP's eligibility filter, which is the one thing this design must not inherit. Container Timing computes its own largest from its own paint feed.

**A shared `ContentfulPaintAggregate` mixin** included by both `ContainerTimingInfo` and `InteractionContentfulPaint`, carrying `size`, `intersectionRect` and `largestContentfulPaint`. Deferred, not rejected. The soft-navigations explainer states that directly exposing Container Timing IDL attributes on `InteractionContentfulPaint` is an aspirational goal, and with the aggregate factored onto `ContainerTimingInfo` this becomes a more natural fit than it was when the aggregate lived on the entry: `ContainerTimingInfo` _is_ the aggregate scoped by container root, and ICP the same aggregate scoped by interaction. The nullability difference would have to be reconciled first.

**A genuine `firstPaint`, for the container's first paint of any kind.** Deferred, and the reason the name above is `firstContentfulPaint` rather than the shorter one. A container's box can be visible well before any content lands in it: a background, a border, a skeleton drawn in CSS. That moment is what a page level `first-paint` marks, and it is a real user-perceived signal that Container Timing cannot report today, because the API is fed exclusively by the image and text paint callbacks. Reporting it would need a new signal from the paint system rather than a new attribute over the existing feed, which is why it is not proposed here. Naming the current attribute `firstContentfulPaint` keeps `firstPaint` free for the concept it actually denotes, so the two could sit side by side later without either being renamed or misread.

**`interactionId` on Container Timing entries**, joining a container's paints to the interaction that caused them. Deferred, not rejected.

## Privacy and security considerations

Most of this exposes nothing new. `largestNewContentfulPaint.element` applies the same filter `lastPaintedElement` applied (`LargestContentfulPaint::element()` and `Performance::CanExposeNode()` are logic-identical), and the nested objects carry the same `paintTime` / `presentationTime` values, at the same resolution, that the entry already carries.

One part is new to Container Timing, though not to the platform. The nested entry carries an image's `url` and `loadTime`, which no `container` entry carries today. Both are already available to the same document through Resource Timing for any image, cross-origin or not: the URL is the resource entry's `name`, and `responseEnd` is not among the attributes gated on the timing allow check. Element Timing reports the same two values for an annotated image, and LCP and ICP nest the same construction without a gate of their own. So this is not a route to data the document could not otherwise obtain, and the question is scale rather than kind: Element Timing asks for a per-element annotation, while one `containertiming` attribute enables reporting for everything painting in the subtree. See the corresponding item under [Semantics that need explicit spec text](#semantics-that-need-explicit-spec-text). Text candidates raise none of this: they carry an empty `url` and a zero `loadTime`.

The repository's `security-review.md` records that the API introduces no new cross-origin information exposure. That conclusion still holds on this analysis, but it was written for an API with no resource fields at all, so it is worth re-confirming against the nested entry rather than carried forward untouched.

The pull API adds availability, not observability: everything it returns is already computed for the entry stream. What changes is that it is reachable when no entry carried it.

## Stakeholder feedback

Review feedback drove the switch from contributed-area selection to the element's own inherent size, on the grounds that a size delta adds paint-order problems across frames. That is applied, and the reasoning is set out under [both maxima are selected and sized by the element's own clipped area](#both-maxima-are-selected-and-sized-by-the-elements-own-clipped-area). The rest of the shape proposed here is still under discussion. No signals from other engines.

## Decisions to take

### 1. Do `container` entries need a timeline buffer at all?

Container Timing is currently the odd one out among the paint-timing APIs. `container` entries are bufferable and retrievable through `performance.getEntriesByType('container')`, like Element Timing. LCP and ICP entries are observer-only: if nobody was listening, they are gone. Now that `getContainerTimingInfo()` answers "what has this container painted so far" without an observer and without any dependence on the buffer, it is worth asking which of the two models Container Timing should be specified in.

**(a) Keep the buffer.** An observer registering after the interesting paints replays the real sequence of updates with `buffered: true`, and non-observer code can read the same entries through `getEntriesByType()`. The cost is a capped, per-document retention of entries that most pages never read, and a cap that is observable: once it is reached, entries are dropped and the sequence a late reader sees is silently incomplete.

**(b) Observer-only, like LCP and ICP.** The timeline carries updates to whoever is listening, and everything else is pulled. No cap to specify, no dropped-entry semantics, and one less way for two readers of the same page to disagree. The cost is real: an observer that registers late can no longer learn _what changed while it was not listening_, only the current state, because the aggregate is state and not history.

_Lean: (a), and say so explicitly in the spec rather than leaving it implicit._ The sequence of updates is the one thing the pull API genuinely cannot reconstruct, and it is the thing a developer wants when a container settles in several steps: which step brought the hero content, which one merely added an icon. Dropping the buffer to match LCP would be tidier but would remove information that has no other route to script. The argument for (b) gets stronger if the trial shows nobody reads container entries outside an observer, which is worth measuring before the trial ends, since we currently have no data on it either way.

Whichever is chosen, the interaction with the aggregate should be stated: entries and `getContainerTimingInfo()` are two views of the same recording, the aggregate is never behind the last entry, and the aggregate keeps answering after the entries for those paints have aged out of the buffer.

### 2. Should an entry also be queued when the container's largest contentful paint grows?

Today a `container` entry is queued when the accumulated painted region grows, and only then. That misses the case this API most wants to report: the real content arriving over its own placeholder. An image that paints entirely inside the region the placeholder already covered adds nothing to `size`, queues nothing, and yet is the most significant thing that has happened to the container. The proposal is to widen the trigger to _the painted region grew, or the container's cumulative largest contentful paint grew_.

The second half has to mean **the cumulative largest increased**, not "a contentful paint happened". Content reaches Container Timing once per thing it paints, so the second reading would queue an entry per contentful element, which in a large widget is hundreds. The first reading is monotonic and bounded, typically a handful of extra entries over a container's life, and it is exactly the trigger ICP already uses: a strictly larger candidate appeared. Adopting it makes the Container Timing trigger a superset of ICP's, so a container can never stay silent about a paint that ICP would have reported.

_Lean: adopt it, and no longer necessarily as separate follow-up work._ It is not additive, since entry counts change for pages already using the API, which is why it was held back. With the trial closing around October 2026 and an implementation landing after that, it can be taken together with the attribute changes instead of queued behind them. What has to be settled first is not the timing but the problem below.

**The problem it creates, which needs an answer before it lands.** An entry queued because the largest grew, with no growth in the painted region, carries a `size` and an `intersectionRect` identical to the previous entry for that container. Nothing on the entry says why it was queued. A consumer that treats "new entry" as "the container got bigger", or that deduplicates on `size`, sees what looks like a repeat. The options, none of them obviously right:

- **Say nothing, and let consumers compare.** An entry whose `size` matches the previous one for the same root was queued because the hero changed. This is inferable but only if you kept the previous entry, which an observer callback receiving a batch does not necessarily have.
- **Add a reason to the entry**, an enumerated attribute naming what triggered it. Honest and self-describing, but it is a new attribute on an entry this proposal otherwise leaves alone, and no other paint-timing entry carries one.
- **Keep the triggers but make the second one visible through the existing fields.** For instance, an entry queued only by a growing largest could report `largestNewContentfulPaint` with the container's `size` unchanged, which is what already happens, and the spec could simply state that this combination is meaningful rather than anomalous.

The trade-off is between an entry that explains itself and an entry surface that stays frozen for the trial. This should be settled in the issue rather than in the implementation.

### 3. Does content whose element was removed still count as the cumulative largest?

_Lean: yes, current behaviour._ Page LCP and ICP say no, and consistency is the argument for that. Against it: `largestContentfulPaint` is specified as monotonic over the container's lifetime, and retracting a candidate when its element goes away breaks that: a widget that swapped its main image for a smaller one would report the smaller one as its largest ever, and the answer would depend on when you ask, which is what the pull API is meant to stop mattering. LCP's reason does not transfer cleanly: LCP is a page-load metric describing a final loaded view, a container is explicitly a thing that keeps changing. The nested object survives either way; only its `element` goes null.

### 4. Should `ContainerTimingInfo` keep `lastPaint` at all?

_Lean: drop it._ What it costs is out of proportion to what it answers, and dropping it removes a whole class of open question rather than settling one.

`firstContentfulPaint` reports `"first-contentful-paint"`, a name Paint Timing already enumerates, so `lastPaint` is the only member of this proposal that asks another specification for a new one. Removing it removes that ask entirely, and with it the fallbacks (an empty `name`, or folding the last paint into `PaintTimingMixin` on `ContainerTimingInfo` itself). It also removes an ambiguity that [Decision 2](#decisions-to-take) would otherwise create: once entries can be queued by a growing largest as well as by a growing region, "last paint" has two defensible meanings, the last paint that grew the region and the last paint that produced an entry, and neither is obviously the one a caller wants. What remains is `{firstContentfulPaint, largestContentfulPaint, size, intersectionRect}`, which is also a cleaner fit for the shared aggregate mixin sketched under future considerations.

The cost is real but conditional on [Decision 1](#decisions-to-take). If `container` entries keep their timeline buffer, "when did this container last change" is answerable from the last entry's `startTime`, and `lastPaint` is a convenience for a question the entry stream already answers. If entries become observer-only, the aggregate is the only route to that question for a caller that was not listening, and `lastPaint` should stay. The two decisions should therefore be taken together, and in the order 1 then 4.

Note what is not affected either way: the first contentful paint stays, on the entry and on the aggregate, because it is the replacement for the deprecated `firstRenderTime` and because it is immutable, which is what lets it sit on a frozen entry at all.

### 5. Should a retired container root keep answering?

_Lean: no._ A root that lost its `containertiming` attribute, changed identifier, or left the document returns null. Keeping a retired record alive raises questions with no good answer: for how long, keyed on what, and what happens when the identifier is reused elsewhere.

Two of those three are how implementations already behave, since changing or removing the attribute drops the record. The third, disconnection, is a change: nothing erases a record on removal today, so a detached root keeps answering for as long as script holds it. Treating all three the same way is the point of the lean, and the full table, including the identifier round trip, is in [a container root's history ends when it stops being a connected container root](#a-container-roots-history-ends-when-it-stops-being-a-connected-container-root).

### 6. Should object identity across calls be normative?

_Lean: document, do not require._ Building nested objects at paint time is forced by navigation identity; identity across calls falls out of it. Requiring it constrains implementations that would rather build at query time, for something script can reproduce by comparing `renderTime` and `size`. Explicitly permitting it, so the difference is not a conformance bug either way, may be the middle.

## References

- Container Timing explainer: https://github.com/WICG/container-timing/
- Container Timing spec draft: https://bloomberg.github.io/container-timing/
- Largest Contentful Paint: https://w3c.github.io/largest-contentful-paint/
- Paint Timing: https://w3c.github.io/paint-timing/
- Element Timing: https://wicg.github.io/element-timing/
- Soft Navigations / Interaction Contentful Paint: https://wicg.github.io/soft-navigations/
- Background on the `lastPaintedElement` semantics: https://github.com/WICG/container-timing/issues/53
