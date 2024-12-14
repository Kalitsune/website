<script lang="ts">
	import * as env from '$env/static/public';

	import SocialCardArray from '$lib/components/SocialCardArray.svelte';
	import Status from '$lib/components/Status.svelte';
	import { theme } from '$lib/themes';

	let dark_theme: boolean;

	theme.subscribe((value) => {
		dark_theme = value === 'dark';
	});
	
  $: innerWidth = 0;
  $: buttonStyle = innerWidth > 768 ? "height: 3.5rem" : "";
</script>

<svelte:window bind:innerWidth />

<div>
  <Status username="kalitsune"/>
	<img src="https://avatars.githubusercontent.com/u/74075397?v=4" alt="My pfp!" />
	<h1>{env.PUBLIC_NAME}</h1>
	<p class="greetings">Hello! Welcome to my website!</p>
	{#key dark_theme}
		<SocialCardArray
			socials={[
				{
					url: 'https://github.com/Kalitsune',
					logo: 'GitHub-Logo.svg',
					text: 'GitHub',
					alt: 'Github',
					back_color: '#000',
					text_color: '#fff',
          style: buttonStyle,
				},
				{
					url: 'https://stackoverflow.com/users/15751382/kalitsune',
					logo: 'https://www.vectorlogo.zone/logos/stackoverflow/stackoverflow-icon.svg',
					text: '<p><span style="font-weight: var(--font-weight-5);">stack</span>overflow</p>',
					alt: 'StackOverflow',
					back_color: 'var(--surface-2)',
          style: buttonStyle,
				},
				{
					url: 'mailto:fanny@kalitsune.net',
					logo: 'icon:Mail',
					text: `Email Me!`,
					alt: 'Email',
					back_color: '#0078e5',
					text_color: '#fff',
          style: buttonStyle,
				},
				{
					url: 'https://discord.gg/s2TrpTD6C2',
					logo: 'https://assets-global.website-files.com/6257adef93867e50d84d30e2/653714c1f22aef3b6921d63d_636e0a6ca814282eca7172c6_icon_clyde_white_RGB.svg',
					text: '@kalitsune.net',
					alt: 'Discord',
					back_color: '#5865F2',
					text_color: '#fff',
          style: buttonStyle,
				},
				{
					url: 'https://hub.docker.com/u/kalithekitsune',
					logo: dark_theme ? 'Docker-Logo-White.svg' : 'Docker-Logo.svg',
					text: 'Dockerhub',
					alt: 'DockerHub',
					back_color: dark_theme ? '#00084D' : '#E5F2FC',
					text_color: dark_theme ? '#fff' : '#1D63ED',
          style: buttonStyle,
				},
				{
					url: 'https://www.instagram.com/kalitsune',
					logo: 'Instagram-Logo.svg',
					text: 'Instagram',
					alt: 'Instagram',
					back_color: 'linear-gradient(-135deg, #1400c8, #b900b4, #f50000)',
					text_color: '#fff',
          style: buttonStyle,
				},
        {
					url: 'https://www.instagram.com/kalitsune_cosplay',
					logo: 'Instagram-Logo.svg',
					text: 'Cosplays',
					alt: 'Instagram',
					back_color: 'linear-gradient(-135deg, #f17b41, #e05ba2, #cd4bc9)',
					text_color: '#fff',
          style: buttonStyle,
				},
			]}
			style="width: {innerWidth < 768 ? "fit-content" : "var(--size-15)"};"
		/>
	{/key}
	<p>Thank you for your time!</p>
</div>

<style>
	div {
		display: flex;
		flex-direction: column;
		justify-content: center;
    align-items: center;
		gap: var(--size-7);
	}

  p {
    font-size: var(--font-size-fluid-1);
    text-align: center;
  }

	img {
		border-radius: var(--radius-round);
		width: var(--size-12);
	}
</style>
