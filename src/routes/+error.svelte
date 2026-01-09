<script lang="ts">
	import { page } from '$app/stores';
</script>

<svelte:head>
	<title>Error {$page.status} | Code Formatter</title>
</svelte:head>

<div class="min-h-screen flex items-center justify-center p-4">
	<div class="text-center max-w-md">
		<h1 class="text-6xl font-bold neon-primary mb-4">{$page.status}</h1>
		<p class="text-xl text-gray-400 mb-6 terminal-prompt">
			{#if $page.status === 404}
				Page not found
			{:else if $page.status === 500}
				Internal server error
			{:else}
				Something went wrong
			{/if}
		</p>
		<p class="text-gray-500 mb-8">
			{$page.error?.message || 'An unexpected error occurred.'}
		</p>
		<a
			href="/"
			class="btn-cyber btn-primary px-6 py-3 text-sm rounded-lg uppercase tracking-wider font-bold inline-block"
		>
			[Return Home]
		</a>
	</div>
</div>
