<template>

  <div class="d-flex justify-content-xl-around justify-content-between align-items-start flex-row">

    <div>
      <div class="text-center py-4">
        <div class="btn-group">
          <button class="btn" :class="{ disabled: currentPageNumber <= 1 }" @click="previous">&lt;</button>
          <button class="border btn btn-light" disabled>{{ currentPageNumber }} / {{ totalPages }}</button>
          <button class="btn" :class="{ disabled: currentPageNumber === totalPages }" @click="next">&gt;</button>
        </div>
      </div>

      <div v-if="hasPdf" class="relative" :class="{ removing: removing }"
           :style="{ width: width+'px', height: height+'px'}">
          <canvas ref="canvas" @click="addPin($event.offsetX, $event.offsetY)"></canvas>
          <pin v-for="(pin, index) in currentPage.pins" :key="pin.id" 
               v-bind="pin" :unit="unit" draggable
               @click.ctrl.native="removePin(index)"
               @mousedown.native="saveDragOffset(pin, $event)"
               @drag.native="movePin(pin, $event)"
               @dragend.native="movePin(pin, $event)"/>
        </div>
    </div>

    <div>
      <div class="text-center py-4">
        <div class="btn-group">
          <button class="btn btn-warning" type="button" @click="showOpenDialog">Pick a new file</button>
          <button class="btn btn-danger" @click="clear">Clear all pins</button>
        </div>
      </div>
      
      <div class="input-group">
        <div class="input-group-prepend">
          <span class="input-group-text">Unit: </span>
        </div>
        <select class="form-control" v-model="unit">
          <option v-for="unit in units" :key="unit.name" :value="unit">{{ unit.name }}</option>
        </select>
      </div>

      <ul class="mt-1 list-unstyled">
        <li v-for="pin in currentPage.pins" :key="pin.id">
          <div class="input-group input-group-sm mb-1">
            <input type="text" class="form-control" v-model="pin.name" placeholder="Pin's name">
            <div class="input-group-append">
              <span class="input-group-text">{{ Math.round(pin.x * unit.factor, 2) }} ; {{ Math.round(pin.y * unit.factor, 2) }}</span>
            </div>
          </div>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
  import Pin from './LandingPage/Pin'
  import PDFJS from 'pdfjs-dist/webpack'
  import { remote } from 'electron'

  const { dialog, BrowserWindow } = remote
  const units = [
    { name: 'mm', factor: 0.2645833 },
    { name: 'px', factor: 1 },
    { name: 'pt', factor: 0.75 }
  ]

  export default {
    name: 'landing-page',
    components: { Pin },
    data () {
      return {
        hasPdf: false,
        width: 0,
        height: 0,
        pages: [],
        removing: false,
        pdf: null,
        currentPageNumber: 0,
        totalPages: 0,
        units: units,
        unit: units[0]
      }
    },
    computed: {
      currentPage () {
        return this.currentPageNumber === 0 ? [] : this.pages[this.currentPageNumber]
      }
    },
    methods: {
      showOpenDialog () {
        const currentWindow = BrowserWindow.getFocusedWindow()
        dialog.showOpenDialog(currentWindow, {
          title: 'Choose a PDF to open in Pin-GUI-n',
          filters: [
            { name: 'PDF Files', extensions: ['pdf'] }
          ],
          properties: ['openFile']
        }, this.loadPdf)
      },
      async loadPdf (files) {
        if (files.length === 0) {
          return
        }

        this.hasPdf = false
        this.clear()

        this.pdf = await PDFJS.getDocument(`file://${files[0]}`)
        this.totalPages = this.pdf.numPages

        this.pages.splice(0, this.pages.length)
        for (let i = 1; i <= this.totalPages; i++) {
          this.$set(this.pages, i, { pins: [] })
        }

        this.hasPdf = true

        await this.displayPage(1)
      },
      async displayPage (pageNumber) {
        if (pageNumber <= 0 || pageNumber > this.totalPages) {
          return
        }

        this.currentPageNumber = pageNumber
        const page = await this.pdf.getPage(pageNumber)

        // https://github.com/mozilla/pdf.js/issues/5628#issuecomment-343299126
        const scale = 96 / 72
        var viewport = page.getViewport(scale)

        var canvas = this.$refs['canvas']
        var context = canvas.getContext('2d')
        this.width = canvas.width = viewport.width
        this.height = canvas.height = viewport.height

        await page.render({ canvasContext: context, viewport: viewport })
      },
      addPin (x, y) {
        this.currentPage.pins.push({
          x,
          y,
          originX: 0,
          originY: 0,
          name: '',
          id: (new Date().valueOf())
        })
      },
      next () {
        if (this.currentPageNumber < this.totalPages) {
          this.displayPage(this.currentPageNumber + 1)
        }
      },
      previous () {
        if (this.currentPageNumber > 1) {
          this.displayPage(this.currentPageNumber - 1)
        }
      },
      clear () {
        if (this.currentPageNumber <= 0) {
          return
        }
        this.currentPage.pins.splice(0, this.currentPage.pins.length)
      },
      removePin (index) {
        this.currentPage.pins.splice(index, 1)
      },
      saveDragOffset: function (pin, event) {
        pin.originX = event.offsetX
        pin.originY = event.offsetY
      },
      movePin: function (pin, event) {
        pin.x += event.offsetX - pin.originX
        pin.y += event.offsetY - pin.originY
      }
    },
    mounted () {
      const self = this
      self.$nextTick(function () {
        self.showOpenDialog()
      })
    }
  }
</script>

<style lang="scss">
.relative {
  position: relative;

  &.removing {
    cursor: crosshair !important;
  }

  canvas {
    border: 1px solid rgba(0, 0, 0, 0.226);
    position: absolute;
  }
}
</style>
