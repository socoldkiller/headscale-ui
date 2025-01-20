<script lang="ts">
	import { Device } from '$lib/common/classes';
	import { slide, fade } from 'svelte/transition';
	import { get } from 'svelte/store';
	import { alertStore } from '$lib/common/stores';
	import { getDevices } from '$lib/common/apiFunctions.svelte';
	
	export let node: Device;

	let editing = false;
	let error: string | null = null;
	let inputValue = '';

	function validateIPs(ips: string[]): boolean {
		const ipPattern = /^((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/;
		return ips.every(ip => ipPattern.test(ip));
	}


	async function saveChanges() {
		const ips = inputValue.split(',').map(ip => ip.trim());
		
		if (!validateIPs(ips)) {
			error = 'Invalid IP address format';
			return;
		}

		try {
			// Update each IP address individually
			for (const ip of ips) {
				const response = await fetch(`/api/v1/node/${node.id}/ip/${ip}`, {
					method: 'PUT',
					headers: {
						'Content-Type': 'application/json',
						'Authorization': `Bearer ${localStorage.getItem('headscaleAPIKey')}`
					},
					body: JSON.stringify({
						node_id: node.id,
						new_ip: ip
					})
				});

				if (!response.ok) {
					throw new Error(await response.text());
				}
			}

			// Update device locally
			node.ipAddresses = ips;
			editing = false;
			error = null;
			
			// Refresh devices list
			getDevices();
		} catch (err: unknown) {
			if (err instanceof Error) {
				$alertStore = err.message;
			} else {
				$alertStore = 'An unknown error occurred';
			}
			error = 'Failed to update IP addresses';
		}
	}

	function startEditing() {
		inputValue = node.ipAddresses.join(', ');
		editing = true;
		error = null;
	}
</script>

<div>
	{#if !editing}
		{node.ipAddresses.join(', ')}
		<button
			on:click={startEditing}
			type="button"
			class="ml-2"
			><svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 inline flex-shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
				<path stroke-linecap="round" stroke-linejoin="round" d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
			</svg></button
		>
	{:else}
		<div class="flex flex-col gap-1">
			<div class="flex gap-1">
				<input
					type="text"
					bind:value={inputValue}
					class="card-select mr-3 {error ? 'input-error' : ''}"
					on:keydown={(e) => {
						if (e.key === 'Enter') {
							saveChanges();
						}
					}}
				/>
				<button type="button" in:fade|global on:click={saveChanges}
					><svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 inline flex-shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
						<path stroke-linecap="round" stroke-linejoin="round" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z" />
					</svg></button
				>
				<button type="button" in:fade|global on:click|stopPropagation={() => (editing = false)}
					><svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 inline flex-shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
						<path stroke-linecap="round" stroke-linejoin="round" d="M10 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z" />
					</svg></button
				>
			</div>
			{#if error}
				<div class="text-error text-xs">{error}</div>
			{/if}
		</div>
	{/if}
</div>
