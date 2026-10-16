<!-- src/lib/pages/notes/Dots.svelte -->
<!-- The background of a Notes page: a dot grid in the wave color, like notebook paper. The note covers the middle,
     so it shows beside and around it. -->
<script module>
  /** The dot grid, in pixels and percent of the wave color. */
  export const DOTS_DEFAULTS = {
    dotSize: 1.2,
    dotGap: 22,
    dotOpacity: 50
  };
</script>

<script>
  const settings = DOTS_DEFAULTS;
</script>

<div
  class="dots"
  aria-hidden="true"
  style:--dot-size="{settings.dotSize}px"
  style:--dot-gap="{settings.dotGap}px"
  style:--dot-color="color-mix(in srgb, var(--wave) {settings.dotOpacity}%, transparent)">
</div>


<style>
  /* Fixed, so it stays put while a long note scrolls. */
  .dots {
    position: fixed;
    inset: 0;
    z-index: -1;
    pointer-events: none;
    background-image: radial-gradient(circle, var(--dot-color) var(--dot-size), transparent calc(var(--dot-size) + 0.5px));
    background-size: var(--dot-gap) var(--dot-gap);
    background-position: center;
  }
</style>
