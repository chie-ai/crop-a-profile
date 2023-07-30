<template>
  <div class="row">
    <div class="col-12">
      <div class="relative-position profile-preview overflow-hidden" :style="{ height: `${profileContainerHeight}px` }" v-touch-pan.prevent.mouse="handlePan">
        <img
          :src="imgPreview"
          alt="profile"
          ref="profilePreview"
          id="profilePreview"
          class="absolute"
          :style="{ top: offsetTop+'px', left: offsetLeft+'px' }"
        />
        <div :style="{ width: `${canvasTargetScale}px`, height: `${canvasTargetScale}px` }" id="targetDimension" class="absolute-center target-profile" />
      </div>
    </div>
    <div class="col-12 q-px-lg">
      <q-slider label label-always v-model="dimensionsResizer" :min="0" @pan="sliderPan" @update:model-value="dimensionsResizer" :max="100" color="app-accent-0"/>
    </div>
    <template v-if="newImg">
      <div class="col-12 text-center">
        <img :src="newImg" alt="canvasProfilePreview" class="canvas-profile-preview">
      </div>
    </template>
    <!-- Start: Web test -->
    <q-file @update:model-value="calculateProfileRatio" ref="selectProfile" class="q-mt-lg hidden" label="Upload picture" />
    <!-- End -->
    <canvas id="canvas" class="hidden"></canvas>
    <canvas id="canvasTargetDimensions" class="hidden"></canvas>
  </div>
  <div class="col-12 q-px-xs">
    <div class="row">
      <div class="col-5 q-px-xs">
        <q-btn color="primary" class="full-width" label="Take a picture" @click="getProfile('CAMERA')" />
        <img :src="imageSrc">
      </div>
      <div class="col-7 q-px-xs">
        <q-btn color="primary" class="full-width" label="Select profile picture" @click="getProfile('PHOTOS')" />
        <img :src="imageSrc">
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch } from 'vue'
import { Camera, CameraResultType } from '@capacitor/camera'
// import { useQuasar } from 'quasar'

export default {
  name: 'CanvasPage',
  setup () {
    // const $q = useQuasar()
    const imageSrc = ref('')
    const selectProfile = ref(null)

    onMounted(async () => {
      console.log('Permission status: ', await Camera.checkPermissions())
    })

    const getProfile = async (action) => {
      try {
        await Camera.requestPermissions()
      } catch (err) {
        console.log(err)
        // Start: Web test
        selectProfile.value.pickFiles()
        // End
      }
      const permission = await Camera.checkPermissions()

      if (permission.camera === 'granted' && permission.photos === 'granted') {
        const image = await Camera.getPhoto({
          quality: 90,
          source: action,
          direction: 'FRONT',
          resultType: CameraResultType.Uri
        })

        canvasInitialConfiguration()

        imageFile.value = image.webPath
        scaleImageIntoCanvas(image.webPath, 0)
      }
    }

    const profilePreview = ref(null)
    const imgPreview = ref(null)
    const profileContainerHeight = ref(window.innerWidth)
    const imageFile = ref(null)
    const profileAndContainerHeightDiff = ref(0)
    const profileAndContainerWidthDiff = ref(0)
    const canvasTargetScale = ref(window.innerWidth * 0.90)
    const newImg = ref(null)
    const dimensionsResizer = ref(0)
    const info = ref(null)
    const panning = ref(false)
    const offsetTop = ref(0)
    const offsetLeft = ref(0)
    let defaultSx = 0
    let defaultSy = 0
    let dynamicSx = 0, dynamicSy = 0, sx = 0, sy = 0
    let canvasAndContWidthDiff = 0
    let canvasAndContHeightDiff = 0
    let sliderPanning = false
    let imagePreviewDynamicWidth = 0
    let imagePreviewDynamicHeight = 0
    let zoomLevel = 0 / 100
    const MAX_WIDTH = window.innerWidth
    const MAX_HEIGHT = window.innerHeight
    let _image = null
    const profileHeader = 0 // Not being used for the meantime

    const canvasInitialConfiguration = () => {
      offsetTop.value = 0
      offsetLeft.value = 0
      dimensionsResizer.value = 0
      canvasAndContWidthDiff = window.innerWidth - canvasTargetScale.value
      canvasAndContHeightDiff = profileContainerHeight.value - canvasTargetScale.value
      defaultSx = canvasAndContWidthDiff / 2
      defaultSy = canvasAndContHeightDiff / 2
      dynamicSx = defaultSx
      dynamicSy = defaultSy
    }

    const sliderPan = (phase) => {
      sliderPanning = (phase === 'start')
      if (!sliderPanning) {
        profilePreview.value.classList.add('transition')
        dynamicSx = defaultSx - offsetLeft.value
        dynamicSy = defaultSy - offsetTop.value
        scaleImageIntoCanvas(imageFile.value, zoomLevel)
      } else {
        profilePreview.value.classList.remove('transition')
      }
    }

    watch(dimensionsResizer, (newVal) => {
      zoomLevel = (newVal / 100)

      // Calculate the scaling factor to resize new image to
      // Fit MAX dimensions without overflow
      let scalingFactor = Math.min(MAX_WIDTH / _image.width, MAX_HEIGHT / _image.height)

      if (_image.width > _image.height) {
        scalingFactor = (profileContainerHeight.value / _image.height)
      }

      // Calculate the resized image dimensions
      const dimWidth = (_image.width * scalingFactor)
      const dimHeight = (_image.height * scalingFactor)

      // Calculate the resized image dimensions with the given percentage;
      sx = (dimWidth + (dimWidth * zoomLevel))
      sy = (dimHeight + (dimHeight * zoomLevel))

      const xDiff = sx - MAX_WIDTH
      const yDiff = sy - profileContainerHeight.value

      profileAndContainerWidthDiff.value = xDiff
      profileAndContainerHeightDiff.value = yDiff

      profilePreview.value.style.width = sx + 'px'
      profilePreview.value.style.height = sy + 'px'

      if (sliderPanning) {
        const diffPreviewWidth = imagePreviewDynamicWidth - xDiff
        const diffPreviewHeight = imagePreviewDynamicHeight - yDiff

        offsetLeft.value = offsetLeft.value + (diffPreviewWidth / 2)
        offsetTop.value = offsetTop.value + (diffPreviewHeight / 2)

        const profilePreview = document.getElementById('profilePreview')
        const offsetLeft_ = profilePreview.getBoundingClientRect().left
        const offsetTop_ = profilePreview.getBoundingClientRect().top - profileHeader // Offset top of the preview container

        preventLeftTip()
        preventRightTip(offsetLeft_)
        preventTopTip()
        preventBottomTip(offsetTop_)
      }

      imagePreviewDynamicWidth = xDiff
      imagePreviewDynamicHeight = yDiff
    })

    // Start: Web test
    const calculateProfileRatio = (file) => {
      canvasInitialConfiguration()
      imageFile.value = URL.createObjectURL(file)

      scaleImageIntoCanvas(imageFile.value, 0 / 100)
    }
    // End

    const scaleImageIntoCanvas = (image, zoomlevel) => {
      _image = new Image()
      _image.crossOrigin = 'anonymous'
      _image.src = image
      _image.onload = async function (event) {
        // Calculate the scaling factor to resize new image to
        // Fit MAX dimensions without overflow
        let scalingFactor = Math.min((MAX_WIDTH / _image.width), (MAX_HEIGHT / _image.height))

        // Use height to get the scalingFactor if height is lesser than the width
        if (_image.width > _image.height) {
          scalingFactor = (profileContainerHeight.value / _image.height)
        }

        // Calculate the resized image dimensions
        const dimWidth = (_image.width * scalingFactor)
        const dimHeight = (_image.height * scalingFactor)

        // Calculate the resized image dimensions with the given zoom level in percentage
        sx = (dimWidth + (dimWidth * zoomlevel))
        // sy = sx * (dimHeight / dimWidth)
        sy = (dimHeight + (dimHeight * zoomlevel))

        // Create a new canvas
        const canvas = document.getElementById('canvas')
        const ctx = canvas.getContext('2d')

        // Resize the canvas to the new dimensions
        canvas.width = sx
        canvas.height = sy

        profilePreview.value.style.width = sx + 'px'
        profilePreview.value.style.height = sy + 'px'

        // Get the vertical and horizontal difference between the image and container
        const xDiff = sx - MAX_WIDTH
        const yDiff = sy - profileContainerHeight.value

        profileAndContainerWidthDiff.value = xDiff
        profileAndContainerHeightDiff.value = yDiff

        imagePreviewDynamicWidth = xDiff
        imagePreviewDynamicHeight = yDiff

        // Scale & draw the image onto the canvas
        ctx.drawImage(_image, 0, 0, sx, sy)
        imgPreview.value = canvas.toDataURL()

        cropTargetCanvasDimensions()
      }
    }

    const cropTargetCanvasDimensions = () => {
      const img = new Image()
      img.crossOrigin = 'anonymous'
      img.src = imgPreview.value
      img.onload = function (evt) {
        // Create a new canvas
        const canvasTargetDimensions = document.getElementById('canvasTargetDimensions')
        const ctxScale = canvasTargetDimensions.getContext('2d')

        // Resize the canvas to the new dimensions
        canvasTargetDimensions.width = canvasTargetScale.value
        canvasTargetDimensions.height = canvasTargetScale.value

        ctxScale.drawImage(
          img,
          dynamicSx,
          dynamicSy,
          canvasTargetScale.value,
          canvasTargetScale.value,
          0, 0,
          canvasTargetScale.value,
          canvasTargetScale.value
        )

        newImg.value = canvasTargetDimensions.toDataURL()
      }
    }
    let topCurrentPosition = 0
    let leftCurrentPosition = 0
    let initialOffsetTop = 0
    let initialOffsetLeft = 0

    const handlePan = async ({ evt, ...newInfo }) => {
      info.value = newInfo

      if (!panning.value) {
        initialOffsetTop = newInfo.position.top
        initialOffsetLeft = newInfo.position.left
        topCurrentPosition = offsetTop.value
        leftCurrentPosition = offsetLeft.value
      }

      const offsetLeft_ = (leftCurrentPosition + (newInfo.position.left - initialOffsetLeft))
      offsetLeft.value = offsetLeft_

      const offsetTop_ = (topCurrentPosition + (newInfo.position.top - initialOffsetTop))
      offsetTop.value = offsetTop_

      if (newInfo.isFirst) {
        panning.value = true
        profilePreview.value.classList.remove('transition')
      } else if (newInfo.isFinal) {
        panning.value = false
        profilePreview.value.classList.add('transition')

        preventLeftTip()
        preventRightTip(offsetLeft_)

        preventTopTip()
        preventBottomTip(offsetTop_)

        initialOffsetTop = offsetTop.value
        initialOffsetLeft = offsetLeft.value
        dynamicSx = defaultSx - offsetLeft.value
        dynamicSy = defaultSy - offsetTop.value
        cropTargetCanvasDimensions(imageFile.value)
      }
    }

    // Prevent the bottom tip of an image to pass through inside the bottom tip of the container
    const preventBottomTip = (offsetTop_) => {
      if (profileAndContainerHeightDiff.value <= -parseFloat(offsetTop_)) {
        offsetTop.value = -parseFloat(profileAndContainerHeightDiff.value)
      }
      // Adjust target canvas image
      if (sliderPanning) {
        dynamicSx = defaultSx - offsetLeft.value
        dynamicSy = defaultSy - offsetTop.value
      }
    }

    // Prevent the top tip of an image to pass through inside the top tip of the container
    const preventTopTip = () => {
      if (parseFloat(offsetTop.value) > 0) {
        offsetTop.value = 0
      }
    }

    // Prevent the right tip of an image to pass through inside the right tip of the container
    const preventRightTip = (offsetLeft_) => {
      if (profileAndContainerWidthDiff.value <= -parseFloat(offsetLeft_)) {
        offsetLeft.value = -parseFloat(profileAndContainerWidthDiff.value)
      }
      // Adjust target canvas image
      if (sliderPanning) {
        dynamicSx = defaultSx - offsetLeft.value
        dynamicSy = defaultSy - offsetTop.value
      }
    }

    // Prevent the left tip of an image to pass through inside the left tip of the container
    const preventLeftTip = () => {
      if (parseFloat(offsetLeft.value) > 0) {
        offsetLeft.value = 0
      }
    }

    return {
      // Capacitor device camera
      imageSrc,
      selectProfile,

      // Canvas
      info,
      profilePreview,
      panning,
      imgPreview,
      newImg,
      canvasTargetScale,
      profileContainerHeight,
      calculateProfileRatio,
      handlePan,
      offsetTop,
      offsetLeft,
      dimensionsResizer,
      sliderPan,
      getProfile
    }
  }
}
</script>
