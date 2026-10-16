<!-- src/lib/pages/board/components/Shade.svelte -->
<!-- The background of a Spread, in the style of a game client: dark with depth instead of one flat color.
     Darker corners, a warm glow in the wave color behind the streaks, faint angular panels up top and fine grain.
     Across the bottom, sharp streaks in the wave color slope down to the right, one layer behind the other, each
     shaded and with a thin bright rim. Every layer drifts a little on its own slow loop.
     Fills the closest positioned parent, behind everything in it. -->
<script>
  // Back to front, the front layers lower and stronger, the front one in the full wave color. Each top edge runs
  // down to the right and turns back a few times, which leaves the pointed streaks. They reach past the sides and
  // the bottom so the drift never shows an edge.
  const LAYERS = [
    { tint: 22, d: 'M -100 640 L -100 18 L 760 130 L 420 152 L 1350 235 L 980 260 L 2020 330 L 2020 640 Z' },
    { tint: 50, d: 'M -100 640 L -100 138 L 560 215 L 250 242 L 1100 320 L 760 348 L 1700 410 L 1420 434 L 2020 458 L 2020 640 Z' },
    { tint: 100, d: 'M -100 640 L -100 292 L 700 360 L 380 394 L 1250 450 L 900 484 L 2020 526 L 2020 640 Z' }
  ];

  // Big angular panels in the upper part, barely lighter than the background, like the scenery behind a client.
  const PANELS = [
    'M 0 0 L 1920 0 L 1920 120 L 900 260 L 0 180 Z',
    'M 1100 0 L 1920 0 L 1920 420 L 1500 300 Z'
  ];

  // Fine grain over everything, so large dark areas don't look flat.
  const GRAIN =
    "url(\"data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='160' height='160'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='2' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)' opacity='0.5'/%3E%3C/svg%3E\")";
</script>

<div class="shade" aria-hidden="true">
  <svg class="panels" viewBox="0 0 1920 1080" preserveAspectRatio="none">
    {#each PANELS as d, index (index)}
      <path {d} />
    {/each}
  </svg>

  <div class="glow"></div>

  <svg class="streaks" viewBox="0 0 1920 600" preserveAspectRatio="none">
    <defs>
      {#each LAYERS as layer, index (index)}
        <!-- Lighter along the top edge and on the left, darker towards the bottom right. -->
        <linearGradient id="shade-streak-{index}" x1="0" y1="0" x2="0.35" y2="1">
          <stop offset="0" style:stop-color="color-mix(in srgb, var(--wave) {layer.tint}%, var(--bg))" />
          <stop offset="1" style:stop-color="color-mix(in srgb, var(--wave) {layer.tint * 0.45}%, var(--bg))" />
        </linearGradient>
      {/each}
    </defs>
    {#each LAYERS as layer, index (index)}
      <path
        d={layer.d}
        fill="url(#shade-streak-{index})"
        style:--rim="color-mix(in srgb, color-mix(in srgb, var(--wave) {Math.min(100, layer.tint + 30)}%, var(--bg)), white 10%)" />
    {/each}
  </svg>

  <div class="grain" style:background-image={GRAIN}></div>
  <div class="vignette"></div>
</div>


<style>
  .shade {
    position: absolute;
    inset: 0;
    z-index: -1;
    overflow: hidden;
    pointer-events: none;
    /* A little lighter at the top, a little warmer near the streaks. */
    background: linear-gradient(
      to bottom,
      color-mix(in srgb, var(--bg), white 3%),
      var(--bg) 40%,
      color-mix(in srgb, var(--wave) 8%, var(--bg))
    );
  }

  .panels {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
  }

  .panels path {
    fill: color-mix(in srgb, var(--bg), black 22%);
  }

  .panels path + path {
    fill: color-mix(in srgb, var(--wave) 5%, var(--bg));
  }

  /* The light the streaks give off, strongest on the left where they rise highest. */
  .glow {
    position: absolute;
    left: -20%;
    bottom: -30%;
    width: 90%;
    height: 90%;
    background: radial-gradient(closest-side, color-mix(in srgb, var(--wave) 22%, transparent), transparent);
  }

  /* The bottom half of the page, the streaks start high on the left and only cover the bottom on the right. */
  .streaks {
    position: absolute;
    left: 0;
    bottom: 0;
    width: 100%;
    height: 50%;
  }

  /* A thin bright rim, slightly softened at the points. Then a slow sway back and forth, every layer at its own
     pace and direction (in the SVG's own units). */
  .streaks path {
    stroke: var(--rim);
    stroke-width: 1.5px;
    stroke-opacity: 0.6;
    stroke-linejoin: round;
    vector-effect: non-scaling-stroke;
    animation: drift 26s ease-in-out infinite alternate;
  }

  .streaks path:nth-of-type(2) {
    animation-duration: 34s;
    animation-direction: alternate-reverse;
  }

  .streaks path:nth-of-type(3) {
    animation-duration: 21s;
    animation-delay: -8s;
  }

  @keyframes drift {
    from { transform: translate(-60px, -6px); }
    to { transform: translate(60px, 6px); }
  }

  .grain {
    position: absolute;
    inset: 0;
    opacity: 0.06;
    mix-blend-mode: overlay;
  }

  /* Darker corners and top edge, so the eye goes to the middle. */
  .vignette {
    position: absolute;
    inset: 0;
    background:
      linear-gradient(to bottom, color-mix(in srgb, black 35%, transparent), transparent 22%),
      radial-gradient(ellipse at 50% 45%, transparent 55%, color-mix(in srgb, black 45%, transparent));
  }

  /* On the light background the black shading would look dirty. */
  :global(:root[data-theme="light"]) .vignette {
    opacity: 0.25;
  }

  @media (prefers-reduced-motion: reduce) {
    .streaks path {
      animation: none;
    }
  }
</style>
