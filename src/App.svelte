<script lang="ts">
  import {onMount} from 'svelte';

  type Background = {
    type: 'solid',
    color: string
  } | {
    type: 'gradient',
    style: string
  } | {
    type: 'image',
    image: HTMLImageElement
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

  let previewCanvasElement: HTMLCanvasElement;
  let imageElement: HTMLInputElement;
  let customBackgroundImageElement: HTMLInputElement;
  let ctxPreview: CanvasRenderingContext2D;

  let image: HTMLImageElement;
  let originalWidth = 0;
  let originalHeight = 0;

  // UI state
  const tolerance: number = 4;
  let paddingPercentage: number = 10;
  let padding: number = 0;
  let marginPercentage: number = 10;
  let margin: number = 0;
  let borderThickness: number = 0;
  let borderColor: string = '#000000';
  let borderRadiusPercentage: number = 10;
  let borderRadius: number = 0;
  let background: Background = {type: 'solid', color: '#ff0000'};

  let croppedCanvas: HTMLCanvasElement;
  let ctxCropped: CanvasRenderingContext2D;

  let croppedRect: { left: number; top: number; width: number; height: number } | null = null;
  let paddingColor: [number, number, number, number] = [255, 255, 255, 255];

  onMount(() => {
    ctxPreview = previewCanvasElement.getContext('2d')!;
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
        image.src = '../test.png';
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
    drawPreview();
  }

  function setGradientBackground(style: string) {
    background = {type: 'gradient', style};
    drawPreview();
  }

  function setImageBackground(event: Event) {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (!file) return;

    const image = new Image();
    image.onload = () => {
      background = {type: "image", image};
      drawPreview();
    };
    image.src = URL.createObjectURL(file);
  }

  function getRotatedGradient(angle: number): CanvasGradient {
    angle += 90
    const w = previewCanvasElement.width
    const h = previewCanvasElement.height
    const rad = (angle * Math.PI) / 180;
    const x0 = w / 2 + Math.cos(rad) * w;
    const y0 = h / 2 + Math.sin(rad) * h;
    const x1 = w / 2 - Math.cos(rad) * w;
    const y1 = h / 2 - Math.sin(rad) * h;
    return ctxPreview.createLinearGradient(x0, y0, x1, y1);
  }

  function drawGradientBackground(style: string) {
    const w = previewCanvasElement.width
    const h = previewCanvasElement.height
    let gradient: CanvasGradient;

    switch (style) {
      case 'pink':
        gradient = getRotatedGradient(55)
        gradient.addColorStop(0, '#f68375');
        gradient.addColorStop(0.5, '#e95a9f');
        gradient.addColorStop(1, '#2f225e');
        break
      case 'purple':
        gradient = getRotatedGradient(130)
        gradient.addColorStop(0, '#192357');
        gradient.addColorStop(1, '#b372cd');
        break
      case 'night':
        gradient = getRotatedGradient(130)
        gradient.addColorStop(0, '#010b46');
        gradient.addColorStop(1, '#1f85af');
        break
      case 'ocean':
        gradient = getRotatedGradient(35)
        gradient.addColorStop(0, '#4fbbbe');
        gradient.addColorStop(1, '#8fcd9f');
        break
      case 'red':
        gradient = getRotatedGradient(65)
        gradient.addColorStop(0, '#fb9857');
        gradient.addColorStop(1, '#ee7496');
        break
      case 'bright-pink':
      default:
        gradient = getRotatedGradient(45)
        gradient.addColorStop(0, '#f9acc3');
        gradient.addColorStop(1, '#d67b99');
    }
    ctxPreview.fillStyle = gradient;
    ctxPreview.fillRect(0, 0, w, h);
  }

  function drawBackground() {
    const ctx = ctxPreview
    const canvas = previewCanvasElement
    const w = canvas.width
    const h = canvas.height
    switch (background.type) {
      case 'solid':
        ctx.fillStyle = background.color;
        ctx.fillRect(0, 0, w, h);
        break;
      case "gradient":
        drawGradientBackground(background.style)
        break;
      case 'image':
        const img = background.image
        const canvasRatio = canvas.width / canvas.height;
        const imgRatio = img.width / img.height;
        let drawWidth, drawHeight, offsetX, offsetY;

        if (imgRatio > canvasRatio) { // Image is wider — fit height, crop sides
          drawHeight = canvas.height;
          drawWidth = img.width * (canvas.height / img.height);
          offsetX = (canvas.width - drawWidth) / 2;
          offsetY = 0;
        } else { // Image is taller — fit width, crop top/bottom
          drawWidth = canvas.width;
          drawHeight = img.height * (canvas.width / img.width);
          offsetX = 0;
          offsetY = (canvas.height - drawHeight) / 2;
        }

        ctx.drawImage(img, offsetX, offsetY, drawWidth, drawHeight);
        break;
        // kept as examples for different types of backgrounds
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

    padding = imgW * (paddingPercentage / 100)
    margin = imgW * (marginPercentage / 100)
    borderRadius = imgW * (borderRadiusPercentage / 100)

    const outW = imgW + padding * 2 + borderThickness * 2 + margin * 2;
    const outH = imgH + padding * 2 + borderThickness * 2 + margin * 2;

    previewCanvasElement.width = outW;
    previewCanvasElement.height = outH;

    ctxPreview.clearRect(0, 0, outW, outH);
    drawBackground();

    // Draw padding area with border-radius
    ctxPreview.save();
    ctxPreview.beginPath();
    // TODO change to squircle
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
    const dataUrl = previewCanvasElement.toDataURL('image/png');
    const a = document.createElement('a');
    a.href = dataUrl;
    a.download = 'autocrop.png';
    document.body.appendChild(a);
    a.click();
    a.remove();
  }

  async function copyToClipboard() {
    try {
      const blob = await new Promise<Blob | null>(resolve => previewCanvasElement.toBlob(resolve, 'image/png'));
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
  #main-image-input, #gradient-backgrounds>* {
    corner-shape: squircle;
  }

  #solid-background-transparent {
    background: conic-gradient(#fff 90deg, #ccc 90deg, #ccc 180deg, #fff 180deg, #fff 270deg, #ccc 270deg);
  }

  #solid-background-color-input {
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

<div class="h-screen flex items-center">
    <div class="bg-gray-200 p-4 rounded-r-lg">
        <h1 class="text-3xl font-bold mb-4">Pro-Padding</h1>

        <label for="file-input" class="block text-sm mb-1">Load image</label>
        <button id="main-image-input" class="w-8 h-8 rounded-xl border border-dashed border-gray-700 bg-center bg-no-repeat bg-cover" style="background-image:url('{image !== null ? image?.src : '/image-preview.png'}')" on:click={() => imageElement.click()} aria-label="Image input"></button>
        <input hidden id="file-input" type="file" accept="image/*" bind:this={imageElement} on:change={handleFileEvent} />

        <label for="gradient-backgrounds" class="block text-sm mt-4 mb-1">Gradient Backgrounds</label>
        <div id="gradient-backgrounds" class="flex gap-1 w-min">
            <button class="w-8 h-8 rounded-xl border border-gray-700 bg-linear-55 from-[#f68375] via-[#e95a9f] to-[#2f225e]" on:click={() => setGradientBackground('pink')} aria-label="Gradient background: pink"></button>
            <button class="w-8 h-8 rounded-xl border border-gray-700 bg-linear-130 from-[#192357] to-[#b372cd]" on:click={() => setGradientBackground('purple')} aria-label="Gradient background: purple"></button>
            <button class="w-8 h-8 rounded-xl border border-gray-700 bg-linear-130 from-[#010b46] to-[#1f85af]" on:click={() => setGradientBackground('night')} aria-label="Gradient background: night"></button>
            <button class="w-8 h-8 rounded-xl border border-gray-700 bg-linear-35 from-[#4fbbbe] to-[#8fcd9f]" on:click={() => setGradientBackground('ocean')} aria-label="Gradient background: ocean"></button>
            <button class="w-8 h-8 rounded-xl border border-gray-700 bg-linear-65 from-[#fb9857] to-[#ee7496]" on:click={() => setGradientBackground('red')} aria-label="Gradient background: red"></button>
            <button class="w-8 h-8 rounded-xl border border-gray-700 bg-linear-135 from-[#f9acc3] to-[#d67b99]" on:click={() => setGradientBackground('bright-pink')} aria-label="Gradient background: bright-pink"></button>

            <button class="w-8 h-8 rounded-xl border border-dashed border-gray-700 bg-center bg-no-repeat bg-cover" style="background-image:url('{background.type === 'image' ? background.image?.src : '/image-preview.png'}')" on:click={() => customBackgroundImageElement.click()} aria-label="Custom background image"></button>
            <input hidden type="file" accept="image/*" bind:this={customBackgroundImageElement} on:change={setImageBackground} />
        </div>

        <label for="solid-backgrounds" class="block text-sm mt-4 mb-1">Solid Backgrounds</label>
        <div id="solid-backgrounds" class="grid grid-rows-2 grid-flow-col gap-1 w-min">
            {#each solidBackgrounds as solidBackground}
                <button class="w-5 h-5 rounded-xl border border-gray-700" style="background:{solidBackground}" on:click={() => setSolidBackground(solidBackground)} aria-label="color: {solidBackground}"></button>
            {/each}

            <button id="solid-background-transparent" class="w-5 h-5 rounded-xl border border-gray-700" aria-label="color: #0000" on:click={() => setSolidBackground('#0000')}></button>

            <input id="solid-background-color-input" type="color" class="w-5 h-5 rounded-xl border border-gray-700" on:input={(e) => setSolidBackground(e.currentTarget.value)} />
        </div>

        <label for="margin-input" class="block text-sm mt-4 mb-1">Margin</label>
        <input id="margin-input" type="range" step="0.1" min="0" max="20" bind:value={marginPercentage} on:input={drawPreview} />

        <label for="padding-input" class="block text-sm mt-4 mb-1">Padding</label>
        <input id="padding-input" type="range" step="0.1" min="0" max="20" bind:value={paddingPercentage} on:input={drawPreview} />

        <label for="border-radius-input" class="block text-sm mt-4 mb-1">Corner</label>
        <input id="border-radius-input" type="range" step="0.1" min="0" max="20" bind:value={borderRadiusPercentage} on:input={drawPreview} />

        <div class="flex gap-2 mt-4">
            <button class="px-3 py-1 bg-gray-300 rounded hover:bg-gray-400" on:click={applyAndExport}>Download</button>
            <button class="px-3 py-1 bg-gray-300 rounded hover:bg-gray-400" on:click={copyToClipboard}>Copy</button>
        </div>
    </div>

    <div class="m-12 flex-1 grid place-items-center">
        <div class="w-[70%] relative">
            <label for="preview-canvas" class="absolute left-0 p-1 font-bold select-none">Preview</label>
            <canvas id="preview-canvas" bind:this={previewCanvasElement} class="border border-gray-300 w-full"></canvas>
        </div>
    </div>
</div>
