<script>
  import PhotoGallery from './lib/PhotoGallery.svelte';
</script>

<main>
  <h1>My Photo Gallery</h1>
  <PhotoGallery />
</main>

<style>
  main {
    width: 100%;
    min-height: 100vh;
    background-color: #f5f5f5;
  }

  h1 {
    text-align: center;
    padding: 2rem;
    color: #333;
    font-size: 2.5rem;
    margin: 0;
  }
</style>