<script lang="ts">
    import type { NumericRange } from "@sveltejs/kit";
    import { onMount } from "svelte";

    class Spaces {
        array: Array<boolean>;
        size: number;

        constructor(size: number) {
            this.array = new Array(size * size).fill(false);
            this.size = size;
        }

        x(i: number) {
            return i % gridSize;
        }

        y(i: number) {
            return (i / gridSize) | 0;
        }

        xy(i: number) {
            return [this.x(i), this.y(i)];
        }
    }

    class Vec2 {
        x: number;
        y: number;

        constructor(x: number, y: number) {
            this.x = x;
            this.y = y;
        }

        xy(x: number, y: number) {
            this.x = x;
            this.y = y;
        }
    }

    class SnakeNode {
        x: number;
        y: number;
        next: SnakeNode | null;

        constructor(x: number, y: number) {
            this.x = x;
            this.y = y;
            this.next = null;
        }
    }

    let canvas: HTMLCanvasElement;

    const canvasSize = 600;

    const gridSize = 12;

    const ratio = canvasSize / gridSize;

    const head = new SnakeNode(1, 2);
    const xs = new Int32Array([1]);
    const ys = new Int32Array([2]);

    const one = new Spaces(gridSize);
    one.array[12] = true;

    let loopId: number;

    const squareSize = 48;

    let direction = new Vec2(1, 0);

    function setDirection(event: KeyboardEvent) {
        event.preventDefault();
        console.log(event);
        switch (event.code) {
            case "ArrowUp":
            case "KeyW":
                direction.xy(0, 1);
                break;
            case "ArrowLeft":
            case "KeyA":
                direction.xy(-1, 0);
                break;
            case "ArrowDown":
            case "KeyS":
                direction.xy(0, -1);
                break;
            case "ArrowRight":
            case "KeyD":
                direction.xy(1, 0);
                break;
        }
    }

    function move() {
        const x = head.x;
        const y = head.y;

        const nx = x + direction.x;
        const ny = y + -direction.y;

        if (nx > gridSize - 1) {
            head.x = 0;
        } else if (nx < 0) {
            head.x = gridSize - 1;
        } else {
            head.x = nx;
        }

        if (ny > gridSize - 1) {
            head.y = 0;
        } else if (ny < 0) {
            head.y = gridSize - 1;
        } else {
            head.y = ny;
        }
    }

    // variable canvas only gets bound once the HTML loads, using it before will give you undefined,
    // which is why we have to wait for the html to "mount"
    onMount(() => {
        const ctx = canvas.getContext("2d");
        if (!ctx) {
            throw new Error("Canvas is null or context is unavailable");
        }

        document.addEventListener("keydown", setDirection);

        function drawSquare(x: number, y: number) {
            if (!ctx) return;
            ctx.fillRect(
                x * ratio + ratio / 2 - squareSize / 2,
                y * ratio + ratio / 2 - squareSize / 2,
                squareSize,
                squareSize,
            );
        }

        function setup() {}
        function main() {
            ctx.fillStyle = "green";
            for (let i = 0; i < gridSize; i++) {
                for (let j = 0; j < gridSize; j++) {
                    drawSquare(i, j);
                }
            }

            move();
            ctx.fillStyle = "red";

            drawSquare(head.x, head.y);

            one.array.forEach((b, i) => {
                if (b) {
                    drawSquare(...one.xy(i));
                }
            });
        }

        main();
        loopId = setInterval(() => {
            main();
        }, 1000);
    });

    // when running the vite development server, HMR is enabled, meaning every time you save a file
    // it reruns all the code fresh. this line of code runs before a rerun, cleaning up any intervals/loops/whatever
    // so they don't persist through reloads
    if (import.meta.hot) {
        import.meta.hot.dispose(() => {
            document.removeEventListener("keydown", setDirection);
            if (loopId) {
                clearInterval(loopId);
            }
        });
    }
</script>

<canvas id="game" bind:this={canvas} width={canvasSize} height={canvasSize}
></canvas>

<style>
    #game {
        border: 5px solid black;
    }
</style>
