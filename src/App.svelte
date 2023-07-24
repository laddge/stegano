<script lang="ts">
  import { encode as b64Encode, fromUint8Array } from 'js-base64'
  import { gzip } from 'pako'

  let text: string = ''
  let files: FileList
  let canvas: HTMLCanvasElement
  let ctx: CanvasRenderingContext2D
  let output: string = ''
  let err: string = ''

  const encode = (text: string) => {
    const b64 = b64Encode(text) + '='
    const gzipped = '=' + fromUint8Array(gzip(text)) + '='
    if (b64.length > gzipped.length) {
      return gzipped
    } else {
      return b64
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

  $: {
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
          stegano()
        }
      }
    }
  }
</script>

<main class="max-w-lg mx-auto p-6">
  <div class="my-6">
    <input type="text" bind:value={text} placeholder="Text" class="input input-primary w-full" />
    <input type="file" accept="image/png" bind:files class="file-input file-input-bordered file-input-primary w-full mt-6" />
  </div>
  <p class="text-error font-bold">{err}</p>
  <canvas bind:this={canvas} class="hidden" />
  {#if output}
    <img src={output} alt="output" class="w-full mt-6" />
  {/if}
</main>
