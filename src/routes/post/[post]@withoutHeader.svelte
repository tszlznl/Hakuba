<script context="module" lang="ts">
	import type { Load } from '@sveltejs/kit';
	import type { Component } from 'svelte';
	import Header from '$lib/components/Header.svelte';
	import type Post from '$lib/types/post';
	import Article from '$lib/components/Article.svelte';
	import Labels from '$lib/components/Labels.svelte';
	import { readableDate } from '$lib/helper/readableDate';
	import { fetchPost } from '$lib/helper/fetchPosts';
	import BackButton from '$lib/components/BackButton.svelte';
	import Giscus from '$lib/components/Giscus.svelte';
	import PageMeta from '$lib/components/PageMeta.svelte';
	import { error } from '@sveltejs/kit';

	export const load: Load = async ({ params }) => {
		try {
			const post = await fetchPost(params.post);
			if (!post) {
				throw error(404, 'Post not found');
			}

			const metadata = post.metadata;

			const timezone = metadata.timezone;

			return {
				...post,
				labels: metadata.labels?.map(({ name }) => name),
				published: readableDate(metadata.published, timezone),
				updated: metadata.updated ? readableDate(metadata.updated, timezone) : undefined
			};
		} catch (e: any) {
			if (e.status === 404) throw e;
			throw error(500, e.message);
		}
	};
</script>

<script lang="ts">
	export let component: Component;
	export let metadata: Post;
	export let labels: string[] | undefined;
	export let published: string | undefined;
	export let updated: string | undefined;
</script>

<PageMeta {metadata} />

<Article lang={metadata.lang}>
	<header class="mb-14 flex flex-col">
		<Header class="group relative">
			<BackButton />
			{metadata.title}
		</Header>
		<div
			class="flex flex-col justify-center [&_*]:!text-sm [&_*]:!font-normal [&_*]:!text-slate-600"
		>
			<div class=" mt-4 self-start border-t border-t-slate-900 pt-2">
				Published: {published}
			</div>
			{#if updated && updated !== published}
				<div>
					Updated: &nbsp;&nbsp;{updated}
				</div>
			{/if}
			<Labels {labels} />
		</div>
	</header>

	<svelte:component this={component} />

	<Giscus config={metadata} />
</Article>
