<script lang="ts">
	import '../app.css';
	// Store for storing the media that is recorded so that we can download it from the list provided.
	// NOTE: This will clear on browser refresh
	import { writable } from 'svelte/store';

	// Reference to the video tag.
	let videoElement: HTMLVideoElement;
	// Reference to the media recorder.
	let mediaRecorder: MediaRecorder | null = null;
	// When the recording starts, the media chunks are pushed here.
	let recordedChunks: Blob[] = [];
	let isRecording = false;
	let previewStream: MediaStream | null = null;
	let isPreviewing = false;

	// List of media that has been recorded.
	const recordedMedia = writable<{ name: string; url: string }[]>([]);

	// Function for starting the recording
	const startRecording = async (isScreen: boolean = false) => {
		try {
			// Setting the Media Constraint based on the user selection of recording video/audio
			// from either webcam or screen recording.
			const constraints = isScreen
				? { video: { mediaSource: 'screen' }, audio: false }
				: { video: true, audio: true };

			// Based on the user selection above, setting the stream using these apis
			// getDisplayMedia() -> For screen recording
			// getUserMedia() -> For webcam
			const stream = isScreen
				? await (navigator.mediaDevices as any).getDisplayMedia(constraints)
				: await (navigator.mediaDevices as any).getUserMedia(constraints);

			videoElement.srcObject = stream;
			videoElement.play();

			// Creating media recorder and attaching event ondataavailable to it.
			mediaRecorder = new MediaRecorder(stream);
			mediaRecorder.ondataavailable = (event) => {
				if (event.data.size > 0) {
					recordedChunks.push(event.data);
				}
			};

			mediaRecorder.onstop = () => {
				const blob = new Blob(recordedChunks, { type: 'video/webm' });
				const url = URL.createObjectURL(blob);
				const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
				const name = isScreen ? `SCREEN_${timestamp}` : `WEBCAM_${timestamp}`;
				// Updating media to local store and resetting the chunks array
				recordedMedia.update((media) => [...media, { name, url }]);
				recordedChunks = [];
			};

			mediaRecorder.start();
			isRecording = true;
		} catch (err) {
			console.error('Error accessing media devices.', err);
		}
	};

	// Handle Stop Recording
	const stopRecording = () => {
		if (mediaRecorder) {
			mediaRecorder.stop();
			isRecording = false;
			if (videoElement && videoElement.srcObject) {
				(<MediaStream>videoElement.srcObject).getTracks().forEach((track) => track.stop());
			}
		}
	};

	// Handle Pause Recording
	const pauseRecording = () => {
		if (mediaRecorder && isRecording) {
			mediaRecorder.pause();
		}
	};

	// Handle Resume Recording
	const resumeRecording = () => {
		if (mediaRecorder && isRecording) {
			mediaRecorder.resume();
		}
	};

	// Handle Toggle Webcam Preview
	const togglePreview = async () => {
		if (isPreviewing) {
			if (previewStream) {
				previewStream.getTracks().forEach((track) => track.stop());
			}
			videoElement.srcObject = null;
			isPreviewing = false;
		} else {
			try {
				previewStream = await navigator.mediaDevices.getUserMedia({ video: true, audio: false });
				videoElement.srcObject = previewStream;
				videoElement.play();
				isPreviewing = true;
			} catch (err) {
				console.error('Error accessing camera for preview.', err);
			}
		}
	};

	// Handle Download Media.
	const downloadMedia = (url: string, name: string) => {
		const a = document.createElement('a');
		a.style.display = 'none';
		a.href = url;
		a.download = `${name}.webm`;
		document.body.appendChild(a);
		// Trigger click event to the 'a' tag created for starting the download.
		a.click();
		window.URL.revokeObjectURL(url);
	};
</script>

<main>
	<div class="flex justify-between items-start gap-2 p-4">
		<div class="flex flex-col gap-2">
			<video class="bg-black" bind:this={videoElement} autoplay muted></video>
			<div class="flex gap-2">
				<button
					class="border border-blue-500 px-2 py-4 max-w-40 rounded-xl"
					on:click={() => startRecording(false)}>Start Camera Recording</button
				>
				<button
					class="border border-blue-500 px-2 py-4 max-w-40 rounded-xl"
					on:click={() => startRecording(true)}>Start Screen Recording</button
				>
				<button
					class="border border-blue-500 px-2 py-4 max-w-40 rounded-xl"
					on:click={stopRecording}>Stop Recording</button
				>
				<button
					class="border border-blue-500 px-2 py-4 max-w-40 rounded-xl"
					on:click={pauseRecording}>Pause Recording</button
				>
				<button
					class="border border-blue-500 px-2 py-4 max-w-40 rounded-xl"
					on:click={resumeRecording}>Resume Recording</button
				>
				<button
					class="border border-blue-500 px-2 py-4 max-w-40 rounded-xl"
					on:click={togglePreview}>{isPreviewing ? 'Stop Preview' : 'Start Preview'}</button
				>
			</div>
		</div>
		{#if $recordedMedia.length > 0}
			<div class="flex flex-col gap-4 justify-center items-center flex-grow">
				<h2 class="font-semibold">Recorded Media</h2>
				<ul class="flex flex-col gap-2 w-full">
					{#each $recordedMedia as { name, url }}
						<li class="flex justify-between items-center border-b first:border-t border-gray-400">
							{name}
							<button
								class="font-medium text-sm hover:text-green-600 text-blue-600"
								on:click={() => downloadMedia(url, name)}>Download</button
							>
						</li>
					{/each}
				</ul>
			</div>
		{/if}
	</div>
</main>
