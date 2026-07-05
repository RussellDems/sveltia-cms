<!--
  @component
  An analog clock time picker rendered in a popup for the DateTime field editor. Shows a Material
  Design-style dial: pick the hour by tapping or dragging on the clock face, which then advances to
  minute selection, with an AM/PM toggle. The Done button (via the `option` role handled by the
  popup service) closes the popup. Operates on a plain 24-hour `HH:mm` string; time zone handling
  is left to the field editor.
-->
<script>
  import { _ } from '@sveltia/i18n';
  import { Button } from '@sveltia/ui';

  /**
   * @typedef {object} Props
   * @property {string} [value] Selected time in 24-hour `HH:mm` format. Empty if none.
   * @property {number} [minuteStep] Step between selectable minutes. Defaults to 1.
   * @property {(value: string) => void} [onSelect] Custom `Select` event handler called with the
   * new time in 24-hour `HH:mm` format whenever the hour, minute or meridiem changes.
   */

  /** @type {Props} */
  let {
    /* eslint-disable prefer-const */
    value = '',
    minuteStep = 1,
    onSelect = undefined,
    /* eslint-enable prefer-const */
  } = $props();

  /** Diameter of the clock dial in pixels. */
  const DIAL_SIZE = 220;
  /** Radius at which the number labels are placed. */
  const NUMBER_RADIUS = 88;

  const hour24 = $derived(Number(value.match(/^(\d{2}):/)?.[1] ?? 12));
  const minute = $derived(Number(value.match(/^\d{2}:(\d{2})/)?.[1] ?? 0));
  const isPM = $derived(hour24 >= 12);
  const hour12 = $derived(hour24 % 12 === 0 ? 12 : hour24 % 12);

  /**
   * Whether the dial is currently selecting the hour or the minute.
   * @type {'hour' | 'minute'}
   */
  let mode = $state('hour');
  /** @type {HTMLElement | undefined} */
  let dial = $state();
  /** Whether a pointer drag on the dial is in progress. */
  let draggingDial = $state(false);

  const handAngle = $derived(mode === 'hour' ? (hour12 % 12) * 30 : minute * 6);
  const labels = $derived(
    mode === 'hour'
      ? Array.from({ length: 12 }, (__, index) => ({
          text: String(index === 0 ? 12 : index),
          angle: index * 30,
          selected: (hour12 % 12) * 30 === index * 30,
        }))
      : Array.from({ length: 12 }, (__, index) => ({
          text: String(index * 5).padStart(2, '0'),
          angle: index * 30,
          selected: minute === index * 5,
        })),
  );

  /**
   * Pad a number to two digits.
   * @param {number} n Number to pad.
   * @returns {string} Padded number.
   */
  const pad = (n) => String(n).padStart(2, '0');

  /**
   * Emit the given time via {@link onSelect}.
   * @param {number} h24 Hour in 24-hour format.
   * @param {number} m Minute.
   */
  const emit = (h24, m) => {
    onSelect?.(`${pad(h24)}:${pad(m)}`);
  };

  /**
   * Update the hour, keeping the current meridiem and minute.
   * @param {number} h12 Hour in 12-hour format (1–12).
   */
  const setHour12 = (h12) => {
    emit((h12 % 12) + (isPM ? 12 : 0), minute);
  };

  /**
   * Update the minute, keeping the current hour.
   * @param {number} m Minute.
   */
  const setMinute = (m) => {
    emit(hour24, ((m % 60) + 60) % 60);
  };

  /**
   * Update the meridiem, keeping the current 12-hour value and minute.
   * @param {boolean} pm `true` for PM, `false` for AM.
   */
  const setMeridiem = (pm) => {
    if (pm !== isPM) {
      emit((hour24 + 12) % 24, minute);
    }
  };

  /**
   * Convert a pointer event position to a value on the dial and select it.
   * @param {PointerEvent} event Pointer event.
   */
  const selectFromPointer = (event) => {
    if (!dial) {
      return;
    }

    const { left, top, width, height } = dial.getBoundingClientRect();
    const x = event.clientX - left - width / 2;
    const y = event.clientY - top - height / 2;
    // Angle in degrees clockwise from the 12 o’clock position
    const degrees = ((Math.atan2(x, -y) * 180) / Math.PI + 360) % 360;

    if (mode === 'hour') {
      const h = Math.round(degrees / 30) % 12;

      setHour12(h === 0 ? 12 : h);
    } else {
      setMinute((Math.round(degrees / (6 * minuteStep)) * minuteStep) % 60);
    }
  };

  /**
   * Adjust the value in the current mode by the given amount. Used for keyboard support.
   * @param {number} delta Amount to adjust by.
   */
  const nudge = (delta) => {
    if (mode === 'hour') {
      const h = ((hour12 - 1 + delta + 12) % 12) + 1;

      setHour12(h);
    } else {
      setMinute(minute + delta * minuteStep);
    }
  };
</script>

<div role="group" class="time-picker" aria-label={_('date_picker.select_time')}>
  <div role="none" class="display">
    <Button
      class="digits {mode === 'hour' ? 'active' : ''}"
      aria-label={_('date_picker.hour')}
      aria-pressed={mode === 'hour'}
      onclick={() => {
        mode = 'hour';
      }}
    >
      {pad(hour12)}
    </Button>
    <span role="none" class="separator">:</span>
    <Button
      class="digits {mode === 'minute' ? 'active' : ''}"
      aria-label={_('date_picker.minute')}
      aria-pressed={mode === 'minute'}
      onclick={() => {
        mode = 'minute';
      }}
    >
      {pad(minute)}
    </Button>
    <div role="none" class="meridiem">
      <Button
        size="small"
        class={!isPM ? 'active' : ''}
        aria-pressed={!isPM}
        label={_('date_picker.am')}
        onclick={() => {
          setMeridiem(false);
        }}
      />
      <Button
        size="small"
        class={isPM ? 'active' : ''}
        aria-pressed={isPM}
        label={_('date_picker.pm')}
        onclick={() => {
          setMeridiem(true);
        }}
      />
    </div>
  </div>
  <div
    role="slider"
    tabindex="0"
    class="dial"
    style:width="{DIAL_SIZE}px"
    style:height="{DIAL_SIZE}px"
    bind:this={dial}
    aria-label={_(mode === 'hour' ? 'date_picker.hour' : 'date_picker.minute')}
    aria-valuemin={mode === 'hour' ? 1 : 0}
    aria-valuemax={mode === 'hour' ? 12 : 59}
    aria-valuenow={mode === 'hour' ? hour12 : minute}
    onpointerdown={(event) => {
      draggingDial = true;
      dial?.setPointerCapture(event.pointerId);
      selectFromPointer(event);
    }}
    onpointermove={(event) => {
      if (draggingDial) {
        selectFromPointer(event);
      }
    }}
    onpointerup={() => {
      draggingDial = false;

      if (mode === 'hour') {
        mode = 'minute';
      }
    }}
    onkeydown={(event) => {
      const { key } = event;

      if (['ArrowUp', 'ArrowRight'].includes(key)) {
        event.preventDefault();
        nudge(1);
      } else if (['ArrowDown', 'ArrowLeft'].includes(key)) {
        event.preventDefault();
        nudge(-1);
      } else if (key === 'Enter' && mode === 'hour') {
        event.preventDefault();
        mode = 'minute';
      }
    }}
  >
    <div role="none" class="hand" style:transform="rotate({handAngle}deg)">
      <div role="none" class="knob"></div>
    </div>
    <div role="none" class="center-dot"></div>
    {#each labels as { text, angle, selected } (`${mode}-${text}`)}
      <div
        role="none"
        class="number"
        class:selected
        style:left="{DIAL_SIZE / 2 + NUMBER_RADIUS * Math.sin((angle * Math.PI) / 180)}px"
        style:top="{DIAL_SIZE / 2 - NUMBER_RADIUS * Math.cos((angle * Math.PI) / 180)}px"
      >
        {text}
      </div>
    {/each}
  </div>
  <div role="none" class="footer">
    <Button variant="tertiary" size="small" role="option" label={_('done')} />
  </div>
</div>

<style>
  .time-picker {
    display: inline-flex;
    flex-direction: column;
    align-items: center;
    gap: 12px;
    padding: 12px;
    -webkit-user-select: none;
    user-select: none;
    cursor: default;
  }

  .display {
    display: flex;
    align-items: center;
    gap: 4px;

    .separator {
      font-size: var(--sui-font-size-x-large, 20px);
      font-weight: var(--sui-font-weight-bold, 600);
    }

    :global(button.digits) {
      justify-content: center;
      margin: 0 !important;
      padding: 4px 8px;
      min-width: 48px;
      border-radius: var(--sui-control-medium-border-radius, 4px);
      font-size: var(--sui-font-size-x-large, 20px);
      font-weight: var(--sui-font-weight-bold, 600);
      font-variant-numeric: tabular-nums;
    }

    :global(button.digits.active) {
      color: var(--sui-primary-accent-color);
      background-color: var(--sui-selected-background-color, var(--sui-hover-background-color));
    }

    .meridiem {
      display: flex;
      flex-direction: column;
      gap: 2px;
      margin-inline-start: 8px;

      :global(button) {
        margin: 0 !important;
        justify-content: center;
      }

      :global(button.active) {
        color: var(--sui-highlight-foreground-color, #fff);
        background-color: var(--sui-primary-accent-color);
      }
    }
  }

  .dial {
    position: relative;
    border-radius: 50%;
    background-color: var(--sui-secondary-background-color);
    touch-action: none;
    cursor: pointer;

    &:focus-visible {
      outline: 2px solid var(--sui-primary-accent-color);
      outline-offset: 2px;
    }

    .number {
      position: absolute;
      display: flex;
      align-items: center;
      justify-content: center;
      width: 32px;
      height: 32px;
      border-radius: 50%;
      transform: translate(-50%, -50%);
      font-size: var(--sui-font-size-small);
      font-variant-numeric: tabular-nums;
      pointer-events: none;

      &.selected {
        color: var(--sui-highlight-foreground-color, #fff);
      }
    }

    .hand {
      position: absolute;
      left: calc(50% - 1px);
      bottom: 50%;
      width: 2px;
      height: 88px;
      background-color: var(--sui-primary-accent-color);
      transform-origin: bottom center;
      pointer-events: none;

      .knob {
        position: absolute;
        top: -16px;
        left: calc(50% - 16px);
        width: 32px;
        height: 32px;
        border-radius: 50%;
        background-color: var(--sui-primary-accent-color);
        opacity: 0.9;
      }
    }

    .center-dot {
      position: absolute;
      left: calc(50% - 3px);
      top: calc(50% - 3px);
      width: 6px;
      height: 6px;
      border-radius: 50%;
      background-color: var(--sui-primary-accent-color);
      pointer-events: none;
    }
  }

  .footer {
    display: flex;
    justify-content: end;
    width: 100%;
  }
</style>
