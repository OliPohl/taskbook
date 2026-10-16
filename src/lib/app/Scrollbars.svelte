<!-- src/lib/app/Scrollbars.svelte -->
<!-- Every scrollbar in the app, floating over what scrolls instead of taking space of its own (the browser's are
     hidden, see global.css), so backgrounds and content reach the edges. Works for the page and for every element
     that scrolls, up and down or sideways: a thin thumb appears while it scrolls or with the mouse near its edge,
     then fades out. The thumb can be dragged. Touch screens keep the browser's own, they already float. -->
<script>
  // Fades out after this long without scrolling.
  const HIDE_AFTER = 1000;
  // How long the fade out takes, the bar is removed after it.
  const FADE = 400;
  // How close to an edge the mouse shows the bar (pixels).
  const EDGE = 14;
  const MIN_THUMB = 30;
  // Thickness of the thumb, and while the mouse is on it.
  const THIN = 5;
  const THICK = 8;
  // Space between the thumb and the edge.
  const INSET = 3;

  /**
   * @typedef {'y' | 'x'} Axis
   * @typedef {{until: number, hover: Axis | '', drag: Axis | '', id: number}} Entry
   * @typedef {{left: number, top: number, width: number, height: number, thumbStart: number, thumbSize: number}} Bar
   * @typedef {{id: number, el: Element, shown: boolean, clip: {left: number, top: number, width: number, height: number},
   *   y: Bar | null, x: Bar | null, hover: Axis | '', drag: Axis | ''}} Shown
   */

  /** @type {Axis[]} */
  const AXES = ['y', 'x'];

  /** @type {Map<Element, Entry>} */
  // eslint-disable-next-line svelte/prefer-svelte-reactivity -- plain bookkeeping, `bars` is what's shown
  const entries = new Map();
  let nextId = 1;
  /** @type {Shown[]} */
  let bars = $state([]);
  let fine = $state(false);

  const root = () => document.scrollingElement ?? document.documentElement;

  /** @param {Element} el @param {Axis} axis */
  function scrollable(el, axis) {
    const size = axis === 'y' ? el.scrollHeight - el.clientHeight : el.scrollWidth - el.clientWidth;
    if (size <= 1) return false;
    if (el === root()) return true;
    const overflow = getComputedStyle(el)[axis === 'y' ? 'overflowY' : 'overflowX'];
    return overflow === 'auto' || overflow === 'scroll';
  }

  /** Where an element is on screen, the window for the page. @param {Element} el */
  function rectOf(el) {
    if (el === root()) return { left: 0, top: 0, right: window.innerWidth, bottom: window.innerHeight };
    const rect = el.getBoundingClientRect();
    // Only the inside, without borders.
    const left = rect.left + el.clientLeft;
    const top = rect.top + el.clientTop;
    return { left, top, right: left + el.clientWidth, bottom: top + el.clientHeight };
  }

  /** The part of an element that can be seen: cut by every parent that clips and by the window. @param {Element} el */
  function visibleRect(el) {
    let { left, top, right, bottom } = rectOf(el);
    for (let parent = el.parentElement; parent && parent !== document.body; parent = parent.parentElement) {
      const style = getComputedStyle(parent);
      if (style.overflowX === 'visible' && style.overflowY === 'visible') continue;
      const rect = parent.getBoundingClientRect();
      left = Math.max(left, rect.left);
      top = Math.max(top, rect.top);
      right = Math.min(right, rect.right);
      bottom = Math.min(bottom, rect.bottom);
    }
    left = Math.max(left, 0);
    top = Math.max(top, 0);
    right = Math.min(right, window.innerWidth);
    bottom = Math.min(bottom, window.innerHeight);
    return { left, top, width: Math.max(0, right - left), height: Math.max(0, bottom - top) };
  }

  /** The thumb of one axis, or null when it doesn't scroll that way. @param {Element} el @param {Axis} axis @param {Entry} entry */
  function barOf(el, axis, entry) {
    if (!scrollable(el, axis)) return null;
    const rect = rectOf(el);
    const thick = entry.hover === axis || entry.drag === axis ? THICK : THIN;
    if (axis === 'y') {
      const length = rect.bottom - rect.top;
      const thumbSize = Math.max(MIN_THUMB, (length * el.clientHeight) / el.scrollHeight);
      const progress = el.scrollTop / (el.scrollHeight - el.clientHeight);
      return {
        left: rect.right - INSET - thick,
        top: rect.top,
        width: thick,
        height: length,
        thumbStart: progress * (length - thumbSize),
        thumbSize
      };
    }
    const length = rect.right - rect.left;
    const thumbSize = Math.max(MIN_THUMB, (length * el.clientWidth) / el.scrollWidth);
    const progress = el.scrollLeft / (el.scrollWidth - el.clientWidth);
    return {
      left: rect.left,
      top: rect.bottom - INSET - thick,
      width: length,
      height: thick,
      thumbStart: progress * (length - thumbSize),
      thumbSize
    };
  }

  /** @type {number | undefined} */
  let frameId;

  /** Follows every bar that is shown or fading out, every frame while there are any. */
  function update() {
    frameId = undefined;
    const now = performance.now();
    /** @type {Shown[]} */
    const next = [];
    for (const [el, entry] of entries) {
      const busy = entry.hover || entry.drag;
      if (!el.isConnected || (!busy && now > entry.until + FADE)) {
        entries.delete(el);
        continue;
      }
      const y = barOf(el, 'y', entry);
      const x = barOf(el, 'x', entry);
      if (!y && !x) {
        entries.delete(el);
        continue;
      }
      next.push({ id: entry.id, el, shown: Boolean(busy) || now < entry.until, clip: visibleRect(el), y, x, hover: entry.hover, drag: entry.drag });
    }
    bars = next;
    if (entries.size) frameId = requestAnimationFrame(update);
  }

  function schedule() {
    frameId ??= requestAnimationFrame(update);
  }

  /** @param {Element} el */
  function entryOf(el) {
    let entry = entries.get(el);
    if (!entry) {
      entry = { until: 0, hover: '', drag: '', id: nextId++ };
      entries.set(el, entry);
    }
    return entry;
  }

  /** @param {Event} event */
  function onScroll(event) {
    const el = event.target === document ? root() : /** @type {Element} */ (event.target);
    if (!(el instanceof Element)) return;
    if (!scrollable(el, 'y') && !scrollable(el, 'x')) return;
    entryOf(el).until = performance.now() + HIDE_AFTER;
    schedule();
  }

  /** @type {{x: number, y: number, target: EventTarget | null} | null} */
  let pointer = null;
  let pointerFrame = false;

  /** @param {PointerEvent} event */
  function onPointerMove(event) {
    pointer = { x: event.clientX, y: event.clientY, target: event.target };
    if (pointerFrame) return;
    pointerFrame = true;
    requestAnimationFrame(findHover);
  }

  /** The innermost element under the mouse with the mouse near its scrollbar's edge gets its bar shown. */
  function findHover() {
    pointerFrame = false;
    if (!pointer) return;
    // On a thumb: that bar keeps its hover.
    if (pointer.target instanceof Element && pointer.target.closest('.scrollbars')) return;

    /** @type {[Element, Axis] | null} */
    let found = null;
    let el = pointer.target instanceof Element ? pointer.target : null;
    for (; el && !found; el = el === root() ? null : (el.parentElement ?? root())) {
      if (el === document.body) el = root();
      const rect = rectOf(el);
      if (pointer.x < rect.left || pointer.x > rect.right || pointer.y < rect.top || pointer.y > rect.bottom) continue;
      if (rect.right - pointer.x <= EDGE && scrollable(el, 'y')) found = [el, 'y'];
      else if (rect.bottom - pointer.y <= EDGE && scrollable(el, 'x')) found = [el, 'x'];
    }

    for (const [other, entry] of entries) {
      if (found && other === found[0]) continue;
      if (entry.hover) {
        entry.hover = '';
        entry.until = performance.now();
      }
    }
    if (found) entryOf(found[0]).hover = found[1];
    schedule();
  }

  function onPointerLeaveWindow() {
    pointer = null;
    for (const entry of entries.values()) {
      if (entry.hover && !entry.drag) {
        entry.hover = '';
        entry.until = performance.now();
      }
    }
    schedule();
  }

  $effect(() => {
    // Like global.css: everything but touch screens.
    const query = window.matchMedia('(pointer: fine), (pointer: none)');
    fine = query.matches;
    const onChange = () => (fine = query.matches);
    query.addEventListener('change', onChange);

    document.addEventListener('scroll', onScroll, { capture: true, passive: true });
    document.addEventListener('pointermove', onPointerMove, { passive: true });
    document.documentElement.addEventListener('pointerleave', onPointerLeaveWindow);
    window.addEventListener('resize', schedule);

    return () => {
      query.removeEventListener('change', onChange);
      document.removeEventListener('scroll', onScroll, { capture: true });
      document.removeEventListener('pointermove', onPointerMove);
      document.documentElement.removeEventListener('pointerleave', onPointerLeaveWindow);
      window.removeEventListener('resize', schedule);
      if (frameId !== undefined) cancelAnimationFrame(frameId);
    };
  });

  /** @type {{el: Element, axis: Axis, start: number, scroll: number, ratio: number} | null} */
  let dragState = null;

  /** @param {PointerEvent} event @param {Shown} bar @param {Axis} axis */
  function grab(event, bar, axis) {
    if (event.button !== 0) return;
    event.preventDefault();
    event.stopPropagation();
    const el = bar.el;
    const shown = /** @type {Bar} */ (axis === 'y' ? bar.y : bar.x);
    const room = (axis === 'y' ? shown.height : shown.width) - shown.thumbSize;
    const scrollRange = axis === 'y' ? el.scrollHeight - el.clientHeight : el.scrollWidth - el.clientWidth;
    dragState = {
      el,
      axis,
      start: axis === 'y' ? event.clientY : event.clientX,
      scroll: axis === 'y' ? el.scrollTop : el.scrollLeft,
      ratio: room > 0 ? scrollRange / room : 0
    };
    entryOf(el).drag = axis;
    /** @type {HTMLElement} */ (event.currentTarget).setPointerCapture(event.pointerId);
    schedule();
  }

  /** @param {PointerEvent} event */
  function drag(event) {
    if (!dragState) return;
    const moved = (dragState.axis === 'y' ? event.clientY : event.clientX) - dragState.start;
    const value = dragState.scroll + moved * dragState.ratio;
    if (dragState.axis === 'y') dragState.el.scrollTop = value;
    else dragState.el.scrollLeft = value;
  }

  function release() {
    if (!dragState) return;
    const entry = entries.get(dragState.el);
    if (entry) {
      entry.drag = '';
      entry.until = performance.now() + HIDE_AFTER;
    }
    dragState = null;
    schedule();
  }
</script>

{#if fine}
  <div class="scrollbars" aria-hidden="true">
    {#each bars as bar (bar.id)}
      <!-- Cut to the part of the element that can be seen, so a bar never shows past it. -->
      <div
        class="clip"
        style:left="{bar.clip.left}px"
        style:top="{bar.clip.top}px"
        style:width="{bar.clip.width}px"
        style:height="{bar.clip.height}px">
        {#each AXES as axis (axis)}
          {@const shown = bar[axis]}
          {#if shown}
            <div
              class="thumb"
              class:shown={bar.shown}
              class:active={bar.hover === axis || bar.drag === axis}
              role="presentation"
              style:left="{shown.left - bar.clip.left + (axis === 'x' ? shown.thumbStart : 0)}px"
              style:top="{shown.top - bar.clip.top + (axis === 'y' ? shown.thumbStart : 0)}px"
              style:width="{axis === 'y' ? shown.width : shown.thumbSize}px"
              style:height="{axis === 'y' ? shown.thumbSize : shown.height}px"
              onpointerdown={(event) => grab(event, bar, axis)}
              onpointermove={drag}
              onpointerup={release}
              onpointercancel={release}>
            </div>
          {/if}
        {/each}
      </div>
    {/each}
  </div>
{/if}


<style>
  /* Above everything, but only the thumbs themselves catch the mouse. */
  .scrollbars {
    position: fixed;
    inset: 0;
    z-index: 100000;
    pointer-events: none;
  }

  .clip {
    position: fixed;
    overflow: hidden;
  }

  .thumb {
    position: absolute;
    border-radius: 6px;
    background-color: #494949;
    opacity: 0;
    transition: opacity 0.4s, background-color 0.15s;
  }

  .thumb.shown {
    opacity: 1;
    pointer-events: auto;
    transition: opacity 0.1s, background-color 0.15s;
  }

  .thumb.active {
    background-color: #6a6a6a;
  }
</style>
