<template>
  <div class="view-home">
    <canvas ref="refCanvas" />
    <canvas ref="refNewCanvas" />
    <input type="file" multiple @change="onChangeFile" />
  </div>
</template>

<script>
import { ref, watch } from 'vue'

export default {
  setup() {
    const imageList = ref([])
    const refCanvas = ref(null)
    const refNewCanvas = ref(null)

    const readFile = (file, length) => {
      const reader = new FileReader()

      reader.onload = (event) => {
        var img = new Image()

        img.onload = () => {
          imageList.value.push({ img, name: file.name })
          if (length === imageList.value.length) {
            imageProcessing()
          }
        }

        img.src = event.target.result
      }

      reader.readAsDataURL(file)
    }

    const onChangeFile = async (e) => {
      for (let i = 0; i < e.target.files.length; i++) {
        readFile(e.target.files[i], e.target.files.length)
      }
    }

    const imageProcessing = async () => {
      for (let i in imageList.value) {
        drawImage(imageList.value[i].img)
        const maxPos = await refineImage(imageList.value[i].img)
        drawRefinedImage(imageList.value[i].img, maxPos)
        downloadImage(imageList.value[i].name)
      }
    }

    const drawImage = async (img) => {
      refCanvas.value.width = img.width
      refCanvas.value.height = img.height
      refCanvas.value.getContext('2d').drawImage(img, 0, 0)
    }

    const refineImage = async (img) => {
      const ctx = refCanvas.value.getContext('2d')
      const { width, height } = img

      const superiorColorRate = []

      const checkSuperiorColor = (colorRate, colorList) => {
        colorList.forEach((el) => {
          const findColorRate = colorRate.find((r) => r.colorStr === el)

          if (findColorRate) {
            findColorRate.num++
          } else {
            colorRate.push({
              colorStr: el,
              num: 1,
            })
          }
        })
      }

      for (let i = 0; i < height; i += 1) {
        const ImgData = ctx.getImageData(width / 3, i, 1, 1).data
        const colorStr = ImgData.join('_')
        checkSuperiorColor(superiorColorRate, [colorStr])
      }

      const superiorColorStr = superiorColorRate.filter((r) => r.num > 4).sort((a, b) => b.num - a.num)[0].colorStr
      const scanWidth = parseInt(width * 0.6)

      const maxPos = {
        sx: -1,
        sy: -1,
        ex: -1,
        ey: -1,
      }

      const scanLine = (refinedImgDatas, type, i) => {
        const colorStrs = refinedImgDatas
          .map((el, index) => {
            if (index % 4 === 0) {
              return [
                refinedImgDatas[index],
                refinedImgDatas[index + 1],
                refinedImgDatas[index + 2],
                refinedImgDatas[index + 3],
              ].join('_')
            }

            return null
          })
          .filter((el) => el)

        const lineColorRate = []
        checkSuperiorColor(lineColorRate, colorStrs)

        if (lineColorRate.length === 1 && lineColorRate[0].colorStr === superiorColorStr && maxPos[type] === -1) {
          maxPos[type] = i
        }
      }

      // Go Up
      for (let i = parseInt(height / 2); i > 0; i--) {
        const refinedImgDatas = []

        ctx.getImageData((width - scanWidth) / 2, i, scanWidth, 1).data.forEach((el) => refinedImgDatas.push(el))

        scanLine(refinedImgDatas, 'sy', i)
      }

      // Go Down
      for (let i = parseInt(height / 2); i < height; i++) {
        const refinedImgDatas = []

        ctx.getImageData((width - scanWidth) / 2, i, scanWidth, 1).data.forEach((el) => refinedImgDatas.push(el))

        scanLine(refinedImgDatas, 'ey', i)
      }

      // Go Left
      for (let i = parseInt(width / 2); i > 0; i--) {
        const refinedImgDatas = []

        ctx.getImageData(i, (height - scanWidth) / 2, 1, scanWidth).data.forEach((el) => refinedImgDatas.push(el))

        scanLine(refinedImgDatas, 'sx', i)
      }

      // Go Right
      for (let i = parseInt(width / 2); i < width; i++) {
        const refinedImgDatas = []

        ctx.getImageData(i, (height - scanWidth) / 2, 1, scanWidth).data.forEach((el) => refinedImgDatas.push(el))

        scanLine(refinedImgDatas, 'ex', i)
      }

      return maxPos
    }

    const drawRefinedImage = (img, maxPos) => {
      const newSize = maxPos.ey - maxPos.sy
      refNewCanvas.value.width = newSize
      refNewCanvas.value.height = newSize

      const ctx = refNewCanvas.value.getContext('2d')

      ctx.fillStyle = '#FBFBFD'
      ctx.fillRect(0, 0, refNewCanvas.value.width, refNewCanvas.value.height)

      ctx.drawImage(img, (newSize - img.width) / 2, -maxPos.sy, img.width, img.height)
    }

    const downloadImage = (name) => {
      let dataURL = refNewCanvas.value.toDataURL('image/png')

      dataURL = dataURL.replace(/^data:image\/[^;]*/, 'data:application/octet-stream')

      dataURL = dataURL.replace(
        /^data:application\/octet-stream/,
        'data:application/octet-stream;headers=Content-Disposition%3A%20attachment%3B%20filename=Canvas.png'
      )

      const aTag = document.createElement('a')
      aTag.download = `resized_${name}`
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
