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
	let duration: number = $state(0);
	// flag to throw errors/prevent functions from accessing a null buffer
	let fileUploaded: boolean = $state(false);
	// this is the literal node that controls playback
	let audioPlaybackNode: AudioBufferSourceNode | null = null;
	// the analyser node is a middleman that will let us extract data from the audio
	// + if I understand correctly it routes to AC.destination for final playback
	let analyserNode: AnalyserNode | null = null;

	// HTML Canvas element for rendering/drawing the waveform
	let canvas: HTMLCanvasElement;
	let line_canvas: HTMLCanvasElement;
	let playback_line_canvas: HTMLCanvasElement;
	let line_canvas_width: number = 0;
	let line_canvas_height: number = 0;
	let canvasWidth: number = 0;
	let canvasHeight: number = 0;
	let truncatedDuration: number = 0;
	let sec_width_ratio: number = 0;
	// each ratio maps some interval of x to .01 of a second

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
			truncatedDuration = Math.floor(duration * 100) / 100;
			canvasWidth = canvas.width;
			sec_width_ratio = canvasWidth / truncatedDuration;
			fileUploaded = true;
		} catch (error) {
			console.error('Error loading audio from file path:', error);
			fileUploaded = false;
		}
	});
	// this function increased the fixed time by .01 second, this is an idea to track playback time in seconds for timeline purposes
	function tickByOne() {
		fixedTime += 0.01;
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

	// function for handling the line
	function handleLineSeek(event: MouseEvent) {
		if (!line_canvas) return;
		const ctx = line_canvas.getContext('2d');
		if (!ctx) return;
		// get the x coordinate of the mouse
		const rect = line_canvas.getBoundingClientRect();
		const x = (event.clientX - rect.left) * (line_canvas.width / rect.width);
		// clear the canvas
		ctx.clearRect(0, 0, line_canvas.width, line_canvas.height);
		// draw the line
		ctx.beginPath();
		ctx.moveTo(x, 0);
		ctx.lineTo(x, line_canvas.height);
		ctx.strokeStyle = 'red';
		ctx.lineWidth = 1; // Adjust this value to make the line thinner
		ctx.stroke();
	}
	function handlePause() {
		if (!audioPlaybackNode) return;
		currentTime = audioContext.currentTime;
		audioPlaybackNode?.stop();
		isPlaying = false;
		if (intervalID) {
			clearInterval(intervalID);
		}
	}
	function handlePlay() {
		audioPlaybackNode = audioContext.createBufferSource();
		audioPlaybackNode.buffer = audioBuffer;
		audioPlaybackNode.connect(analyserNode);
		audioPlaybackNode.start(0, currentTime);
		isPlaying = true;
		intervalID = setInterval(tickByOne, 10);
	}
	function handleStop() {
		if (!audioPlaybackNode) return;
		if (intervalID) {
			clearInterval(intervalID);
		}
		currentTime = 0;
		audioPlaybackNode?.stop();
		isPlaying = false;
		fixedTime = 0;
	}
	// animate line to show playback
	$effect(() => {
		if (!playback_line_canvas) return;
		const ctx = playback_line_canvas.getContext('2d');
		if (!ctx) return;
		// clear the canvas
		ctx.clearRect(0, 0, playback_line_canvas.width, playback_line_canvas.height);
		const playback_line_x = fixedTime * sec_width_ratio;
		// draw the line
		ctx.beginPath();
		ctx.moveTo(playback_line_x, 0);
		ctx.lineTo(playback_line_x, playback_line_canvas.height);
		ctx.strokeStyle = 'gray';
		ctx.lineWidth = 1; // Adjust this value to make the line thinner
		ctx.stroke();
	});
</script>

<button class="bg-green-400" onclick={handlePlay}>Play</button>
<button class="bg-sky-400" onclick={handlePause}>Pause </button>
<button class="bg-red-400" onclick={handleStop}>Stop </button>
<h1>
	{#if isPlaying}
		Playing {currentTime.toFixed(3)} / {duration.toFixed(3)}
		Playing {fixedTime.toFixed(3)} / {duration.toFixed(3)}
	{:else}
		Paused
	{/if}
</h1>
<button onclick={drawBasicWaveform}>Draw Random Waveform</button>
<button onclick={clearCanvas} class="border-black bg-red-400">Clear Canvas</button>
<div class="canvas-container">
	<canvas bind:this={canvas} class="waveform-canvas bg-gray-200"></canvas>
	<canvas bind:this={playback_line_canvas} class="playback-line-canvas bg-opacity-0"></canvas>
	<canvas
		bind:this={line_canvas}
		class="line-canvas bg-opacity-0"
		onmousemove={handleLineSeek}
		onclick={(e: MouseEvent) => {
			const rect = line_canvas.getBoundingClientRect();
			const x = (e.clientX - rect.left) * (line_canvas.width / rect.width);
			fixedTime = x / sec_width_ratio;
			console.log('jumping to: ', fixedTime);
			handlePause();
			currentTime = fixedTime;
			handlePlay();
		}}
	></canvas>
</div>

<style>
	.canvas-container {
		position: relative;
		width: 100%;
		height: 200px; /* Adjust this height as needed */
	}
	.waveform-canvas,
	.playback-line-canvas,
	.line-canvas {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
	}
</style>
