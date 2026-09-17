# Container Timing: contentful paint attribution and the aggregate view

**Author:** Jose Dapena Paz (Igalia) &lt;jdapena@igalia.com&gt;

**Last updated:** 2026-09-16

**Status:** Proposal for discussion.

**Audience:** assumes working knowledge of Container Timing, Element Timing, LCP and ICP. Background on those is not repeated.

**Participate:**

- Spec draft: https://wicg.github.io/container-timing/
- Repository / issues: https://github.com/WICG/container-timing/
- Related: [WICG/container-timing#53 `lastPaintedElement` semantics](https://github.com/WICG/container-timing/issues/53)

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
- [References and acknowledgements](#references-and-acknowledgements)

## Introduction

The starting point was for this explainer was: align Container Timing's attribution with the shape ICP converged on. ICP attributes contentful paints to a scope by nesting a whole `LargestContentfulPaint` in the attributing entry; Container Timing does something similar with `lastPaintedElement`, an `Element?` that carries no size, no URL, no load time and no render time of its own. Adopting ICP's shape is mostly mechanical, and the paint feed already carries every field the nested object needs.

Working through it surfaced two things that are not mechanical, so this document will also cover those.

#### The entry cannot carry the container's accumulated state

`lastPaintedElement` resets its comparison on each emission, so it describes one update. But a developer also wants the container hero content over its whole life, and that is not derivable from an entry stream you may not have received.

One concrete case today: a contentful paint landing entirely inside the already-painted region queues no entry at all (the container did not grow) while still being able to be the largest thing the container ever painted. [Decision 2](#decisions-to-take) proposes closing that hole at the trigger, which would be a good change on its own, but it does not remove the need for the aggregate: an entry reports what changed at the moment it was queued, and the state has to be reachable later, by code that was not listening then.

The entry buffer is a second case: it is capped, so on a busy page the entries carrying the state are eventually dropped, and code that reads them later sees a truncated history.

#### So the division of labour should be explicit

The timeline carries *updates* (what changed, when, and what content drove it). Accumulated state is reached by *asking* ( `Element.getContainerTimingInfo()`) with no observer required and no dependence on buffering.

Every attribute below follows from that split, and it is also the framing under which [Decision 1](#decisions-to-take) (do `container` entries still need a timeline buffer at all?) becomes answerable.

#### Origin trial

Everything proposed is additive for the duration of the origin trial:

- `lastPaintedElement` and `firstRenderTime` are deprecated but keep working.
- `startTime`, `duration`, `name`, `entryType`, `identifier`, `rootElement`, `intersectionRect` and the emission trigger are untouched.

## Goals

- Align Container Timing's attribution with ICP's: a nested `LargestContentfulPaint` instead of an element reference, with `size` measured the way LCP measures it so the values are comparable across the three APIs.
- Split per-update reporting from accumulated state, and make the accumulated state reachable without an observer and without depending on the entry buffer.
- Keep every currently shipping behaviour intact for origin trial participants.

## Non-goals

- **Changing `startTime` / `duration` anchoring.** ICP anchors `startTime` to the interaction that caused the paint. Adopting that here would silently change two attributes trial participants already measure against. Out of scope.
- **Adopting LCP's candidate eligibility rules.** Container Timing deliberately reports paints LCP declines to treat as candidates. See [Container Timing's contentful set is not LCP's](#container-timings-contentful-set-is-not-lcps).
- **Removing the deprecated attributes now.** They go when the origin trial ends.
- **Exposing a list of everything newly painted**, LoAF-`scripts`-style. A container filling with hundreds of images in one frame would carry hundreds of sub-entries.
- **Unifying with ICP's aggregate shape.** A shared `ContentfulPaintAggregate` mixin is discussed in [Considered alternatives](#considered-alternatives) as future work, not as part of this change.

## What is wrong with the entry today

Three problems, all consequences of putting everything on a per-update entry:

1. **The attribution is a bare `Element?`.** No size, no URL, no load time, no render time. Everything a developer would correlate by hand against Resource Timing, and everything ICP's nested LCP already carries.
2. **There is no cumulative hero content.** `lastPaintedElement` resets on emission by design. The only way to answer to "what is the biggest thing this container has ever painted?" is getting all the `PerformanceContainerTiming` entries. Worse compared to the solution in ICP and less comparable with LCP.
3. **Entries are not always there when the state is wanted.** The entry buffer is capped, so a long-lived page loses its early entries, and code that never registered an observer has no interest in the stream at all but may still want the numbers. A contentful paint inside the already-painted region also queues nothing today, because the container did not grow; that particular gap is what [Decision 2](#decisions-to-take) would close.

## Motivating use cases

- **Attributing a slow widget:** which element dominates it, which resource backed it, when that resource loaded, and when it rendered, without correlating against Resource Timing by hand.
- **Comparing a container against page LCP or against an interaction's ICP:** requires the same `size` definition and a cumulative, not per-update, largest.
- **Reading final numbers at teardown**, from an SPA that tore the widget out, with no observer alive for the widget's lifetime.
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

    // NEW. This entry's container root's aggregate view, queried live.
    ContainerTimingInfo? getContainerTimingInfo();

    // DEPRECATED. Removed when the origin trial ends.
    readonly attribute Element? lastPaintedElement;
    readonly attribute DOMHighResTimeStamp firstRenderTime;

    object toJSON();
};
PerformanceContainerTiming includes PaintTimingMixin;   // this update's paint

// NEW. The aggregate view of a container root, accumulated over its whole
// lifetime rather than describing a single update.
[Exposed=Window, RuntimeEnabled=ContainerTiming]
interface ContainerTimingInfo {
    // When the container first painted content, and the most recent paint that
    // grew its painted region. firstPaint supersedes firstRenderTime.
    readonly attribute PerformancePaintTiming firstPaint;
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

| To answer... | Use |
|---|---|
| What content grew this container just now? | `entry.largestNewContentfulPaint` |
| How big is the container now, and where? | `entry.size` / `entry.intersectionRect` (as of this entry), or `info.size` / `info.intersectionRect` (as of the last presented frame) |
| What is this container's hero content, overall? | `info.largestContentfulPaint` |
| When did this container first render content? | `info.firstPaint` |
| When did it last grow? | `info.lastPaint` |
| Any of the above, with no observer attached | `element.getContainerTimingInfo()` |

...where `info` is `element.getContainerTimingInfo()` or `entry.getContainerTimingInfo()`.

### Four things to notice

**The nested objects are never queued**, as in ICP and LoAF. Their `entryType` and `name` are the ordinary ones for their interface (`"largest-contentful-paint"`; `"paint"` named `"first-paint"` / `"last-paint"`) and `toJSON()` behaves normally. No spec currently says an instance may exist unqueued, so that needs naming somewhere. See see [Semantics that need explicit spec text](#semantics-that-need-explicit-spec-text).

**`getContainerTimingInfo()` is a live query, not a snapshot.** Both overloads return the root's state *now*, which on a `PerformanceEntry` could be unusual enough to be worth stating twice: an entry buffered ten seconds ago answers with today's aggregate, not with the aggregate as of its own `startTime`. Same pattern as `PerformanceSoftNavigation.getLargestInteractionContentfulPaint()`. It is a method rather than an attribute partly to signal this. It never throws: a non-root element, a retired root, and a root that has painted nothing all return null.

**The nested paint attributes are non-nullable because content is reported once.** An entry exists only because the painted region grew, so there is always newly painted content to nominate; a container has an aggregate view only once something painted in it. "Nothing yet" is `getContainerTimingInfo()` returning null, not null attributes inside a `ContainerTimingInfo`.

**`"last-paint"` is a real ask on Paint Timing**, which enumerates two names and would have a third. It would never show on the timeline, since these objects are never queued, but it is another spec enumeration member, and it is the only one this proposal asks for. Whether `lastPaint` earns that is [Decision 4](#decisions-to-take), where the lean is to drop the attribute.

## Key scenarios

#### The update stream

The stream, which is what the timeline is for:

```js
new PerformanceObserver(list => {
  for (const entry of list.getEntries()) {
    const paint = entry.largestNewContentfulPaint;   // never null
    log(`${entry.identifier} grew to ${entry.size} pixels; ` +
        `driven by <${paint.element?.localName}> ${paint.size} pixels, ` +
        `url=${paint.url || '(text)'}, loaded=${paint.loadTime}, ` +
        `rendered=${paint.renderTime}`);
  }
}).observe({type: 'container', buffered: true});
```

#### The accumulated state

This way we can implement, using the pull API, cases as this in teardown:

```js
function teardownWidget(widget) {
  const info = widget.getContainerTimingInfo();   // query *before* removing
  if (info) {
    beacon({
      id: widget.getAttribute('containertiming'),
      firstPaint: info.firstPaint.startTime,
      lastPaint: info.lastPaint.startTime,
      hero: info.largestContentfulPaint.element?.id,
      heroSize: info.largestContentfulPaint.size,
      area: info.size,
    });
  }
  widget.remove();
}
```

Detaching the element is fine: a detached root keeps answering for as long as script holds a reference.

Removing the `containertiming` attribute or changing its identifier is not: that ends the root's history and `getContainerTimingInfo()` returns null. See [A container root's history ends when it stops being a container root](#a-container-roots-history-ends-when-it-stops-being-a-container-root).

The same method exists on the entry, so an observer can reach the aggregate without holding the element: `entry.getContainerTimingInfo()`, answering about the root *now*.

## Detailed design discussion

### Two maxima, because the entry and the aggregate answer different questions

| | `entry.largestNewContentfulPaint` | `info.largestContentfulPaint` |
|---|---|---|
| Window | Since this container's previous entry | The container root's whole lifetime |
| Reset | On every emission | Never |
| Where | On the entry | On `ContainerTimingInfo` |
| Nullability | Non-nullable | Non-nullable (the containing `ContainerTimingInfo` is nullable) |

A Container Timing entry fires on *aggregate area growth*, which is neither of these. Without the cumulative one you cannot identify a container's hero content; without the per-update one you cannot tell what changed. Because both use the same selection rule and differ only in the reset, the lifetime maximum is exactly the maximum of the per-entry ones.

**On naming:** `lastContentfulPaint` was considered for the per-update one and rejected. `last*` reintroduces the "last in paint order" misreading that [WICG/container-timing#53](https://github.com/WICG/container-timing/issues/53) set out to remove. The attribute selects the *largest* new contributor, not the most recent one. Giving both attributes the same name in their two places was also rejected: two identically named attributes with different accumulation windows would make this API hard to read.

### Both maxima are selected and sized by the element's own clipped area

Neither uses "the area this element newly contributed to the container". Both use the element's own clipped painted area: the measure LCP uses for its candidates.

This matters for three reasons:

1. **Paint order stops mattering.** Selecting by contributed area hands the overlap between two elements to whichever happened to paint first, both within a frame and    across loads of the same page.
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

A pull API has neither problem, and it lets script ask at teardown time rather than keeping an observer alive.

One implementation consequence, worth carrying into any engine: **recording must not be gated on whether an entry can be queued.** Chromium's text path used to consult "can I report to container timing?", which is false once the buffer is full with nobody observing, so text paints stopped reaching the aggregate while image paints kept arriving. With a pull API that becomes web-visible.

### `firstPaint` and `lastPaint` are objects, not timestamps

`PerformancePaintTiming` is `PerformanceEntry` plus `PaintTimingMixin` and nothing else. Nesting one gives the container's first render its own `paintTime` / `presentationTime` split, which the bare `firstRenderTime` scalar, a presentation time only, cannot express.

`lastPaint` is the most recent paint that actually **grew** the container's painted region. Together the two bracket the interval over which the container was still settling, so a caller can answer "has anything new been damaged in here since T?" without having received an entry. That is the case for keeping it; [Decision 4](#decisions-to-take) sets out the stronger case for dropping it, and this section describes the shape as currently implemented rather than as recommended.

Note the asymmetry this creates with the cumulative largest: a fully-contained paint can advance `largestContentfulPaint` without advancing `lastPaint`, so the cumulative largest's `renderTime` may be **later** than `lastPaint`.

Whether that is worth a third name in Paint Timing's enumeration is a fair question, and the fallbacks are listed in [Four things to notice](#four-things-to-notice). The case for paying it: `firstRenderTime` is a presentation timestamp with no way to say when the frame was painted, and the two can be tens of milliseconds apart on a loaded machine, which is exactly the situation a developer profiling a slow widget is in. A scalar cannot be extended later without another attribute; an object can.

Named `firstPaint` rather than `firstContentfulPaint`: strictly this is a contentful paint, since Container Timing only tracks images and text, but from the container's point of view it is simply the first paint the API knows about, and the shorter name is clearer. To be explicit for spec readers, since the names collide: this is **not** the page-level `first-paint` entry scoped to a container. The page's `first-paint` marks the first time the document painted anything at all, including non-contentful pixels; `ContainerTimingInfo.firstPaint` marks the first image or text Container Timing observed inside this container root. Only the `name` string is shared.

### The nested paint attributes are non-nullable

An entry is queued only when the region grew, which means content painted inside the container, which means there is a largest such paint. A container has an aggregate view only once something has painted in it. `ContainerTimingInfo` itself is nullable, and that is where "nothing yet" is expressed.

This holds because Container Timing defines its own contentful set; see below.

### Container Timing's contentful set is not LCP's

Reusing the `LargestContentfulPaint` *interface* will be read as importing LCP's *behaviour*. It does not. Paints that LCP declines to treat as candidates (viewport-covering images, low-entropy images, background images that fail LCP's "contentful background" test) do reach Container Timing, and Container Timing keeps reporting them. This is already true today and is deliberate.

It is also load-bearing: it is part of why the nested attributes can be non-nullable.

Related, and equally needing explicit spec text: **Container Timing does not stop recording at first input or scroll.** Containers legitimately keep updating for the document's lifetime; that is the point of the API. "Same as LCP" is not a usable shorthand here, because the four consumers all differ: page LCP stops on first input unconditionally and permanently; ICP stops per soft-navigation context, and only once that context has painted; Element Timing never stops. Container Timing should be specified alongside Element Timing.

### Object identity across calls

The nested objects are built when the paint happens, not when the query is made. There is a hard reason for that: `PerformanceEntry`'s identity includes the navigation it belongs to, so an object built on demand during a later soft navigation would carry the wrong one.

Building eagerly has an observable consequence. When a container's cumulative largest has not changed, consecutive `getContainerTimingInfo()` calls hand back the **same** `LargestContentfulPaint` object (likewise `firstPaint`, which never changes for a container), so script can `===`-compare across calls to detect "the hero content did not change". The enclosing `ContainerTimingInfo` is a fresh object per call, since `size` and `intersectionRect` move.

This is presented as a consequence worth documenting, not as a contract this proposal asks to have specified. Requiring it normatively would constrain implementations that would rather build the nested objects at query time, for a convenience script can reproduce by comparing `renderTime` and `size`. Whether the spec should guarantee it, merely permit it, or say nothing is [decision 6](#decisions-to-take). What the spec does need to say either way is the related timing point: a nested object keeps the `paintTime` / `presentationTime` of the paint that made it the largest, so those legitimately differ from `lastPaint` and from the enclosing entry's. That is informative, since it tells you when the hero landed, possibly many frames earlier, but only if it is intentional.

### The aggregate can be ahead of the last entry, never behind it

`ContainerTimingInfo.size` and `.intersectionRect` are measured as of the last presented frame rather than as of the last emitted entry, so they keep advancing while emission is throttled.

### A container root's history ends when it stops being a container root

What is being accumulated is the history of **a container root under a given identifier**, and it ends when either half of that stops holding. Precisely:

| Event | Effect on the history |
|---|---|
| `containertiming` attribute removed | Ends. `getContainerTimingInfo()` returns null. |
| `containertiming` set to a **different** identifier | Ends. A fresh history starts under the new identifier. |
| `containertiming` set to the **same** identifier | No effect; the history continues. |
| Identifier changed away and later **changed back** | The original history ended at the first change and does not come back. The second change starts a third history, empty until something paints. |
| `containertimingignore` toggled | No effect on the history; it changes propagation to ancestors. |
| Element removed from the document | No effect. A detached root keeps answering for as long as script holds a reference to it. |
| Element becomes unreachable from script | The record is collected with it. |

The round-trip case is the one worth spelling out because "cumulative over the container's lifetime" invites the opposite reading. There is no memory of a retired identifier: a container root's history is not resurrectable, and an author who toggles the attribute to pause reporting will find the numbers gone rather than paused.

All of this is already today's behaviour for the entry stream (changing or removing the attribute already drops the container's record), but with a pull API it becomes directly observable, so it has to be specified rather than left as an implementation detail. The corollary for authors: query final numbers **before** teardown, and treat the attribute as write-once.

### Why the entry reaches its root unfiltered

The entry's `getContainerTimingInfo()` deliberately does **not** go through `rootElement`. `rootElement` applies the usual node-exposure filter, which rejects a disconnected node, so it is already null once a widget has been torn out of the document, which is exactly the moment the final numbers are wanted. Routing the method through it would make the aggregate unreachable in the teardown case the pull API exists for.

This exposes nothing new. The method returns a `ContainerTimingInfo`, never the root element itself, and the root is always in the same document as the performance object that queued the entry, so a caller learns only about content it could already observe. `rootElement` keeps its current filtering, unchanged.

### `size` becomes `unsigned long long`

The IDL declared `size` as `double` while the spec draft says `unsigned long long`. This aligns on `unsigned long long`. The value is an integer pixel count in every existing implementation, so no observable value changes; `toJSON()` now emits an integer.

### Content is reported once, and growing does not re-report it

An element that is small at first paint and later grows never produces a larger candidate. Combined with "Container Timing keeps recording forever", this produces a mild surprise: `size` keeps growing while `largestContentfulPaint` can stay pinned to something small. This matches page LCP and is probably acceptable, but it should be called out.

The once-only rule is not absolute, and the two content types differ. Text can be re-armed: an interaction can make an already-painted aggregator eligible again, which is how soft navigation re-measures text. Images cannot, but their record is keyed on the pair (element, image request), so a new load is a new record. Container Timing is excluded from the text re-arm, which is the behaviour we want: a re-armed repaint is withheld from both Element Timing and Container Timing, reaching only LCP and ICP, so the same text cannot be credited twice to one container.

## Semantics that need explicit spec text

These are the contract, not afterthoughts. Each is behaviour the design above depends on, currently either unstated or stated only in a spec that consumes it rather than defines it. Three of the six are about rules Container Timing *inherits*, which is exactly where "obviously it works like LCP" would produce a wrong implementation.

1. **Container Timing's candidate set is not LCP's**, and the nested attributes' non-nullability depends on that.
2. **Container Timing never stops recording**, not at first input and not at first scroll. Specify it alongside Element Timing, not alongside LCP.
3. **"Lifetime" means the container root's record lifetime**, which ends when the `containertiming` attribute is removed or its identifier changes.
4. **The once-only rule.** Today it is an emergent property of two Paint Timing internal data structures (a persistent "set of elements with rendered text" and a consume-once "images pending rendering" list), and the only prose asserting it as a *guarantee* lives in the LCP spec, which consumes the rule rather than implementing it. Container Timing now depends on it for its selection rule, so it should be stated somewhere Container Timing can cite.
5. **Nested entries exist without being queued.** `LargestContentfulPaint` and `PerformancePaintTiming` instances reachable only through these attributes are never added to the performance timeline, never delivered to an observer, and never returned by `getEntries()`. ICP and LoAF already rely on this, but neither the LCP nor the Paint Timing spec currently says an instance may exist unqueued, so the construction should be named ("synthetic" or "unqueued" entries) in whichever spec is the right home, and Container Timing should cite it rather than re-describe it.
6. **The soft-navigation re-arm.** Soft Navigations' "reset paint tracking" algorithm is an explicit placeholder whose issue text mentions Element Timing and LCP but not Container Timing. Whatever lands should say that a re-armed repaint contributes to neither `largestContentfulPaint` nor `largestNewContentfulPaint`, because otherwise the same text can be credited twice to one container, and the per-update attribute in particular would report repainted content as newly painted.

## Deprecations and migration

Both deprecated attributes stay in the IDL and keep working, annotated so that access increments a use counter and raises a DevTools issue naming the replacement. They are removed once the origin trial ends. This mirrors the pattern already used in this feature for the `containertiming-ignore` → `containertimingignore` attribute rename.

| Deprecated | Replacement | Why |
|---|---|---|
| `entry.lastPaintedElement` | `entry.largestNewContentfulPaint.element` | Same element, same exposure filtering, plus size, url, loadTime and renderTime |
| `entry.firstRenderTime` | `entry.getContainerTimingInfo()?.firstPaint.startTime` | An object with a `paintTime` / `presentationTime` split instead of a bare presentation timestamp |

Before:

```js
new PerformanceObserver(list => {
  for (const entry of list.getEntries()) {
    report({
      id: entry.identifier,
      element: entry.lastPaintedElement?.id,
      firstRender: entry.firstRenderTime,
      area: entry.size,
    });
  }
}).observe({type: 'container', buffered: true});
```

After:

```js
new PerformanceObserver(list => {
  for (const entry of list.getEntries()) {
    const paint = entry.largestNewContentfulPaint;   // never null
    const info = entry.getContainerTimingInfo();     // null if no longer a root
    report({
      id: entry.identifier,
      element: paint.element?.id,       // same element lastPaintedElement gave
      firstRender: info?.firstPaint.startTime,
      area: entry.size,
      // newly available, no extra bookkeeping:
      elementArea: paint.size,
      resource: paint.url,
      resourceLoaded: paint.loadTime,
      hero: info?.largestContentfulPaint.element?.id,
    });
  }
}).observe({type: 'container', buffered: true});
```

`lastPaintedElement` is now derived from `largestNewContentfulPaint.element`, so the two cannot disagree during the deprecation window, and that migration is a pure rename.

**The `firstRenderTime` migration is not a pure rename, and it costs something.** A scalar read becomes a method call, an optional chain and a property walk, and unlike the old attribute it can come back null for a root that has stopped being a container root, which is exactly the case where the old attribute still worked, since the entry carried its own copy. Three options were weighed:

1. **Keep `firstRenderTime` undeprecated.** Rejected: it is a presentation-time-only scalar, which is the limitation the object form exists to remove, and keeping it means shipping two ways to ask the same question permanently.
2. **Add a non-deprecated `firstPaint` convenience to the entry**, alongside `largestNewContentfulPaint`. This is a genuine option and is not foreclosed. It costs one attribute and removes the regression entirely, at the price of duplicating aggregate state onto a per-update object, which is the conflation this proposal is trying to undo, and it would mean two `firstPaint` objects with the same value on two surfaces.
3. **Aggregate state lives on the aggregate view. (Chosen.)** The first render time is a property of the container, not of an update; every entry for a container repeating the same timestamp is a symptom of the original conflation. The verbosity is real but small, and the null case is the same rule as everything else on the aggregate view.

If trial feedback says the null case bites in practice, option 2 remains available without any breaking change.

## Considered alternatives

**Keep `lastPaintedElement` and add scalar attributes next to it** (`lastPaintedSize`, `lastPaintedUrl`, ...). Rejected: it re-derives, attribute by attribute, an interface the platform already has, and the result would not be comparable with LCP or ICP entries without callers knowing the correspondence by hand.

**Put the cumulative largest on the entry as a second attribute.** Rejected: the entry buffer is capped, and an entry only exists at the moments the container happened to change, so an entry attribute cannot answer for a reader that arrived later. Widening the trigger, as in [Decision 2](#decisions-to-take), adds moments but does not make them reachable afterwards.

**Select the largest by newly contributed area** (the "delta credit" rule that shipped first). Rejected after review: it makes the result depend on paint processing order, both within a frame and across loads, and it double-counts overlaps.

**Restrict candidates to elements that grow the painted region**, per [issue #53](https://github.com/WICG/container-timing/issues/53). Rejected: one rule has to cover both accumulation windows, or the lifetime largest stops being the maximum of the per-entry ones. Besides, content reaches Container Timing only once, so the guard is not needed.

**Keep a non-deprecated `firstPaint` on the entry** so that migrating off `firstRenderTime` stays a one-line change. Not rejected outright (see [Deprecations and migration](#deprecations-and-migration)), but not proposed, because per-container state repeated on every per-update entry is the conflation this change sets out to undo.

**Adopt ICP's `startTime` / `duration` anchoring.** Rejected for compatibility, and there is an independent design argument: under ICP's convention `startTime` would become constant per container root, so every entry for a long-lived container would carry the same timestamp. Unlike `interaction-contentful-paint` and `largest-contentful-paint`, which are observer-only, `container` entries are retrievable through `performance.getEntriesByType()`, which sorts by `startTime`, so the result would be grouped by container root rather than chronological.

**Reuse LCP's `LargestContentfulPaintCalculator` to compute the container's largest.** Rejected: it would drag in LCP's eligibility filter, which is the one thing this design must not inherit. Container Timing computes its own largest from its own paint feed.

**A shared `ContentfulPaintAggregate` mixin** included by both `ContainerTimingInfo` and `InteractionContentfulPaint`, carrying `size`, `intersectionRect` and `largestContentfulPaint`. Deferred, not rejected. The soft-navigations explainer states that directly exposing Container Timing IDL attributes on `InteractionContentfulPaint` is an aspirational goal, and with the aggregate factored onto `ContainerTimingInfo` this becomes a more natural fit than it was when the aggregate lived on the entry: `ContainerTimingInfo` *is* the aggregate scoped by container root, and ICP the same aggregate scoped by interaction. The nullability difference would have to be reconciled first.

**`interactionId` on Container Timing entries**, joining a container's paints to the interaction that caused them. Deferred, not rejected.

## Privacy and security considerations

Nothing new is exposed. `largestNewContentfulPaint.element` applies the same filter `lastPaintedElement` applied (`LargestContentfulPaint::element()` and `Performance::CanExposeNode()` are logic-identical); the nested `url` follows LCP's existing rules; the nested objects carry the same `paintTime` / `presentationTime` values, at the same resolution, that the entry already carries.

The one deliberate relaxation: the entry's `getContainerTimingInfo()` reaches its root without the node-exposure filter, because that filter rejects disconnected nodes and would make the teardown case unreachable. It returns a `ContainerTimingInfo`, never the element, and the root is always in the same document as the performance object that queued the entry.

The pull API adds availability, not observability: everything it returns is already computed for the entry stream. What changes is that it is reachable when no entry carried it.

## Stakeholder feedback

Michal Mocny's review drove the switch from contributed-area selection to the element's own inherent size ("doing size delta just adds paint-order issues across multiple frames"); that is applied, and is the subject of [issue #53](https://github.com/WICG/container-timing/issues/53). The shape proposed here is otherwise still under discussion. No signals from other engines.

## Decisions to take

### 1. Do `container` entries need a timeline buffer at all?

Container Timing is currently the odd one out among the paint-timing APIs. `container` entries are bufferable and retrievable through `performance.getEntriesByType('container')`, like Element Timing. LCP and ICP entries are observer-only: if nobody was listening, they are gone. Now that `getContainerTimingInfo()` answers "what has this container painted so far" without an observer and without any dependence on the buffer, it is worth asking which of the two models Container Timing should be specified in.

**(a) Keep the buffer.** An observer registering after the interesting paints replays the real sequence of updates with `buffered: true`, and non-observer code can read the same entries through `getEntriesByType()`. The cost is a capped, per-document retention of entries that most pages never read, and a cap that is observable: once it is reached, entries are dropped and the sequence a late reader sees is silently incomplete.

**(b) Observer-only, like LCP and ICP.** The timeline carries updates to whoever is listening, and everything else is pulled. No cap to specify, no dropped-entry semantics, and one less way for two readers of the same page to disagree. The cost is real: an observer that registers late can no longer learn *what changed while it was not listening*, only the current state, because the aggregate is state and not history.

*Lean: (a), and say so explicitly in the spec rather than leaving it implicit.* The sequence of updates is the one thing the pull API genuinely cannot reconstruct, and it is the thing a developer wants when a container settles in several steps: which step brought the hero content, which one merely added an icon. Dropping the buffer to match LCP would be tidier but would remove information that has no other route to script. The argument for (b) gets stronger if the trial shows nobody reads container entries outside an observer, which is worth measuring before the trial ends, since we currently have no data on it either way.

Whichever is chosen, the interaction with the aggregate should be stated: entries and `getContainerTimingInfo()` are two views of the same recording, the aggregate is never behind the last entry, and the aggregate keeps answering after the entries for those paints have aged out of the buffer.

### 2. Should an entry also be queued when the container's largest contentful paint grows?

Today a `container` entry is queued when the accumulated painted region grows, and only then. That misses the case this API most wants to report: the real content arriving over its own placeholder. An image that paints entirely inside the region the placeholder already covered adds nothing to `size`, queues nothing, and yet is the most significant thing that has happened to the container. The proposal is to widen the trigger to *the painted region grew, or the container's cumulative largest contentful paint grew*.

The second half has to mean **the cumulative largest increased**, not "a contentful paint happened". Content reaches Container Timing once per thing it paints, so the second reading would queue an entry per contentful element, which in a large widget is hundreds. The first reading is monotonic and bounded, typically a handful of extra entries over a container's life, and it is exactly the trigger ICP already uses: a strictly larger candidate appeared. Adopting it makes the Container Timing trigger a superset of ICP's, so a container can never stay silent about a paint that ICP would have reported.

*Lean: adopt it, as follow-up work rather than as part of this change.* It is not additive: entry counts change for pages already using the API, so it belongs after the origin trial rather than inside it, and after the attribute changes above have landed.

**The problem it creates, which needs an answer before it lands.** An entry queued because the largest grew, with no growth in the painted region, carries a `size` and an `intersectionRect` identical to the previous entry for that container. Nothing on the entry says why it was queued. A consumer that treats "new entry" as "the container got bigger", or that deduplicates on `size`, sees what looks like a repeat. The options, none of them obviously right:

- **Say nothing, and let consumers compare.** An entry whose `size` matches the previous one for the same root was queued because the hero changed. This is inferable but only if you kept the previous entry, which an observer callback receiving a batch does not necessarily have.
- **Add a reason to the entry**, an enumerated attribute naming what triggered it. Honest and self-describing, but it is a new attribute on an entry this proposal otherwise leaves alone, and no other paint-timing entry carries one.
- **Keep the triggers but make the second one visible through the existing fields.** For instance, an entry queued only by a growing largest could report `largestNewContentfulPaint` with the container's `size` unchanged, which is what already happens, and the spec could simply state that this combination is meaningful rather than anomalous.

The trade-off is between an entry that explains itself and an entry surface that stays frozen for the trial. This should be settled in the issue rather than in the implementation.

### 3. Does content whose element was removed still count as the cumulative largest?

*Lean: yes, current behaviour.* Page LCP and ICP say no, and consistency is the argument for that. Against it: `largestContentfulPaint` is specified as monotonic over the container's lifetime, and retracting a candidate when its element goes away breaks that: a widget that swapped its main image for a smaller one would report the smaller one as its largest ever, and the answer would depend on when you ask, which is what the pull API is meant to stop mattering. LCP's reason does not transfer cleanly: LCP is a page-load metric describing a final loaded view, a container is explicitly a thing that keeps changing. The nested object survives either way; only its `element` goes null.

### 4. Should `ContainerTimingInfo` keep `lastPaint` at all?

*Lean: drop it.* What it costs is out of proportion to what it answers, and dropping it removes a whole class of open question rather than settling one.

`firstPaint` reports `"first-paint"`, a name Paint Timing already enumerates. `lastPaint` is the only reason this proposal asks another specification to add an enumeration member, so removing it removes the cross-specification ask entirely, and with it the fallbacks (an empty `name` on both nested objects, or folding the last paint into `PaintTimingMixin` on `ContainerTimingInfo` itself). It also removes an ambiguity that [Decision 2](#decisions-to-take) would otherwise create: once entries can be queued by a growing largest as well as by a growing region, "last paint" has two defensible meanings, the last paint that grew the region and the last paint that produced an entry, and neither is obviously the one a caller wants. What remains is `{firstPaint, largestContentfulPaint, size, intersectionRect}`, which is also a cleaner fit for the shared aggregate mixin sketched under future considerations.

The cost is real but conditional on [Decision 1](#decisions-to-take). If `container` entries keep their timeline buffer, "when did this container last change" is answerable from the last entry's `startTime`, and `lastPaint` is a convenience for a question the entry stream already answers. If entries become observer-only, the aggregate is the only route to that question for a caller that was not listening, and `lastPaint` should stay. The two decisions should therefore be taken together, and in the order 1 then 4.

Note what is not affected either way: `firstPaint` stays, because the first render time is genuinely unreachable once the early entries have aged out of the buffer, and because it is the replacement for the deprecated `firstRenderTime`.

### 5. Should a retired container root keep answering?

*Lean: no, current behaviour.* A root that lost its `containertiming` attribute or changed identifier returns null. Keeping a retired record alive raises questions with no good answer: for how long, keyed on what, and what happens when the identifier is reused elsewhere. Full table, including the change-back case, in [A container root's history ends when it stops being a container root](#a-container-roots-history-ends-when-it-stops-being-a-container-root).

### 6. Should object identity across calls be normative?

*Lean: document, do not require.* Building nested objects at paint time is forced by navigation identity; identity across calls falls out of it. Requiring it constrains implementations that would rather build at query time, for something script can reproduce by comparing `renderTime` and `size`. Explicitly permitting it, so the difference is not a conformance bug either way, may be the middle.

## References and acknowledgements

- Container Timing explainer: https://github.com/WICG/container-timing/
- Container Timing spec draft: https://bloomberg.github.io/container-timing/
- Largest Contentful Paint: https://w3c.github.io/largest-contentful-paint/
- Paint Timing: https://w3c.github.io/paint-timing/
- Element Timing: https://wicg.github.io/element-timing/
- Soft Navigations / Interaction Contentful Paint: https://wicg.github.io/soft-navigations/
- WICG/container-timing#53: `lastPaintedElement` semantics

Thanks to Michal Mocny and Scott Haseley for review feedback on the Chromium implementation, and to the Bloomberg Container Timing authors for the original design.
