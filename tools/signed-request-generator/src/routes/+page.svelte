<script lang="ts">
	import Form from '../components/Form.svelte';
	import Results from '../components/Results.svelte';

	let encodedRequest = '';
	let requestJson = '';
	let error = '';
	let chainEnv = '';
</script>

<div class="mb-4">
	<Form bind:encodedRequest bind:requestJson bind:error bind:chainEnv />
</div>

{#if error}
	<h2 class="text-red">Error</h2>
	<p class="text-sm">{error}</p>
{:else}
	<Results {encodedRequest} {requestJson} {chainEnv} />
{/if}
