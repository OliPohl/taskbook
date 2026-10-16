<!-- src/lib/dev/DevTuner.svelte -->
<!-- Only while developing: a panel on the right with a slider for every setting of a background (like Swirl.svelte),
     to try out looks live. The values are kept in this browser until reset. "Copy" puts them on the clipboard, to
     paste into the background's defaults. -->
<script module>
  /** @typedef {{key: string, label: string, min: number, max: number, step: number}} Slider */
</script>

<script>
  /**
   * @type {{title: string, settings: Record<string, number>, defaults: Record<string, number>, sliders: Slider[],
   *   storageKey: string}}
   */
  let { title, settings = $bindable(), defaults, sliders, storageKey } = $props();

  let open = $state(true);
  let copied = $state(false);

  /** Values from the last session in this browser, once when the panel opens. */
  function load() {
    try {
      const saved = JSON.parse(localStorage.getItem(storageKey) ?? 'null');
      // Only settings that still exist, older ones are left out.
      if (saved) for (const key of Object.keys(defaults)) if (key in saved) settings[key] = saved[key];
    } catch {
      // Nothing saved, or storage is blocked: start from the defaults.
    }
  }
  load();

  $effect(() => {
    const json = JSON.stringify(settings);
    try {
      localStorage.setItem(storageKey, json);
    } catch {
      // Storage is blocked, the values just don't survive a reload.
    }
  });

  function reset() {
    Object.assign(settings, defaults);
  }

  async function copy() {
    await navigator.clipboard.writeText(JSON.stringify(settings, null, 2));
    copied = true;
    setTimeout(() => (copied = false), 1200);
  }
</script>

<aside class="tuner" class:closed={!open}>
  <header>
    <button class="title" onclick={() => (open = !open)}>
      <span class="m3-icon">{open ? 'chevron_right' : 'tune'}</span>
      {#if open}{title}{/if}
    </button>
    {#if open}
      <button onclick={copy}>{copied ? 'Copied' : 'Copy'}</button>
      <button onclick={reset}>Reset</button>
    {/if}
  </header>

  {#if open}
    <div class="rows">
      {#each sliders as slider (slider.key)}
        <label>
          <span class="label">{slider.label}</span>
          <input type="range" min={slider.min} max={slider.max} step={slider.step} bind:value={settings[slider.key]} />
          <input type="number" min={slider.min} max={slider.max} step={slider.step} bind:value={settings[slider.key]} />
        </label>
      {/each}
    </div>
  {/if}
</aside>


<style>
  .tuner {
    position: fixed;
    top: 80px;
    right: 16px;
    z-index: 5000;
    box-sizing: border-box;
    width: 300px;
    max-height: calc(100vh - 100px);
    display: flex;
    flex-direction: column;
    background-color: var(--surface);
    border: 1px solid var(--line);
    border-radius: 14px;
    box-shadow: 0 12px 40px var(--shadow);
    color: var(--text);
    font-family: system-ui, sans-serif;
    font-size: 12px;
    user-select: none;
  }

  .tuner.closed {
    width: auto;
  }

  header {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 8px;
  }

  button {
    padding: 4px 10px;
    background-color: var(--surface-raised);
    border: 1px solid var(--line);
    border-radius: 8px;
    color: var(--text);
    font: inherit;
    cursor: pointer;
  }

  button:hover {
    background-color: var(--surface-active);
  }

  .title {
    flex: 1;
    display: flex;
    align-items: center;
    gap: 4px;
    font-weight: 600;
    text-align: left;
  }

  .closed .title {
    flex: none;
  }

  .rows {
    overflow-y: auto;
    padding: 4px 12px 12px;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  label {
    display: grid;
    grid-template-columns: 96px 1fr 56px;
    align-items: center;
    gap: 8px;
  }

  .label {
    color: var(--text-soft);
  }

  input[type="range"] {
    width: 100%;
    accent-color: var(--wave);
  }

  input[type="number"] {
    box-sizing: border-box;
    width: 100%;
    padding: 2px 4px;
    background-color: var(--bg);
    border: 1px solid var(--line);
    border-radius: 6px;
    color: var(--text);
    font: inherit;
  }
</style>
