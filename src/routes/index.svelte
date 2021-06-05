<script lang="typescript">
    import { onMount } from 'svelte';
    import { toPixelData } from 'html-to-image';


    class Vector {
        x: number;
        y: number;

        constructor(x, y) {
            this.x = x;
            this.y = y;
        }

        magnitude():number {
            return Math.sqrt(this.x ** 2 + this.y ** 2);
        }

        scale(n:number):Vector {
            return new Vector(this.x * n, this.y * n);
        }

        add(o:Vector):Vector {
            return new Vector(this.x + o.x, this.y + o.y);
        }

        normal():Vector {
            let magnitude = this.magnitude();
            return new Vector(this.x / magnitude, this.y / magnitude);
        }
    };
    type Firefly = {
        pos: Vector,
        vel: Vector,
        perched?: boolean
    }

    const numFireflies:number = 200;
    const numWingmates:number = 40; // max number of wingmates considered by each firefly
    const perceptionRadiusSquared:number = 10000; // radius to look for wingmates within
    const cohesionFactor:number = 0.003; // factor applied to attraction between fireflies
    const spacing = 300; // distance within which fireflies are considered close
    const repulsionFactor:number = 0.02; // factor applied to repulsion between close fireflies
    const flowFactor:number = 0.01; // factor applied to velocity matching of nearby fireflies
    const offscreenSpace:number = 50; // distance allowed offscreen before dV is applied
    const offscreenFactor:number = 3; // dV applied to offscreen fireflies (to return them to the rendered area)
    const maxSpeed = 20;
    const maxSpeedSquared:number = maxSpeed ** 2; // drag is applied above the sqrt of this speed
    const dragFactor:number = 0.9; // TODO: not yet defined
    const simulationSpeed:number = 0.1; // simulation speed applied to dt
    const perchChance = 0.6;
    const unperchChance = 0.01;

    let pagePixels:Uint8ClampedArray;

    let lastTimestamp:DOMHighResTimeStamp;

    let fireflies:Array<Firefly>;

    let sourceNode:HTMLElement;
    let refCanvas:HTMLCanvasElement;
    let refCtx:CanvasRenderingContext2D
    let targetCanvas:HTMLCanvasElement;
    let targetCtx:CanvasRenderingContext2D;

    onMount(() => {
        refCanvas = document.createElement('canvas');
        refCtx = refCanvas.getContext("2d");
        targetCtx = targetCanvas.getContext("2d");
        resizeCanvas();

    });

    function createFireflies():Array<Firefly> {
        let ffs:Array<Firefly> = []; // variable naming lol
        for (let i = 0; i < numFireflies; i++) {
            ffs.push({
                pos: new Vector(targetCanvas.width * Math.random(), targetCanvas.height * Math.random()),
                vel: new Vector(100 * Math.random() - 50, 100 * Math.random() - 50), // TODO: better random starting velocity?
            })
        }
        return ffs;
    }

    function distSquared(a:Firefly, b:Firefly):number {
        return (a.pos.x - b.pos.x) ** 2 + (a.pos.y - b.pos.y) ** 2;
    }

    function avgFf(ffs:Array<Firefly>):Firefly {
        if (!ffs || !ffs.length) {
            return null;
        }
        let avgPos = ffs.reduce((a, b) => a.add(b.pos), new Vector(0, 0)).scale(1 / ffs.length);
        let avgVel = ffs.reduce((a, b) => a.add(b.vel), new Vector(0, 0)).scale(1 / ffs.length);
        return {pos: avgPos, vel: avgVel};
    }

    function getWingmates(f:Firefly):Array<Firefly> {
        // TODO: this algorithm is really, _really_ bad
        let sorted = fireflies.slice();
        sorted.sort((a:Firefly, b:Firefly) => distSquared(f, a) - distSquared(f, b));
        sorted = sorted.filter(o => distSquared(o, f) < perceptionRadiusSquared);
        return sorted.slice(1, numWingmates + 1);
    }

    function getPixelVal(loc:Vector):Uint8ClampedArray {
        let pi = (4 * Math.floor(loc.y) * sourceNode.offsetWidth) + (4 * Math.floor(loc.x));
        return pagePixels.slice(pi, pi + 4);
    }

    function checkPerch() {
        for (let i = 0; i < fireflies.length; i++) {
            let f = fireflies[i];
            if (f.perched || f.pos.x < 5) {
                continue;
            }
            let pxVal = getPixelVal(f.pos);
            if (pxVal[0] > 200 && Math.random() < perchChance) {
                f.perched = true;
                f.vel.x = 0, f.vel.y = 0;
            }
        }
    }

    function applyFormation(dt:DOMHighResTimeStamp) {
        let f:Firefly;
        let wingmates:Array<Firefly> = [];
        for (let i = 0; i < fireflies.length; i++) {
            f = fireflies[i]; // for convenience
            if (f.perched) {
                if (Math.random() < unperchChance) {
                    f.perched = false;
                } else {
                    continue;
                }
            }
            wingmates = getWingmates(f);
            if (!wingmates || wingmates.length == 0) {
                continue;
            }
            let wingCenter = avgFf(wingmates);
            if (!wingCenter) {
                continue;
            }
            

            // cohesion
            let ds = distSquared(f, wingCenter);
            if (ds > spacing) {

                f.vel = f.vel.add(wingCenter.pos.add(f.pos.scale(-1)).scale(cohesionFactor * dt));
            }

            // repulsion
            for (let j = 0; j < wingmates.length; j++) {
                let w = wingmates[j];
                let ds = distSquared(f, w);
                if (ds < spacing) {
                    let dPos = f.pos.add(w.pos.scale(-1));
                    f.vel = f.vel.add(dPos.scale(repulsionFactor * dt));
                }
            }

            // velocity matching
            f.vel = f.vel.add(f.vel.add(wingCenter.vel.scale(1)).scale(flowFactor * dt));
        }
    }

    function applyOffscreenBounce(dt:DOMHighResTimeStamp) {
        for (let i = 0; i < fireflies.length; i++) {
            if (fireflies[i].pos.x < -1 * offscreenSpace) {
                fireflies[i].vel.x += offscreenFactor * dt;
            } else if (fireflies[i].pos.x > targetCanvas.width + offscreenSpace) {
                fireflies[i].vel.x -= offscreenFactor * dt;
            }
            if (fireflies[i].pos.y < -1 * offscreenSpace) {
                fireflies[i].vel.y += offscreenFactor * dt;
            } else if (fireflies[i].pos.y > targetCanvas.height + offscreenSpace) {
                fireflies[i].vel.y -= offscreenFactor * dt;
            }
        }
    }

    // very rough drag; TODO: consider changes here
    function applyDrag(dt:DOMHighResTimeStamp) {
        for (let i = 0; i < fireflies.length; i++) {
            if (fireflies[i].vel.x * fireflies[i].vel.x + fireflies[i].vel.y * fireflies[i].vel.y > maxSpeedSquared) {
                fireflies[i].vel = fireflies[i].vel.normal().scale(maxSpeed);
            }
        }
    }

    function drawFirefly(ctx, f) {
        // const angle = Math.atan2(f.vel.y, f.vel.x);
        // targetCtx.translate(f.pos.x, f.pos.y);
        // targetCtx.rotate(angle);
        // targetCtx.translate(-f.pos.x, -f.pos.y);
        // targetCtx.beginPath();
        // targetCtx.moveTo(f.pos.x, f.pos.y);
        // targetCtx.lineTo(f.pos.x - 15, f.pos.y + 5);
        // targetCtx.lineTo(f.pos.x - 15, f.pos.y - 5);
        // targetCtx.lineTo(f.pos.x, f.pos.y);
        // targetCtx.fill();
        // targetCtx.setTransform(1, 0, 0, 1, 0, 0);
        targetCtx.fillRect(f.pos.x, f.pos.y, 3, 3);
    }

    function renderFrame(timestamp:DOMHighResTimeStamp) {
        if (!lastTimestamp) {
            lastTimestamp = timestamp;
        }
        let dt = (timestamp - lastTimestamp) / 20; // const factor just makes other constants more manageable
        lastTimestamp = timestamp;

        checkPerch();
        applyFormation(dt);
        applyDrag(dt);
        applyOffscreenBounce(dt);

        for (let i = 0; i < fireflies.length; i++) {
            fireflies[i].pos.x += fireflies[i].vel.x * dt * simulationSpeed;
            fireflies[i].pos.y += fireflies[i].vel.y * dt * simulationSpeed;
        }

        targetCtx.clearRect(0, 0, targetCanvas.width, targetCanvas.height);

        // targetCtx.fillStyle = 'blue';
        // for (let i = 0; i < sourceNode.offsetWidth; i++) {
        //     for (let j = 0; j < sourceNode.offsetHeight; j++) {
        //         if (getPixelVal(new Vector(i, j))[0] > 200)
        //         targetCtx.fillRect(i, j, 1, 1);
        //     }
        // }

        // targetCtx.drawImage(refCanvas, 0, 0); // TODO: remove debug
        targetCtx.fillStyle = 'white';
        for (let i = 0; i < fireflies.length; i++) {
            drawFirefly(targetCtx, fireflies[i]);
        }

        window.requestAnimationFrame(renderFrame);
    }

    function resizeCanvas() {
        if (refCtx.canvas.width != targetCanvas.offsetWidth) {refCtx.canvas.width = targetCanvas.offsetWidth;}
        if (refCtx.canvas.height != targetCanvas.offsetHeight) {refCtx.canvas.height = targetCanvas.offsetHeight;}
        if (targetCtx.canvas.width != targetCanvas.offsetWidth) {targetCtx.canvas.width = targetCanvas.offsetWidth;}
        if (targetCtx.canvas.width != targetCanvas.offsetHeight) {targetCtx.canvas.height = targetCanvas.offsetHeight;}
    }

    function updateRef() {
        console.log("updating reference canvas from source!");

        if (!sourceNode || !targetCanvas) {
            return;
        }

        toPixelData(sourceNode, {pixelRatio: 1})
            .then(function (pixels) {
                pagePixels = pixels;
                refCtx.clearRect(0, 0, refCanvas.width, refCanvas.height);
                targetCtx.clearRect(0, 0, targetCanvas.width, targetCanvas.height);
                resizeCanvas();
                fireflies = createFireflies();
                window.requestAnimationFrame(renderFrame);
            })
            .catch(function (error) {
                console.error('oops, something went wrong!', error);
            });
    }
</script>

<main>
    <div id="source" class="test" bind:this={sourceNode} on:click={updateRef}>
        <div id="title"><h1>Test</h1></div>
    </div>
    <canvas id="target" class="test" bind:this={targetCanvas}></canvas>
</main>

<style>
    :global(html, body) {
        background-color: black;
        color: white;
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
        height: 100vh;

        display: flex;
        flex-direction: column;
        align-content: space-around;
    }

    #source {
        border-right: 1px solid white;
    }

    #title {
        margin: 5em auto;
        padding: 1em;
        
        border: 1px solid white;
        border-radius: 100px;
    }

    h1 {
    }
</style>