<!--
  @component
  Implement the preview for the File and Image field types.
  @see https://decapcms.org/docs/widgets/#File
  @see https://decapcms.org/docs/widgets/#Image
  @see https://sveltiacms.app/en/docs/fields/file
  @see https://sveltiacms.app/en/docs/fields/image
-->
<script>
  import FilePreviewItem from '$lib/components/contents/details/fields/file/file-preview-item.svelte';
  import ImageSlideshowPreview from '$lib/components/contents/details/fields/file/image-slideshow-preview.svelte';
  import { isMultiple } from '$lib/services/integrations/media-libraries/shared';

  /**
   * @import { FieldPreviewProps } from '$lib/types/private';
   * @import { MediaField } from '$lib/types/public';
   */

  /**
   * @typedef {object} Props
   * @property {MediaField} fieldConfig Field configuration.
   * @property {string | string[] | undefined} currentValue Field value.
   */

  /** @type {FieldPreviewProps & Props} */
  let {
    /* eslint-disable prefer-const */
    locale,
    typedKeyPath,
    fieldConfig,
    currentValue,
    /* eslint-enable prefer-const */
  } = $props();

  /** Whether to render multiple images as a site-style slideshow (fork-specific option). */
  const slideshow = $derived(
    fieldConfig.widget === 'image' &&
      !!(/** @type {{ preview_slideshow?: boolean }} */ (fieldConfig).preview_slideshow),
  );
</script>

{#if isMultiple(fieldConfig)}
  {#if Array.isArray(currentValue)}
    {#if slideshow}
      <ImageSlideshowPreview {locale} {typedKeyPath} {fieldConfig} {currentValue} />
    {:else}
      {#each currentValue as value, index (`${value}-${index}`)}
        <FilePreviewItem {value} {fieldConfig} {typedKeyPath} />
      {/each}
    {/if}
  {/if}
{:else if typeof currentValue === 'string' && currentValue}
  <FilePreviewItem value={currentValue} {fieldConfig} {typedKeyPath} />
{/if}
