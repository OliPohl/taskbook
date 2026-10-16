<!-- src/lib/pages/widget-pages/Burst.svelte -->
<!-- The background of the calculator and image pages: a comic burst on the wave color. Thin spikes in the page's
     gray shoot out of the middle, widening outwards. Their edges are jagged, with teeth that point outwards like
     flames. Its seed glides on slowly, so the spikes slowly morph
     into new shapes and the page isn't completely still. -->
<script module>
  /** The look of the burst. Ranges are picked from at random for every spike (or every corner). Some ranges run
   * from high to low: that's fine, and swapping them would change the look. */
  export const BURST_DEFAULTS = {
    spikes: 19,
    // Always the same "random" spikes, so the burst looks the same on every visit.
    seed: 29,
    // How far a spike may turn away from its even place around the circle (part of the gap between two).
    angleJitter: 0.1,
    lengthMin: 180,
    lengthMax: 165,
    // Width at the far end, past the corners, the SVG is 200 wide.
    widthMin: 40,
    widthMax: 40,
    // Length of one zigzag step, and how far the middle line swings to the side on each.
    segmentMin: 41.5,
    segmentMax: 18.5,
    zigzagMin: 2.1,
    zigzagMax: 2.2,
    // How many corners get a tooth, and how far it sticks out.
    toothChance: 0.4,
    toothMin: 1.4,
    toothMax: 5.6,
    // How much of the page's gray the spikes have (the rest is the wave color).
    color: 100,
    // Seconds the seed takes to glide on to the next whole one, 0 keeps it still.
    seedSeconds: 40
  };

  /** @typedef {typeof BURST_DEFAULTS} BurstSettings */

  /** @param {number} r @param {number} deg @returns {[number, number]} */
  const polar = (r, deg) => {
    const rad = (deg * Math.PI) / 180;
    return [r * Math.cos(rad), r * Math.sin(rad)];
  };

  /** @param {number} x @param {number} y */
  const xy = (x, y) => `${x.toFixed(3)} ${y.toFixed(3)}`;

  /** The highest seed, after it the seeds start over at 0 and the burst blends back into seed 0's look. */
  export const MAX_SEED = 300;

  /** A fixed "random" number from 0 to 1 for a draw `n` at the whole seed `k`. @param {number} n @param {number} k */
  const hash = (n, k) => {
    const x = Math.sin(n * 12.9898 + (k % MAX_SEED) * 78.233) * 43758.5453;
    return x - Math.floor(x);
  };

  /**
   * The path of every spike. The seed may be a decimal: every random value then glides through its values at the
   * whole seeds around it on a smooth curve (Catmull-Rom), without slowing down at the whole seeds, so moving the
   * seed slowly morphs the burst instead of jumping.
   * What can't glide (how many zigzag steps a spike has, which way it starts) stays the same for every seed.
   * @param {BurstSettings} s
   */
  export function burstPath(s) {
    const below = Math.floor(s.seed);
    const t = s.seed - below;
    const t2 = t * t;
    const t3 = t2 * t;
    // The curve's weights for the values at the seeds before, below, above and after `seed`.
    const w0 = (-t3 + 2 * t2 - t) / 2;
    const w1 = (3 * t3 - 5 * t2 + 2) / 2;
    const w2 = (-3 * t3 + 4 * t2 + t) / 2;
    const w3 = (t3 - t2) / 2;
    const before = below + MAX_SEED - 1;
    let draw = 0;
    const random = () => {
      draw++;
      return (
        w0 * hash(draw, before) + w1 * hash(draw, below) + w2 * hash(draw, below + 1) + w3 * hash(draw, below + 2)
      );
    };
    /** @param {number} min @param {number} max */
    const between = (min, max) => min + random() * (max - min);
    /** The same for every seed. @param {number} n @param {number} min @param {number} max */
    const fixed = (n, min, max) => min + hash(n + 0.5, 0) * (max - min);

    const shapes = [];
    const count = Math.max(1, Math.round(s.spikes));
    // The far ends are past the corners, so on the page every spike just gets wider towards the edge.
    for (let i = 0; i < count; i++) {
      const step = 360 / count;
      const angle = step * i + between(-s.angleJitter, s.angleJitter) * step;
      const length = between(s.lengthMin, s.lengthMax);
      const width = between(s.widthMin, s.widthMax);

      // A jagged spike from the middle, from a point to `width` at its far end. Its middle line zigzags, and on
      // the outside of some corners a tooth sticks out.
      const [ax, ay] = polar(1, angle);
      // Across the spike.
      const nx = -ay;
      const ny = ax;
      const stepLength = Math.max(1, fixed(i * 3, s.segmentMin, s.segmentMax));
      const segments = Math.max(2, Math.round(fixed(i * 3 + 1, s.lengthMin, s.lengthMax) / stepLength));
      const left = [];
      const right = [];
      let side = i % 2 ? -1 : 1;
      for (let k = 0; k <= segments; k++) {
        const along = k / segments;
        const corner = k > 0 && k < segments;
        // No zigzag at the start, all spikes meet in the middle.
        const swing = k === 0 ? 0 : side * between(s.zigzagMin, s.zigzagMax);
        side = -side;
        const cx = ax * length * along + nx * swing;
        const cy = ay * length * along + ny * swing;
        const half = (width * along) / 2;
        // A tooth sticks out the same amount on thin and wide parts, so wide parts don't turn into blobs. It grows
        // in and out over a short range of its draw instead of popping, so it glides with the seed too.
        const chance = random();
        const size = between(s.toothMin, s.toothMax);
        const tooth = corner ? Math.min(1, Math.max(0, (s.toothChance - chance) / 0.15)) * size : 0;
        const leftHalf = half + (swing > 0 ? tooth : 0);
        const rightHalf = half + (swing < 0 ? tooth : 0);
        left.push(xy(cx + nx * leftHalf, cy + ny * leftHalf));
        right.push(xy(cx - nx * rightHalf, cy - ny * rightHalf));
      }
      shapes.push(`M ${left.join(' L ')} L ${right.reverse().join(' L ')} Z`);
    }
    return shapes.join(' ');
  }
</script>

<script>
  // Without a tuner the settings never change, they are just the defaults.
  const settings = BURST_DEFAULTS;
  const path = burstPath(settings);
  /** @type {SVGPathElement} */
  let pathEl;

  // The seed glides on, one whole seed every `seedSeconds`. Every frame draws straight into the path.
  $effect(() => {
    if (!(settings.seedSeconds > 0) || window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;
    let seed = settings.seed;
    let last = performance.now();
    let frameId = requestAnimationFrame(frame);

    /** @param {number} now */
    function frame(now) {
      // Capped, so a tab coming back from the background doesn't jump.
      const dt = Math.min(0.1, (now - last) / 1000);
      last = now;
      seed = (seed + dt / settings.seedSeconds) % MAX_SEED;
      pathEl.setAttribute('d', burstPath({ ...settings, seed }));
      frameId = requestAnimationFrame(frame);
    }

    return () => cancelAnimationFrame(frameId);
  });
</script>

<div class="burst" aria-hidden="true">
  <svg
    viewBox="-100 -100 200 200"
    preserveAspectRatio="xMidYMid slice"
    style:--spike="color-mix(in srgb, var(--bg) {settings.color}%, var(--wave))">
    <path d={path} bind:this={pathEl} />
  </svg>
</div>


<style>
  .burst {
    position: absolute;
    inset: 0;
    z-index: -1;
    overflow: hidden;
    pointer-events: none;
    /* Half the wave color, like every wave colored background. */
    background-color: color-mix(in srgb, var(--wave) 50%, var(--bg));
  }

  svg {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
  }

  /* Smooth, anti-aliased edges, also while the shapes morph. */
  path {
    fill: var(--spike);
    shape-rendering: geometricPrecision;
  }

</style>
