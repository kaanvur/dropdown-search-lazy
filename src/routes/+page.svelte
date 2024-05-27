<script lang="ts">
	import { onMount, tick } from 'svelte';
	import * as Command from '$lib/components/ui/command/index.js';
	import * as Popover from '$lib/components/ui/popover/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	import { cn } from '$lib/utils.js';
	import * as Avatar from '$lib/components/ui/avatar';
	import viewport from '$lib/useViewportAction';
	import { badgeVariants } from '$lib/components/ui/badge/index.js';
	import { LoaderPinwheel, Check, ChevronsUpDown, X } from 'lucide-svelte';

	interface Character {
		id: number;
		name: string;
		image: string;
		episode: string[];
		loaded: boolean;
		label: string;
		value: string;
	}
	let characters: Character[] = [];
	let selectedCharacters: Character[] = [];
	let open = false;
	let searchQuery = '';
	let errorMessage = '';
	let nextUrl: string | null = null;

	let searchTimeout: number | undefined;
	async function handleSearchInput() {
		errorMessage = '';
		nextUrl = null;
		try {
			clearTimeout(searchTimeout);
			searchTimeout = setTimeout(async () => {
				const response = await fetch(
					`https://rickandmortyapi.com/api/character/?name=${searchQuery}`
				);
				const data = await response.json();
				if (!data.results) {
					return (errorMessage = data.error);
				}
				characters = data.results.map((character: Character) => ({
					...character,
					value: character.id.toString(),
					label: character.name,
					loaded: false
				}));
				if (data.info && data.info.next) {
					nextUrl = data.info.next;
				}
			}, 400);
		} catch (error: any) {
			console.error(error);
		}
	}
	onMount(handleSearchInput);

	async function loadNextPage() {
		try {
			if (!nextUrl) {
				return;
			}
			const response = await fetch(nextUrl);
			const data = await response.json();
			if (!data.results) {
				return (errorMessage = data.error);
			}
			characters = [
				...characters,
				...data.results.map((character: Character) => ({
					...character,
					value: character.name,
					label: character.name,
					loaded: false
				}))
			];
			if (data.info && data.info.next) {
				nextUrl = data.info.next;
			} else {
				nextUrl = null;
			}
		} catch (error: any) {
			console.error(error);
		}
	}
	function removeCharacter(valueToRemove: string) {
		selectedCharacters = selectedCharacters.filter(
			(character) => character.value !== valueToRemove
		);
	}
</script>

<div class="max-w-lg mx-auto grid min-h-screen place-items-center relative combobox-container">
	<Popover.Root bind:open let:ids portal="null">
		<Popover.Trigger asChild let:builder>
			<Button
				builders={[builder]}
				variant="outline"
				role="combobox"
				aria-expanded={open}
				class="w-full h-auto"
			>
				{#if selectedCharacters.length > 0}
					<span class="flex gap-1 flex-wrap items-center">
						{#each selectedCharacters as selectedCharacter}
							<Button
								class={`${badgeVariants()} h-auto`}
								on:click={() => removeCharacter(selectedCharacter.value)}
								>{selectedCharacter.label} <X class="h-4 w-4" />
							</Button>
						{/each}
					</span>
				{:else}
					Select a character
				{/if}
				<ChevronsUpDown class="ml-auto h-4 w-4 shrink-0 opacity-50" />
			</Button>
		</Popover.Trigger>
		<input hidden bind:value={selectedCharacters} name="selectedCharacter" />

		<Popover.Content class="w-full p-0">
			<Command.Root>
				<Command.Input placeholder="Search character..." />
				<Command.Empty>No character found.</Command.Empty>
				<Command.Group class="overflow-y-auto py-1 h-48">
					{#if errorMessage}
						<Command.Item>{errorMessage}</Command.Item>
					{:else}
						{#each characters as character}
							<Command.Item
								class="gap-2"
								value={character.value}
								onSelect={(currentValue) => {
									if (selectedCharacters.some((character) => character.value === currentValue)) {
										removeCharacter(currentValue);
									} else {
										selectedCharacters = [
											...selectedCharacters,
											{ value: currentValue, label: character.label }
										];
									}
								}}
							>
								<Check
									class={cn(
										'h-4 w-4 border rounded',
										selectedCharacters.some(
											(selectedCharacter) => selectedCharacter.value === character.value
										)
											? ''
											: 'text-transparent'
									)}
								/>
								<Avatar.Root>
									<Avatar.Image src={character.image} alt={character.name} />
									<Avatar.Fallback>A</Avatar.Fallback>
								</Avatar.Root>
								<div class="flex align-middle flex-col">
									{character.label}
									<span class="text-sm text-slate-500">({character.episode.length}) Episodes</span>
								</div>
							</Command.Item>
						{/each}
					{/if}
					{#if nextUrl}
						<Command.Item class="flex flex-col justify-center">
							<p use:viewport on:enterViewport={loadNextPage}>load more</p>
							<LoaderPinwheel class="h-18 w-18 animate-spin" />
						</Command.Item>
					{/if}
				</Command.Group>
			</Command.Root>
		</Popover.Content>
	</Popover.Root>
</div>
