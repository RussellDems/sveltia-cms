<!--
  @component
  A site-matching slideshow preview for multi-value Image fields, replacing the default stacked
  image list. Enabled with the custom `preview_slideshow: true` field option. Replicates the
  RussellDems `post.html` gallery: sliding track, prev/next arrows, photo counter, thumbnail strip,
  and 5-second auto-advance that pauses on hover.

  If the custom `featured_toggle_field` option names a sibling Boolean field, the front matter
  logic from the site template is mirrored: when the toggle is OFF, the first image is treated as
  the featured/cover image and only the remaining images appear in the slideshow; when it’s ON,
  all images appear in the slideshow. A single remaining image renders as a plain cover image,
  matching the template’s fallback.
-->
<script>
  import { untrack } from 'svelte';

  import { getMediaFieldURL } from '$lib/services/assets/info';
  import { entryDraft } from '$lib/services/contents/draft';

  /**
   * @import { FieldPreviewProps } from '$lib/types/private';
   * @import { MediaField } from '$lib/types/public';
   */

  /**
   * @typedef {object} Props
   * @property {MediaField} fieldConfig Field configuration.
   * @property {string[]} currentValue Field value: list of image paths.
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

  /** Interval between automatic slide advances, matching the site template. */
  const AUTOPLAY_INTERVAL = 5000;

  /** Name of the sibling Boolean field that toggles the featured image split. */
  const toggleFieldName = $derived(
    /** @type {{ featured_toggle_field?: string }} */ (fieldConfig).featured_toggle_field,
  );
  /** Whether the “gallery with featured image” toggle is on. */
  const toggleOn = $derived(
    !!toggleFieldName && !!$entryDraft?.currentValues[locale]?.[toggleFieldName],
  );

  const images = $derived(currentValue.filter((value) => !!value?.trim()));
  /**
   * Mirror the Liquid logic in `post.html`: `featured` is the first image and `slides` the rest,
   * unless the toggle is on, in which case all images are slides and there is no featured image.
   */
  const slides = $derived(toggleOn ? images : images.slice(1));
  const featured = $derived(toggleOn ? undefined : images[0]);

  /**
   * Resolved blob/public URLs for {@link images}, keyed by path.
   * @type {Record<string, string>}
   */
  let urlMap = $state({});
  /** Index of the current slide. */
  let currentIndex = $state(0);
  /** Whether the pointer is over the slider, pausing the auto-advance. */
  let hovering = $state(false);

  const entry = $derived($entryDraft?.originalEntry);
  const collectionName = $derived($entryDraft?.collectionName ?? '');
  const fileName = $derived($entryDraft?.fileName);

  $effect(() => {
    void [images];

    untrack(async () => {
      urlMap = Object.fromEntries(
        await Promise.all(
          images.map(async (value) => [
            value,
            await getMediaFieldURL({
              value,
              entry,
              collectionName,
              fileName,
              fieldConfig,
              typedKeyPath,
            }),
          ]),
        ),
      );
    });
  });

  $effect(() => {
    // Clamp the index when images are removed or the toggle changes the slide count
    if (currentIndex >= slides.length) {
      currentIndex = 0;
    }
  });

  $effect(() => {
    // Auto-advance every 5 seconds like the live site, pausing on hover
    if (slides.length > 1 && !hovering) {
      const timer = setInterval(() => {
        currentIndex = (currentIndex + 1) % slides.length;
      }, AUTOPLAY_INTERVAL);

      return () => clearInterval(timer);
    }

    return undefined;
  });

  /**
   * Go to the slide at the given index, wrapping around.
   * @param {number} index New slide index.
   */
  const goTo = (index) => {
    currentIndex = (index + slides.length) % slides.length;
  };
</script>

{#if slides.length > 0}
  <div role="none" class="post-gallery">
    <div
      role="none"
      class="slider"
      onmouseenter={() => {
        hovering = true;
      }}
      onmouseleave={() => {
        hovering = false;
      }}
    >
      <div role="none" class="track" style:transform="translateX(-{currentIndex * 100}%)">
        {#each slides as image, index (`${image}-${index}`)}
          <div role="none" class="slide">
            {#if urlMap[image]}
              <img src={urlMap[image]} alt="Photo {index + 1}" loading="lazy" />
            {/if}
          </div>
        {/each}
      </div>
      {#if slides.length > 1}
        <button
          type="button"
          class="nav prev"
          aria-label="Previous photo"
          onclick={() => {
            goTo(currentIndex - 1);
          }}
        >
          &#8592;
        </button>
        <button
          type="button"
          class="nav next"
          aria-label="Next photo"
          onclick={() => {
            goTo(currentIndex + 1);
          }}
        >
          &#8594;
        </button>
        <div role="none" class="counter">{currentIndex + 1} / {slides.length}</div>
      {/if}
    </div>
    {#if slides.length > 1}
      <div role="none" class="thumbs">
        {#each slides as image, index (`${image}-${index}`)}
          <button
            type="button"
            class="thumb"
            class:active={index === currentIndex}
            aria-label="Photo {index + 1}"
            onclick={() => {
              goTo(index);
            }}
          >
            {#if urlMap[image]}
              <img src={urlMap[image]} alt="Photo {index + 1}" loading="lazy" />
            {/if}
          </button>
        {/each}
      </div>
    {/if}
  </div>
{:else if featured && urlMap[featured]}
  <p>
    <img class="featured" src={urlMap[featured]} alt="Featured" />
  </p>
{/if}

<style>
  /* Replicate the live site’s `.post-gallery` styles (components-additions.css) so the preview
     matches what Jekyll generates. Site design tokens are inlined because the preview pane
     doesn’t load the site’s stylesheets. */
  .post-gallery {
    margin: 8px 0;
    border: 1px solid #e8edf5;
    border-radius: 4px;
    overflow: hidden;
  }

  .slider {
    position: relative;
    overflow: hidden;
    background: #000;
  }

  .track {
    display: flex;
    transition: transform 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .slide {
    min-width: 100%;
    background: #000;
    display: flex;
    align-items: center;
    justify-content: center;
    min-height: 200px;

    img {
      width: 100%;
      max-height: 400px;
      object-fit: contain;
      display: block;
    }
  }

  .nav {
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    background: rgba(0, 0, 0, 0.45);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: white;
    width: 36px;
    height: 36px;
    border-radius: 50%;
    cursor: pointer;
    font-size: 0.9rem;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: background 0.22s cubic-bezier(0.4, 0, 0.2, 1);
    z-index: 2;

    &:hover {
      background: rgba(0, 0, 0, 0.7);
    }

    &.prev {
      left: 0.75rem;
    }

    &.next {
      right: 0.75rem;
    }
  }

  .counter {
    position: absolute;
    bottom: 0.6rem;
    right: 0.75rem;
    background: rgba(0, 0, 0, 0.5);
    color: white;
    font-size: 0.75rem;
    padding: 0.2rem 0.5rem;
    border-radius: 99px;
    z-index: 2;
  }

  .thumbs {
    display: flex;
    gap: 0.4rem;
    padding: 0.5rem;
    background: #f5f4f0;
    flex-wrap: wrap;
  }

  .thumb {
    width: 64px;
    height: 48px;
    border-radius: 4px;
    overflow: hidden;
    cursor: pointer;
    border: 2px solid transparent;
    padding: 0;
    background: none;
    transition: border-color 0.22s cubic-bezier(0.4, 0, 0.2, 1);
    flex-shrink: 0;

    &.active {
      border-color: #e8a020;
    }

    img {
      width: 100%;
      height: 100%;
      object-fit: cover;
      display: block;
    }
  }

  .featured {
    width: 100%;
    border-radius: 4px;
    max-height: 480px;
    object-fit: cover;
  }
</style>
