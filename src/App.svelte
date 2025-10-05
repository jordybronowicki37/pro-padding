<script lang="ts">
  import {onMount} from 'svelte';

  type Background = {
    type: 'solid',
    color: string
  } | {
    type: 'gradient',
    option: string
  } | {
    type: 'image',
    file: string
  }

  const solidBackgrounds = [
    '#131311',
    '#333031',
    '#ffffff',
    '#eaeaea',
    '#e52b36',
    '#f7b2a6',
    '#f7791f',
    '#fdbc64',
    '#f09f1b',
    '#f8d983',
    '#128544',
    '#99e6b0',
    '#0c7eec',
    '#9ecff5',
    '#8032f8',
    '#ceacf8'
  ]

  let fileInput: HTMLInputElement;
  let previewCanvas: HTMLCanvasElement;
  let ctxPreview: CanvasRenderingContext2D;

  let image = new Image();
  let originalWidth = 0;
  let originalHeight = 0;

  // UI state
  let tolerance: number = 4;
  let padding: number = 20;
  let margin: number = 40;
  let borderThickness: number = 0;
  let borderColor: string = '#000000';
  let borderRadius: number = 20;
  let background: Background = {type: 'solid', color: '#ff0000'};

  let croppedCanvas: HTMLCanvasElement;
  let ctxCropped: CanvasRenderingContext2D;

  let croppedRect: { left: number; top: number; width: number; height: number } | null = null;
  let paddingColor: [number, number, number, number] = [255, 255, 255, 255];

  onMount(() => {
    ctxPreview = previewCanvas.getContext('2d')!;
    croppedCanvas = document.createElement('canvas');
    ctxCropped = croppedCanvas.getContext('2d')!;
    // TODO: for testing purposes
    handleFile(null)
  });

  function handleFileEvent(e: Event) {
    const target = e.target as HTMLInputElement;
    const f = target.files?.[0];
    if (!f) return;
    handleFile(f)
  }

  function handleFile(f: File | null) {
    image = new Image();
    image.onload = () => {
      originalWidth = image.width;
      originalHeight = image.height;
      croppedCanvas.width = originalWidth;
      croppedCanvas.height = originalHeight;
      ctxCropped.clearRect(0, 0, croppedCanvas.width, croppedCanvas.height);
      ctxCropped.drawImage(image, 0, 0);
      detectAndCrop();
      drawPreview();
    };
    // TODO: for testing purposes
    if (f === null) {
        image.src = '../public/test.png';
    } else {
        image.src = URL.createObjectURL(f);
    }
  }

  function colorDistance(a: number[], b: number[]): number {
    return Math.sqrt(
      (a[0] - b[0]) ** 2 +
      (a[1] - b[1]) ** 2 +
      (a[2] - b[2]) ** 2
    );
  }

  function getPixel(data: Uint8ClampedArray, width: number, x: number, y: number): number[] {
    const i = (y * width + x) * 4;
    return [data[i], data[i + 1], data[i + 2], data[i + 3]];
  }

  function rowMatches(data: Uint8ClampedArray, width: number, y: number, borderColor: number[], tol: number): boolean {
    for (let x = 0; x < width; x++) {
      const px = getPixel(data, width, x, y);
      if (colorDistance(px, borderColor) > tol) return false;
    }
    return true;
  }

  function colMatches(data: Uint8ClampedArray, width: number, height: number, x: number, borderColor: number[], tol: number): boolean {
    for (let y = 0; y < height; y++) {
      const px = getPixel(data, width, x, y);
      if (colorDistance(px, borderColor) > tol) return false;
    }
    return true;
  }

  function detectAndCrop() {
    const w = croppedCanvas.width;
    const h = croppedCanvas.height;
    const imgData = ctxCropped.getImageData(0, 0, w, h);
    const data = imgData.data;

    const corners = [getPixel(data, w, 0, 0), getPixel(data, w, w - 1, 0), getPixel(data, w, 0, h - 1), getPixel(data, w, w - 1, h - 1)];
    let borderColor = [0, 0, 0, 255];
    for (let c of corners) {
      borderColor[0] += c[0];
      borderColor[1] += c[1];
      borderColor[2] += c[2];
    }
    borderColor[0] = Math.round(borderColor[0] / 4);
    borderColor[1] = Math.round(borderColor[1] / 4);
    borderColor[2] = Math.round(borderColor[2] / 4);

    let top = 0, bottom = h - 1, left = 0, right = w - 1;
    const tol = Number(tolerance);

    while (top <= bottom && rowMatches(data, w, top, borderColor, tol)) top++;
    while (bottom >= top && rowMatches(data, w, bottom, borderColor, tol)) bottom--;
    while (left <= right && colMatches(data, w, h, left, borderColor, tol)) left++;
    while (right >= left && colMatches(data, w, h, right, borderColor, tol)) right--;

    if (right < left || bottom < top) {
      croppedRect = {left: 0, top: 0, width: w, height: h};
      paddingColor = [borderColor[0], borderColor[1], borderColor[2], 255];
      return;
    }

    const cw = right - left + 1;
    const ch = bottom - top + 1;

    paddingColor = [borderColor[0], borderColor[1], borderColor[2], 255];

    croppedRect = {left, top, width: cw, height: ch};
    const croppedData = ctxCropped.getImageData(left, top, cw, ch);
    croppedCanvas.width = cw;
    croppedCanvas.height = ch;
    ctxCropped.putImageData(croppedData, 0, 0);
  }

  function setSolidBackground(color: string) {
    background = {type: 'solid', color: color};
    drawPreview()
  }

  function drawBackground(ctx: CanvasRenderingContext2D, w: number, h: number) {
    switch (background.type) {
      case 'solid':
        ctx.fillStyle = background.color;
        ctx.fillRect(0, 0, w, h);
        break;
      // case 'linear-gradient':
      //   const gradient = ctx.createLinearGradient(0, 0, w, h);
      //   gradient.addColorStop(0, '#ff9a9e');
      //   gradient.addColorStop(1, '#fad0c4');
      //   ctx.fillStyle = gradient;
      //   ctx.fillRect(0, 0, w, h);
      //   break;
      // case 'radial-gradient':
      //   const radial = ctx.createRadialGradient(w/2, h/2, 10, w/2, h/2, w/2);
      //   radial.addColorStop(0, '#a1c4fd');
      //   radial.addColorStop(1, '#c2e9fb');
      //   ctx.fillStyle = radial;
      //   ctx.fillRect(0, 0, w, h);
      //   break;
      // case 'blobs':
      //   ctx.fillStyle = '#54dcef';
      //   ctx.fillRect(0, 0, w, h);
      //   for (let i = 0; i < 5; i++) {
      //     const x = Math.random() * w;
      //     const y = Math.random() * h;
      //     const r = 100 + Math.random() * 150;
      //     const blobGradient = ctx.createRadialGradient(x, y, 0, x, y, r);
      //     blobGradient.addColorStop(0, `hsla(${Math.random()*360}, 70%, 70%, 0.5)`);
      //     blobGradient.addColorStop(1, 'transparent');
      //     ctx.fillStyle = blobGradient;
      //     ctx.beginPath();
      //     ctx.arc(x, y, r, 0, Math.PI * 2);
      //     ctx.fill();
      //   }
      //   break;
    }
  }

  function drawPreview() {
    if (!croppedRect) return;
    const imgW = croppedCanvas.width;
    const imgH = croppedCanvas.height;

    const outW = imgW + padding * 2 + borderThickness * 2 + margin * 2;
    const outH = imgH + padding * 2 + borderThickness * 2 + margin * 2;

    previewCanvas.width = outW;
    previewCanvas.height = outH;

    ctxPreview.clearRect(0, 0, outW, outH);
    drawBackground(ctxPreview, outW, outH);

    // Draw padding area with border-radius
    ctxPreview.save();
    ctxPreview.beginPath();
    ctxPreview.roundRect(margin + borderThickness, margin + borderThickness, imgW + padding * 2, imgH + padding * 2, borderRadius);
    ctxPreview.clip();
    ctxPreview.fillStyle = `rgba(${paddingColor[0]},${paddingColor[1]},${paddingColor[2]},${paddingColor[3] / 255})`;
    ctxPreview.fillRect(margin + borderThickness, margin + borderThickness, imgW + padding * 2, imgH + padding * 2);

    // Draw image inside padding
    ctxPreview.drawImage(croppedCanvas, margin + borderThickness + padding, margin + borderThickness + padding, imgW, imgH);
    ctxPreview.restore();

    if (borderThickness > 0) {
      ctxPreview.lineWidth = borderThickness;
      ctxPreview.strokeStyle = borderColor;
      ctxPreview.roundRect(margin + borderThickness / 2, margin + borderThickness / 2, imgW + padding * 2 + borderThickness - 1, imgH + padding * 2 + borderThickness - 1, borderRadius);
      ctxPreview.stroke();
    }
  }

  function applyAndExport() {
    drawPreview();
    const dataUrl = previewCanvas.toDataURL('image/png');
    const a = document.createElement('a');
    a.href = dataUrl;
    a.download = 'autocrop.png';
    document.body.appendChild(a);
    a.click();
    a.remove();
  }

  async function copyToClipboard() {
    try {
      const blob = await new Promise<Blob | null>(resolve => previewCanvas.toBlob(resolve, 'image/png'));
      if (!blob) return;
      await navigator.clipboard.write([
        new ClipboardItem({ 'image/png': blob })
      ]);
    } catch (err) {
      console.error('Failed to copy image:', err);
    }
  }
</script>

<style>
  .app {
    font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial;
    padding: 1rem;
    max-width: 980px;
    margin: auto;
  }

  .controls {
    display: flex;
    gap: 1rem;
    flex-wrap: wrap;
  }

  .panel {
    background: #eee;
    padding: 1rem;
    border-radius: 8px;
    flex: 1;
  }

  canvas {
    border: 1px solid #ddd;
    max-width: 100%;
    height: auto;
    display: block;
  }

  label {
    display: block;
    font-size: 0.9rem;
    margin-bottom: 0.25rem;
  }

  #solid-backgrounds {
    display: grid;
    grid-template-rows: 1fr 1fr;
    grid-auto-flow: column;
    width: min-content;
    gap: 5px;
  }

  #solid-backgrounds * {
    background: var(--bg-color);
    width: 20px;
    height: 20px;
    border: 1px solid #333;
    border-radius: 10px;
    padding: 0;

    &:hover {
      border: 2px solid #111;
    }
  }

  #solid-backgrounds .transparent {
    background: conic-gradient(#fff 90deg, #ccc 90deg, #ccc 180deg, #fff 180deg, #fff 270deg, #ccc 270deg);
  }

  #solid-backgrounds input[type="color"] {
    background: conic-gradient(#f00, #ff0, #0f0, #0ff, #00f, #f0f);
    &::-webkit-color-swatch-wrapper {
      display: none;
    }
    &::-webkit-color-swatch {
      display: none;
    }
    &::-moz-color-swatch {
      display: none;
    }
  }
</style>

<div class="app">
    <h1>Pro-Padding</h1>
    <div class="controls">
        <div class="panel">
            <label for="file-input">Load image</label>
            <input id="file-input" bind:this={fileInput} type="file" accept="image/*" on:change={handleFileEvent}/>

            <label for="gradient-backgrounds">Gradient Backgrounds</label>
            <div id="gradient-backgrounds">

            </div>

            <label for="solid-backgrounds">Solid Backgrounds</label>
            <div id="solid-backgrounds">
                {#each solidBackgrounds as solidBackground}
                    <button style="--bg-color:{solidBackground}" on:click={() => setSolidBackground(solidBackground)} aria-label="color: {solidBackground}"></button>
                {/each}
                <button class="transparent" style="--bg-color:#0000" on:click={() => setSolidBackground('#0000')} aria-label="color: #0000"></button>
                <input id="solid-background-color-input" type="color"
                       on:input={(e) => setSolidBackground(e.currentTarget.value)}
                       on:click={(e) => setSolidBackground(e.currentTarget.value)}/>
            </div>

            <label for="margin-input">Margin: {margin}px</label>
            <input id="margin-input" type="range" min="0" max="200" bind:value={margin} on:input={drawPreview}/>

            <label for="padding-input">Padding: {padding}px</label>
            <input id="padding-input" type="range" min="0" max="100" bind:value={padding} on:input={drawPreview}/>

            <label for="border-radius-input">Border radius: {borderRadius}px</label>
            <input id="border-radius-input" type="range" min="0" max="60" bind:value={borderRadius} on:input={drawPreview}/>

            <div class="button-group">
                <button on:click={applyAndExport}>Download</button>
                <button on:click={copyToClipboard}>Copy</button>
            </div>
        </div>

        <div class="panel">
            <label for="preview-canvas">Preview</label>
            <canvas id="preview-canvas" bind:this={previewCanvas}></canvas>
        </div>
    </div>
</div>
