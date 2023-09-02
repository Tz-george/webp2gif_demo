<script setup lang="ts">
import { ref, reactive, computed } from "vue";
import JSZip from 'jszip'
import {saveAs} from 'file-saver';
import WebPXMux, {Frame, Frames} from 'webpxmux'
import GIF from 'gif.js'

const xMux = WebPXMux("./webpxmux.wasm")

const type = {
  mimeType: 'image/gif',
  ext: '.gif'
}
const converted: { fileName: string, blob: Blob }[] = []

async function onFileSelect(event: InputEvent) {
  const files = (event.target as HTMLInputElement).files
  if (!files || files.length < 1) {
    return
  }
  await Promise.all(Array.from(files).map((file: File) => convert(file)))
  save()
}

function fileReader(file: File): Promise<Uint8Array> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = () => {
      resolve(new Uint8Array(reader.result as ArrayBuffer))
    }
    reader.onerror = () => {
      reject(reader.error)
    }
    reader.readAsArrayBuffer(file)
  })
}

async function convert(file: File) {
  const frames = await decode(file)
  const fileName = file.name.split('.').shift()!
  const blob = await drawFrames(frames, fileName)
  return {fileName, blob}
}

// 解码
async function decode(file: File) {
  await xMux.waitRuntime()
  const data = await fileReader(file)
  const frames = await xMux.decodeFrames(data)
  return frames
}

function drawFrames(frames: Frames, fileName: string) {
  if (frames.frameCount < 1) {
    return
  }
  return new Promise((resolve) => {
    const height = frames.height
    const width = frames.width
    const canvas = canvasFactory(width, height)
    const ctx = canvas.getContext("2d")!

    const gif = new GIF({
      workerScript: "./gif.worker.js",
      workers: 2,
      quality: 10,
      width,
      height,
    })
    for (let i = 0; i < frames.frameCount; i++) {
      const frame = frames.frames[i]
      ctx.clearRect(0, 0, width, height)
      const {r, g, b, a} = getRGBA(frames.bgColor)
      ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${a})`
      ctx.fillRect(0, 0, width, height)
      drawFrame(frame, width, height, ctx)
      gif.addFrame(ctx, {copy: true, delay: i === 0 ? 0.00000001 : frame.duration})
    }
    gif.on('finished', function (blob: Blob) {
      converted.push({ fileName, blob })
      resolve({
        fileName,
        blob,
      })
    })
    gif.render()
  })
}

function canvasFactory(width: number, height: number): HTMLCanvasElement {
  const canvas = document.createElement("canvas")
  canvas.height = height
  canvas.width = width
  return canvas
}

function drawFrame(frame: Frame, width: number, height: number, ctx: CanvasRenderingContext2D) {
  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const bitColor = frame.rgba[x + y * width]
      const {r, g, b, a} = getRGBA(bitColor)
      ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${a})`
      ctx.fillRect(x, y, 1, 1)
    }
  }
}

function canvasToImage(canvas: HTMLCanvasElement) {
  const image = new Image(canvas.width, canvas.height)
  image.src = canvas.toDataURL('image/gif', 1)
  return image
}

function getRGBA(uint32: number): { r: number, g: number, b: number, a: number } {
  const r = (uint32 & (0xFF << 24)) >>> 24
  const g = (uint32 & (0xFF << 16)) >>> 16
  const b = (uint32 & (0xFF << 8)) >>> 8
  const a = (uint32 & (0xFF)) / 255.0

  return {
    r, g, b, a
  }
}

function save() {
  const zip = new JSZip()
  for (const convert of converted) {
    zip.file(convert.fileName + type.ext, convert.blob)
  }
  zip.generateAsync({type: 'blob'})
      .then((content) => {
        saveAs(content, 'results.zip')
      })
}

function saveBlob(blob: Blob, fileName: string) {
  console.log(blob, fileName, type);
  const url = URL.createObjectURL(blob);
  const linker = document.createElement("a");
  linker.href = url;
  linker.download = fileName + type.ext;
  document.body.appendChild(linker);
  linker.click();
  document.body.removeChild(linker);
  URL.revokeObjectURL(url);
}
</script>

<template>
  <input type="file" multiple @input="onFileSelect"/>
</template>
