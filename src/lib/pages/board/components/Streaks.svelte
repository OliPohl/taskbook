<!-- src/lib/pages/board/components/Streaks.svelte -->
<!-- The background of a Spread: brush strokes in the wave color across the bottom of the page, generated from a
     seed. Long strokes follow a bowl shaped curve and cross each other in pairs from opposite sides, their sharp
     tips overlapping, which leaves thin dark crevices between them: the band is a big pair from both sides, a short
     stroke lies on top at each side, and pairs float above the band. The seed glides on slowly, so everything
     flexes like the burst. Drawn on a 1293 x 911 page and stretched to the page's size.
     Fills the closest positioned parent, behind everything in it. While developing, a panel on the right (DevTuner)
     changes the settings live. -->
<script module>
  export const STREAKS_DEFAULTS = {
    seed: 0,
    // Seconds it takes to glide on to the next seed, 0 keeps it still.
    seedSeconds: 30,
    // The bowl the strokes follow: its height from the bottom at the left, the middle and the right (out of 911).
    left: 135,
    middle: 90,
    right: 160,
    // The band's two big strokes: their thickness, how far towards the other side their tips reach (percent of the
    // width), and how far apart up and down they cross.
    bandThickness: 34,
    bandReach: 70,
    bandSplit: 22,
    // The short strokes on top at each side: how many layers, how long (percent of the width), how thick, and how
    // far above the band.
    sideLayers: 1,
    sideLength: 20,
    sideThickness: 26,
    sideStep: 26,
    // The floating pairs: how many, how long one stroke is (percent of the width), how thick, how far above the
    // band, and how far apart the pair's two strokes are up and down.
    floaters: 1,
    floaterLength: 36,
    floaterThickness: 32,
    floaterGap: 60,
    floaterSplit: 20,
    // How much the tips lift up towards their point, and how wobbly the strokes are.
    tipLift: 12,
    roughness: 3,
    // 1 mirrors everything left to right.
    flip: 1,
    // How much of the wave color they have (the rest is the background), and their opacity, in percent.
    color: 50,
    opacity: 100
  };

  /** @type {import('$lib/dev/DevTuner.svelte').Slider[]} */
  const SLIDERS = [
    { key: 'seed', label: 'Seed', min: 0, max: 300, step: 0.01 },
    { key: 'seedSeconds', label: 'Seed step seconds', min: 0, max: 120, step: 0.5 },
    { key: 'left', label: 'Height left', min: 0, max: 700, step: 5 },
    { key: 'middle', label: 'Height middle', min: 0, max: 700, step: 5 },
    { key: 'right', label: 'Height right', min: 0, max: 700, step: 5 },
    { key: 'bandThickness', label: 'Band thickness', min: 4, max: 150, step: 1 },
    { key: 'bandReach', label: 'Band reach %', min: 30, max: 100, step: 1 },
    { key: 'bandSplit', label: 'Band split', min: 0, max: 120, step: 1 },
    { key: 'sideLayers', label: 'Side layers', min: 0, max: 4, step: 1 },
    { key: 'sideLength', label: 'Side length %', min: 5, max: 60, step: 1 },
    { key: 'sideThickness', label: 'Side thickness', min: 4, max: 120, step: 1 },
    { key: 'sideStep', label: 'Side step', min: 0, max: 120, step: 1 },
    { key: 'floaters', label: 'Floating pairs', min: 0, max: 4, step: 1 },
    { key: 'floaterLength', label: 'Floating length %', min: 5, max: 80, step: 1 },
    { key: 'floaterThickness', label: 'Floating thickness', min: 4, max: 120, step: 1 },
    { key: 'floaterGap', label: 'Floating gap', min: 0, max: 400, step: 1 },
    { key: 'floaterSplit', label: 'Floating split', min: 0, max: 120, step: 1 },
    { key: 'tipLift', label: 'Tip lift', min: -60, max: 60, step: 1 },
    { key: 'roughness', label: 'Roughness', min: 0, max: 20, step: 0.5 },
    { key: 'flip', label: 'Flip', min: 0, max: 1, step: 1 },
    { key: 'color', label: 'Wave color %', min: 0, max: 100, step: 1 },
    { key: 'opacity', label: 'Opacity %', min: 0, max: 100, step: 1 }
  ];

  const WIDTH = 1293;
  const HEIGHT = 911;
  /** After it the seeds start over at 0, blending back into seed 0. */
  const MAX_SEED = 300;
  const STEPS = 40;

  /** A fixed "random" number from 0 to 1 for a draw `n` at the whole seed `k`. @param {number} n @param {number} k */
  const hash = (n, k) => {
    const x = Math.sin(n * 12.9898 + (((k % MAX_SEED) + MAX_SEED) % MAX_SEED) * 78.233) * 43758.5453;
    return x - Math.floor(x);
  };

  /** A smooth curve through a, b, c, d, at t between b and c (Catmull-Rom). @param {number} a @param {number} b @param {number} c @param {number} d @param {number} t */
  const curve = (a, b, c, d, t) =>
    0.5 * (2 * b + (c - a) * t + (2 * a - 5 * b + 4 * c - d) * t * t + (3 * b - a - 3 * c + d) * t * t * t);

  /** A value from 0 to 1 for the draw `n` that glides smoothly with the seed. @param {number} n @param {number} seed */
  function glide(n, seed) {
    const k = Math.floor(seed);
    return curve(hash(n, k - 1), hash(n, k), hash(n, k + 1), hash(n, k + 2), seed - k);
  }

  /** Smooth noise from -1 to 1 along x (control points `spacing` apart) that glides with the seed. @param {number} x @param {number} seed @param {number} spacing */
  function noise(x, seed, spacing) {
    const position = x / spacing;
    const i = Math.floor(position);
    const value = (/** @type {number} */ n) => glide(5000 + n, seed) * 2 - 1;
    return curve(value(i - 1), value(i), value(i + 1), value(i + 2), position - i);
  }

  /** @param {[number, number][]} points */
  const area = (points) =>
    points.reduce((sum, [x, y], i) => {
      const [nx, ny] = points[(i + 1) % points.length];
      return sum + x * ny - nx * y;
    }, 0);

  /** A closed part, always turning the same way so overlapping parts add up instead of cutting holes. @param {[number, number][]} points */
  function part(points) {
    if (area(points) < 0) points.reverse();
    return 'M ' + points.map(([x, y]) => `${x.toFixed(1)} ${y.toFixed(1)}`).join(' L ') + ' Z';
  }

  /** Every part of the streaks at a seed. @param {typeof STREAKS_DEFAULTS} s @param {number} seed */
  export function streaksPath(s, seed) {
    let draw = 0;
    /** The next random value, gliding with the seed. Always drawn in the same order. */
    const random = () => glide(++draw, seed);
    /** @param {number} min @param {number} max */
    const between = (min, max) => min + random() * (max - min);

    // The bowl: a smooth curve through the three heights.
    const heights = [s.left * between(0.85, 1.15), s.middle * between(0.8, 1.2), s.right * between(0.85, 1.15)];
    /** The bowl's y at x. @param {number} x */
    const bowl = (x) => {
      const t = x / WIDTH;
      const height = heights[0] * 2 * (t - 0.5) * (t - 1) - heights[1] * 4 * t * (t - 1) + heights[2] * 2 * t * (t - 0.5);
      return HEIGHT - height;
    };

    /** @type {string[]} */
    const parts = [];

    /**
     * One stroke along the bowl, from `from` to its sharp tip at `to` (either way), `offset` below the bowl.
     * Its tail is off the page (`edge`) or a blunt end. Full thickness most of the way, thinning to the tip,
     * which lifts a little.
     * @param {number} from @param {number} to @param {number} offset @param {number} thickness @param {boolean} edge
     */
    const stroke = (from, to, offset, thickness, edge) => {
      const wobble = 1000 + draw * 37;
      const upper = [];
      const lower = [];
      for (let k = 0; k <= STEPS; k++) {
        const t = k / STEPS;
        const x = from + (to - from) * t;
        const y = bowl(x) + offset - s.tipLift * t * t + noise(x + wobble, seed, 90) * s.roughness;
        const tail = edge ? 1 : Math.sqrt(Math.min(1, t / 0.1));
        const tip = t < 0.5 ? 1 : Math.pow(1 - (t - 0.5) / 0.5, 0.75);
        const half = (thickness / 2) * tail * tip;
        upper.push(/** @type {[number, number]} */ ([x, y - half]));
        lower.push(/** @type {[number, number]} */ ([x, y + half]));
      }
      parts.push(part([...upper, ...lower.reverse()]));
    };

    // The band: solid below its two big strokes, which come from both sides and cross with their tips. The solid
    // part reaches up into both, so a crevice only opens where a stroke thins out to its tip.
    const band = s.bandThickness * between(0.85, 1.15);
    const split = s.bandSplit * between(0.7, 1.3);
    /** @type {[number, number][]} */
    const base = [];
    for (let x = -10; x <= WIDTH + 10; x += 10) base.push([x, bowl(x) + split / 2 - band * 0.3]);
    base.push([WIDTH + 10, HEIGHT + 10], [-10, HEIGHT + 10]);
    parts.push(part(base));
    const reach = (s.bandReach / 100) * between(0.9, 1.1);
    stroke(-100, WIDTH * reach, -split / 2, band, true);
    stroke(WIDTH + 100, WIDTH * (1 - reach * between(0.9, 1.05)), split / 2, band, true);

    // Short strokes on top at each side, each layer higher and shorter.
    for (let layer = 1; layer <= Math.round(s.sideLayers); layer++) {
      const length = (s.sideLength / 100) * WIDTH;
      const thickness = s.sideThickness * between(0.8, 1.2);
      const step = (s.sideStep + s.sideThickness * 0.5) * layer;
      stroke(-100, length * between(0.7, 1.3) / layer, -split / 2 - step * between(0.8, 1.2), thickness, true);
      stroke(WIDTH + 100, WIDTH - (length * between(0.7, 1.3)) / layer, split / 2 - step * between(0.8, 1.2), thickness, true);
    }

    // Floating pairs above the band: an upper stroke with its tip to one side, a lower one crossing it from the
    // other side, both with blunt tails.
    const count = Math.round(s.floaters);
    for (let i = 0; i < count; i++) {
      const center = WIDTH * ((i + between(0.35, 0.65)) / count);
      const length = (s.floaterLength / 100) * WIDTH * between(0.8, 1.2);
      const thickness = s.floaterThickness * between(0.8, 1.2);
      const gap = -split / 2 - band / 2 - s.floaterGap * between(0.8, 1.2) - thickness / 2;
      const pairSplit = s.floaterSplit * between(0.7, 1.3);
      // Alternating which way the upper stroke points.
      const way = i % 2 === 0 ? 1 : -1;
      const overlap = length * between(0.25, 0.4);
      stroke(center - way * (length - overlap / 2), center + way * (overlap / 2), gap, thickness, false);
      stroke(center + way * (length - overlap / 2), center - way * (overlap / 2), gap + pairSplit, thickness * between(0.8, 1), false);
    }
    return parts.join(' ');
  }
</script>

<script>
  import { dev } from '$app/environment';
  import DevTuner from '$lib/dev/DevTuner.svelte';

  let settings = $state({ ...STREAKS_DEFAULTS });
  let path = $derived(streaksPath(settings, settings.seed));
  /** @type {SVGPathElement} */
  let pathEl;

  // The seed glides on, one whole seed every `seedSeconds`. Every frame draws straight into the path, outside of
  // Svelte. Restarts when a setting changes, from the seed the settings have then.
  $effect(() => {
    const current = { ...settings };
    if (!(current.seedSeconds > 0) || window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;
    let seed = current.seed;
    let last = performance.now();
    let frameId = requestAnimationFrame(frame);

    /** @param {number} now */
    function frame(now) {
      // Capped, so a tab coming back from the background doesn't jump.
      const dt = Math.min(0.1, (now - last) / 1000);
      last = now;
      seed = (seed + dt / current.seedSeconds) % MAX_SEED;
      pathEl.setAttribute('d', streaksPath(current, seed));
      frameId = requestAnimationFrame(frame);
    }

    return () => cancelAnimationFrame(frameId);
  });
</script>

<div class="streaks" aria-hidden="true" style:opacity={settings.opacity / 100}>
  <svg
    viewBox="0 0 {WIDTH} {HEIGHT}"
    preserveAspectRatio="none"
    style:transform={settings.flip >= 1 ? 'scaleX(-1)' : undefined}
    style:--fill="color-mix(in srgb, var(--wave) {settings.color}%, var(--bg))">
    <path d={path} bind:this={pathEl} />
  </svg>
</div>

{#if dev}
  <DevTuner
    title="Spread background"
    bind:settings
    defaults={STREAKS_DEFAULTS}
    sliders={SLIDERS}
    storageKey="taskbook_streaks_strokes_tuner" />
{/if}


<style>
  .streaks {
    position: absolute;
    inset: 0;
    z-index: -1;
    overflow: hidden;
    pointer-events: none;
  }

  svg {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
  }

  path {
    fill: var(--fill);
    shape-rendering: geometricPrecision;
  }
</style>
