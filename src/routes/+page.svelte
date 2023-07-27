<script>
	import {tweened} from 'svelte/motion';
	import {onDestroy, onMount} from 'svelte';
	import {cubicOut} from 'svelte/easing';
	import ColorCard from "$lib/ColorCard.svelte";
	import {colorStatuses, firstLoad, loading, placeholders, totalGenerations} from "../stores.js";
	import {ProgressRadial} from "@skeletonlabs/skeleton";
	import axios from "axios";

	const scale = tweened(1, { duration: 180, easing: cubicOut });


	let colorCodes = [];
	let colors = [];
	let input = "";
	let lastSent = "";
	let gennedFont = "Baloo Tamma 2";
	let stop = false;
	let inputUp = tweened(0, {
		duration: 700,
		easing: cubicOut
	});
	let placeholder = "";
	let titleSize = '10px';

	let runningFontErrors = 0;

	let fontLink;

	let runningColorErrors = 0;

	let hasVisitedBefore = false;


	onMount(() => {
		$: if ($totalGenerations < 1) {
			console.log($totalGenerations)
			stop = false;
			animateText();
		} else {
			stop = true;
		}
		if (localStorage.getItem('hasVisited')) {
			hasVisitedBefore = true;
		} else {
			localStorage.setItem('hasVisited', 'true');
		}
		console.log("Been Here "+hasVisitedBefore);
	});


	$: if($totalGenerations>0){
		stop=true;
		placeholder = "";
	}

	onDestroy(() => {
		stop = true;
	});

	function handleClick() {
		$loading=true
		let APICall = genColors(input);
		getFont(input);
		lastSent = input;
		APICall.then(processAPIData);
		scale.set(0.9)
				.then(() => scale.set(1));
		inputUp.set(-300)
		setTimeout(function(){
			$firstLoad=true
			$colorStatuses = ["loading", "loading", "loading", "loading", "loading"];
		}, 150);

	}

	async function getFont(input) {
		let messages =
				[
					{
						role: 'system',
						content: 'You are a Google Fonts Font Finder. You will be given a theme, and no matter how weird the font is, you will return exclusively the name of a real Google Font that matches that theme.'
					},
					{
						role: 'user',
						content: 'Theme: Sailing the ocean waves.'
					},
					{
						role: 'assistant',
						content: 'Lobster Two'
					},
					{
						role: 'user',
						content: 'Theme: ' + input
					}];
		try {
			const result = await axios.post("https://nwe0dbevfl.execute-api.us-east-2.amazonaws.com/prod/chat", {
				"messages": messages
			});
			loadFont(result.data.content);
		} catch (error) {

		}
	}

	function loadFont(fontName) {
		const link = document.createElement('link');
		link.href = `https://fonts.googleapis.com/css2?family=${encodeURIComponent(fontName)}&display=swap`;
		fontLink = `https://fonts.google.com/specimen/${encodeURIComponent(fontName)}`;
		link.rel = 'stylesheet';

		// Error handling
		link.onerror = function() {
			console.error(`Failed to load font: ${fontName}`);
			runningFontErrors++;
			if(runningFontErrors<3) {
				getFont(input);
			}
		};

		// If the font loads successfully, set gennedFont
		link.onload = function() {
			runningFontErrors=0;
			gennedFont = fontName;
			console.log(`Font loaded: ${fontName}`);
		};

		document.head.appendChild(link);
	}

	async function swapColor(event) {
		let colorString = "";
		for (let i = 0; i < colors.length; i++) {
			colorString += `${colors[i]}|${colorCodes[i]}\n`;
		}
		let messages =
				[
					{
						role: 'system',
						content: 'You are a color palette alternative generator. You are given a theme and a list of 5 colors and their hex codes and the color that is being replaced. You will return 1 new color and the corresponding hex code that compliments the other colors and the theme, in the same format. That is all you will return, you will return no other text.'
					},
					{
						role: 'user',
						content: 'Theme: Sailing the ocean waves.\nColors:\nOcean Blue|#0077be\nSandy Beach|#f0d9b5\nSeashell White|#fff5ee\nSailboat Gray|#6c7b8b\nCoral Sunset|#ff6f61\nReplace: Ocean Blue'
					},
					{
						role: 'assistant',
						content: 'Turquoise Waters|#1abc9c'
					},
					{
						role: 'user',
						content: 'Theme: ' + lastSent + '\nColors:\n' + colorString + 'Replace: ' + colors[event.detail.text]
					}];
		console.log(messages);
		try {
			const result = await axios.post("https://nwe0dbevfl.execute-api.us-east-2.amazonaws.com/prod/chat", {
				"messages": messages
			});
			[colors[event.detail.text], colorCodes[event.detail.text]] = result.data.content.split('|');
			$colorStatuses[event.detail.text] = "loaded";
		} catch (error) {
			$colorStatuses[event.detail.text] = "loaded";
			errorMessage();
		}
	}

	function processAPIData(data) {
		let tempColors=[];
		let tempColorCodes=[];
		let lines = data.split('\n');

		lines.forEach(line => {
			let parts = line.split('|');
			tempColors.push(parts[0]);
			tempColorCodes.push(parts[1]);
		});

		if(tempColors.length<5){
			errorMessage();
			$loading=false
			return;
		}
		const hexColorRegex = /^#([0-9A-F]{3}){1,2}$/i;
		let validHex = true;
		tempColorCodes.forEach(code => {
			if (!hexColorRegex.test(code)) {
				validHex = false;
			}
		});
		if(validHex) {
			colors = tempColors;
			colorCodes = tempColorCodes;
			$colorStatuses = ["loaded", "loaded", "loaded", "loaded", "loaded"];
			$loading = false
			$totalGenerations++;
			console.log(colors)
			console.log($colorStatuses)
			runningColorErrors=0;
		} else {
			console.log("trying again...")
			runningColorErrors++;
			if(runningColorErrors<3) {
				genColors(input).then(processAPIData);
			}else{
				errorMessage();
			}
		}
	}

	async function genColors(input) {
		let messages =
				[
					{
						role: 'system',
						content: 'You are a color palette generator. You will take in a description of a theme and return exactly 5 colors that match the theme and compliment one another. They will be on separate lines. They will come with the color name in the format: ColorName|ColorHexCode . There will be on text aside from the Color names and color hex codes and the separating lines. They will not be numbered, there will be no other sentences. Try and use softer colors that compliment one another. Make sure every hex code is complete. Avoid overly vibrant colors for the most part'
					},
					{
						role: 'user',
						content: 'Theme: Sailing the ocean waves.'
					},
					{
						role: 'assistant',
						content: 'Ocean Blue|#0077be\nSandy Beach|#f0d9b5\nSeashell White|#fff5ee\nSailboat Gray|#6c7b8b\nCoral Sunset|#ff6f61'
					},
					{
						role: 'user',
						content: 'Theme: '+input
					}]

		;
		try {
			const result = await axios.post("https://nwe0dbevfl.execute-api.us-east-2.amazonaws.com/prod/chat", {
				"messages": messages
			});
			console.log(messages);
			return result.data.content;
		} catch (error) {

			$loading=false
			$totalGenerations++;
			errorMessage();
		}
	}

	function errorMessage() {
		let fullMessage = "Something Went Wrong";
		input = "";
		for (let i = 0; i < fullMessage.length; i++) {
			setTimeout(function () {
				input += fullMessage[i];
			}, 50 * i);
		}
		setTimeout(function () {
			input="";
		}, 2000);
	}

	function checkForEnter(event) {
		if (event.key === 'Enter') {
			handleClick();
		}
	}

	async function animateText() {
		while (!stop) {
			for (let i = 0; i < $placeholders.length; i++) {
				const word = $placeholders[i];

				// Write the word one character at a time
				for (let j = 0; j < word.length; j++) {
					if (stop) return;
					placeholder = placeholder + word.charAt(j);
					await new Promise(resolve => setTimeout(resolve, 10)); // delay
				}

				if (stop) return;

				// Waits a second at the end of the word
				await new Promise(resolve => setTimeout(resolve, 800));

				// Remove the word one character at a time
				for (let j = word.length - 1; j >= 0; j--) {
					if (stop) return;
					placeholder = placeholder.substring(0, j);
					await new Promise(resolve => setTimeout(resolve, 10)); // delay
				}

				if (stop) return;

				// Wait a second before the next word
				await new Promise(resolve => setTimeout(resolve, 800));
			}
		}
	}

	function diceRoll(){
		input = $placeholders[Math.floor(Math.random() * $placeholders.length)];
		handleClick();
	}

</script>





<div class="flex items-center justify-center h-screen bg-transparent relative w-full transform-gpu"
	 style="transform: translate({0}px,{$inputUp-10}px); padding-top: {($inputUp*-1-100)}px;">

	<div class="relative z-30 flex flex-col items-center w-full">

		<div style="transform: translate({0}px,{(($inputUp/300)*-20)-30}px); font-family: {gennedFont}" class="text-6xl sm:text-8xl font-bold text-center w-full">
			paletteBot
		</div>
		{#if $inputUp}
			<div class="text-base sm:text-2xl font-bold mb-5 sm:mb-2 text-center w-full">
				{#if $totalGenerations>0}
					<a href={fontLink} target="_blank" rel="noopener noreferrer" class="">{gennedFont}</a>
				{:else}
					&nbsp;
				{/if}
			</div>
		{/if}

		<div class="w-10/12 sm:w-10/12 md:w-2/3 lg:w-1/2 xl:w-1/3 relative">
			<input class="input w-full py-4 pl-8 pr-24 text-m sm:text-xl" placeholder={placeholder} bind:value={input} on:keydown={checkForEnter}/>

			{#if $loading}
				<div class="absolute right-4 top-1/2 transform-gpu" style="transform: scale({$scale}) translateY(-50%);">
					<ProgressRadial ... stroke={100} meter="stroke-secondary-500" track="stroke-secondary-500/30" class="h-7 w-7 sm:h-8 sm:w-8" />
				</div>
			{:else}
				<button class="absolute right-12 top-1/2 transform-gpu"
						on:click="{diceRoll}" style="transform: scale({$scale}) translateY(-40%);">
					<svg class="w-12 h-12"viewBox="0 0 100 125" version="1.1" x="px" y="0px"><g transform="translate(0,-952.36216)"><path d="m 50.002435,957.36458 c -3.49829,0 -6.32881,1.80127 -11.41427,4.73737 l -17.74689,10.24567 c -5.08544,2.9361 -8.06014,4.48618 -9.80928,7.51575 -1.74913,3.02963 -1.60497,6.38097 -1.60497,12.25312 l 0,20.49131 c 0,5.8722 -0.14424,9.2236 1.60497,12.2531 1.74916,3.0297 4.72382,4.5797 9.80928,7.5158 l 17.74689,10.2457 c 5.08546,2.9361 7.91598,4.7374 11.41427,4.7373 3.49829,10e-5 6.32394,-1.8012 11.4094,-4.7373 l 17.75174,-10.2457 c 5.08546,-2.9361 8.05527,-4.4861 9.80443,-7.5158 1.74913,-3.0296 1.60497,-6.3809 1.60497,-12.2531 l 0,-20.49131 c 0,-5.8722 0.14424,-9.22347 -1.60497,-12.25312 -1.74916,-3.02959 -4.71897,-4.57965 -9.80443,-7.51575 l -17.75174,-10.24567 c -5.08546,-2.9361 -7.91111,-4.73734 -11.4094,-4.73737 z m 0,4.96525 c 1.58717,0 3.84131,1.137 8.92677,4.07307 l 17.75174,10.24567 c 3.01336,1.73978 4.89812,2.9075 6.16779,3.8791 l -23.91953,13.81447 c -5.08546,2.93608 -7.3396,4.06821 -8.92677,4.06821 -1.58717,0 -3.84618,-1.13213 -8.93164,-4.06821 l -23.91952,-13.8096 c 1.26946,-0.97239 3.15482,-2.14161 6.17262,-3.88397 l 17.7469,-10.24567 c 5.08546,-2.9361 7.34447,-4.07302 8.93164,-4.07307 z m -20.04526,14.58543 c -3.30554,1.2e-4 -5.98333,1.54878 -5.98353,3.45725 2.3e-4,1.90844 2.67801,3.45716 5.98353,3.45725 3.30554,-9e-5 5.98816,-1.54878 5.98836,-3.45725 10e-4,-0.93421 -0.65335,-1.83138 -1.81348,-2.48262 -1.11798,-0.62622 -2.61723,-0.97525 -4.17488,-0.97463 z m 20.04526,0 c -3.30552,1.2e-4 -5.98814,1.54881 -5.98837,3.45725 2e-4,1.90847 2.68283,3.45231 5.98837,3.45241 3.30551,-10e-5 5.9833,-1.54397 5.9835,-3.45241 -1.8e-4,-0.89901 -0.6073,-1.76502 -1.69226,-2.40988 -1.12696,-0.66996 -2.67361,-1.04722 -4.29124,-1.04737 z m 20.04039,0 c -3.30551,1.2e-4 -5.98816,1.54881 -5.98836,3.45725 2e-4,1.90847 2.68282,3.45231 5.98836,3.45241 3.30552,-10e-5 5.9833,-1.54397 5.98353,-3.45241 -2e-4,-0.93394 -0.65308,-1.83195 -1.81349,-2.48262 -1.11695,-0.6264 -2.61288,-0.97448 -4.17004,-0.97463 z m 15.28853,7.91336 c 0.20655,1.58598 0.27637,3.80314 0.27637,7.28787 l 0,20.49131 c 0,5.8722 -0.14226,8.396 -0.93583,9.7705 -0.79357,1.3746 -2.90549,2.7614 -7.99095,5.6975 l -17.75174,10.2456 c -3.01492,1.7407 -4.9685,2.79 -6.44415,3.4039 l 0,-27.6192 c 0,-5.8721 0.14226,-8.3959 0.93583,-9.7704 0.79357,-1.3745 2.90547,-2.7614 7.99095,-5.69747 l 23.91952,-13.80961 z m -70.65784,0.005 23.91465,13.80474 c 5.08546,2.93614 7.19736,4.32294 7.99095,5.69744 0.7936,1.3745 0.94069,3.8983 0.94069,9.7705 l 0,27.6192 c -1.47703,-0.614 -3.43411,-1.6633 -6.44901,-3.4039 l -17.7469,-10.2457 c -5.08546,-2.9361 -7.19735,-4.3229 -7.99095,-5.6975 -0.79357,-1.3744 -0.94067,-3.8983 -0.94067,-9.7704 l 0,-20.49138 c 0,-3.48108 0.0737,-5.69717 0.28124,-7.283 z m 65.9835,5.72168 c -0.51951,0.0566 -1.0816,0.25658 -1.65345,0.5964 -1.10094,0.6541 -2.15005,1.77421 -2.92873,3.12267 -1.65266,2.86272 -1.65266,5.95523 0,6.90963 1.65289,0.9541 4.33067,-0.5897 5.98353,-3.45237 1.65266,-2.86272 1.65266,-5.96012 0,-6.91451 -0.40445,-0.23337 -0.88183,-0.31845 -1.40135,-0.26182 z m -41.33183,11.5646 c -0.49779,-0.044 -0.9538,0.047 -1.34313,0.2715 -1.65265,0.9544 -1.65265,4.0469 0,6.9096 1.65286,2.8627 4.33067,4.4114 5.98353,3.4573 1.65266,-0.9544 1.65266,-4.0518 0,-6.9145 -0.80889,-1.4008 -1.90626,-2.5495 -3.04996,-3.1906 -0.55047,-0.3085 -1.0926,-0.4889 -1.59044,-0.5333 z m 21.34961,0 c -0.49779,0.044 -1.03511,0.2199 -1.58558,0.5285 -1.1437,0.641 -2.24588,1.7946 -3.0548,3.1954 -1.65266,2.8627 -1.65266,5.9552 0,6.9096 1.65286,0.954 4.33067,-0.5897 5.98353,-3.4524 1.65266,-2.8627 1.65266,-5.9601 0,-6.9144 -0.38933,-0.2247 -0.84536,-0.3112 -1.34315,-0.2667 z m 20.04042,0 c -0.49779,0.044 -1.03493,0.2199 -1.58558,0.5285 -1.1437,0.641 -2.24588,1.7946 -3.0548,3.1954 -1.65266,2.8627 -1.65266,5.9601 0,6.9145 1.65289,0.954 4.33067,-0.5946 5.98353,-3.4573 1.65266,-2.8627 1.65266,-5.9601 0,-6.9144 -0.38933,-0.2247 -0.84536,-0.3112 -1.34315,-0.2667 z m -61.43043,11.5694 c -0.49781,-0.044 -0.95402,0.046 -1.34315,0.2715 -1.65265,0.9544 -1.65265,4.0518 0,6.9145 1.65289,2.8626 4.33549,4.4064 5.98837,3.4524 1.65266,-0.9544 1.65266,-4.0469 0,-6.9096 -0.80849,-1.4017 -1.91073,-2.5535 -3.0548,-3.1955 -0.55064,-0.3082 -1.09262,-0.4894 -1.59042,-0.5333 z m 41.39001,0 c -0.49779,0.044 -1.03511,0.2249 -1.58558,0.5333 -1.1437,0.6411 -2.24588,1.7898 -3.0548,3.1906 -1.65266,2.8627 -1.65266,5.9601 0,6.9145 1.65289,0.954 4.33064,-0.5946 5.98353,-3.4573 1.65266,-2.8627 1.65266,-5.9552 0,-6.9096 -0.38933,-0.2247 -0.84534,-0.3159 -1.34315,-0.2715 z m 20.04042,0 c -0.49779,0.044 -1.03493,0.2251 -1.58558,0.5333 -1.14407,0.642 -2.24631,1.7938 -3.0548,3.1955 -1.65266,2.8627 -1.65266,5.9552 0,6.9096 1.65289,0.9541 4.33067,-0.5897 5.98353,-3.4524 1.65268,-2.8627 1.65268,-5.9601 0,-6.9145 -0.38913,-0.2254 -0.84536,-0.3155 -1.34315,-0.2715 z m -20.09859,11.5791 c -0.51954,0.056 -1.08143,0.252 -1.65348,0.5916 -1.10129,0.655 -2.15047,1.7782 -2.92873,3.1275 -1.65266,2.8627 -1.65266,5.9552 0,6.9096 1.65286,0.9541 4.33067,-0.5898 5.98353,-3.4524 1.65268,-2.8627 1.65268,-5.9601 0,-6.9145 -0.40422,-0.2341 -0.88181,-0.3179 -1.40132,-0.2618 z" style="color:#FFFFFF;font-style:normal;font-variant:normal;font-weight:normal;font-stretch:normal;font-size:medium;line-height:normal;font-family:sans-serif;text-indent:0;text-align:start;text-decoration:none;text-decoration-line:none;text-decoration-style:solid;text-decoration-color:#FFFFFF;letter-spacing:normal;word-spacing:normal;text-transform:none;direction:ltr;block-progression:tb;writing-mode:lr-tb;baseline-shift:baseline;text-anchor:start;white-space:normal;clip-rule:nonzero;display:inline;overflow:visible;visibility:visible;opacity:1;isolation:auto;mix-blend-mode:normal;color-interpolation:sRGB;color-interpolation-filters:linearRGB;solid-color:#FFFFFF;solid-opacity:1;fill:#FFFFFF;fill-opacity:1;fill-rule:evenodd;stroke:none;stroke-width:2;stroke-linecap:butt;stroke-linejoin:miter;stroke-miterlimit:4;stroke-dasharray:none;stroke-dashoffset:0;stroke-opacity:1;color-rendering:auto;image-rendering:auto;shape-rendering:auto;text-rendering:auto;enable-background:accumulate"/></g></svg>
				</button>
				<button class="absolute right-4 top-1/2 transform-gpu"
						on:click="{handleClick}" style="transform: scale({$scale}) translateY(-50%);">
					<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-8 h-8">
						<path stroke-linecap="round" stroke-linejoin="round" d="M6 12L3.269 3.126A59.768 59.768 0 0121.485 12 59.77 59.77 0 013.27 20.876L5.999 12zm0 0h7.5" />
					</svg>
				</button>
			{/if}
		</div>
	</div>




	{#if $firstLoad}
		<ColorCard position={1} color={colorCodes[0]} colorName={colors[0]} on:swap={swapColor}/>
	{/if}
	{#if $firstLoad}
		<ColorCard position={2} color={colorCodes[1]} colorName={colors[1]} on:swap={swapColor}/>
	{/if}
	{#if $firstLoad}
		<ColorCard position={3} color={colorCodes[2]} colorName={colors[2]} on:swap={swapColor}/>
	{/if}
	{#if $firstLoad}
		<ColorCard position={4} color={colorCodes[3]} colorName={colors[3]} on:swap={swapColor}/>
	{/if}
	{#if $firstLoad}
		<ColorCard position={5} color={colorCodes[4]} colorName={colors[4]} on:swap={swapColor}/>
	{/if}



</div>





