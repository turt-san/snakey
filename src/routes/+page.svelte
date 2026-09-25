<script lang="ts">
    import { onMount } from "svelte";

    let canvas;

    const width = 600;
    const height = 600;

    onMount(() => {
        const ctx = canvas.getContext("2d");
        ctx.fillStyle = "green";
        ctx.fillRect(10, 10, 150, 100);
    });
</script>

<canvas id="game" bind:this={canvas} {width} {height}></canvas>

<style>
    #game {
        border: 5px solid black;
    }
</style>
