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
  let fileDialog: HTMLDialogElement

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
  let background: Background = {type: 'gradient', style: 'purple'};

  let croppedCanvas: HTMLCanvasElement;
  let ctxCropped: CanvasRenderingContext2D;

  let croppedRect: { left: number; top: number; width: number; height: number } | null = null;
  let paddingColor: [number, number, number] = [255, 255, 255];

  onMount(() => {
    ctxPreview = previewCanvasElement.getContext('2d')!;
    croppedCanvas = document.createElement('canvas');
    ctxCropped = croppedCanvas.getContext('2d', { willReadFrequently: true })!;
    fileDialog.showModal()
  });

  function handleFileEvent(e: Event) {
    const target = e.target as HTMLInputElement;
    const f = target.files?.[0];
    if (!f) return;
    handleFile(f)
    fileDialog.close()
  }

  function handleFile(f: File) {
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
    image.src = URL.createObjectURL(f);
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
      paddingColor = [borderColor[0], borderColor[1], borderColor[2]];
      return;
    }

    const cw = right - left + 1;
    const ch = bottom - top + 1;

    paddingColor = [borderColor[0], borderColor[1], borderColor[2]];

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

  function squircleRect(ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number, r: number) {
    const SQUIRCLE = 0.25;
    const c = r * SQUIRCLE;

    ctx.beginPath();
    ctx.moveTo(x + r, y);

    ctx.lineTo(x + w - r, y);
    ctx.bezierCurveTo(x + w - c, y, x + w, y + c, x + w, y + r);

    ctx.lineTo(x + w, y + h - r);
    ctx.bezierCurveTo(x + w, y + h - c, x + w - c, y + h, x + w - r, y + h);

    ctx.lineTo(x + r, y + h);
    ctx.bezierCurveTo(x + c, y + h, x, y + h - c, x, y + h - r);

    ctx.lineTo(x, y + r);
    ctx.bezierCurveTo(x, y + c, x + c, y, x + r, y);

    ctx.closePath();
  }

  function drawPreview() {
    if (!croppedRect) return;

    // Set variables and constants
    const imgW = croppedCanvas.width;
    const imgH = croppedCanvas.height;
    padding = imgW * (paddingPercentage / 100)
    margin = imgW * (marginPercentage / 100)
    borderRadius = imgW * (borderRadiusPercentage / 100)
    const newOutputW = imgW + padding * 2 + borderThickness * 2 + margin * 2;
    const newOutputH = imgH + padding * 2 + borderThickness * 2 + margin * 2;
    const totalMargin = margin + borderThickness;
    const boxW = imgW + padding * 2;
    const boxH = imgH + padding * 2;

    // Setup canvas
    previewCanvasElement.width = newOutputW;
    previewCanvasElement.height = newOutputH;

    // Draw background
    ctxPreview.clearRect(0, 0, newOutputW, newOutputH);
    drawBackground();

    // Draw shadow shape
    ctxPreview.save();
    ctxPreview.shadowColor = 'rgba(0,0,0,0.5)';
    ctxPreview.shadowBlur = 20;
    ctxPreview.shadowOffsetX = imgW * 0.02;
    ctxPreview.shadowOffsetY = imgH * 0.05;
    squircleRect(ctxPreview, totalMargin, totalMargin, boxW, boxH, borderRadius);
    ctxPreview.fillStyle = '#000';
    ctxPreview.fill();
    ctxPreview.restore();

    // Draw padding box
    ctxPreview.save();
    squircleRect(ctxPreview, totalMargin, totalMargin, boxW, boxH, borderRadius);
    ctxPreview.clip();
    ctxPreview.fillStyle = `rgb(${paddingColor[0]},${paddingColor[1]},${paddingColor[2]})`;
    ctxPreview.fillRect(totalMargin, totalMargin, boxW, boxH);
    ctxPreview.restore();

    // Draw image
    ctxPreview.drawImage(croppedCanvas, totalMargin + padding, totalMargin + padding, imgW, imgH);
    ctxPreview.restore();

    // Draw border around image
    if (borderThickness > 0) {
      ctxPreview.lineWidth = borderThickness;
      ctxPreview.strokeStyle = borderColor;
      squircleRect(ctxPreview, totalMargin, totalMargin, boxW, boxH, borderRadius);
      ctxPreview.stroke();
      ctxPreview.restore();
    }
  }

  async function applyAndExport() {
    drawPreview();

    if ('showSaveFilePicker' in window) {
      const blob = await new Promise<Blob>((resolve) =>
        previewCanvasElement.toBlob((b) => resolve(b!), 'image/png')
      );
      // @ts-ignore
      const handle = await window.showSaveFilePicker({
        suggestedName: 'pro-padding-image.png',
        types: [
          {
            description: 'PNG Image',
            accept: { 'image/png': ['.png'] }
          }
        ]
      });
      const writable = await handle.createWritable();
      await writable.write(blob);
      await writable.close();
    } else { // Fallback for browsers that do not support window.showSaveFilePicker
      const dataUrl = previewCanvasElement.toDataURL('image/png');
      const a = document.createElement('a');
      a.href = dataUrl;
      a.download = 'pro-padding-image.png';
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(dataUrl)
    }
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
  #preview-canvas {
    max-height: 90vh;
  }
  #main-image-input, #gradient-backgrounds>* {
    corner-shape: squircle;
  }
  #github-container {
    corner-top-left-shape: squircle;
  }
  #solid-background-transparent {
    background: conic-gradient(#fff 90deg, #ccc 90deg, #ccc 180deg, #fff 180deg, #fff 270deg, #ccc 270deg);
  }
  #upload-file-dialog::backdrop {
    backdrop-filter: blur(10px);
    background: #0004;
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
  #gradient-backgrounds {
    .pink {
      background: linear-gradient(55deg, #f68375, #e95a9f 50%, #2f225e) !important;
    }
    .purple {
      background: linear-gradient(130deg, #192357, #b372cd) !important;
    }
    .night {
      background: linear-gradient(130deg, #010b46, #1f85af) !important;
    }
    .ocean {
      background: linear-gradient(35deg, #4fbbbe, #8fcd9f) !important;
    }
    .red {
      background: linear-gradient(65deg, #fb9857, #ee7496) !important;
    }
    .bright-pink {
      background: linear-gradient(135deg, #f9acc3, #d67b99) !important;
    }
  }
</style>

<div class="h-screen relative flex items-center bg-gray-950">
  <div class="bg-slate-800 border border-slate-500 border-l-0 rounded-r-lg">
    <div class="p-4">
      <h1 class="text-3xl font-bold mb-4 text-white">Pro-Padding</h1>

      <label for="file-input" class="block text-sm mb-1 text-slate-100">Load image</label>
      <button id="main-image-input" class="w-8 h-8 rounded-xl border border-dashed border-gray-700 bg-center bg-no-repeat bg-cover" style="background-image:url('{image !== null ? image?.src : '/image-preview.png'}')" on:click={() => imageElement.click()} aria-label="Image input"></button>
      <input hidden id="file-input" type="file" accept="image/*" bind:this={imageElement} on:change={handleFileEvent} />

      <label for="gradient-backgrounds" class="block text-sm mt-4 mb-1 text-slate-100">Gradient Backgrounds</label>
      <div id="gradient-backgrounds" class="flex gap-1 w-min">
        <button class="cursor-pointer w-8 h-8 rounded-xl border border-gray-700 pink" on:click={() => setGradientBackground('pink')} aria-label="Gradient background: pink"></button>
        <button class="cursor-pointer w-8 h-8 rounded-xl border border-gray-700 purple" on:click={() => setGradientBackground('purple')} aria-label="Gradient background: purple"></button>
        <button class="cursor-pointer w-8 h-8 rounded-xl border border-gray-700 night" on:click={() => setGradientBackground('night')} aria-label="Gradient background: night"></button>
        <button class="cursor-pointer w-8 h-8 rounded-xl border border-gray-700 ocean" on:click={() => setGradientBackground('ocean')} aria-label="Gradient background: ocean"></button>
        <button class="cursor-pointer w-8 h-8 rounded-xl border border-gray-700 red" on:click={() => setGradientBackground('red')} aria-label="Gradient background: red"></button>
        <button class="cursor-pointer w-8 h-8 rounded-xl border border-gray-700 bright-pink" on:click={() => setGradientBackground('bright-pink')} aria-label="Gradient background: bright-pink"></button>

        <button class="w-8 h-8 rounded-xl border border-dashed border-gray-700 bg-center bg-no-repeat bg-cover" style="background-image:url('{background.type === 'image' ? background.image?.src : '/image-preview.png'}')" on:click={() => customBackgroundImageElement.click()} aria-label="Custom background image"></button>
        <input hidden type="file" accept="image/*" bind:this={customBackgroundImageElement} on:change={setImageBackground} />
      </div>

      <label for="solid-backgrounds" class="block text-sm mt-4 mb-1 text-slate-100">Solid Backgrounds</label>
      <div id="solid-backgrounds" class="grid grid-rows-2 grid-flow-col gap-1 w-min">
        {#each solidBackgrounds as solidBackground}
          <button class="cursor-pointer w-5 h-5 rounded-xl border border-gray-700" style="background:{solidBackground}" on:click={() => setSolidBackground(solidBackground)} aria-label="color: {solidBackground}"></button>
        {/each}

        <button id="solid-background-transparent" class="cursor-pointer w-5 h-5 rounded-xl border border-gray-700" aria-label="color: #0000" on:click={() => setSolidBackground('#0000')}></button>

        <input id="solid-background-color-input" type="color" class="cursor-pointer w-5 h-5 rounded-xl border border-gray-700" on:input={(e) => setSolidBackground(e.currentTarget.value)} />
      </div>

      <label for="margin-input" class="block text-sm mt-4 mb-1 text-slate-100">Margin</label>
      <input id="margin-input" type="range" step="0.1" min="0" max="20" bind:value={marginPercentage} on:input={drawPreview} />

      <label for="padding-input" class="block text-sm mt-4 mb-1 text-slate-100">Padding</label>
      <input id="padding-input" type="range" step="0.1" min="0" max="20" bind:value={paddingPercentage} on:input={drawPreview} />

      <label for="border-radius-input" class="block text-sm mt-4 mb-1 text-slate-100">Corner</label>
      <input id="border-radius-input" type="range" step="0.1" min="0" max="20" bind:value={borderRadiusPercentage} on:input={drawPreview} />
    </div>

    <div class="grid grid-cols-2 mt-2 border-t border-t-slate-500">
      <button class="cursor-pointer p-4 border-r border-r-slate-500 hover:bg-slate-700 text-slate-100" on:click={applyAndExport}>Download</button>
      <button class="cursor-pointer p-4 hover:bg-slate-700 text-slate-100" on:click={copyToClipboard}>Copy</button>
    </div>
  </div>

  <div class="m-12 flex-1 grid place-items-center">
    <div class="w-[70%] relative">
      <label for="preview-canvas" class="absolute text-white left-0 p-1 font-bold select-none">Preview</label>
      <canvas id="preview-canvas" bind:this={previewCanvasElement} class="border border-slate-600 w-full"></canvas>
    </div>
  </div>

  <div id="github-container" class="absolute bottom-0 right-0 text-white text-sm p-2 bg-slate-800 hover:bg-slate-700 border-t border-l border-slate-500 rounded-tl-xl">
    <a class="hover:underline" href="https://github.com/jordybronowicki37/pro-padding" target="_blank">View project on Github</a>
  </div>
</div>

<dialog bind:this={fileDialog} id="upload-file-dialog" class="place-self-center border border-slate-500 rounded-xl" closedby="none">
  <div class="w-96 p-4 bg-slate-800 flex flex-col items-center">
    <h2 class="text-white text-2xl">Upload a file</h2>
    <p class="text-white text-sm mt-2 mb-2">To start editing your screenshot, you will need to upload the screenshot which you want to upload.</p>
    <button class="cursor-pointer text-white p-1 hover:bg-slate-700 border border-slate-500 rounded" on:click={() => imageElement.click()}>Upload file</button>
  </div>
</dialog>
