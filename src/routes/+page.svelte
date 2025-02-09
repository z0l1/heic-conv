<script lang="ts">
	import { onMount } from 'svelte';
	import type { ChangeEventHandler } from 'svelte/elements';

	let heic2any;

	onMount(async () => {
		if (typeof window !== 'undefined') {
			const module = await import('heic2any');
			heic2any = module.default;
		}
	});

	let files = $state<FileList | null | undefined>();
	let convertType = $state('png');

	$effect(() => {
		console.log('files changed', files);
	});

	const onFileChange: ChangeEventHandler<HTMLInputElement> = (e) => {
		console.log(e);
	};

	let convertCnt = $state(0);
	// let convertedCnt = $state(0);
	let convertedFiles: File[] = $state([]);

	let convertStarted = $state(false);
	let converting = $derived(convertedFiles.length !== convertCnt);

	const onConvert = async () => {
		convertCnt = 0;
		// convertedCnt = 0;
		// converting = false;
		convertedFiles = [];

		if (!files) {
			return;
		}

		const filesToConvert: File[] = [];

		for (const file of files) {
			if (file.name.toLowerCase().endsWith('.heic')) {
				filesToConvert.push(file);
			}
		}

		if (filesToConvert.length === 0) {
			alert('no selected file is *.HEIC');
			return;
		}

		convertStarted = true;
		convertCnt = filesToConvert.length;
		// converting = true;

		for (const fileToConvert of filesToConvert) {
			heic2any!({
				blob: fileToConvert,
				toType: `image/${convertType}`,
				quality: 1
				// quality: 0.5, // cuts the quality and size by half
			})
				.then((conversionResult) => {
					// conversionResult is a BLOB
					// of the JPEG formatted image

					const fileName = fileToConvert.name.replace(/heic/gi, convertType);

					convertedFiles.push(new File([conversionResult], fileName));
				})
				.catch((e) => {
					// see error handling section

					alert('error, kialts');
					console.log(e);
				});
		}
	};

	const onDownloadConverted = () => {
		for (const converted of convertedFiles) {
			const blobUrl = URL.createObjectURL(converted);
			const link = document.createElement('a');
			link.href = blobUrl;
			link.download = converted.name;
			document.body.appendChild(link);
			link.dispatchEvent(
				new MouseEvent('click', {
					bubbles: true,
					cancelable: true,
					view: window
				})
			);
			// Remove link from body
			document.body.removeChild(link);
		}
	};
</script>

<h1 class="text-lg font-bold">SELECT FILES</h1>

<input bind:files accept=".heic" onchange={onFileChange} type="file" multiple />

{#if !files || files.length === 0}
	<!-- <h1 class="text-lg font-bold">NO FILES SELECTED</h1> -->
{:else}
	<div class="flex w-full">
		<!-- <h1 class="text-lg font-bold">SELECTED FILES ({files.length})</h1> -->
		<div class="flex-1">
			<ul>
				{#each files as file}
					<li>
						<span class="font-mono">{file.name}</span>
						<span class="font-mono">{(file.size / 1000 / 1000).toPrecision(2)} MB</span>
					</li>
				{/each}
			</ul>
		</div>

		<div class="flex-1">
			<div>
				<span>CONVERT HEIC TO</span>
				<select disabled={converting} bind:value={convertType}>
					<option value="png" label="PNG"></option>
					<option value="jpeg" label="JPEG"></option>
				</select>
			</div>

			<button
				disabled={converting}
				onclick={onConvert}
				type="button"
				class="border bg-slate-300 p-2 px-4"
			>
				CONVERT
			</button>

			{#if convertStarted}
				<h1>Converting {convertCnt} .heic images to .{convertType}</h1>

				<h1>{convertedFiles.length} / {convertCnt}</h1>
				{#if converting}
					<div class="flex flex-row items-center gap-1">
						converting
						<div class="max-w-min animate-spin">...</div>
					</div>
				{:else}
					<div>finished...</div>

					<button type="button" onclick={onDownloadConverted} class="border bg-slate-300 p-2 px-4"
						>DOWNLOAD FILES</button
					>
				{/if}
			{/if}
		</div>
	</div>
{/if}
