<!-- src/lib/pages/dashboard/Shards.svelte -->
<!-- The background of the dashboard and the "Add New" page: the window broken into shards, like cracked earth, each
     a different shade of the wave color and without a border. The shards are the cells around points scattered over
     a grid (a Voronoi diagram), generated from a seed, and their edges are bent by a noise filter so they wobble
     like drawn by hand.
     Fixed, so it stays put while the dashboard scrolls. While developing, a panel on the right (DevTuner) changes
     the settings live. -->
<script module>
  export const SHARDS_DEFAULTS = {
    seed: 12,
    // The size of a shard in pixels, and how far its point strays from the grid (0 is a regular grid, 1 the most).
    size: 430,
    jitter: 0.6,
    // How much of the wave color a shard has, the rest is the background: the faintest and the strongest, in percent.
    minColor: 0,
    maxColor: 30,
    // How far the edges bend in pixels, and how wide the bends are (bigger is wider).
    wobble: 0,
    wobbleSize: 40
  };

  /** @type {import('$lib/dev/DevTuner.svelte').Slider[]} */
  const SLIDERS = [
    { key: 'seed', label: 'Seed', min: 0, max: 100, step: 1 },
    { key: 'size', label: 'Shard size', min: 80, max: 800, step: 10 },
    { key: 'jitter', label: 'Jitter', min: 0, max: 1, step: 0.05 },
    { key: 'minColor', label: 'Faintest %', min: 0, max: 60, step: 1 },
    { key: 'maxColor', label: 'Strongest %', min: 0, max: 60, step: 1 },
    { key: 'wobble', label: 'Wobble', min: 0, max: 120, step: 1 },
    { key: 'wobbleSize', label: 'Wobble size', min: 40, max: 800, step: 10 }
  ];

  /** A fixed "random" number from 0 to 1 for a draw `n` at a seed. @param {number} n @param {number} seed */
  const hash = (n, seed) => {
    const x = Math.sin(n * 12.9898 + seed * 78.233) * 43758.5453;
    return x - Math.floor(x);
  };

  /** Which side of the line between `a` and `b` a point is on: negative on `a`'s side. @param {[number, number]} a @param {[number, number]} b @param {[number, number]} p */
  const side = (a, b, p) => (p[0] - (a[0] + b[0]) / 2) * (b[0] - a[0]) + (p[1] - (a[1] + b[1]) / 2) * (b[1] - a[1]);

  /**
   * The part of a polygon on the side of the line between `a` and `b` that is closer to `a`.
   * @param {[number, number][]} polygon @param {[number, number]} a @param {[number, number]} b
   */
  function clip(polygon, a, b) {
    /** @type {[number, number][]} */
    const result = [];
    for (let i = 0; i < polygon.length; i++) {
      const p = polygon[i];
      const q = polygon[(i + 1) % polygon.length];
      const sp = side(a, b, p);
      const sq = side(a, b, q);
      if (sp <= 0) result.push(p);
      if (sp <= 0 !== sq <= 0) {
        const t = sp / (sp - sq);
        result.push([p[0] + (q[0] - p[0]) * t, p[1] + (q[1] - p[1]) * t]);
      }
    }
    return result;
  }

  /** How many shades a shard picks from: the one furthest from its neighbors' shades wins. */
  const CANDIDATES = 16;

  /**
   * The shards covering a `width` x `height` window, reaching `margin` past every edge so the bent edges never
   * show the page behind them.
   * @param {typeof SHARDS_DEFAULTS} s @param {number} width @param {number} height @param {number} margin
   */
  export function shards(s, width, height, margin) {
    const columns = Math.ceil((width + 2 * margin) / s.size) + 2;
    const rows = Math.ceil((height + 2 * margin) / s.size) + 2;
    /** @type {[number, number][]} */
    const points = [];
    for (let row = 0; row < rows; row++)
      for (let column = 0; column < columns; column++) {
        const n = row * 997 + column;
        points.push([
          -margin + (column - 1 + 0.5 + (hash(n, s.seed) - 0.5) * s.jitter) * s.size,
          -margin + (row - 1 + 0.5 + (hash(n + 0.5, s.seed) - 0.5) * s.jitter) * s.size
        ]);
      }

    /** @type {[number, number][]} */
    const bounds = [[-margin, -margin], [width + margin, -margin], [width + margin, height + margin], [-margin, height + margin]];
    const reach = 2.5 * s.size;
    const cells = points.map((point) => {
      // Only points close enough can share an edge.
      const near = points.filter(
        (other) => other !== point && Math.abs(other[0] - point[0]) < reach && Math.abs(other[1] - point[1]) < reach
      );
      let polygon = bounds;
      for (const other of near) polygon = clip(polygon, point, other);
      // A neighbor shares an edge: two corners lie on the line halfway between the points (within half a pixel).
      const neighbors = near.filter((other) => {
        const distance = Math.hypot(other[0] - point[0], other[1] - point[1]);
        return polygon.filter((p) => Math.abs(side(point, other, p)) / distance < 0.5).length >= 2;
      });
      return { polygon, neighbors: neighbors.map((other) => points.indexOf(other)) };
    });

    // Each shard in turn takes the shade, out of a few random ones, that is furthest from the shades its neighbors
    // already have, so shards next to each other stand apart.
    /** @type {number[]} */
    const shades = [];
    cells.forEach((cell, i) => {
      let best = 0;
      let bestDistance = -1;
      for (let k = 0; k < CANDIDATES; k++) {
        const shade = s.minColor + hash(i * CANDIDATES + k + 0.25, s.seed) * (s.maxColor - s.minColor);
        const distance = Math.min(Infinity, ...cell.neighbors.filter((j) => j < i).map((j) => Math.abs(shades[j] - shade)));
        if (distance > bestDistance) [best, bestDistance] = [shade, distance];
      }
      shades.push(best);
    });

    return cells.map(({ polygon }, i) => ({
      d: polygon.length < 3 ? '' : 'M ' + polygon.map(([x, y]) => `${x.toFixed(1)} ${y.toFixed(1)}`).join(' L ') + ' Z',
      fill: `color-mix(in srgb, var(--wave) ${shades[i].toFixed(1)}%, var(--bg))`
    }));
  }
</script>

<script>
  import { dev } from '$app/environment';
  import DevTuner from '$lib/dev/DevTuner.svelte';

  let settings = $state({ ...SHARDS_DEFAULTS });
  let width = $state(0);
  let height = $state(0);
  let margin = $derived(settings.wobble * 2 + 20);
  let parts = $derived(width > 0 ? shards(settings, width, height, margin) : []);
</script>

<svelte:window bind:innerWidth={width} bind:innerHeight={height} />

<div class="shards" aria-hidden="true">
  <svg viewBox="0 0 {width} {height}">
    <filter id="shards-wobble" filterUnits="userSpaceOnUse" x={-margin} y={-margin} width={width + 2 * margin} height={height + 2 * margin}>
      <feTurbulence type="fractalNoise" baseFrequency={1 / settings.wobbleSize} numOctaves="2" seed={settings.seed} />
      <feDisplacementMap in="SourceGraphic" scale={settings.wobble} xChannelSelector="R" yChannelSelector="G" />
    </filter>
    <g filter="url(#shards-wobble)">
      {#each parts as part, i (i)}
        <path d={part.d} style:fill={part.fill} />
      {/each}
    </g>
  </svg>
</div>

{#if dev}
  <DevTuner
    title="Dashboard background"
    bind:settings
    defaults={SHARDS_DEFAULTS}
    sliders={SLIDERS}
    storageKey="taskbook_shards_tuner" />
{/if}


<style>
  .shards {
    position: fixed;
    inset: 0;
    z-index: -1;
    overflow: hidden;
    pointer-events: none;
  }

  svg {
    display: block;
    width: 100%;
    height: 100%;
  }

  path {
    shape-rendering: geometricPrecision;
  }
</style>
