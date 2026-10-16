<!-- src/lib/pages/widget-pages/WidgetPage.svelte -->
<!-- A widget as a minimal page: icon, title and settings, then the widget itself (see WidgetPageView).
     Clock, timer and pomodoro pages have the swirl behind them, it spins while the widget counts down.
     Calculator and image pages have the burst. -->
<script>
  import WidgetPageView from './WidgetPageView.svelte';
  import Swirl from './Swirl.svelte';
  import Burst from './Burst.svelte';
  import { widgetOf } from './store.svelte.js';
  import { WIDGET_TYPES } from '$lib/widgets/store.svelte.js';
  import PageHeader from '$lib/app/PageHeader.svelte';

  /** @type {{page: import('../types.js').Page}} */
  let { page } = $props();

  let icon = $derived(WIDGET_TYPES[/** @type {keyof typeof WIDGET_TYPES} */ (page.type)].icon);
  let swirl = $derived(['clock', 'timer', 'pomodoro'].includes(page.type));
  let burst = $derived(['calculator', 'image'].includes(page.type));
  // A running timer or pomodoro, a clock whose alarm is set, or anything that rings.
  let active = $derived.by(() => {
    const widget = widgetOf(page);
    return !!(widget && (widget.endsAt || widget.ringing || (widget.type === 'clock' && widget.nextAt)));
  });
</script>

<main class="widget-page">
  {#if swirl}
    <Swirl {active} />
  {:else if burst}
    <Burst />
  {/if}
  <PageHeader {page} {icon} centered />
  <WidgetPageView {page} />
</main>


<style>
  .widget-page {
    position: relative;
    box-sizing: border-box;
    min-height: 100vh;
    padding: 80px 10px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }
</style>
