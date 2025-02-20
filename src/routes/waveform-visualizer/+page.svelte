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
	let fixedTime: number = $state(0);
	let intervalID: number | null = null;
	// total number of seconds in song
	let duration: number = 0;
	// flag to throw errors/prevent functions from accessing a null buffer
	let fileUploaded: boolean = $state(false);
	// this is the literal node that controls playback
	let audioPlaybackNode: AudioBufferSourceNode | null = null;
	// the analyser node is a middleman that will let us extract data from the audio
	// + if I understand correctly it routes to AC.destination for final playback
	let analyserNode: AnalyserNode | null = null;

	// HTML Canvas element for rendering/drawing the waveform
	let canvas: HTMLCanvasElement;
	let canvasWidth: number = 0;
	let canvasHeight: number = 0;
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
	// this function increased the fixed time by 1 second, this is an idea to track playback time in seconds for timeline purposes
	function tickByOne() {
		fixedTime += 1;
	}
	function handleMouseMove(event: MouseEvent) {
		if (!canvas || !audioBuffer) return;
		const rect = canvas.getBoundingClientRect();
		const mouseX = event.clientX - rect.left;
	}
	// basic function to draw generic waveform/practice how to
	function drawBasicWaveform() {
		if (!canvas) return;
		const ctx = canvas.getContext('2d');
		if (!ctx) return;
		// set dimension variables
		canvasWidth = canvas.width;
		canvasHeight = canvas.height;
		// style settings
		ctx.strokeStyle = 'black';
		ctx.lineWidth = 1;
		ctx.beginPath();
		let curX: number = 0;
		//let curY: number = 0;
		// let's draw 60 lines of variable heights
		for (let i = 1; i < 251; i++) {
			// get random height
			const y = (Math.random() * canvasHeight) / 2;
			// move to next x coordinate, height/2 is middle of canvas
			ctx.moveTo(curX, canvasHeight / 2);
			ctx.lineTo(curX, y);
			ctx.moveTo(curX, canvasHeight / 2);
			ctx.lineTo(curX, canvasHeight - y);
			curX = (canvasWidth / 250) * i;
			ctx.stroke();
		}
	}
	// clear the canvas
	function clearCanvas() {
		if (!canvas) return;
		const ctx = canvas.getContext('2d');
		if (!ctx) return;
		ctx.clearRect(0, 0, canvas.width, canvas.height);
	}
</script>

<button
	class="bg-green-400"
	onclick={() => {
		audioPlaybackNode = audioContext.createBufferSource();
		audioPlaybackNode.buffer = audioBuffer;
		audioPlaybackNode.connect(analyserNode);
		audioPlaybackNode.start(0, currentTime);
		isPlaying = true;
		intervalID = setInterval(tickByOne, 1000);
	}}>Play</button
>
<button
	class="bg-sky-400"
	onclick={() => {
		if (!audioPlaybackNode) return;
		currentTime = audioContext.currentTime;
		audioPlaybackNode?.stop();
		isPlaying = false;
		if (intervalID) {
			clearInterval(intervalID);
		}
	}}
	>Pause
</button>
<button
	class="bg-red-400"
	onclick={() => {
		if (!audioPlaybackNode) return;
		if (intervalID) {
			clearInterval(intervalID);
		}
		currentTime = 0;
		audioPlaybackNode?.stop();
		isPlaying = false;
		fixedTime = 0;
	}}
	>Stop
</button>
<h1>
	{#if isPlaying}
		Playing {currentTime} / {duration}
		Playing {fixedTime} / {duration}
	{:else}
		Paused
	{/if}
</h1>
<button onclick={drawBasicWaveform}>Draw Random Waveform</button>
<button onclick={clearCanvas} class="border-black bg-red-400">Clear Canvas</button>
<canvas
	bind:this={canvas}
	class="waveform-canvas block h-[200px] w-full bg-gray-200"
	onmousemove={handleMouseMove}
></canvas>
