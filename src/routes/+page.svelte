<script lang="ts">
	import { onDestroy, onMount } from 'svelte';
	import * as monaco from 'monaco-editor';
	import { Button } from '$lib/components/ui/button/index.js';
	import {
		ArrowRightLeft,
		BrushCleaning,
		SunIcon,
		MoonIcon,
		BadgeCheckIcon,
		BadgeAlert,
		GitCompareArrows
	} from '@lucide/svelte';
	import { resetMode, setMode, mode } from 'mode-watcher';
	import * as DropdownMenu from '$lib/components/ui/dropdown-menu/index.js';
	import { buttonVariants } from '$lib/components/ui/button/index.js';
	import { Badge } from '$lib/components/ui/badge/index.js';

	let editorElement: HTMLDivElement;
	let editor: monaco.editor.IStandaloneDiffEditor;
	let borderColorClass = 'border-accent'; // Initial border color class
	let statusLabelText = '';
	let statusLabelColorClass = '';
	let isTextIdentical = false;

	// Function to update border color and status label based on text comparison
	function updateStatus() {
		const originalModel = editor.getModel()?.original;
		const modifiedModel = editor.getModel()?.modified;

		if (originalModel && modifiedModel) {
			const originalValue = originalModel.getValue();
			const modifiedValue = modifiedModel.getValue();

			// Rimuovi spazi e tabulazioni prima del confronto
			const normalize = (str: string) => str.replace(/[ \t]/g, '');
			isTextIdentical = normalize(originalValue) === normalize(modifiedValue);

			if (isTextIdentical) {
				borderColorClass = 'border-green-200'; // Texts are identical
				statusLabelText = 'Identical';
				statusLabelColorClass = 'text-green-500';
			} else {
				borderColorClass = 'border-red-200'; // Texts are different
				statusLabelText = 'Different';
				statusLabelColorClass = 'text-red-500';
			}
		} else {
			borderColorClass = 'border-accent'; // Fallback if models are not ready
			statusLabelText = '';
			statusLabelColorClass = '';
		}
	}

	onMount(async () => {
		editor = monaco.editor.createDiffEditor(editorElement, {
			automaticLayout: true,
			theme: $mode === 'dark' ? 'vs-dark' : 'vs-light',
			originalEditable: true,
			renderSideBySide: true,
			renderSideBySideInlineBreakpoint: 10,
			wordWrap: 'on'
		});
		editor.setModel({
			original: monaco.editor.createModel('', 'plaintext'),
			modified: monaco.editor.createModel('', 'plaintext')
		});

		// Listen for content changes in both models
		editor.getModel()?.original.onDidChangeContent(() => updateStatus());
		editor.getModel()?.modified.onDidChangeContent(() => updateStatus());

		// Initial check for border color and status
		updateStatus();
	});

	onDestroy(() => {
		editor?.dispose();
	});

	function changeTheme(newMode: string) {
		if (newMode === 'dark') {
			setMode('dark');
		} else if (newMode === 'light') {
			setMode('light');
		} else {
			// 'system' mode
			resetMode();
		}
		monaco.editor.setTheme($mode === 'dark' ? 'vs-dark' : 'vs-light');
	}

	function swapText() {
		const originalModel = editor.getModel()?.original;
		const modifiedModel = editor.getModel()?.modified;

		if (originalModel && modifiedModel) {
			const originalValue = originalModel.getValue();
			const modifiedValue = modifiedModel.getValue();

			originalModel.setValue(modifiedValue);
			modifiedModel.setValue(originalValue);
			// updateStatus(); // No need to call here, onDidChangeContent will trigger it
		}
	}

	function clearText() {
		const originalModel = editor.getModel()?.original;
		const modifiedModel = editor.getModel()?.modified;

		if (originalModel && modifiedModel) {
			originalModel.setValue('');
			modifiedModel.setValue('');
			// updateStatus(); // No need to call here, onDidChangeContent will trigger it
		}
	}
</script>

<div
	class="bg-base-100 dark:bg-base-900 flex h-screen w-full flex-col items-center justify-between"
>
	<div class="flex h-[80px] w-full items-center justify-around text-lg font-semibold">
		<div>
			<GitCompareArrows class="inline-block h-6 w-6 mr-2" />
			Text Comparison Tool
		</div>
		<DropdownMenu.Root>
			<DropdownMenu.Trigger class={buttonVariants({ variant: 'outline', size: 'icon' })}>
				<SunIcon
					class="h-[1.2rem] w-[1.2rem] scale-100 rotate-0 transition-all dark:scale-0 dark:-rotate-90"
				/>
				<MoonIcon
					class="absolute h-[1.2rem] w-[1.2rem] scale-0 rotate-90 transition-all dark:scale-100 dark:rotate-0"
				/>
				<span class="sr-only">Toggle theme</span>
			</DropdownMenu.Trigger>
			<DropdownMenu.Content align="end">
				<DropdownMenu.Item onclick={() => changeTheme('light')}>Light</DropdownMenu.Item>
				<DropdownMenu.Item onclick={() => changeTheme('dark')}>Dark</DropdownMenu.Item>
				<DropdownMenu.Item onclick={() => changeTheme('system')}>System</DropdownMenu.Item>
			</DropdownMenu.Content>
		</DropdownMenu.Root>
	</div>
	<div
		class="mb-0 flex h-[calc(100svh-80px)] w-full flex-col rounded border-2 shadow-lg md:m-[20px] md:mt-0 md:mb-0 md:h-[calc(100svh-100px)] md:w-[calc(100%-40px)] {isTextIdentical
			? 'border-green-200 dark:border-green-700'
			: 'border-red-200 dark:border-red-800'}"
	>
		<div class="flex-grow" bind:this={editorElement}></div>
	</div>
	<div class="flex h-[80px] w-full items-center justify-center gap-2">
		{#if statusLabelText}
			<Badge
				variant="secondary"
				class="text-white {isTextIdentical
					? 'bg-green-500 dark:bg-green-600'
					: 'bg-red-400 dark:bg-red-500'}"
			>
				{#if isTextIdentical}
					<BadgeCheckIcon />
					Identical
				{:else}
					<BadgeAlert />
					Different
				{/if}
			</Badge>
		{/if}
		<Button onclick={swapText}>
			<ArrowRightLeft />
			Swap
		</Button>
		<Button onclick={clearText}>
			<BrushCleaning />
			Clear
		</Button>
	</div>
</div>
