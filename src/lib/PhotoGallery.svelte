<script>
  import { fade, scale, crossfade } from "svelte/transition";
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
  let currentHeaderImage = null;
  let headerImages = [];
  let headerImageTimer;

  const [send, receive] = crossfade({
    duration: (d) => d * 0.5,
    fallback(node, params) {
      return fade(node, { duration: 400 });
    },
  });

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

  // Function to get random image
  function getRandomImage() {
    if (headerImages.length === 0) return null;
    const randomIndex = Math.floor(Math.random() * headerImages.length);
    const image = headerImages[randomIndex];
    // Preload the full resolution version
    preloadImage(image.full);
    return image;
  }

  // Function to update header image
  function updateHeaderImage() {
    let newImage;
    do {
      newImage = getRandomImage();
    } while (newImage === currentHeaderImage && headerImages.length > 1);
    currentHeaderImage = newImage;
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

    // Filter for landscape images (aspect ratio > 1.3)
    headerImages = processedPhotos.filter((photo) => {
      const img = new Image();
      img.src = photo.min;
      // Preload the full resolution version right away
      preloadImage(photo.full);
      return img.width / img.height > 1.3;
    });

    // Set initial image
    updateHeaderImage();

    // Start rotation timer
    headerImageTimer = setInterval(updateHeaderImage, 7000); // Increased duration to allow for loading

    return () => {
      clearInterval(headerImageTimer);
    };
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

<div class="header">
  <div class="header-photo-container">
    {#if currentHeaderImage}
      {#key currentHeaderImage.full}
        <div
          class="photo-wrapper"
          in:receive={{ key: currentHeaderImage.full }}
          out:send={{ key: currentHeaderImage.full }}
        >
          <img
            src={loadedImages.has(currentHeaderImage.full)
              ? currentHeaderImage.full
              : currentHeaderImage.min}
            alt=""
            class="header-photo"
          />
        </div>
      {/key}
      <div class="photo-overlay" />
    {/if}
  </div>
</div>

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
              <div class="photo-hover-overlay">
                <span class="view-text">View</span>
              </div>
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
      transition:fade={{ duration: 400 }}
    >
      <img
        src={loadedImages.has(selectedPhoto.full)
          ? selectedPhoto.full
          : selectedPhoto.min}
        alt=""
        transition:scale={{ duration: 400 }}
      />
      <button class="close-button" on:click={handleClosePhoto}>×</button>
    </div>
  {/if}
</div>

<style>
  .header {
    width: 100%;
    position: relative;
    margin-bottom: 3rem;
  }

  .header-photo-container {
    width: 100%;
    height: 70vh;
    min-height: 400px;
    max-height: 800px;
    overflow: hidden;
    position: relative;
    background-color: rgb(26, 26, 26);
  }

  .photo-wrapper {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 1;
  }

  .header-photo {
    width: 100%;
    height: 100%;
    object-fit: cover;
    object-position: center;
  }

  .photo-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: linear-gradient(
      to bottom,
      rgba(26, 26, 26, 0.4) 0%,
      rgba(26, 26, 26, 0.2) 50%,
      rgb(26, 26, 26) 100%
    );
    pointer-events: none;
    z-index: 2;
  }

  .gallery {
    width: 100%;
    max-width: 1400px;
    margin: 0 auto;
    padding: 1rem;
  }

  .folder-section {
    margin-bottom: 4rem;
    opacity: 0;
    animation: fadeIn 0.8s ease-out forwards;
  }

  .folder-section h2 {
    margin: 2rem 1rem;
    font-size: 2rem;
    color: var(--color-text);
    font-weight: 600;
    letter-spacing: -0.02em;
  }

  .grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
    gap: 2rem;
    padding: 1rem;
  }

  .photo-container {
    aspect-ratio: 16/9;
    cursor: pointer;
    overflow: hidden;
    border-radius: 0;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
    transition: all 0.3s ease;
    position: relative;
    background-color: var(--color-bg);
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .photo-container img {
    width: 100%;
    height: 100%;
    object-fit: contain;
    background-color: var(--color-bg);
  }

  .photo-container:hover {
    transform: translateY(-5px);
    box-shadow: 0 8px 30px rgba(0, 0, 0, 0.4);
  }

  .photo-hover-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 0;
    transition: opacity 0.3s ease;
  }

  .photo-container:hover .photo-hover-overlay {
    opacity: 1;
  }

  .view-text {
    color: white;
    font-size: 1.2rem;
    font-weight: 600;
    letter-spacing: 0.1em;
    text-transform: uppercase;
  }

  .fullscreen {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.95);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
  }

  .fullscreen img {
    max-width: 95%;
    max-height: 95vh;
    object-fit: contain;
    border-radius: 8px;
    box-shadow: 0 8px 40px rgba(0, 0, 0, 0.5);
  }

  .close-button {
    position: absolute;
    top: 2rem;
    right: 2rem;
    width: 3rem;
    height: 3rem;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.1);
    border: none;
    color: white;
    font-size: 2rem;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s ease;
  }

  .close-button:hover {
    background: rgba(255, 255, 255, 0.2);
    transform: rotate(90deg);
  }

  @keyframes fadeIn {
    from {
      opacity: 0;
      transform: translateY(20px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }
</style>
