<script lang="ts">
    import type { NumericRange } from "@sveltejs/kit";
    import { onMount } from "svelte";

    class FreeSpaces {
        array: Array<boolean>;
        size: number;

        constructor(size: number) {
            this.array = new Array(size * size).fill(false);
            this.size = size;
        }

        empty() {
            this.array.fill(false);
        }

        indexFromXY(x: number, y: number) {
            if (x + 1 > this.size || y + 1 > this.size) {
                throw new Error("Index out of bounds");
            }
            return x + y * this.size;
        }

        mask(x: number, y: number) {
            const index = this.indexFromXY(x, y);
            this.array[index] = true;
        }

        isHit(x: number, y: number) {
            const index = this.indexFromXY(x, y);
            return this.array[index];
        }

        x(i: number) {
            return i % this.size;
        }

        y(i: number) {
            // coerces the float into an int, Math.trunc() can also be used
            return (i / this.size) | 0;
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

    // simple one way linked list
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
    const interval = 1000;

    const head = new SnakeNode(5, 5);
    head.next = new SnakeNode(4, 5);
    head.next.next = new SnakeNode(3, 5);

    const freeSpaces = new FreeSpaces(gridSize);

    let loopId: number;

    const squareSize = ratio * 0.9;

    let direction = new Vec2(1, 0);

    function keyListener(event: KeyboardEvent) {
        // event.preventDefault();
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
        freeSpaces.empty();
        let x = head.x;
        let y = head.y;

        let nx = x + direction.x;
        let ny = y + -direction.y;

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

        let next = head.next;
        while (next) {
            nx = next.x;
            ny = next.y;
            next.x = x;
            next.y = y;
            freeSpaces.mask(x, y);
            x = nx;
            y = ny;
            next = next.next;
        }

        if (freeSpaces.isHit(head.x, head.y)) {
            console.log("DEATH");
        }
    }

    function drawSquare(ctx: CanvasRenderingContext2D, x: number, y: number) {
        ctx.beginPath();
        ctx.rect(
            x * ratio + ratio / 2 - squareSize / 2,
            y * ratio + ratio / 2 - squareSize / 2,
            squareSize,
            squareSize,
        );
        ctx.fill();
    }

    function drawApple(ctx: CanvasRenderingContext2D, x: number, y: number) {
        ctx.beginPath();
        ctx.roundRect(
            x * ratio + ratio / 2 - squareSize / 2,
            y * ratio + ratio / 2 - squareSize / 2,
            squareSize,
            squareSize,
            20,
        );
        ctx.fill();
    }

    function genApple(ctx) {
        const ar: number[] = [];
        freeSpaces.array.forEach((b, i) => {
            if (!b) {
                ar.push(i);
            }
        });
        const index = Math.floor(Math.random() * ar.length);
        const r = ar[index];
        ctx.fillStyle = "blue";
        drawApple(ctx, ...freeSpaces.xy(r));
    }

    // variable canvas only gets bound once the HTML loads, using it before will give you undefined,
    // which is why we have to wait for the html to "mount"
    onMount(() => {
        if (!canvas.getContext("2d"))
            throw new Error("Canvas is either undefined or context is null");
        const ctx = canvas.getContext("2d") as CanvasRenderingContext2D;

        document.addEventListener("keydown", keyListener);

        function setup() {}

        function main() {
            ctx.fillStyle = "green";
            for (let i = 0; i < gridSize; i++) {
                for (let j = 0; j < gridSize; j++) {
                    drawSquare(ctx, i, j);
                }
            }

            move();
            genApple(ctx);

            ctx.fillStyle = "red";
            drawSquare(ctx, head.x, head.y);
            let next = head.next;
            while (next) {
                drawSquare(ctx, next.x, next.y);
                next = next.next;
            }

            ctx.fillStyle = "orange";
            freeSpaces.array.forEach((b, i) => {
                if (b) {
                    drawSquare(ctx, ...freeSpaces.xy(i));
                }
            });
        }

        main();
        loopId = setInterval(() => {
            main();
        }, interval);
    });

    // when running the vite development server, HMR is enabled, meaning every time you save a file
    // it reruns all the code fresh. this line of code runs before a rerun, cleaning up any intervals/loops/whatever
    // so they don't persist through reloads
    if (import.meta.hot) {
        import.meta.hot.dispose(() => {
            document.removeEventListener("keydown", keyListener);
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
