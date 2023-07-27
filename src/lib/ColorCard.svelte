<script>

    import {fly, fade} from "svelte/transition";
    import {colorStatuses, firstLoad, totalGenerations} from "../stores.js";
    import {tweened} from "svelte/motion";
    import {cubicOut, backInOut, cubicIn} from "svelte/easing";
    import {createEventDispatcher, onMount} from 'svelte';
    const xPositionMaker = tweened(0, { duration: 1750, easing: cubicOut });
    const yPositionMaker = tweened(0, { duration: 1750, easing: cubicOut });
    let scalePulse = tweened(1, { duration: 300, easing: cubicOut });
    let copying=false;
    let swapping = false;

    const dispatch = createEventDispatcher();


    export let position = 3;
    export let color = "#F0F0F0";

    export let textColor = "black"

    export let colorName = "Color Name";

    let xOffset = 0;
    let yOffset = 0;
    let screenWidth = window.innerWidth;
    let cardGap = 10;

    let bigGap = 400;

    let cardWidth = 270;
    let cardHeight = 360;

    let widthClass = `w-[${cardWidth}px]`;
    let heightClass = `h-[${cardHeight}px]`;

    if(position === 1) xOffset = -600;
    if(position === 2) xOffset = -300;
    if(position === 4) xOffset = 300;
    if(position === 5) xOffset = 600;


    $: setReadableTextColor(color);

    $: setPosition(screenWidth);

    $: widthClass = `w-[${cardWidth}px]`;
    $: heightClass = `h-[${cardHeight}px]`;

    setPosition(screenWidth);


    function setPosition(){
        if(450 < screenWidth && screenWidth < 1350){
            if(position === 1){
                xOffset = -105-(cardGap/2);
                yOffset = 0;
            }
            if(position === 2){
                xOffset = 105+(cardGap/2);
                yOffset = 0;
            }
            if(position === 3){
                xOffset = -105-(cardGap/2);
                yOffset = 280+cardGap;
            }
            if(position === 4){
                xOffset = 105+(cardGap/2);
                yOffset = 280+cardGap;
            }
            if(position === 5){
                xOffset = 0;
                yOffset = 560+cardGap*2;
            }
            cardWidth = screenWidth;
            cardHeight = screenWidth/3 * (360/270);
            bigGap = 260;

        }else if(screenWidth <= 450) {
            if (position === 1) {
                xOffset = -75 - (cardGap / 2);
                yOffset = 0;
            }
            if (position === 2) {
                xOffset = 75 + (cardGap / 2);
                yOffset = 0;
            }
            if (position === 3) {
                xOffset = -75 - (cardGap / 2);
                yOffset = 200 + cardGap;
            }
            if (position === 4) {
                xOffset = 75 + (cardGap / 2);
                yOffset = 200 + cardGap;
            }
            if (position === 5) {
                xOffset = 0;
                yOffset = 400 + cardGap * 2;
            }
            cardWidth = screenWidth/3;
            cardHeight = screenWidth / 2 * (360 / 270);
            bigGap = 260;
        }
        else{
            if(position === 1){
                xOffset = -600;
                yOffset = 0;
            }
            if(position === 2){
                xOffset = -300;
                yOffset = 0;
            }
            if(position === 3){
                xOffset = 0;
                yOffset = 0;
            }
            if(position === 4){
                xOffset = 300;
                yOffset = 0;
            }
            if(position === 5){
                xOffset = 600;
                yOffset = 0;
            }
            bigGap = 400;
        }
        setTimeout(function(){
            xPositionMaker.set(xOffset);
        }, 1050);

        setTimeout(function(){
            yPositionMaker.set(yOffset);
        }, 1050);
    }

    function updateWidth() {
        screenWidth = window.innerWidth;
    }

    onMount(() => {
        window.addEventListener('resize', updateWidth);
        updateWidth(); // Initial update

        // Remove event listener on component destroy
        return () => {
            window.removeEventListener('resize', updateWidth);
        };
    });

    function X() {
        swapping=true;

        dispatch('swap', {
            text: position-1
        });


            scalePulse = tweened(1, {duration: 200, easing: cubicOut});
            scalePulse.set(0.93);
            setTimeout(function () {
                scalePulse = tweened(0.93, {duration: 200, easing: cubicIn});
                scalePulse.set(1);
                setTimeout(function () {
                    $colorStatuses[position - 1] = "loading";
                    swapping=false;
                }, 200);
            }, 200);
    }



    console.log(widthClass)

    function setReadableTextColor(hexColor) {
        // convert hex color code to RGB
        let r = parseInt(hexColor.slice(1, 3), 16);
        let g = parseInt(hexColor.slice(3, 5), 16);
        let b = parseInt(hexColor.slice(5, 7), 16);

        // determine brightness of the color using a common formula
        let brightness = Math.round(((r * 299) + (g * 587) + (b * 114)) / 1000);

        // return black for bright colors and white for dark colors
        textColor = (brightness > 125) ? 'black' : 'white';
    }

    async function clickedCard() {
        scalePulse = tweened(1, {duration: 200, easing: cubicOut});
        scalePulse.set(0.93);
        setTimeout(function () {
            scalePulse = tweened(0.93, {duration: 200, easing: cubicIn});
            scalePulse.set(1);
        }, 200);
        if (swapping) return;
        console.log("clicked card")
        copying = true;
        try {
            await navigator.clipboard.writeText(color);
            console.log('Text copied to clipboard');
        } catch (err) {
            console.error('Failed to copy text: ', err);
            // Fallback for older browsers or in case the Clipboard API fails
            try {
                const textarea = document.createElement('textarea');
                textarea.value = color;
                document.body.appendChild(textarea);
                textarea.select();
                document.execCommand('copy');
                document.body.removeChild(textarea);
                console.log('Text copied to clipboard using fallback');
            } catch (err) {
                console.error('Failed to copy text using fallback: ', err);
            }
        }
        setTimeout(function () {
            copying = false;
        }, 200);
    }

</script>

<style>
    .background-animate {
        background-size: 400%;

        -webkit-animation: AnimationName 3s ease infinite;
        -moz-animation: AnimationName 3s ease infinite;
        animation: AnimationName 3s ease infinite;
    }

    @keyframes AnimationName {
        0%,
        100% {
            background-position: 0% 50%;
        }
        50% {
            background-position: 100% 50%;
        }
    }


</style>



{#if $firstLoad && $colorStatuses[position-1] === "loading"}
    <div style="transform: translate({$xPositionMaker}px,{bigGap+$yPositionMaker}px) scale({$scalePulse})" in:fly={$totalGenerations===0 ? {y: -300, duration: 1000} : {}} class="absolute z-10 {screenWidth<1350 ? (screenWidth<450 ? 'h-[200px]' : 'h-[280px]') : 'h-[360px]'} {screenWidth<1350 ? (screenWidth<450 ? 'w-[150px]' : 'w-[210px]') : 'w-[270px]'} background-animate bg-gradient-to-r from-pink-500 via-red-500 to-yellow-500 rounded-3xl flex flex-col justify-end p-2 shadow-sm opacity-50"></div>
{/if}
{#if $colorStatuses[position-1] === "loaded"}

    <div on:click={clickedCard} style="transform: translate({$xPositionMaker}px,{bigGap+$yPositionMaker}px) scale({$scalePulse});  background-color: {color}" class="absolute {screenWidth<1350 ? (screenWidth<450 ? 'h-[200px]' : 'h-[280px]') : 'h-[360px]'} {screenWidth<1350 ? (screenWidth<450 ? 'w-[150px]' : 'w-[210px]') : 'w-[270px]'} rounded-3xl flex flex-col justify-end p-2 shadow-sm">
    <button class="absolute top-2 right-2" on:click={X}>
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke={textColor} class="h-6 w-6 opacity-50">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
        </svg>
    </button>
    <p style = "color: {textColor}" class="text-sm">{colorName}</p>
    <p style = "color: {textColor}" class="text-sm">{color}</p>


        {#if copying}
            <div
                    class="opacity-30 absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 w-1/4 h-1/4"
                    style="transform: translate({-80}px,{-80}px)"
                    in:fade="{{ duration: 200 }}"
                    out:fade="{{ duration: 200 }}"
            >
                <svg width="160px" height="160px" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                    <path d="M6 11C6 8.17157 6 6.75736 6.87868 5.87868C7.75736 5 9.17157 5 12 5H15C17.8284 5 19.2426 5 20.1213 5.87868C21 6.75736 21 8.17157 21 11V16C21 18.8284 21 20.2426 20.1213 21.1213C19.2426 22 17.8284 22 15 22H12C9.17157 22 7.75736 22 6.87868 21.1213C6 20.2426 6 18.8284 6 16V11Z" stroke={textColor} stroke-width="1.5"/>
                    <path d="M6 19C4.34315 19 3 17.6569 3 16V10C3 6.22876 3 4.34315 4.17157 3.17157C5.34315 2 7.22876 2 11 2H15C16.6569 2 18 3.34315 18 5" stroke={textColor} stroke-width="1.5"/>
                </svg>
            </div>
        {/if}

</div>

{/if}
