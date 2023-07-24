<script lang="ts">
  import { encode as b64Encode, fromUint8Array } from 'js-base64'
  import { gzip } from 'pako'

  let text: string = ''
  let files: FileList
  let canvas: HTMLCanvasElement

  $: {
    if (canvas && files) {
      const [file] = files
      const img = new Image()
      const reader = new FileReader()
      const ctx = canvas.getContext('2d')
      reader.readAsDataURL(file)
      reader.onload = () => {
        img.src = reader.result as string
        img.onload = () => {
          canvas.width = img.width
          canvas.height = img.height
          ctx.drawImage(img, 0, 0)
        }
      }
    }
  }

  const encode = (text: string) => {
    const b64 = b64Encode(text)
    const gzipped = '=' + fromUint8Array(gzip(text))
    if (b64.length > gzipped.length) {
      return gzipped
    } else {
      return b64
    }
  }
</script>

<main class="max-w-lg mx-auto p-6">
  <div>
    <input type="text" bind:value={text} placeholder="Text" class="input input-primary w-full" />
    <input type="file" accept="image/png" bind:files class="file-input file-input-bordered file-input-primary w-full mt-6" />
  </div>
  {encode(text)}
  <canvas bind:this={canvas} class="w-full mt-6" />
</main>
