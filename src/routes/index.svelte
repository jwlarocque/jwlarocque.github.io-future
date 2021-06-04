<script lang="typescript">
    import { onMount } from 'svelte';
    import { toCanvas } from 'html-to-image';

    type Coords = {x: Number, y: Number};
    type Firefly = {
        pos: Coords,
        velocity: Coords
    }
    type BHNode = {
        centerMass: Coords,
        totalMass: Number,
        averageVelocity: Coords,
        nw?: BHNode,
        ne?: BHNode,
        se?: BHNode,
        sw?: BHNode
    }

    let fireflies:Array<Firefly>;

    let sourceNode:HTMLElement;
    let refCanvas:HTMLCanvasElement;
    let refCtx:CanvasRenderingContext2D
    $: if (refCanvas) {
        refCtx = refCanvas.getContext("2d");
    };
    let targetNode:HTMLCanvasElement;
    let targetCtx:CanvasRenderingContext2D;
    $: if (targetNode) {
        targetCtx = targetNode.getContext("2d");
    };

    onMount(() => {
        refCanvas = document.createElement('canvas');
    });

    function createFireflies() {
        const n:Number = 10;

        for (let i = 0; i < n; i++) {
            fireflies.push({
                pos: {x: 0, y: 0},
                velocity: {x: 0, y: 0}
            })
        }
    }

    function renderFrame() {

    }

    function updateRef() {
        console.log("updating reference canvas from source!");

        if (!sourceNode || !targetNode) {
            return;
        }

        toCanvas(sourceNode, {pixelRatio: 1})
            .then(function (canvas) {
                refCtx.canvas.width = targetNode.offsetWidth;
                refCtx.canvas.height = targetNode.offsetHeight;
                targetCtx.canvas.width = targetNode.offsetWidth;
                targetCtx.canvas.height = targetNode.offsetHeight;
                refCtx.drawImage(canvas, 0, 0);
                targetCtx.drawImage(refCanvas, 0, 0); // TODO: remove debug
            })
            .catch(function (error) {
                console.error('oops, something went wrong!', error);
            });
    }
</script>

<main>
    <div id="source" class="test" bind:this={sourceNode} on:click={updateRef}>
        <div id="title"><h1>This Is Only a Test</h1></div>
    </div>
    <canvas id="target" class="test" bind:this={targetNode}></canvas>
</main>

<style>
    :global(html, body) {
        margin: 0;
        padding: 0;

        font-family: Haettenschweiler, 'Arial Narrow Bold', sans-serif;
    }

    main {
        display: flex;
        flex-direction: row;
    }

    .test {
        margin: 0;
        width: 50%;
        height: 150vh;

        display: flex;
        flex-direction: column;
        align-content: space-around;
    }

    #source {
        border-right: 1px solid black;
    }

    #title {
        margin: 5em auto;
        padding: 1em;
        
        border: 3px solid black;
        border-radius: 100px;
    }

    h1 {
    }
</style>