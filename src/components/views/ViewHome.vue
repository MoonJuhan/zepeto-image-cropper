<template>
  <div class="view-home">
    <canvas ref="refCanvas"></canvas>
    <canvas ref="refNewCanvas"></canvas>
    <input type="file" @change="onChangeFile" />
  </div>
</template>

<script>
import { ref } from 'vue'

export default {
  setup() {
    const refCanvas = ref(null)
    const refNewCanvas = ref(null)

    const onChangeFile = (e) => {
      const reader = new FileReader()

      reader.onload = function (event) {
        var img = new Image()

        img.onload = function () {
          console.log(img.height)
          refCanvas.value.width = img.width
          refCanvas.value.height = img.height
          refCanvas.value.getContext('2d').drawImage(img, 0, 0)

          refineImage(img)
        }
        img.src = event.target.result
      }

      reader.readAsDataURL(e.target.files[0])
    }

    const refineImage = (img) => {
      const ctx = refCanvas.value.getContext('2d')
      const { width, height } = img

      let startY = 0
      let endY = 0

      console.log(width, height)
      for (let i = 0; i < height; i += 1) {
        const ImgData = ctx.getImageData(width / 2, i, 1, 1).data
        if (ImgData[0] === 255 && ImgData[1] === 255 && ImgData[2] === 255) {
          if (i > 130 && i < 170) startY = i
          if (i > height / 2) {
            if (endY === 0) endY = i
          }
        }
        console.log(`${width / 2},${i} : ${ImgData[0]} ${ImgData[1]} ${ImgData[2]} ${ImgData[3]}`)
        // ctx.putImageData(ctx.getImageData(width / 2, i, 1, 1), (width / 2) + 50, i);
      }

      console.log(startY, endY)

      drawRefinedImage(img, startY + 40, endY - 40)
    }

    const drawRefinedImage = (img, sy, ey) => {
      const newSize = ey - sy
      refNewCanvas.value.width = newSize
      refNewCanvas.value.height = newSize

      const ctx = refNewCanvas.value.getContext('2d')

      ctx.fillStyle = '#FBFBFD'
      ctx.fillRect(0, 0, refNewCanvas.value.width, refNewCanvas.value.height)

      ctx.drawImage(img, (newSize - img.width) / 2, -sy, img.width, img.height)

      downloadImage()
    }

    const downloadImage = () => {
      var dataURL = refNewCanvas.value.toDataURL('image/png')

      dataURL = dataURL.replace(/^data:image\/[^;]*/, 'data:application/octet-stream')

      dataURL = dataURL.replace(
        /^data:application\/octet-stream/,
        'data:application/octet-stream;headers=Content-Disposition%3A%20attachment%3B%20filename=Canvas.png'
      )

      const aTag = document.createElement('a')
      aTag.download = 'from_canvas.png'
      aTag.href = dataURL
      aTag.click()
    }

    return {
      refCanvas,
      refNewCanvas,
      onChangeFile,
    }
  },
}
</script>

<style lang="scss" scoped>
canvas {
  border: 2px solid red;
}
</style>
