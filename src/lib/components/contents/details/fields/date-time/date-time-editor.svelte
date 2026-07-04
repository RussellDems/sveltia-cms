<!--
  @component
  Implement the editor for a DateTime field.
  @see https://decapcms.org/docs/widgets/#Datetime
  @see https://sveltiacms.app/en/docs/fields/datetime
  @todo Replace the native `<input>` with a custom component.
-->
<script>
  import { _ } from '@sveltia/i18n';
  import { Button, Icon } from '@sveltia/ui';
  import { untrack } from 'svelte';

  import DatePicker from '$lib/components/contents/details/fields/date-time/date-picker.svelte';
  import TimePicker from '$lib/components/contents/details/fields/date-time/time-picker.svelte';
  import { parseDateTimeConfig } from '$lib/services/contents/fields/date-time/config';
  import {
    getCurrentDateTime,
    getCurrentValue,
    getDate,
    getInputValue,
  } from '$lib/services/contents/fields/date-time/helper';
  import {
    getInitialTimeZone,
    getTimeZoneLabel,
  } from '$lib/services/contents/fields/date-time/timezone';

  /**
   * @import { FieldEditorProps } from '$lib/types/private';
   * @import { DateTimeField } from '$lib/types/public';
   */

  /**
   * @typedef {object} Props
   * @property {DateTimeField} fieldConfig Field configuration.
   * @property {string | undefined} currentValue Field value.
   */

  /** @type {FieldEditorProps & Props} */
  let {
    /* eslint-disable prefer-const */
    fieldId,
    fieldConfig,
    currentValue = $bindable(),
    required = true,
    readonly = false,
    invalid = false,
    /* eslint-enable prefer-const */
  } = $props();

  let inputValue = $state('');
  let isInputFocused = $state(false);

  const { type, min, max, step, dateOnly, utc, singleCustomTimeZone } = $derived(
    parseDateTimeConfig(fieldConfig),
  );
  const timeZone = $derived(getInitialTimeZone(currentValue, fieldConfig));

  /**
   * Update {@link inputValue} based on {@link currentValue}. Only update if the input is not
   * currently focused to avoid interfering with user typing.
   */
  const setInputValue = () => {
    if (isInputFocused) {
      return;
    }

    const _inputValue = getInputValue({ currentValue, fieldConfig, timeZone });

    // Avoid a cycle dependency & infinite loop
    if (_inputValue !== undefined && _inputValue !== inputValue) {
      inputValue = _inputValue;
    }
  };

  /**
   * Update {@link currentValue} based on {@link inputValue}.
   */
  const setCurrentValue = () => {
    const _currentValue = getCurrentValue({ inputValue, currentValue, fieldConfig, timeZone });

    // Avoid a cycle dependency & infinite loop
    if (
      _currentValue !== undefined &&
      _currentValue !== currentValue &&
      // Compare the actual date/time: if a user edits an existing entry in a different location
      // than where it was originally written, `inputValue` and `_currentValue` may shift to the
      // current timezone, but the epoch won’t change. Don’t update `currentValue` in that case.
      Number(getDate(_currentValue, fieldConfig)) !== Number(getDate(currentValue, fieldConfig))
    ) {
      currentValue = _currentValue;
    }
  };

  $effect(() => {
    // Keep the displayed value in sync with the stored entry value.
    void [currentValue];

    untrack(() => {
      setInputValue();
    });
  });

  $effect(() => {
    // Only update currentValue when inputValue changes (not when timezone changes)
    void [inputValue];

    untrack(() => {
      setCurrentValue();
    });
  });

  /** Date part (`YYYY-MM-DD`) of the current input value, if any. */
  const datePart = $derived(type === 'time' ? '' : (inputValue?.split('T')[0] ?? ''));
  /** Time part (`HH:mm`) of the current input value, if any. */
  const timePart = $derived(
    type === 'time' ? inputValue.slice(0, 5) : (inputValue?.split('T')[1]?.slice(0, 5) ?? ''),
  );
  /** Minute step for the time picker, derived from the field’s `step` option (in seconds). */
  const minuteStep = $derived(
    typeof step === 'number' && Number.isInteger(step / 60) && step / 60 >= 1 && step / 60 <= 30
      ? step / 60
      : 1,
  );

  /**
   * Update the date part of {@link inputValue}, preserving the time part. If the field also has a
   * time and no time is set yet, fall back to the current time.
   * @param {string} newDate New date in `YYYY-MM-DD` format.
   */
  const setDatePart = (newDate) => {
    if (type === 'date') {
      inputValue = newDate;
    } else {
      const fallbackTime = getCurrentDateTime(fieldConfig, timeZone).split('T')[1] ?? '00:00';

      inputValue = `${newDate}T${timePart || fallbackTime}`;
    }
  };

  /**
   * Update the time part of {@link inputValue}, preserving the date part. If the field also has a
   * date and no date is set yet, fall back to the current date.
   * @param {string} newTime New time in `HH:mm` format.
   */
  const setTimePart = (newTime) => {
    if (type === 'time') {
      inputValue = newTime;
    } else {
      const fallbackDate = getCurrentDateTime(fieldConfig, timeZone).split('T')[0];

      inputValue = `${datePart || fallbackDate}T${newTime}`;
    }
  };

  /**
   * Handle input focus event.
   */
  const handleFocus = () => {
    isInputFocused = true;
  };

  /**
   * Handle input blur event - sync values when user finishes editing.
   */
  const handleBlur = () => {
    isInputFocused = false;
    // After losing focus, ensure inputValue is synced with currentValue
    setInputValue();
  };
</script>

<div role="none">
  <input
    {...{ type, min, max, step }}
    bind:value={inputValue}
    {readonly}
    aria-readonly={readonly}
    aria-required={required}
    aria-invalid={invalid}
    aria-labelledby="{fieldId}-label"
    aria-errormessage="{fieldId}-error"
    onfocus={handleFocus}
    onblur={handleBlur}
  />
  {#if !readonly && type !== 'time'}
    <Button
      iconic
      variant="ghost"
      aria-label={_('date_picker.select_date')}
      popupPosition="bottom-left"
    >
      {#snippet startIcon()}
        <Icon name="calendar_month" />
      {/snippet}
      {#snippet popup()}
        <DatePicker value={datePart} {min} {max} onSelect={setDatePart} />
      {/snippet}
    </Button>
  {/if}
  {#if !readonly && type !== 'date'}
    <Button
      iconic
      variant="ghost"
      aria-label={_('date_picker.select_time')}
      popupPosition="bottom-left"
    >
      {#snippet startIcon()}
        <Icon name="schedule" />
      {/snippet}
      {#snippet popup()}
        <TimePicker value={timePart} {minuteStep} onSelect={setTimePart} />
      {/snippet}
    </Button>
  {/if}
  {#if !readonly}
    <Button
      variant="tertiary"
      label={_(dateOnly ? 'today' : 'now')}
      onclick={() => {
        inputValue = getCurrentDateTime(fieldConfig, timeZone);
      }}
    />
  {/if}
  {#if !readonly && !required}
    <Button
      variant="tertiary"
      label={_('clear')}
      disabled={!currentValue}
      onclick={() => {
        currentValue = '';
      }}
    />
  {/if}
</div>

{#if singleCustomTimeZone}
  <div role="none" class="timezone">
    {getTimeZoneLabel(singleCustomTimeZone, getDate(currentValue, fieldConfig))}
  </div>
{:else if utc}
  <div role="none" class="timezone">UTC</div>
{/if}

<style>
  div {
    display: flex;
    align-items: center;
  }

  .timezone {
    margin: 4px 8px 0;
    color: var(--sui-secondary-foreground-color);
    font-size: var(--sui-font-size-small);
    white-space: nowrap;
  }
</style>
