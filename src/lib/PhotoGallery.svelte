<script>
  import { fade, scale } from "svelte/transition";
  import { flip } from "svelte/animate";
  import { onMount, afterUpdate } from "svelte";

  // Get all image files from the photos directory
  const imageFiles = import.meta.glob("/src/assets/images/**/*", {
    eager: true,
    query: "?url",
    import: "default",
  });

  // List of supported image extensions
  const supportedExtensions = [".jpg", ".jpeg", ".png", ".gif", ".webp"];

  // Process image files to create groups of related images
  const processedPhotos = Object.keys(imageFiles)
    .filter((path) => {
      const extension = path.toLowerCase().substring(path.lastIndexOf("."));
      return (
        supportedExtensions.includes(extension) &&
        !path.includes(".min.") &&
        !path.includes(".placeholder.")
      );
    })
    .map((path) => {
      const basePath = path.substring(0, path.lastIndexOf("."));
      const extension = path.substring(path.lastIndexOf("."));
      const pathParts = path.split("/");
      const folderName = pathParts[pathParts.length - 2] || "Uncategorized";

      return {
        full: imageFiles[path],
        min: imageFiles[`${basePath}.min${extension}`] || imageFiles[path],
        fileName: pathParts.pop(),
        folder: folderName,
      };
    })
    .filter((group) => group.min); // Only ensure min version exists

  // Group photos by folder
  const photoGroups = processedPhotos.reduce((groups, photo) => {
    if (!groups[photo.folder]) {
      groups[photo.folder] = [];
    }
    groups[photo.folder].push(photo);
    return groups;
  }, {});

  let selectedPhoto = null;
  let loadedImages = new Set();
  let scrollPosition;

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
    return photoGroup.min; // Always use min version as fallback
  }

  onMount(() => {
    // Handle initial deep link if present
    const hash = window.location.hash.slice(1); // Remove the # symbol
    if (hash) {
      const photoToShow = processedPhotos.find(
        (photo) => photo.fileName.replace(/\.[^/.]+$/, "") === hash
      );
      if (photoToShow) {
        handleOpenPhoto(photoToShow);
      }
    }

    // Preload min versions
    processedPhotos.forEach((group) => {
      preloadImage(group.min);
    });
  });

  async function handlePhotoClick(photoGroup) {
    selectedPhoto = photoGroup;
    // Update URL with the photo filename (without extension)
    const baseFileName = photoGroup.fileName.replace(/\.[^/.]+$/, "");
    window.history.pushState(null, "", `#${baseFileName}`);

    // Preload full resolution image when photo is clicked
    if (!loadedImages.has(photoGroup.full)) {
      await preloadImage(photoGroup.full);
    }
  }

  function handleOpenPhoto(photoGroup) {
    scrollPosition = window.scrollY;
    handlePhotoClick(photoGroup);
  }

  function handleClosePhoto() {
    selectedPhoto = null;
    // Remove the hash from the URL
    window.history.pushState(null, "", window.location.pathname);
    // Wait for the transition to complete before restoring scroll position
    setTimeout(() => {
      window.scrollTo(0, scrollPosition);
    }, 250);
  }
</script>

<div class="gallery">
  {#if !selectedPhoto}
    {#each Object.entries(photoGroups) as [folderName, photos]}
      <div class="folder-section">
        <h2>{folderName}</h2>
        <div class="grid">
          {#each photos as photoGroup, i (photoGroup.full)}
            <div
              class="photo-container"
              animate:flip={{ duration: 300 }}
              on:click={() => handleOpenPhoto(photoGroup)}
              on:keydown={(e) =>
                e.key === "Enter" && handleOpenPhoto(photoGroup)}
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
      </div>
    {/each}
  {:else}
    <div
      class="fullscreen"
      on:click={handleClosePhoto}
      on:keydown={(e) => e.key === "Escape" && handleClosePhoto()}
      role="button"
      tabindex="0"
      transition:fade={{ duration: 200 }}
    >
      <img
        src={loadedImages.has(selectedPhoto.full)
          ? selectedPhoto.full
          : selectedPhoto.min}
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

  .folder-section {
    margin-bottom: 2rem;
  }

  .folder-section h2 {
    margin: 1rem;
    font-size: 1.5rem;
    color: #333;
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
