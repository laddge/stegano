<script lang="ts">
  import { encode as b64Encode, decode as b64Decode, fromUint8Array, toUint8Array } from 'js-base64'
  import { gzip, ungzip } from 'pako'

  let text: string = ''
  let files: FileList
  let canvas: HTMLCanvasElement
  let ctx: CanvasRenderingContext2D
  let output: string = ''
  let err: string = ''

  const encode = (text: string) => {
    if (text == '') return
    const b64 = b64Encode(text) + '='
    const gzipped = '=' + fromUint8Array(gzip(text)) + '='
    if (b64.length > gzipped.length) {
      return gzipped
    } else {
      return b64
    }
  }

  const decode = (encoded: string) => {
    if (encoded.slice(0, 1) == '=') {
      return new TextDecoder().decode(ungzip(toUint8Array(encoded.slice(1, encoded.length - 1))))
    } else {
      return b64Decode(encoded.slice(0, encoded.length - 1))
    }
  }

  const stegano = () => {
    output = ''
    const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height)
    const encoded = encode(text)
    let c: number = 0
    for (let i: number = 0; i < imgData.data.length / 4; i++) {
      if (c >= encoded.length) {
        break
      }
      const [r, g, b, a] = imgData.data.slice(i * 4, i * 4 + 4)
      if (a == 0) continue
      const b64Index = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/='.split('').indexOf(encoded[c])
      const n2 = Math.floor(b64Index / 16)
      const n1 = Math.floor((b64Index - 16 * n2) / 4)
      const n0 = b64Index - 16 * n2 - 4 * n1
      const nr = r - r % 5 + n2
      const ng = g - g % 4 + n1
      const nb = b - b % 4 + n0
      imgData.data[i * 4] = nr < 256 ? nr : nr - 5
      imgData.data[i * 4 + 1] = ng < 256 ? ng : ng - 4
      imgData.data[i * 4 + 2] = nb < 256 ? nb : nb - 4
      c++
    }
    if (c < encoded.length) {
      err = 'overflow'
    } else {
      err = ''
      ctx.putImageData(imgData, 0, 0);
      output = canvas.toDataURL()
    }
  }

  const unstegano = () => {
    text = ''
    const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height)
    let encoded: string = ''
    for (let i: number = 0; i < imgData.data.length / 4; i++) {
      if (i > 1 && i % 4 == 1 && encoded.slice(-1) == '=') break
      const [r, g, b, a] = imgData.data.slice(i * 4, i * 4 + 4)
      if (a == 0) continue
      const b64Index = r % 5 * 16 + g % 4 * 4 + b % 4
      encoded += 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/='.split('')[b64Index]
    }
    text = decode(encoded)
  }

  const process = (decodeMode: boolean) => {
    if (canvas && files) {
      const [file] = files
      const img = new Image()
      const reader = new FileReader()
      ctx = canvas.getContext('2d')
      reader.readAsDataURL(file)
      reader.onload = () => {
        img.src = reader.result as string
        img.onload = () => {
          canvas.width = img.width
          canvas.height = img.height
          ctx.drawImage(img, 0, 0)
          if (decodeMode) {
            unstegano()
          } else {
            stegano()
          }
        }
      }
    }
  }
</script>

<main class="max-w-lg mx-auto p-6">
  <div class="mb-6">
    <input type="text" bind:value={text} placeholder="Text" class="input input-primary w-full mt-6" />
    <input type="file" accept="image/png" bind:files class="file-input file-input-bordered file-input-primary w-full mt-6" />
    <div class="flex gap-3 justify-center mt-6">
      <button on:click={() => process(false)} disabled={!text || !files} class="btn btn-primary grow">
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2.5" stroke="currentColor" class="w-6 h-6">
          <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 13.5L12 21m0 0l-7.5-7.5M12 21V3" />
        </svg>
        Text =&gt; Img
      </button>
      <button on:click={() => process(true)} disabled={!files} class="btn btn-primary grow">
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2.5" stroke="currentColor" class="w-6 h-6">
          <path stroke-linecap="round" stroke-linejoin="round" d="M4.5 10.5L12 3m0 0l7.5 7.5M12 3v18" />
        </svg>
        Img =&gt; Text
      </button>
    </div>
  </div>
  <p class="text-error font-bold">{err}</p>
  <canvas bind:this={canvas} class="hidden" />
  {#if output}
    <img src={output} alt="output" class="w-full my-6" />
    <a href={output} download="image.png">
      <button class="btn btn-primary w-full">
        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
          <path stroke-linecap="round" stroke-linejoin="round" d="M3 16.5v2.25A2.25 2.25 0 005.25 21h13.5A2.25 2.25 0 0021 18.75V16.5M16.5 12L12 16.5m0 0L7.5 12m4.5 4.5V3" />
        </svg>
        Download
      </button>
    </a>
  {/if}
</main>
