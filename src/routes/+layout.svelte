<script>
	import 'open-props/style';
	import 'open-props/normalize';
	import 'open-props/buttons';

	import { Home, NotepadText, KeyRound } from 'lucide-svelte';

	import '../app.css';

	import Header from '$lib/components/Header.svelte';
	import Footer from '$lib/components/Footer.svelte';
</script>

<div class="layout">
	<Header
		links={[
			{ name: 'Home', path: '/', icon: Home },
			{ name: 'Blog', path: '/blog', icon: NotepadText },
		]}
	/>

	<div class="main">
		<slot />
	</div>

	<Footer />
</div>

<style>
	.layout {
		max-inline-size: 1440px;
		margin-inline: auto;
		padding-inline: var(--size-7);
		display: flex;
		flex-direction: column;
		min-height: 100vh;
	}

	.main {
		padding: var(--size-9) 0;
		max-inline-size: var(--size-content-3);
		margin-inline: auto;
		flex: 1;
    max-width: 85vw;
	}

	@media (min-width: 1440px) {
		.layout {
			padding-inline: var(--size-4);
		}
	}
</style>
