<!--
  @component
  A visual calendar date picker rendered in a popup for the DateTime field editor. Modeled on the
  `Calendar` component in `@sveltia/ui`, with working month/year navigation and selection state.
  Operates on a plain `YYYY-MM-DD` string; time zone handling is left to the field editor.
-->
<script>
  import { _, isRTL } from '@sveltia/i18n';
  import { Button, Icon } from '@sveltia/ui';
  import { untrack } from 'svelte';

  /**
   * @typedef {object} Props
   * @property {string} [value] Selected date in `YYYY-MM-DD` format. Empty if none.
   * @property {string} [min] Minimum selectable date in `YYYY-MM-DD` format.
   * @property {string} [max] Maximum selectable date in `YYYY-MM-DD` format.
   * @property {(value: string) => void} [onSelect] Custom `Select` event handler called with the
   * new date in `YYYY-MM-DD` format when a day is picked.
   */

  /** @type {Props} */
  let {
    /* eslint-disable prefer-const */
    value = '',
    min = undefined,
    max = undefined,
    onSelect = undefined,
    /* eslint-enable prefer-const */
  } = $props();

  /**
   * List of month names for the header label. Use a fixed date to avoid issues with daylight
   * saving time.
   */
  const MONTH_NAMES = Array.from({ length: 12 }, (__, i) =>
    new Date(2000, i, 10).toLocaleDateString(undefined, { month: 'short' }),
  );

  const now = new Date();
  const today = $derived(
    [now.getFullYear(), now.getMonth() + 1, now.getDate()]
      .map((n, i) => String(n).padStart(i ? 2 : 4, '0'))
      .join('-'),
  );

  /**
   * Parse a `YYYY-MM-DD` string into year/month/day numbers.
   * @param {string} dateStr Date string.
   * @returns {[number, number, number] | undefined} Year, month (0–11) and day, or `undefined` if
   * the string cannot be parsed.
   */
  const parseDate = (dateStr) => {
    const match = dateStr?.match(/^(\d{4})-(\d{2})-(\d{2})/);

    return match
      ? [Number(match[1]), Number(match[2]) - 1, Number(match[3])]
      : undefined;
  };

  /**
   * Format year/month/day numbers as a `YYYY-MM-DD` string.
   * @param {number} year Full year.
   * @param {number} month Month index (0–11).
   * @param {number} day Day of the month.
   * @returns {string} Formatted date.
   */
  const formatDate = (year, month, day) =>
    `${String(year).padStart(4, '0')}-${String(month + 1).padStart(2, '0')}-${String(
      day,
    ).padStart(2, '0')}`;

  /** Year currently shown in the calendar view. */
  let viewYear = $state(now.getFullYear());
  /** Month index (0–11) currently shown in the calendar view. */
  let viewMonth = $state(now.getMonth());

  $effect(() => {
    // Keep the calendar view in sync with the selected date, e.g. when the user edits the date in
    // the text input while the picker is mounted
    const parsed = parseDate(value);

    if (parsed) {
      untrack(() => {
        [viewYear, viewMonth] = parsed;
      });
    }
  });

  const dayGrid = $derived.by(() => {
    const firstOfMonth = new Date(viewYear, viewMonth, 1);
    // Start the grid on the Sunday on or before the 1st of the month. The cursor is local to this
    // computation and doesn’t need to be reactive.
    // eslint-disable-next-line svelte/prefer-svelte-reactivity
    const cursor = new Date(viewYear, viewMonth, 1 - firstOfMonth.getDay());

    return Array.from({ length: 42 }, () => {
      const date = formatDate(cursor.getFullYear(), cursor.getMonth(), cursor.getDate());

      const cell = {
        date,
        label: cursor.getDate(),
        otherMonth: cursor.getMonth() !== viewMonth,
        disabled: (!!min && date < min.slice(0, 10)) || (!!max && date > max.slice(0, 10)),
      };

      cursor.setDate(cursor.getDate() + 1);

      return cell;
    });
  });

  const weekdayLabels = $derived(
    dayGrid.slice(0, 7).map(({ date }) => {
      const [year, month, day] = /** @type {[number, number, number]} */ (parseDate(date));

      return new Date(year, month, day).toLocaleDateString(undefined, { weekday: 'narrow' });
    }),
  );

  /**
   * Move the calendar view by the given number of months.
   * @param {number} delta Number of months to move. Negative to go back.
   */
  const moveMonth = (delta) => {
    const date = new Date(viewYear, viewMonth + delta, 1);

    viewYear = date.getFullYear();
    viewMonth = date.getMonth();
  };
</script>

<div role="group" class="date-picker" aria-label={_('date_picker.select_date')}>
  <div role="none" class="header">
    <Button
      iconic
      size="small"
      aria-label={_('date_picker.previous_year')}
      onclick={() => {
        viewYear -= 1;
      }}
    >
      <Icon name={isRTL() ? 'keyboard_double_arrow_right' : 'keyboard_double_arrow_left'} />
    </Button>
    <Button
      iconic
      size="small"
      aria-label={_('date_picker.previous_month')}
      onclick={() => {
        moveMonth(-1);
      }}
    >
      <Icon name={isRTL() ? 'chevron_right' : 'chevron_left'} />
    </Button>
    <div role="none" class="view-label">{MONTH_NAMES[viewMonth]} {viewYear}</div>
    <Button
      iconic
      size="small"
      aria-label={_('date_picker.next_month')}
      onclick={() => {
        moveMonth(1);
      }}
    >
      <Icon name={isRTL() ? 'chevron_left' : 'chevron_right'} />
    </Button>
    <Button
      iconic
      size="small"
      aria-label={_('date_picker.next_year')}
      onclick={() => {
        viewYear += 1;
      }}
    >
      <Icon name={isRTL() ? 'keyboard_double_arrow_left' : 'keyboard_double_arrow_right'} />
    </Button>
  </div>
  <div role="listbox" class="grid">
    {#each weekdayLabels as label, index (index)}
      <div role="none" class="weekday">{label}</div>
    {/each}
    {#each dayGrid as { date, label, otherMonth, disabled } (date)}
      <div
        role="none"
        class:other-month={otherMonth}
        class:today={date === today}
        class:selected={date === value.slice(0, 10)}
      >
        <Button
          role="option"
          {disabled}
          aria-selected={date === value.slice(0, 10)}
          onclick={() => {
            onSelect?.(date);
          }}
        >
          {label}
        </Button>
      </div>
    {/each}
  </div>
  <div role="none" class="footer">
    <Button
      variant="tertiary"
      size="small"
      role="option"
      label={_('today')}
      onclick={() => {
        onSelect?.(today);
      }}
    />
  </div>
</div>

<style>
  .date-picker {
    display: inline-flex;
    flex-direction: column;
    gap: 8px;
    padding: 8px;
    -webkit-user-select: none;
    user-select: none;
    cursor: default;

    & > * {
      flex: none;
    }
  }

  .header {
    display: flex;
    gap: 4px;
    align-items: center;

    .view-label {
      flex: auto;
      text-align: center;
      font-size: var(--sui-font-size-small);
      font-weight: var(--sui-font-weight-bold, 600);
    }
  }

  .footer {
    display: flex;
    justify-content: center;
  }

  .grid {
    display: grid !important;
    grid-template-columns: repeat(7, 28px);
    gap: 2px;

    div {
      display: flex;
      justify-content: center;
      align-items: center;
      width: 28px;
      height: 28px;
      font-size: var(--sui-font-size-small);

      &.weekday {
        color: var(--sui-secondary-foreground-color);
      }

      &.other-month {
        color: var(--sui-tertiary-foreground-color);
      }

      &.today :global(button) {
        border-width: 1px;
        border-color: var(--sui-primary-accent-color);
      }

      &.selected :global(button) {
        color: var(--sui-highlight-foreground-color, #fff);
        background-color: var(--sui-primary-accent-color);
      }

      :global(button) {
        justify-content: center;
        margin: 0 !important;
        width: 100%;
        height: 28px;
        border-radius: 50%;
      }

      :global(button:hover) {
        background-color: var(--sui-hover-background-color);
      }
    }
  }
</style>
