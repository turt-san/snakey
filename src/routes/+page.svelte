<script lang="ts">
    import type { NumericRange } from "@sveltejs/kit";
    import { onMount } from "svelte";

    // Uint8Array is more efficient, because it guarantees each member ofthe array is exactly 1 byte
    // with how JavaScript works, a boolean can take up to 4 bytes
    class FreeSpaces {
        array: Uint8Array;
        size: number;

        constructor(size: number) {
            this.array = new Array(size * size).fill(false);
            this.size = size;
        }

        empty() {
            this.array.fill(0);
        }

        indexFromXY(x: number, y: number) {
            if (x + 1 > this.size || y + 1 > this.size) {
                throw new Error("Index out of bounds");
            }
            return x + y * this.size;
        }

        mask(x: number, y: number, mode: number = 1) {
            const index = this.indexFromXY(x, y);
            this.array[index] = mode;
        }

        isHit(x: number, y: number) {
            const index = this.indexFromXY(x, y);
            return this.array[index];
        }

        x(i: number) {
            return i % this.size;
        }

        y(i: number) {
            // bitwise or coerces the float into an int, Math.trunc() can also be used
            return (i / this.size) | 0;
        }

        xy(i: number): [number, number] {
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

        get xy() {
            return [this.x, this.y];
        }

        set xy(ar: [number, number]) {
            this.x = ar[0];
            this.y = ar[1];
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
    const loopInterval = 1000;

    const freeSpaces = new FreeSpaces(gridSize);

    const head = new SnakeNode(5, 5);
    head.next = new SnakeNode(4, 5);
    head.next.next = new SnakeNode(3, 5);

    let loopId: number;

    let growSize = 0;

    let score = $state(0);

    const apples: Vec2[] = [];

    const squareSize = ratio * 0.9;

    let direction = new Vec2(1, 0);

    let appleRequired = true;

    let death = false;

    function keyListener(event: KeyboardEvent) {
        // event.preventDefault();
        switch (event.code) {
            case "ArrowUp":
            case "KeyW":
                direction.xy = [0, 1];
                break;
            case "ArrowLeft":
            case "KeyA":
                direction.xy = [-1, 0];
                break;
            case "ArrowDown":
            case "KeyS":
                direction.xy = [0, -1];
                break;
            case "ArrowRight":
            case "KeyD":
                direction.xy = [1, 0];
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

        apples.forEach((vec, i) => {
            if (vec.x === head.x && vec.y === head.y) {
                apples.splice(i, 1);
                growSize += 1;
                score += 1;
            }
            freeSpaces.mask(...vec.xy, 3);
        });

        let next = head.next;
        while (next) {
            nx = next.x;
            ny = next.y;
            next.x = x;
            next.y = y;
            freeSpaces.mask(x, y);
            x = nx;
            y = ny;
            if (next.next === null && growSize > 0) {
                const newNode = new SnakeNode(x, y);
                next.next = newNode;
                next = next.next;
                growSize -= 1;
            } else {
                next = next.next;
            }
        }

        switch (freeSpaces.isHit(head.x, head.y)) {
            case 1:
                death = true;
                break;
            case 3:
                console.log("apple");
                break;
        }

        freeSpaces.mask(head.x, head.y, 2);
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
        console.log("drawn apple");
    }

    function genApple() {
        if (appleRequired) {
            const ar: number[] = [];
            freeSpaces.array.forEach((b, i) => {
                if (b === 0) {
                    ar.push(i);
                }
            });
            const index = Math.floor(Math.random() * ar.length);
            const r = ar[index];
            apples.push(new Vec2(...freeSpaces.xy(r)));
        }
    }

    function gameOver(ctx: CanvasRenderingContext2D) {
        ctx.fillStyle = "magenta";
        ctx.fillRect(0, 0, canvasSize, canvasSize);
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
            move();
            if (death) {
                gameOver(ctx);
                clearInterval(loopId);
                return;
            }
            if (apples.length < 1) {
                genApple();
            }

            drawSquare(ctx, head.x, head.y);

            freeSpaces.array.forEach((n, i) => {
                switch (n) {
                    case 0:
                        ctx.fillStyle = "green";
                        drawSquare(ctx, ...freeSpaces.xy(i));
                        break;
                    case 1:
                        ctx.fillStyle = "orange";
                        drawSquare(ctx, ...freeSpaces.xy(i));
                        break;
                    case 2:
                        ctx.fillStyle = "#24f404";
                        drawSquare(ctx, ...freeSpaces.xy(i));
                        break;
                    case 3:
                        ctx.fillStyle = "green";
                        drawSquare(ctx, ...freeSpaces.xy(i));
                        ctx.fillStyle = "blue";
                        drawApple(ctx, ...freeSpaces.xy(i));
                        break;
                }
            });
        }

        main();
        loopId = setInterval(() => {
            main();
        }, loopInterval);
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

<h1 id="counter">{score}</h1>
<canvas id="game" bind:this={canvas} width={canvasSize} height={canvasSize}
></canvas>

<style>
    #game {
        border: 5px solid black;
    }
</style>
