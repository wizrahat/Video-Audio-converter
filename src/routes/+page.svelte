<script lang="ts">
	// --- Package Imports ---
	import type {
		Mp4OutputFormat,
		WebMOutputFormat,
		Mp3OutputFormat,
		WavOutputFormat
	} from 'mediabunny';

	import {
		Conversion,
		Input,
		BlobSource,
		BufferTarget,
		Output,
		Mp4OutputFormat as Mp4,
		WebMOutputFormat as WebM,
		Mp3OutputFormat as Mp3,
		WavOutputFormat as Wav,
		ALL_FORMATS,
		canEncodeAudio
	} from 'mediabunny';
	import { registerMp3Encoder } from '@mediabunny/mp3-encoder';

	// --- Type Definitions ---
	type OutputFormatKey = 'mp4' | 'webm' | 'mp3' | 'wav';

	// Type that represents any of the Output Format class constructors
	type OutputFormatClass = typeof Mp4 | typeof WebM | typeof Mp3 | typeof Wav;

	interface FormatOption {
		value: OutputFormatKey;
		label: string;
	}

	// --- State Variables (Svelte reactive declarations) ---
	let file: File | null = null;
	let outputFormat: OutputFormatKey = 'mp4';
	let isConverting: boolean = false;
	let conversionProgress: number = 0;
	let convertedFileBlob: Blob | null = null;
	let convertedFileUrl: string = '';
	let conversionError: string = '';
	let downloadFileName: string = 'converted_file.mp4';

	// Available formats
	const availableFormats: FormatOption[] = [
		{ value: 'mp4', label: 'MP4 (Video/Audio)' },
		{ value: 'webm', label: 'WebM (Video/Audio)' },
		{ value: 'mp3', label: 'MP3 (Audio Only)' },
		{ value: 'wav', label: 'WAV (Audio Only)' }
	];

	// --- UI Handlers ---

	function clearFile(): void {
		file = null;
		clearResult();
	}

	/**
	 * Clears the conversion results and revokes the temporary download URL.
	 * This function is now called by a dedicated "Start New" button, not the download link.
	 */
	function clearResult(): void {
		conversionError = '';
		convertedFileBlob = null;
		if (convertedFileUrl) {
			URL.revokeObjectURL(convertedFileUrl);
			convertedFileUrl = '';
		}
	}

	/** * Handles file selection from the input element.
	 * @param {Event & { currentTarget: EventTarget & HTMLInputElement }} event
	 */
	function handleFileSelect(
		event: Event & { currentTarget: EventTarget & HTMLInputElement }
	): void {
		clearResult();
		const files = event.currentTarget.files;
		const selectedFile = files ? files[0] : null;
		if (selectedFile) {
			file = selectedFile;
		}
	}

	/** * Handles file dropping into the drop area.
	 * @param {DragEvent} event
	 */
	function handleDrop(event: DragEvent): void {
		clearResult();
		const droppedFile = event.dataTransfer?.files[0];
		if (droppedFile) {
			file = droppedFile;
		}
	}

	// --- Mediabunny Conversion Logic ---

	/**
	 * Maps the format key to the corresponding Mediabunny Output Format Class.
	 * @param {OutputFormatKey} formatKey
	 * @returns {OutputFormatClass}
	 */
	function getOutputFormatClass(formatKey: OutputFormatKey): OutputFormatClass {
		switch (formatKey) {
			case 'mp4':
				return Mp4;
			case 'webm':
				return WebM;
			case 'mp3':
				return Mp3;
			case 'wav':
				return Wav;
			default:
				throw new Error(`Unsupported format: ${formatKey}`);
		}
	}

	async function startConversion(): Promise<void> {
		if (!file) {
			conversionError = 'No file selected.';
			return;
		}

		isConverting = true;
		conversionProgress = 0;
		clearResult();

		try {
			// 1. MP3 Encoder Check & Registration (if needed)
			if (outputFormat === 'mp3' && !(await canEncodeAudio('mp3'))) {
				registerMp3Encoder();
			}

			// 2. Setup Input
			const input = new Input({
				source: new BlobSource(file),
				formats: ALL_FORMATS
			});

			// 3. Setup Output
			const OutputFormatClass = getOutputFormatClass(outputFormat);
			const output = new Output({
				format: new OutputFormatClass(),
				target: new BufferTarget()
			});

			// 4. Setup Conversion and Progress Listener
			const conversion = await Conversion.init({ input, output });

			conversion.onProgress = (ratio: number) => {
				// ratio is a number between 0 and 1
				conversionProgress = ratio * 100;
			};

			// 5. Execute Conversion
			await conversion.execute();

			// 6. Finalize and Get Result
			const target = output.target as BufferTarget;
			const buffer = target.buffer;

			if (!buffer) {
				throw new Error('Conversion finished but no buffer data was returned.');
			}

			const formatInstance = output.format as
				| Mp4OutputFormat
				| WebMOutputFormat
				| Mp3OutputFormat
				| WavOutputFormat;
			const mimeType = (formatInstance as { mimeType: string }).mimeType;

			convertedFileBlob = new Blob([buffer], { type: mimeType });
			convertedFileUrl = URL.createObjectURL(convertedFileBlob);

			// Create download file name
			const baseName = file.name.substring(0, file.name.lastIndexOf('.')) || 'converted';
			downloadFileName = `${baseName}.${outputFormat}`;
		} catch (e) {
			console.error('Conversion failed:', e);
			const error = e as Error;
			conversionError =
				error.message || 'An unknown error occurred during conversion. Check console for details.';
		} finally {
			isConverting = false;
			conversionProgress = 100; // Ensure progress bar is full or error is shown
		}
	}
</script>

<svelte:head>
	<!-- Styling links -->
	<script src="https://cdn.tailwindcss.com"></script>
	<link
		href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap"
		rel="stylesheet"
	/>

	<style>
		:global(:root) {
			font-family: 'Inter', sans-serif;
		}
		.app-bg {
			background: linear-gradient(135deg, #1f2937 0%, #0e131d 100%);
		}
		:global(body) {
			margin: 0;
			min-height: 100vh;
		}
	</style>
</svelte:head>

<div class="app-bg flex min-h-screen items-center justify-center p-4 text-gray-100">
	<div class="w-full max-w-lg rounded-2xl border border-gray-700 bg-gray-800 p-8 shadow-2xl">
		<h1 class="mb-6 text-center text-3xl font-bold text-indigo-400">
			<span class="text-indigo-300">🐰</span> In-Browser Media Converter
		</h1>
		<p class="mb-8 text-center text-gray-400">
			Powered by Mediabunny: conversion happens instantly on your device.
		</p>

		{#if !file}
			<!-- File Drop/Input Area -->
			<label for="file-input" class="block w-full cursor-pointer">
				<div
					class="rounded-xl border-4 border-dashed border-gray-600 p-12 text-center transition duration-300 hover:border-indigo-500 hover:bg-gray-700/50"
					on:dragover|preventDefault
					on:drop|preventDefault={handleDrop}
				>
					<svg
						class="mx-auto h-12 w-12 text-gray-500"
						fill="none"
						viewBox="0 0 24 24"
						stroke="currentColor"
					>
						<path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 014 4v2a2 2 0 01-2 2h-2.5l-.25.5a3.5 3.5 0 01-6.1 0L10 14.5l-.25-.5H8a2 2 0 01-2-2v-2a4 4 0 01-2-2z"
						/>
					</svg>
					<p class="mt-2 text-sm text-gray-400">
						Drag and drop a video or audio file here, or <span class="font-semibold text-indigo-400"
							>click to select</span
						>
					</p>
					<p class="mt-1 text-xs text-gray-500">(MP4, WebM, WAV, MP3, etc. supported)</p>
				</div>
				<input
					id="file-input"
					type="file"
					class="hidden"
					on:change={handleFileSelect}
					accept="video/*,audio/*"
				/>
			</label>
		{:else}
			<!-- Conversion UI -->
			<div class="space-y-6">
				<!-- File Info -->
				<div
					class="flex items-center justify-between rounded-xl border border-gray-600 bg-gray-700 p-4 shadow-inner"
				>
					<div class="min-w-0 flex-1">
						<p class="truncate text-lg font-medium text-white">{file.name}</p>
						<p class="text-sm text-gray-400">Size: {(file.size / (1024 * 1024)).toFixed(2)} MB</p>
					</div>
					<button
						on:click={clearFile}
						class="rounded-full p-2 text-red-400 transition hover:bg-gray-600 hover:text-red-300"
						title="Remove file"
					>
						<svg
							xmlns="http://www.w3.org/2000/svg"
							class="h-6 w-6"
							fill="none"
							viewBox="0 0 24 24"
							stroke="currentColor"
						>
							<path
								stroke-linecap="round"
								stroke-linejoin="round"
								stroke-width="2"
								d="M6 18L18 6M6 6l12 12"
							/>
						</svg>
					</button>
				</div>

				<!-- Format Selection -->
				<div>
					<label for="output-format" class="mb-2 block text-sm font-medium text-gray-300"
						>Convert To</label
					>
					<select
						id="output-format"
						bind:value={outputFormat}
						class="w-full rounded-xl border border-gray-600 bg-gray-700 px-4 py-3 text-white shadow-md transition focus:border-indigo-500 focus:ring-indigo-500"
						disabled={isConverting}
					>
						{#each availableFormats as format}
							<option value={format.value}>{format.label}</option>
						{/each}
					</select>
				</div>

				<!-- Conversion Button -->
				<button
					on:click={startConversion}
					disabled={isConverting}
					class="w-full transform rounded-xl px-4 py-3.5 font-bold text-white transition duration-300 active:scale-[0.98]
                    {isConverting
						? 'cursor-not-allowed bg-indigo-600'
						: 'bg-indigo-500 shadow-xl shadow-indigo-500/50 hover:bg-indigo-600'}"
				>
					{#if isConverting}
						<span class="flex items-center justify-center">
							<svg
								class="mr-3 -ml-1 h-5 w-5 animate-spin text-white"
								xmlns="http://www.w3.org/2000/svg"
								fill="none"
								viewBox="0 0 24 24"
							>
								<circle
									class="opacity-25"
									cx="12"
									cy="12"
									r="10"
									stroke="currentColor"
									stroke-width="4"
								></circle>
								<path
									class="opacity-75"
									fill="currentColor"
									d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
								></path>
							</svg>
							Converting... ({conversionProgress.toFixed(0)}%)
						</span>
					{:else}
						Convert Now
					{/if}
				</button>

				<!-- Progress Bar & Result -->
				{#if isConverting || convertedFileBlob || conversionError}
					<div class="space-y-4 border-t border-gray-700 pt-4">
						{#if conversionError}
							<div class="rounded-xl border border-red-700 bg-red-900/50 p-4 text-red-300">
								<p class="mb-1 font-bold">Conversion Error:</p>
								<p class="text-sm">{conversionError}</p>
							</div>
						{:else if isConverting}
							<!-- <div class="h-3 overflow-hidden rounded-full bg-gray-700 shadow-inner">
								<div
									class="h-full bg-indigo-500 transition-all duration-100 ease-linear"
									style="width: {conversionProgress}%;"
								></div>
							</div> -->
							<p class="text-center text-sm text-gray-400">Processing media data...</p>
						{:else if convertedFileBlob}
							<!-- FIX: Removed on:click={clearResult} from the download link -->
							<div class="rounded-xl border border-green-700 bg-green-900/50 p-4 text-green-300">
								<span class="mb-3 block font-semibold">✅ Conversion Complete!</span>
								<div class="flex flex-col items-center justify-end gap-3 sm:flex-row">
									<p class="flex-1 text-sm text-gray-300">
										Ready to download <span
											class="rounded bg-green-800/50 px-2 py-0.5 font-mono text-xs"
											>{downloadFileName}</span
										>
									</p>

									<!-- New button to clear state and start over -->
									<!-- <button
										on:click={clearResult}
										class="w-full rounded-xl bg-gray-600 px-4 py-2 font-semibold text-white shadow-md transition hover:bg-gray-500 sm:w-auto"
									>
										Start New
									</button> -->

									<!-- Download link (now safe from immediate cleanup) -->
									<a
										href={convertedFileUrl}
										download={downloadFileName}
										class="w-full rounded-xl bg-green-500 px-4 py-2 text-center font-semibold text-white shadow-lg transition hover:bg-green-600 sm:w-auto"
									>
										Download
									</a>
								</div>
							</div>
						{/if}
					</div>
				{/if}
			</div>
		{/if}
	</div>
</div>
