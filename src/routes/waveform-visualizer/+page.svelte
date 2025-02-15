<script lang="ts">
	import { onMount } from 'svelte';

	// Audio Context basically controls everything/stores all the relevant data we need
	let audioContext: AudioContext;
	// the audio buffer will hold the audio data of the file in memory
	let audioBuffer: AudioBuffer | null = null;
	// boolean for currently playing, various use cases
	let isPlaying: boolean = $state(false);
	// track current time in seconds for play/pause + seeker bar
	let currentTime: number = $state(0);
	// total number of seconds in song
	let duration: number = 0;
	// flag to throw errors/prevent functions from accessing a null buffer
	let fileUploaded: boolean = $state(false);
	// this is the literal node that controls playback
	let audioPlaybackNode: AudioBufferSourceNode | null = null;
	// the analyser node is a middleman that will let us extract data from the audio
	// + if I understand correctly it routes to AC.destination for final playback
	let analyserNode: AnalyserNode | null = null;

	// want to make sure to CSR audio file, need to set up context as well
	// we can handle this all on mount
	onMount(async () => {
		try {
			// initialize audio context
			audioContext = new AudioContext();
			// create analyser node
			// this will stick around and we will use it to link to destination for audio
			analyserNode = audioContext.createAnalyser();
			analyserNode.connect(audioContext.destination);
			// standard example fast fourier transform window sample size
			analyserNode.fftSize = 2048;
			// fetch the file from static dir with some eror/console debug logging
			const file_fetch_resp = await fetch('netkick.wav');
			if (!file_fetch_resp.ok) {
				throw new Error(`HTTP error! status: ${file_fetch_resp.status}`);
			}
			console.log('Fetched: netkick.wav');
			// read the file into an array buffer
			const file_array_buffer = await file_fetch_resp.arrayBuffer();
			console.log('Buffer generated');
			// decode the audio data from the array buffer
			audioBuffer = await audioContext.decodeAudioData(file_array_buffer);
			console.log('Decoded audio data');
			// set duration
			duration = audioBuffer.duration;
			fileUploaded = true;
		} catch (error) {
			console.error('Error loading audio from file path:', error);
			fileUploaded = false;
		}
	});
</script>

<button
	class="bg-green-400"
	onclick={() => {
		audioPlaybackNode = audioContext.createBufferSource();
		audioPlaybackNode.buffer = audioBuffer;
		audioPlaybackNode.connect(analyserNode);
		audioPlaybackNode.start(0, currentTime);
		isPlaying = true;
	}}>Play</button
>
<button
	class="bg-sky-400"
	onclick={() => {
		if (!audioPlaybackNode) return;
		currentTime = audioContext.currentTime;
		audioPlaybackNode?.stop();
		isPlaying = false;
	}}
	>Pause
</button>
<h1>
	{#if isPlaying}
		Playing {audioContext.currentTime} / {duration}
	{:else}
		Paused
	{/if}
</h1>
