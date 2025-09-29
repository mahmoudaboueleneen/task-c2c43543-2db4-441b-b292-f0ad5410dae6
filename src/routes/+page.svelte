<script lang="ts">
  import Map from '$lib/Map.svelte';
</script>

<svelte:head>
  <title>Delivery Location</title>
  <meta name="description" content="Confirm your delivery location on the map." />
</svelte:head>

<main>
  <Map />
</main>

<style>
  main {
    height: 100vh;
    width: 100vw;
  }
</style>
