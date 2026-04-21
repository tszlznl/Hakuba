<script lang="ts">
	import classname from 'clsx';
	import { DOMAIN } from '$lib/constants';
	import type Page from '../types/page';
	let clazz: string | undefined = undefined;
	let pages: Page[];
	export { clazz as class, pages };
</script>

<nav class="z-10">
	<ul
		class={classname(
			'child:px-1 first:child:pl-0 last:child:pr-0 -ml-2 flex flex-row divide-x divide-dotted leading-4 [&_a]:px-2 [&_a]:py-4',
			clazz
		)}
	>
		<li>
			<a data-sveltekit-preload-data href="/">Home</a>
		</li>
		{#each pages || [] as { title, path } (title)}
			<li>
				<a data-sveltekit-preload-data href={`/${path || title?.toLowerCase()}`}>{title}</a>
			</li>
		{/each}
		{#if DOMAIN}
			<li>
				<a data-sveltekit-preload-data target="_blank" href="/atom.xml">Feed</a>
			</li>
		{/if}
	</ul>
</nav>
