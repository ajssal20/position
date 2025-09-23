<script>
	import favicon from '$lib/assets/favicon.svg';
	import{getDistance } from 'geolib';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

{@render children?.()}

