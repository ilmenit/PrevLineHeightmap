# PrevLineHeightmap
## Algorithm to generate infinite heightmap using single line of data
For one project I needed to generate an infinite terrain heightmap. The platform was an old 8bit computer without enough computational power for sophisticated algorithms like Perlin/Simplex noise. Diamond-square algorithm https://en.wikipedia.org/wiki/Diamond-square_algorithm is fast but requires all the heightmap available in the memory therefore cannot be easily used for infinite terrain generation (there are variants for this purpose though https://stackoverflow.com/questions/4977946/making-the-diamond-square-fractal-algorithm-infinite but it would be overkill for this platform).

My first attempt was done by implementing an "old-school" plasma effect, by sum of 2D sine functions of different amplitudes and periods, similarly to https://www.bidouille.org/prog/plasma . The effect however was too regular, and I needed something more random. 

A simple terrain I implemented previously in small, 256 bytes long intro https://www.pouet.net/prod.php?which=86217 where height lines were independent (1D) and were changing naively by adding random [-1,0,+1] value to the height of previous column. Considering straightforward approach (and code size constrains) the effect was fine, but the landscape generated this way was quite flat.

For the new project I wanted steeper and peaky mountains so the local change (let’s call this value delta) of height had to be in range [-3,-3] instead of just [-1,1]. The delta value between heightmap pixels can be totally random, each time between [-3,3], which generates “rough” terrain oscillating around the average value. Let’s write a simple code that generates 1D lines with delta [-3,3] by assigning it value 6*Math.random()-3

```javascript
var canvas = document.getElementById("canvas");
var canvasWidth = canvas.width;
var canvasHeight = canvas.height;
var ctx = canvas.getContext("2d");
const height_max = canvasHeight;

document.getElementById("draw").onclick = draw;

function modify_delta(delta) {
    return 6 * Math.random() - 3;
};

function modify_height(height, delta) {
    var new_height = height + delta;
    return new_height;
}

function draw() {
    var deltaX = 0;
    var x = 0;
    var height = height_max / 2;

    ctx.beginPath();
    ctx.fillStyle = '#AAA';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.beginPath();
    ctx.moveTo(x, height);

    for (var x = 0; x < canvasWidth; ++x) {
        deltaX = modify_delta(deltaX);
        height = modify_height(height, deltaX);
        ctx.lineTo(x, height_max - height);
    }
    ctx.stroke();
}
```
 Generated 1D heightmap is not as good-looking as with midpoint displacement algorithm https://bitesofcode.wordpress.com/2016/12/23/landscape-generation-using-midpoint-displacement/ because of oscillation close to the average value. How can we achieve steeper mountains, still with delta in range [-3,3]? We can add inertia to delta, so the big changes are not instant. Instead of assigning it a random value, we can modify the delta value by [-1,1], keeping it clamped to [-3,3] range. This way we achieve higher, less rough and more sine-like mountains.
```javascript
function modify_delta(delta) {
    const max_delta = 3;
    var r = Math.random();
    if (Math.random() >= 0.5)
        r *= -1;
    delta += r;
    if (delta > max_delta)
        delta = max_delta;
    if (delta < -max_delta)
        delta = -max_delta;
    return delta;
};
```
The effect is better, however we need to clamp also the height. Simple clamping of height to [0,height_max] is not enough, because the mountains become flat reaching the height_max.
```javascript
function modify_height(height, delta) {
    var new_height = height + delta;
    if (new_height < 0)
        new_height = 0;
    else if (new_height > height_max)
        new_height = height_max;
    return new_height;
}
```
We need an improved clamping. The aim was to have peaky mountains and to achieve it we can revert the delta value when it reaches the height_max. The higher the delta was before reverting, the peakier the mountain will be.
```javascript
function modify_height(height, delta) {
    var new_height = height + delta;
    var new_delta = delta;
    if (new_height < 0) {
        new_height = 0;
        // delta = delta; // more islands
        // delta=0; // less water
        // delta = -delta; // even less water
    } else if (new_height > height_max) {
        new_height = height_max;
        new_delta = -delta; // peaks
    }
    return [new_height, new_delta];
}
```
Having 1D generation we can move to 2D. The main constrain stays as we want to keep the steepness (delta value) in range [3,3], but in 2D on both axes X and Y.
Lets start with the full algorithm:
```javascript
var deltas = new Array(canvasWidth);

// init first line
for (var x = 0; x < canvasWidth; ++x) {
    deltaX = modify_delta(deltaX);
    deltas[x] = deltaX;
    values = modify_height(height, deltaX);
    height = values[0];
    deltaX = values[1];
    setHeight(x, 0, height);
}

// the rest of heightmap

for (var y = 1; y < canvasHeight; ++y) {
    deltaX = deltas[0];
    for (var x = 0; x < canvasWidth; ++x) {
        // Step 1. Modify delta
        deltaY = deltas[x];

        // if equal, we can modify it
        if (deltaY == deltaX) {
            deltaX = modify_delta(deltaX);
        } else {
            // if not equal, then becomes one of them
            // delta we have from previous X, so it may become one from y
            if (Math.random() >= 0.5)
                deltaX = deltaY;
        }

        // Step 2. set height
        height = getHeight((x > 0 ? x - 1 : 0), y - 1); // clamped x-1
        height += getHeight((x < canvasWidth ? x + 1 : canvasWidth), y - 1); // clamped x+1
        height /= 2;

        var v = modify_height(height, deltaX);
        height = v[0];
        deltaX = v[1];
        setHeight(x, y, height);
        deltas[x] = deltaX;
    }
}
```
The algorithm is reading data only from the last generated line and is using one extra array for deltas, with size equal width of the map:
```javascript
var deltas = new Array(canvasWidth);
```
The algorithm first fills the first line, exactly like we did for the 1D heightmap:
```javascript
var deltas = new Array(canvasWidth);

// init first line
for (var x = 0; x < canvasWidth; ++x) {
    deltaX = modify_delta(deltaX);
    deltas[x] = deltaX;
    values = modify_height(height, deltaX);
    height = values[0];
    deltaX = values[1];
    setHeight(x, 0, height);
}
```

Then we are using data from the first line to generate the next lines as follows:
1.	If previous deltaX in the current line equals deltaY (from the previous, line for X), we can modify it by adding [-1,1].
2.	If previous deltaX is not equal deltaY (from the line Y-1 for X), then we set deltaX as one of these deltas with 50% probability.
```javascript
        // Step 1. Modify delta
        deltaY = deltas[x];

        // if equal, we can modify it
        if (deltaY == deltaX) {
            deltaX = modify_delta(deltaX);
        } else {
            // if not equal, then becomes one of them
            // delta we have from previous X, so it may become one from y
            if (Math.random() >= 0.5)
                deltaX = deltaY;
        }
```
Then we are setting the height as averaged heights of points (x-1,y-1) and (x+1,y-1) with added newly set delta value:
```javascript
        // Step 2. set height
        height = getHeight((x > 0 ? x - 1 : 0), y - 1); // clamped x-1
        height += getHeight((x < canvasWidth ? x + 1 : canvasWidth), y - 1); // clamped x+1
        height /= 2;

        var v = modify_height(height, deltaX);
        height = v[0];
        deltaX = v[1];
        setHeight(x, y, height);
```
Let’s look at the algorithm visualized.
We are filling the first line heights (big black numbers) and deltas (small red numbers).

![step1](https://github.com/ilmenit/PrevLineHeightmap/blob/main/2d-1.png)

In the first row, the deltaX is set to be equal deltaY, therefore we modify it (here from 0 to -1).

![step2](https://github.com/ilmenit/PrevLineHeightmap/blob/main/2d-2.png)

Second step is setting of height, which is averaged value of 2 cells from previous line (14+15)/2 = 14, with added delta (-1) = 13.

![step3](https://github.com/ilmenit/PrevLineHeightmap/blob/main/2d-3.png)

In the next cell, previous deltas vertically and horizontally are not qual, therefore with 50% probability we set deltaX as one of them (here as deltaY, so deltaX = +1).

![step4](https://github.com/ilmenit/PrevLineHeightmap/blob/main/2d-4.png)

Then we are setting the height as averaged height of cells from previous line (14+16)/2 = 15, modified by deltaX = 16.

![step5](https://github.com/ilmenit/PrevLineHeightmap/blob/main/2d-5.png)

For the next cell previous deltas are equal, therefore we modify deltaX by [-1,1], here by +1 to +2.

![step6](https://github.com/ilmenit/PrevLineHeightmap/blob/main/2d-6.png)

The height is averaged value from previous cells modified by delta:

![step7](https://github.com/ilmenit/PrevLineHeightmap/blob/main/2d-7.png)

Algorithm continues until the whole heightmap is filled.

![finished](https://github.com/ilmenit/PrevLineHeightmap/blob/main/canvas.png)

For 3D visualization of the heightmap I modified terrain example from 
 [“PlayfulJS Terrain” by Hunter Loftis](http://www.playfuljs.com/realistic-terrain-in-130-lines/)

![3d](https://github.com/ilmenit/PrevLineHeightmap/blob/main/index.png)

The algorithm generates bumpy terrain without local details, which was preferred for my purpose. For extra details a second “rough” layer could be added to the generated heightmap, using the same algorithm and delta values between [-1,1] instead of [-3,3].
