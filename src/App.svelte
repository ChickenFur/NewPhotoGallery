<script>
  import PhotoGallery from "./lib/PhotoGallery.svelte";
</script>

<svelte:head>
  <link
    href="https://fonts.googleapis.com/css2?family=Montserrat:wght@400;600;800&display=swap"
    rel="stylesheet"
  />
</svelte:head>

<main>
  <h1>BitByBit Adventure</h1>
  <PhotoGallery />
</main>

<style>
  main {
    width: 100%;
    min-height: 100vh;
  }

  h1 {
    text-align: center;
    padding: 2rem;
    color: var(--color-text);
    font-size: 2.5rem;
    font-weight: 800;
    margin: 0;
    letter-spacing: -0.03em;
    opacity: 0;
    animation: fadeIn 0.8s ease-out forwards;
    animation-delay: 0.2s;
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
