<script lang="ts">
	import favicon from '$lib/assets/favicon.svg';
	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;700&family=JetBrains+Mono:wght@400;600&family=Anton&family=Syne:wght@700;800&display=swap" rel="stylesheet" />
	<title>FLUX — Kinetic Sound Arcade</title>
	<meta name="description" content="FLUX — a physics-and-sound arcade. Throw, pluck, splatter. Svelte 5 + Rapier + Web Audio + shaders." />
</svelte:head>

{@render children()}

<style>
	:global(*) { box-sizing: border-box; }
	:global(html), :global(body) { margin:0; min-height:100%; background:#07090f; color:#f2f5ff; }
</style>
