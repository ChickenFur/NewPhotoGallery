<script>
  import { fade, scale } from 'svelte/transition';
  import { flip } from 'svelte/animate';
  import { onMount } from 'svelte';

  // Get all image files from the photos directory
  const imageFiles = import.meta.glob('/src/assets/photos/*', { 
    eager: true,
    query: '?url',
    import: 'default'
  });
  
  // Process image files to create groups of related images
  const photoGroups = Object.keys(imageFiles)
    .filter(path => !path.includes('.min.') && !path.includes('.placeholder.'))
    .map(path => {
      const basePath = path.substring(0, path.lastIndexOf('.'));
      const extension = path.substring(path.lastIndexOf('.'));
      return {
        full: imageFiles[path],
        min: imageFiles[`${basePath}.min${extension}`] || imageFiles[path],
        placeholder: imageFiles[`${basePath}.placeholder${extension}`] || imageFiles[path],
        fileName: path.split('/').pop()
      };
    })
    .filter(group => group.min && group.placeholder); // Ensure all required versions exist

  let selectedPhoto = null;
  let loadedImages = new Set();

  function preloadImage(src) {
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.onload = () => {
        loadedImages.add(src);
        loadedImages = loadedImages; // Trigger reactivity
        resolve();
      };
      img.onerror = reject;
      img.src = src;
    });
  }

  function getDisplayImage(photoGroup) {
    if (loadedImages.has(photoGroup.min)) {
      return photoGroup.min;
    }
    return photoGroup.placeholder;
  }

  onMount(() => {
    // Preload min versions
    photoGroups.forEach(group => {
      preloadImage(group.min);
    });
  });

  async function handlePhotoClick(photoGroup) {
    selectedPhoto = photoGroup;
    // Preload full resolution image when photo is clicked
    if (!loadedImages.has(photoGroup.full)) {
      await preloadImage(photoGroup.full);
    }
  }
</script>

<div class="gallery">
  {#if !selectedPhoto}
    <div class="grid">
      {#each photoGroups as photoGroup, i (photoGroup.full)}
        <div
          class="photo-container"
          animate:flip={{ duration: 300 }}
          on:click={() => handlePhotoClick(photoGroup)}
          on:keydown={(e) => e.key === 'Enter' && handlePhotoClick(photoGroup)}
          role="button"
          tabindex="0"
        >
          <img
            src={getDisplayImage(photoGroup)}
            alt=""
            transition:scale={{ duration: 300 }}
            loading="lazy"
          />
          <span class="sr-only">View {photoGroup.fileName}</span>
        </div>
      {/each}
    </div>
  {:else}
    <div
      class="fullscreen"
      on:click={() => selectedPhoto = null}
      on:keydown={(e) => e.key === 'Escape' && (selectedPhoto = null)}
      role="button"
      tabindex="0"
      transition:fade={{ duration: 200 }}
    >
      <img
        src={loadedImages.has(selectedPhoto.full) ? selectedPhoto.full : selectedPhoto.min}
        alt=""
        transition:scale={{ duration: 300 }}
      />
      <span class="sr-only">Close {selectedPhoto.fileName}</span>
    </div>
  {/if}
</div>

<style>
  .gallery {
    width: 100%;
    max-width: 1200px;
    margin: 0 auto;
    padding: 1rem;
  }

  .grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 1rem;
    padding: 1rem;
  }

  .photo-container {
    aspect-ratio: 4/3;
    cursor: pointer;
    overflow: hidden;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    transition: transform 0.2s ease;
    position: relative;
  }

  .photo-container:hover {
    transform: scale(1.02);
  }

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .fullscreen {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.9);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    cursor: pointer;
  }

  .fullscreen img {
    max-width: 90%;
    max-height: 90vh;
    object-fit: contain;
    border-radius: 4px;
  }

  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
  }
</style>