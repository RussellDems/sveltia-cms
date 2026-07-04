<!--
  @component
  A visual time picker rendered in a popup for the DateTime field editor. Shows scrollable hour and
  minute columns. Picking an hour keeps the popup open; picking a minute closes it (via the
  `option` role handled by the popup service). Operates on a plain `HH:mm` string; time zone
  handling is left to the field editor.
-->
<script>
  import { _ } from '@sveltia/i18n';
  import { Button } from '@sveltia/ui';

  /**
   * @typedef {object} Props
   * @property {string} [value] Selected time in `HH:mm` format. Empty if none.
   * @property {number} [minuteStep] Step between selectable minutes. Defaults to 1.
   * @property {(value: string) => void} [onSelect] Custom `Select` event handler called with the
   * new time in `HH:mm` format when an hour or minute is picked.
   */

  /** @type {Props} */
  let {
    /* eslint-disable prefer-const */
    value = '',
    minuteStep = 1,
    onSelect = undefined,
    /* eslint-enable prefer-const */
  } = $props();

  const HOURS = Array.from({ length: 24 }, (__, i) => String(i).padStart(2, '0'));
  const minutes = $derived(
    Array.from({ length: Math.ceil(60 / minuteStep) }, (__, i) =>
      String(i * minuteStep).padStart(2, '0'),
    ),
  );

  const selectedHour = $derived(value.match(/^(\d{2}):/)?.[1] ?? '');
  const selectedMinute = $derived(value.match(/^\d{2}:(\d{2})/)?.[1] ?? '');

  /** @type {HTMLElement | undefined} */
  let hourColumn = $state();
  /** @type {HTMLElement | undefined} */
  let minuteColumn = $state();

  /**
   * Scroll the selected item in a column into view, centered.
   * @param {HTMLElement | undefined} column Column element.
   */
  const scrollToSelected = (column) => {
    const selected = /** @type {HTMLElement | null} */ (
      column?.querySelector('[aria-selected="true"]')
    );

    if (column && selected) {
      column.scrollTop = selected.offsetTop - column.clientHeight / 2 + selected.clientHeight / 2;
    }
  };

  $effect(() => {
    scrollToSelected(hourColumn);
    scrollToSelected(minuteColumn);
  });
</script>

<div role="group" class="time-picker" aria-label={_('date_picker.select_time')}>
  <div role="none" class="columns">
    <div role="listbox" class="column" aria-label={_('date_picker.hour')} bind:this={hourColumn}>
      {#each HOURS as hour (hour)}
        <Button
          aria-selected={hour === selectedHour}
          class={hour === selectedHour ? 'selected' : ''}
          onclick={() => {
            onSelect?.(`${hour}:${selectedMinute || '00'}`);
          }}
        >
          {hour}
        </Button>
      {/each}
    </div>
    <div
      role="listbox"
      class="column"
      aria-label={_('date_picker.minute')}
      bind:this={minuteColumn}
    >
      {#each minutes as minute (minute)}
        <Button
          role="option"
          aria-selected={minute === selectedMinute}
          class={minute === selectedMinute ? 'selected' : ''}
          onclick={() => {
            onSelect?.(`${selectedHour || '00'}:${minute}`);
          }}
        >
          {minute}
        </Button>
      {/each}
    </div>
  </div>
</div>

<style>
  .time-picker {
    display: inline-flex;
    padding: 8px;
    -webkit-user-select: none;
    user-select: none;
    cursor: default;
  }

  .columns {
    display: flex;
    gap: 4px;
  }

  .column {
    display: flex;
    flex-direction: column;
    gap: 2px;
    height: 200px;
    overflow-y: auto;
    scrollbar-width: thin;
    padding: 2px;

    :global(button) {
      flex: none;
      justify-content: center;
      margin: 0 !important;
      width: 40px;
      height: 28px;
      border-radius: var(--sui-control-medium-border-radius, 4px);
      font-size: var(--sui-font-size-small);
      font-variant-numeric: tabular-nums;
    }

    :global(button:hover) {
      background-color: var(--sui-hover-background-color);
    }

    :global(button.selected) {
      color: var(--sui-highlight-foreground-color, #fff);
      background-color: var(--sui-primary-accent-color);
    }
  }
</style>
