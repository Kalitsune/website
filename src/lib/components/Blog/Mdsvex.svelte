<script lang="ts" context="module">
	import img from '$lib/components/Blog/LazyImg.svelte';
	import Mermaid from '$lib/components/Blog/Mermaid.svelte';
  import table from '$lib/components/Blog/ScrollingTable.svelte';

	export { img, Mermaid, table };
</script>

<slot />
