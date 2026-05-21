<script lang="ts">
	import { onDestroy, onMount } from 'svelte';
	import * as monaco from 'monaco-editor';
	import { Button } from '$lib/components/ui/button/index.js';
	import { ArrowRightLeft, BrushCleaning, MoonIcon, Settings, SunIcon } from '@lucide/svelte';
	import { resetMode, setMode, mode } from 'mode-watcher';
	import * as DropdownMenu from '$lib/components/ui/dropdown-menu/index.js';
	import { buttonVariants } from '$lib/components/ui/button/index.js';
	import { Checkbox } from '$lib/components/ui/checkbox/index.js';
	import { Label } from '$lib/components/ui/label/index.js';
	import * as Popover from '$lib/components/ui/popover/index.js';

	let editorElement: HTMLDivElement;
	let editor: monaco.editor.IStandaloneDiffEditor;
	let isTextIdentical = false;
	let checkSpaces = true;
	let checkNewLines = true;
	let checkCapitalizations = true;

	function normalizeText(value: string) {
		let normalizedValue = value;

		if (!checkSpaces) {
			normalizedValue = normalizedValue.replace(/[ \t]/g, '');
		}

		if (!checkNewLines) {
			normalizedValue = normalizedValue.replace(/[\r\n]/g, '');
		}

		if (!checkCapitalizations) {
			normalizedValue = normalizedValue.toLocaleLowerCase();
		}

		return normalizedValue;
	}

	// Function to update border color and status label based on text comparison
	function updateStatus() {
		if (!editor) {
			isTextIdentical = false;
			return;
		}

		const originalModel = editor.getModel()?.original;
		const modifiedModel = editor.getModel()?.modified;

		if (originalModel && modifiedModel) {
			const originalValue = originalModel.getValue();
			const modifiedValue = modifiedModel.getValue();

			isTextIdentical = normalizeText(originalValue) === normalizeText(modifiedValue);
		} else {
			isTextIdentical = false;
		}
	}

	function updateDiffOptions() {
		editor?.updateOptions({ ignoreTrimWhitespace: !checkSpaces });
		updateStatus();
	}

	$: if (editor) {
		localStorage.setItem('check-spaces', checkSpaces.toString());
		localStorage.setItem('check-newlines', checkNewLines.toString());
		localStorage.setItem('check-capitalizations', checkCapitalizations.toString());
		updateDiffOptions();
	}

	onMount(async () => {
		checkSpaces = localStorage.getItem('check-spaces') !== 'false';
		checkNewLines = localStorage.getItem('check-newlines') !== 'false';
		checkCapitalizations = localStorage.getItem('check-capitalizations') !== 'false';

		editor = monaco.editor.createDiffEditor(editorElement, {
			automaticLayout: true,
			ignoreTrimWhitespace: !checkSpaces,
			theme: mode.current === 'dark' ? 'vs-dark' : 'vs-light',
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
		monaco.editor.setTheme(mode.current === 'dark' ? 'vs-dark' : 'vs-light');
	}

	function swapText() {
		const originalModel = editor.getModel()?.original;
		const modifiedModel = editor.getModel()?.modified;

		if (originalModel && modifiedModel) {
			const originalValue = originalModel.getValue();
			const modifiedValue = modifiedModel.getValue();

			originalModel.setValue(modifiedValue);
			modifiedModel.setValue(originalValue);
		}
	}

	function clearText() {
		const originalModel = editor.getModel()?.original;
		const modifiedModel = editor.getModel()?.modified;

		if (originalModel && modifiedModel) {
			originalModel.setValue('');
			modifiedModel.setValue('');
		}
	}
</script>

<div class="flex h-screen w-full flex-col items-center justify-between">
	<div class="flex h-screen w-full">
		<div
			class="grow"
			class:ignored-diffs={isTextIdentical}
			class:ignore-line-feed-diffs={!checkNewLines}
			bind:this={editorElement}
		></div>
	</div>
	<div class="absolute bottom-0 mb-4 flex w-full items-center justify-center gap-2">
		<div
			class="relative flex w-fit gap-1.25 rounded-md border-2 bg-white p-2 dark:bg-gray-800 {isTextIdentical
				? 'border-green-300 dark:border-green-600'
				: 'border-red-400 dark:border-red-600'}"
		>
			<Button variant="outline" onclick={swapText}>
				<ArrowRightLeft />
				Swap
			</Button>
			<Button variant="outline" onclick={clearText}>
				<BrushCleaning />
				Clear
			</Button>
			<Popover.Root>
				<Popover.Trigger class={buttonVariants({ variant: 'outline', size: 'icon' })}>
					<Settings />
				</Popover.Trigger>
				<Popover.Content class="w-fit">
					<div class="flex flex-col gap-4">
						<div class="flex items-center gap-3">
							<Checkbox id="toggle-spaces" bind:checked={checkSpaces} />
							<Label for="toggle-spaces">Spaces or tabs diffs</Label>
						</div>
						<div class="flex items-center gap-3">
							<Checkbox id="toggle-newlines" bind:checked={checkNewLines} />
							<Label for="toggle-newlines">Newline or line feed diffs</Label>
						</div>
						<div class="flex items-center gap-3">
							<Checkbox id="toggle-capitalizations" bind:checked={checkCapitalizations} />
							<Label for="toggle-capitalizations">Capitalization diffs</Label>
						</div>
					</div>
				</Popover.Content>
			</Popover.Root>
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
	</div>
</div>

<style>
	:global(.ignored-diffs .line-insert),
	:global(.ignored-diffs .line-delete),
	:global(.ignored-diffs .char-insert),
	:global(.ignored-diffs .char-delete),
	:global(.ignored-diffs .gutter-insert),
	:global(.ignored-diffs .gutter-delete),
	:global(.ignored-diffs .insert-sign),
	:global(.ignored-diffs .delete-sign),
	:global(.ignored-diffs .diff-hidden-lines),
	:global(.ignored-diffs .diffOverview) {
		background-color: transparent !important;
		border-color: transparent !important;
	}

	:global(.ignored-diffs .insert-sign),
	:global(.ignored-diffs .delete-sign) {
		color: transparent !important;
	}
</style>
