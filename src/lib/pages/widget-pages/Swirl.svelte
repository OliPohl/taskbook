<!-- src/lib/pages/widget-pages/Swirl.svelte -->
<!-- The background of the clock, timer and pomodoro pages: a swirl of curved blades in the wave color,
     faded out in the middle so it's only strong near the edges. It sways a little while idle and spins
     while the widget counts down (`active`), easing in and out of the spin. -->
<script module>
  /** The look and motion of the swirl. */
  export const SWIRL_DEFAULTS = {
    blades: 15,
    // How far a blade turns from its inner tip to its outer end (degrees), and how the turn is spread along it
    // (below 1 it turns most near the middle).
    twist: 10,
    twistCurve: 1.25,
    // How wide a blade is at its inner tip and outer end (degrees), and how the width grows along it.
    widthIn: 0,
    widthOut: 9.5,
    widthCurve: 1.6,
    // Where the blades start, out of 100 to the outer end.
    innerRadius: 11,
    // The swirl's size, in percent of the page's diagonal, so it looks the same on every screen size and shape.
    size: 110,
    // How much of the wave color the blades have (the rest is the background), and their opacity in percent.
    color: 50,
    opacity: 100,
    // The fade in the middle (percent of the swirl's size): clear up to `fadeStart`, `fadeMidOpacity` percent at
    // `fadeMid`, fully there from `fadeEnd`.
    fadeStart: 0,
    fadeMid: 0,
    fadeMidOpacity: 0,
    fadeEnd: 0,
    // Spin while active (degrees per second) and how quickly the speed follows (per second).
    spinSpeed: 3,
    speedEase: 1.05,
    // The idle sway (degrees, and how fast) and breathing (percent bigger and smaller, and how fast).
    sway: 0,
    swaySpeed: 0.6,
    breathe: 4.5,
    breatheSpeed: 0.8
  };

  const STEPS = 48;

  /** @param {number} r @param {number} deg */
  const point = (r, deg) => {
    const rad = (deg * Math.PI) / 180;
    return `${(r * Math.cos(rad)).toFixed(2)} ${(r * Math.sin(rad)).toFixed(2)}`;
  };

  /** Every blade: its leading edge out from the middle, then its trailing edge back in. @param {typeof SWIRL_DEFAULTS} s */
  function swirlPath(s) {
    const count = Math.max(1, Math.round(s.blades));
    const blades = [];
    for (let b = 0; b < count; b++) {
      const start = (360 / count) * b;
      const lead = [];
      const trail = [];
      for (let i = 0; i <= STEPS; i++) {
        const t = i / STEPS;
        const r = s.innerRadius + (100 - s.innerRadius) * t;
        const theta = start + s.twist * Math.pow(t, s.twistCurve);
        lead.push(point(r, theta));
        trail.push(point(r, theta + s.widthIn + (s.widthOut - s.widthIn) * Math.pow(t, s.widthCurve)));
      }
      blades.push(`M ${lead.join(' L ')} L ${trail.reverse().join(' L ')} Z`);
    }
    return blades.join(' ');
  }
</script>

<script>
  /** @type {{active?: boolean}} */
  let { active = false } = $props();

  const settings = SWIRL_DEFAULTS;
  const path = swirlPath(settings);
  const mask = (
    `radial-gradient(circle, transparent ${settings.fadeStart}%, rgba(0, 0, 0, ${settings.fadeMidOpacity / 100}) ` +
      `${settings.fadeMid}%, black ${settings.fadeEnd}%)`
  );

  /** @type {SVGSVGElement} */
  let svgEl;

  $effect(() => {
    if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;

    let angle = 0;
    let speed = 0;
    let time = 0;
    let last = performance.now();
    let frameId = requestAnimationFrame(frame);

    /** @param {number} now */
    function frame(now) {
      // Capped, so a tab coming back from the background doesn't jump.
      const dt = Math.min(0.1, (now - last) / 1000);
      last = now;
      time += dt;

      speed += ((active ? settings.spinSpeed : 0) - speed) * Math.min(1, dt * settings.speedEase);
      angle = (angle + speed * dt) % 360;

      // Always a slow sway and breathing, so the page never stands completely still.
      const sway = Math.sin(time * settings.swaySpeed) * settings.sway;
      const scale = 1 + Math.sin(time * settings.breatheSpeed) * (settings.breathe / 100);
      svgEl.style.transform = `translate(-50%, -50%) rotate(${angle + sway}deg) scale(${scale})`;

      frameId = requestAnimationFrame(frame);
    }

    return () => cancelAnimationFrame(frameId);
  });
</script>

<div class="swirl" aria-hidden="true">
  <svg
    viewBox="-100 -100 200 200"
    bind:this={svgEl}
    style:--size={settings.size / 100}
    style:--blade="color-mix(in srgb, var(--wave) {settings.color}%, var(--bg))"
    style:opacity={settings.opacity / 100}
    style:mask-image={mask}>
    <path d={path} />
  </svg>
</div>


<style>
  /* A container, so the swirl can be sized by the page's own width and height (not the window's, the sidebar may
     take part of it). */
  .swirl {
    position: absolute;
    inset: 0;
    z-index: -1;
    overflow: hidden;
    pointer-events: none;
    container-type: size;
  }

  /* Sized by the page's diagonal, so the blades reach the corners the same way on every screen. The middle can be
     masked out, so the time stays on a calm background. */
  svg {
    position: absolute;
    top: 50%;
    left: 50%;
    width: calc(hypot(100cqw, 100cqh) * var(--size));
    height: calc(hypot(100cqw, 100cqh) * var(--size));
    transform: translate(-50%, -50%);
    will-change: transform;
  }

  path {
    fill: var(--blade);
  }
</style>
