<!-- SPDX-License-Identifier: GPL-3.0-or-later
License: GNU GPLv3 or later. See the license file in the project root for more information.
Copyright © 2021 - present Aleksey Hoffman. All rights reserved.
-->

<template>
  <div
    @mousedown="handleDirItemMouseDown($event, source, index)"
    @mouseover="handleDirItemMouseOver($event, source)"
    :id="`item-index-${index}`"
    :data-item-id="source.id"
    :index="index"
    :data-selected="isDirItemSelected(source)"
    :data-item-path="`${source.path}`"
    :data-item-real-path="`${source.realPath}`"
    :data-layout="specifiedNavigatorLayout"
    :data-type="type"
    :data-file-type="$utils.getFileType(source.realPath).mimeDescription"
    :data-item-hover-is-paused="status.itemHover.isPaused"
    :data-hover-effect="dirItemHoverEffect"
    data-two-line="false"
    class="dir-item-card dir-item-node"
    :class="{
      'drop-target drag-target': type && type.includes('directory'),
      'drag-target': type && type.includes('file')
    }"
    :cursor="navigatorOpenDirItemWithSingleClick && !inputState.alt ? 'pointer' : 'default'"
  >
    <v-layout
      class="dir-item-card__content-container"
      align-center
      v-ripple
    >
      <!-- card::overlays -->
      <div class="dir-item-card__overlay-container">
        <div
          v-if="isDirItemSelected(source)"
          class="dir-item-card__overlay--selected"
        ></div>
        <div
          :class="{'is-visible': source.isHighlighted}"
          class="dir-item-card__overlay--highlighted"
        ></div>
        <div
          class="dir-item-card__overlay--hover"
        ></div>
        <div
          :class="{'is-visible': showDragOverOverlay(source)}"
          class="overlay--drag-over"
        ></div>
      </div>

      <!-- card::thumb-container -->
      <v-layout
        :data-item-real-path="`${source.realPath}`"
        class="dir-item-card__thumb-container"
        align-center justify-center
      >
        <!-- card::thumb: {type: directory} -->
        <v-icon
          v-if="type.includes('directory')"
          class="dir-item-card__icon"
          size="28px"
        >{{getThumbIcon(source)}}
        </v-icon>

        <!-- card::thumb: {type: file && !isImage} -->
        <!-- Will be inserted when it enters viewport bounds (intersectionObserver) -->
        <!-- && $utils.getFileType(source.realPath).mimeDescription !== 'image' -->
        <v-layout
          v-if="type.includes('file')"
          column
        >
          <template v-if="specifiedNavigatorLayout === 'list'">
            <v-icon
              class="dir-item-card__icon pt-1"
              size="22px"
              style="height: 22px;"
            >{{getThumbIcon(source)}}
            </v-icon>
            <div class="dir-item-card__ext">
              {{$utils.getExt(source.path)}}
            </div>
          </template>

          <template v-if="specifiedNavigatorLayout === 'grid'">
            <v-icon
              class="dir-item-card__icon"
              size="48px"
            >{{getThumbIcon(source)}}
            </v-icon>
          </template>
        </v-layout>
      </v-layout>

      <!-- {layout: 'list'} -->
      <template v-if="specifiedNavigatorLayout === 'list'">
        <!-- card::name-->
        <div class="dir-item-card__name">
          <!-- card::name::line-1-->
          <div class="dir-item-card__name__line-1">
            <span
              class="inline-code--light mr-2"
              v-if="showScore"
            >
              score: {{source.score}}
            </span>
            <v-tooltip bottom>
              <template v-slot:activator="{ on }">
                <v-icon
                  v-on="on"
                  v-show="source.isInaccessible"
                  color="red"
                  size="12px"
                >
                  mdi-circle-medium
                </v-icon>
              </template>
              <span>Item is inaccessible</span>
            </v-tooltip>
            {{source.name}}
          </div>

          <!-- card::name::line-2-->
          <div
            class="dir-item-card__name__line-2"
            v-if="showDir && source.dir !== source.path"
          >
            {{source.dir}}
          </div>
        </div>

        <!-- card::date-->
        <div class="dir-item-card__date">
          {{$utils.formatDateTime(source.stat.ctime, 'D MMM YYYY')}}
        </div>

        <!-- {type: (directory|directory-symlink)} -->
        <!-- card::item-count -->
        <div
          v-if="type === 'directory' || type === 'directory-symlink'"
          class="dir-item-card__item-count"
        >{{source.dirItemCount}} {{$localizeUtils.pluralize(source.dirItemCount, 'item')}}
        </div>

        <!-- {type: (file|file-symlink)} -->
        <!-- card::item-count -->
        <div
          v-else-if="type === 'file' || type === 'file-symlink'"
          class="dir-item-card__item-count"
        >{{$utils.prettyBytes(source.stat.size, 1)}}
        </div>
      </template>

      <!-- {layout: 'grid'} -->
      <template v-if="specifiedNavigatorLayout === 'grid'">
        <!-- {type: (directory|directory-symlink)} -->
        <v-layout column v-if="type === 'directory' || type === 'directory-symlink'">
          <!-- card::name -->
          <div class="dir-item-card__name">
            <v-tooltip bottom>
              <template v-slot:activator="{ on }">
                <v-icon
                  v-on="on"
                  v-show="source.isInaccessible"
                  color="red"
                  size="12px"
                >
                  mdi-circle-medium
                </v-icon>
              </template>
              <span>Item is inaccessible</span>
            </v-tooltip>
            {{source.name}}
          </div>

          <!-- card::item-count -->
          <div class="dir-item-card__item-count">
            {{source.dirItemCount}} {{$localizeUtils.pluralize(source.dirItemCount, 'item')}}
          </div>
        </v-layout>

        <!-- {type: (file|file-symlink)} -->
        <v-layout
          v-else-if="type === 'file' || type === 'file-symlink'"
          class="dir-item-card__description-container"
          :data-path="source.path"
          justify-center column
        >
          <div class="dir-item-card__overlay"></div>

          <div class="dir-item-card__bottom-container">
            <!-- card::name -->
            <div class="dir-item-card__name">
              <v-tooltip bottom>
                <template v-slot:activator="{ on }">
                  <v-icon
                    v-on="on"
                    v-show="source.isInaccessible"
                    color="red"
                    size="12px"
                    style="margin-left: -16px"
                  >
                    mdi-circle-medium
                  </v-icon>
                </template>
                <span>Item is inaccessible</span>
              </v-tooltip>
              {{source.name}}
            </div>

            <!-- card::item-count -->
            <div class="dir-item-card__item-count">
              {{$utils.getExt(source.path)}} | {{$utils.prettyBytes(source.stat.size, 1)}}
            </div>
          </div>
        </v-layout>
      </template>

      <div class="dir-item-card__actions">
        <slot name="actions"></slot>
      </div>

      <!-- card::checkbox -->
      <!-- Note: using v-if to improve performance -->
      <div
        v-if="isDirItemSelected(source)"
        class="dir-item-card__checkbox"
      >
        <v-icon @mousedown.stop="handleDirItemCheckboxMouseDown($event, source)">
          mdi-check-box-outline
        </v-icon>
      </div>
    </v-layout>
  </div>
</template>

<script>
import { mapGetters, mapState } from 'vuex'
import { mapFields } from 'vuex-map-fields'
import Filehasher from '../utils/fileHasher.js'

const fs = require('fs')
const PATH = require('path')

export default {
  name: 'dir-item',
  props: {
    index: Number,
    source: {
      type: Object,
      default () {
        return {}
      }
    },
    type: String,
    layout: String,
    status: Object,
    thumbLoadingIsPaused: Boolean,
    forceThumbLoad: Boolean,
    showScore: Boolean,
    showDir: Boolean,
    thumbLoadSchedule: Array
  },
  data () {
    return {
      dirItemAwaitsSecondClick: false,
      dirItemAwaitsSecondClickTimeout: null,
      mouseDown: {
        item: {},
        leftClick: false,
        rightClick: false,
        downCoordX: null,
        downCoordY: null,
        clickedItemIsSelected: false,
        noneItemsSelected: false,
        singleItemSelected: false,
        multipleItemsSelected: false
      }
    }
  },
  mounted () {
    this.loadThumb()
  },
  beforeDestroy () {
    try {
      this.$emit('removeFromThumbLoadSchedule', { item: this.source })
    }
    catch (error) { console.log(error) }
  },
  computed: {
    ...mapState({
      navigatorLayout: state => state.storageData.settings.navigatorLayout,
      appPaths: state => state.appPaths
    }),
    ...mapGetters([
      'selectedDirItems',
      'selectedDirItemsPaths'
    ]),
    ...mapFields({
      thumbsInProcessing: 'thumbsInProcessing',
      currentDir: 'navigatorView.currentDir',
      inputState: 'inputState',
      openDirItemSecondClickDelay: 'storageData.settings.navigator.openDirItemSecondClickDelay',
      navigatorOpenDirItemWithSingleClick: 'storageData.settings.navigator.openDirItemWithSingleClick',
      dirItemHoverEffect: 'storageData.settings.dirItemHoverEffect',
      visibleDirItems: 'navigatorView.visibleDirItems',
      dirItemsInfoIsFetched: 'navigatorView.dirItemsInfoIsFetched',
      dragStartedInsideWindow: 'drag.startedInsideWindow',
      dragTargetType: 'drag.targetType',
      dirItemDropTargetItem: 'drag.dropTargetItem',
      dragMouseIsMoving: 'drag.mouseIsMoving',
      dragMoveTreshold: 'drag.moveTreshold',
      dirItemDragMoveTresholdReached: 'drag.moveTresholdReached',
      cursorLeftWindow: 'drag.cursorLeftWindow',
      inboundDragOverlay: 'overlays.inboundDrag',
      dirItemDragOverlay: 'overlays.dirItemDrag',

      drag: 'drag'
    }),
    specifiedNavigatorLayout () {
      return this.layout
        ? this.layout
        : this.navigatorLayout
    }
  },
  methods: {
    async loadThumbHandler (path) {
      if (this.source.path === path) {
        await this.loadThumb()
      }
    },
    async loadSkippedThumbs () {
      const dirItem = document.querySelector(`div[data-item-real-path="${this.source.realPath}"].dir-item-card`)
      if (dirItem) {
        const dirItemThumbContainer = dirItem.querySelector('.dir-item-card__thumb-container')
        const dirItemFileType = dirItem.dataset.fileType
        const dirItemRealPath = dirItem.dataset.itemRealPath
        const hasImage = dirItemThumbContainer.getElementsByTagName('img').length > 0
        if (!hasImage) {
          await this.loadThumb()
        }
      }
    },
    async loadThumb (options = {}) {
      return new Promise((resolve, reject) => {
        const virtuallyLoadedDirItems = document.querySelectorAll('.dir-item-card')
        // const dirItem = specifiedDirItem ?? document.querySelector(`div[data-item-path="${this.source.realPath}"].dir-item-card`)
        const dirItemNode = document.querySelector(`div[data-item-real-path="${this.source.realPath}"].dir-item-card`)
        if (dirItemNode) {
          const dirItemIndex = parseInt(dirItemNode.getAttribute('index'))
          const dirItemThumbContainer = dirItemNode.querySelector('.dir-item-card__thumb-container')
          const dirItemFileType = dirItemNode.dataset.fileType
          const dirItemPath = dirItemNode.dataset.itemPath
          const dirItemRealPath = dirItemNode.dataset.itemRealPath
          // this.animateDirItemNode({dirItemNode, options})
          this.addThumb(dirItemThumbContainer, dirItemRealPath, dirItemFileType, dirItemNode)
            .then(() => {
              resolve()
            })
        }
      })
    },
    animateDirItemNode (params) {
      if (!params.options.skipTransition) {
        params.dirItemNode.animate(
          [
            { opacity: 0, transform: 'translateY(10px)' },
            { opacity: 1, transform: 'translateY(0px)' }
          ],
          {
            easing: 'ease',
            duration: 500
          }
        )
      }
    },
    async addThumb (dirItemThumbContainer, dirItemRealPath, dirItemFileType, dirItemNode) {
      return new Promise((resolve, reject) => {
        const itemType = this.$utils.getFileType(dirItemRealPath)
        const disallowedMimes = [
          'image/svg+xml',
          'image/heic',
          'image/vnd.adobe.photoshop'
        ]
        if (dirItemFileType === 'image' && !disallowedMimes.includes(itemType.mime)) {
          this.fetchImageThumb(dirItemThumbContainer, dirItemRealPath, dirItemNode)
            .then(() => {
              resolve()
            })
        }
        else {
          resolve()
        }
      })
    },
    removeThumb (dirItemThumbContainer) {
      dirItemThumbContainer
        .querySelectorAll('.dir-item-card__thumb')
        .forEach(element => element.remove())
    },
    async fetchImageThumb (dirItemThumbContainer, dirItemRealPath, dirItemNode) {
      // NOTES:
      // - Changed the method of checking for thumbnail existance.
      //   Blocking issue: 'WASM out of memory':
      //   https://github.com/Daninet/hash-wasm/issues/7
      //   Now, instead of comparing files by hash, it compares 3 parameters:
      //   ${fileSize}_${fileDateModified}_${fileNameBase}
      //   It's enough to figure out if a file has changed and needs a new thumb
      return new Promise((resolve, reject) => {
        // Check if thumb dir exists. If not, create it
        if (!fs.existsSync(this.appPaths.storageDirectories.appStorageNavigatorThumbs)) {
          fs.mkdirSync(this.appPaths.storageDirectories.appStorageNavigatorThumbs, { recursive: true })
        }
        // Parse image path
        const parsedFileName = PATH.parse(dirItemRealPath)
        const fileExt = parsedFileName.ext
        const fileNameBase = parsedFileName.base
        const fileSize = this.source.stat.size
        const fileDateModified = this.source.stat.mtimeMs
        const thumbPath = this.getThumbPath(fileSize, fileNameBase, fileDateModified)

        // If thumb already exist, append it, otherwise generate it
        if (fs.existsSync(thumbPath)) {
          this.appendImageThumb(dirItemThumbContainer, thumbPath, dirItemRealPath, dirItemNode)
            .then(() => {
              resolve()
            })
        }
        else {
          this.generateImageThumb(dirItemThumbContainer, thumbPath, dirItemRealPath, dirItemNode)
            .then(() => {
              this.appendImageThumb(dirItemThumbContainer, thumbPath, dirItemRealPath, dirItemNode)
                .then(() => {
                  resolve()
                })
            })
        }
      })
    },
    getThumbPath (fileSize, fileNameBase, fileDateModified) {
      // Replace last occurance of '.symlink' so that it loads symlink thumbs as well
      const thumbDir = this.appPaths.storageDirectories.appStorageNavigatorThumbs
      return this.specifiedNavigatorLayout === 'list'
        ? `${thumbDir}/48x48_${fileSize}_${fileDateModified}_${fileNameBase}`.replace(/\\/g, '/')
        : `${thumbDir}/280x158_${fileSize}_${fileDateModified}_${fileNameBase}`.replace(/\\/g, '/')
    },
    async appendImageThumb (dirItemThumbContainer, thumbPath, dirItemRealPath, dirItemNode) {
      return new Promise((resolve, reject) => {
        const image = new Image()
        // image.style.position = 'absolute'
        image.classList.add('dir-item-card__thumb')
        image.setAttribute('src', this.$storeUtils.getSafePath(this.$utils.getUrlSafePath(thumbPath)))
        if (this.specifiedNavigatorLayout === 'list') {
          image.animate(
            [
              { opacity: 0, transform: 'translateY(4px)' },
              { opacity: 1, transform: 'translateY(0px)' }
            ],
            {
              easing: 'ease',
              duration: 2000
            }
          )
        }
        else if (this.specifiedNavigatorLayout === 'grid') {
          image.animate(
            [
              { opacity: 0 },
              { opacity: 1 }
            ],
            {
              easing: 'ease',
              duration: 2000
            }
          )
        }
        image.decode()
          .then(() => {
            this.loadImageThumb(image, dirItemThumbContainer, dirItemNode)
              .then(() => {
                resolve()
              })
          })
          .catch((error) => {
            resolve()
          })
      })
    },
    async loadImageThumb (image, dirItemThumbContainer, dirItemNode) {
      return new Promise((resolve, reject) => {
        // Remove previous thumb
        while (dirItemThumbContainer.hasChildNodes()) {
          dirItemThumbContainer.removeChild(dirItemThumbContainer.lastChild)
        }
        // Append new thumb
        dirItemThumbContainer.appendChild(image)
        dirItemNode.dataset.thumbIsLoaded = true
        resolve()
      })
    },
    async generateImageThumb (dirItemThumbContainer, thumbPath, dirItemRealPath, dirItemNode) {
      return new Promise((resolve, reject) => {
        this.$emit('addToThumbLoadSchedule', {
          item: this.source,
          thumbPath,
          onEnd: () => {
            resolve()
          }
        })
      })
    },
    isDirItemSelected (item) {
      return this.selectedDirItemsPaths.includes(item.path)
    },
    showDragOverOverlay (item) {
      const isItemHovered = this.inputState.pointer.hoveredItem.path === item.path
      const isOfTypeDirItem = ['dirItem'].includes(this.dragTargetType)
      const dirItemIsOverlapped = isItemHovered && (isOfTypeDirItem || this.drag.dirItemInbound.value)
      return dirItemIsOverlapped
    },
    getThumbIcon (item) {
      const mimeDescription = this.$utils.getFileType(item.path).mimeDescription
      if (mimeDescription === 'video') {
        return 'mdi-play-circle-outline'
      }
      else if (mimeDescription === 'audio') {
        return 'mdi-play-outline'
      }
      else if (mimeDescription === 'image') {
        return 'mdi-image-outline'
      }
      else if (item.type === 'file') {
        return 'mdi-file-outline'
      }
      else if (item.type === 'file-symlink') {
        return 'mdi-file-move-outline'
      }
      else if (item.type === 'directory') {
        return 'mdi-folder-outline'
      }
      else if (item.type === 'directory-symlink') {
        return 'mdi-folder-move-outline'
      }
    },
    handleDirItemMouseDown (event, item, index) {
      this.dragTargetType = 'dirItem'
      this.mouseDown.item = item
      this.mouseDown.leftClick = event.button === 0
      this.mouseDown.rightClick = event.button === 2
      this.mouseDown.downCoordX = event.clientX
      this.mouseDown.downCoordY = event.clientY
      this.mouseDown.clickedItemIsSelected = this.isDirItemSelected(item)
      this.mouseDown.noneItemsSelected = this.selectedDirItems.length === 0
      this.mouseDown.singleItemSelected = this.selectedDirItems.length === 1
      this.mouseDown.multipleItemsSelected = this.selectedDirItems.length > 1

      this.handleMouseDownActions()
      this.handleDirItemMouseUp()
    },
    handleMouseUpActions () {
      const {
        item,
        leftClick,
        rightClick,
        downCoordX,
        downCoordY,
        clickedItemIsSelected,
        singleItemSelected,
        multipleItemsSelected
      } = this.mouseDown

      if (!this.dirItemDragMoveTresholdReached) {
        // Handle pointer_btn_1_up
        if ((this.inputState.ctrl || this.inputState.shift) && leftClick) {
          if (clickedItemIsSelected) {
            this.$store.dispatch('DESELECT_DIR_ITEM', item)
          }
          else if (!clickedItemIsSelected) {
            this.$store.dispatch('DESELECT_DIR_ITEM', this.currentDir)
            this.$store.dispatch('ADD_TO_SELECTED_DIR_ITEMS', item)
          }
        }
        else if (leftClick && !this.inputState.shift && !this.inputState.ctrl) {
          // Handle 2nd pointer_btn_1_up
          clearTimeout(this.dirItemAwaitsSecondClickTimeout)
          if (this.dirItemAwaitsSecondClick) {
            this.$store.dispatch('OPEN_DIR_ITEM', item)
            this.dirItemAwaitsSecondClick = false
          }
          // Handle 1st pointer_btn_1_up
          else {
            if (this.navigatorOpenDirItemWithSingleClick && !this.inputState.alt) {
              this.$store.dispatch('OPEN_DIR_ITEM', item)
            }
            else {
              this.dirItemAwaitsSecondClick = true
              this.dirItemAwaitsSecondClickTimeout = setTimeout(() => {
                this.dirItemAwaitsSecondClick = false
              }, this.openDirItemSecondClickDelay)
            }
            if ((multipleItemsSelected || singleItemSelected) && !clickedItemIsSelected) {
              // this.$store.dispatch('DESELECT_ALL_DIR_ITEMS')
              // this.$store.dispatch('ADD_TO_SELECTED_DIR_ITEMS', item)
            }
            else if (multipleItemsSelected && clickedItemIsSelected) {
              this.$store.dispatch('DESELECT_ALL_DIR_ITEMS')
              this.$store.dispatch('ADD_TO_SELECTED_DIR_ITEMS', item)
            }
            else if (singleItemSelected && clickedItemIsSelected) {
              this.$store.dispatch('DESELECT_DIR_ITEM', item)
            }
          }
        }
        // Handle pointer_btn_2_up
        else if (rightClick) {
          if (!clickedItemIsSelected) {
            this.$store.dispatch('DESELECT_ALL_DIR_ITEMS')
            this.$store.dispatch('ADD_TO_SELECTED_DIR_ITEMS', item)
          }
          this.$store.dispatch('SET_CONTEXT_MENU', {
            x: downCoordX,
            y: downCoordY
          })
        }
      }
    },
    handleMouseDownActions () {
      const { item, leftClick, clickedItemIsSelected } = this.mouseDown
      const isSelectingNotSelectedDirItem = leftClick &&
        !clickedItemIsSelected &&
        !this.inputState.ctrl &&
        !this.inputState.shift
      const isSelectingDirItemRange = leftClick && this.inputState.shift
      // Handle pointer_btn_1_down
      if (isSelectingNotSelectedDirItem) {
        this.$store.dispatch('DESELECT_ALL_DIR_ITEMS')
        this.$store.dispatch('ADD_TO_SELECTED_DIR_ITEMS', item)
      }
      else if (isSelectingDirItemRange) {
        this.$store.dispatch('SELECT_DIR_ITEM_RANGE', item)
      }
    },
    handleDirItemMouseUp () {
      // Creating a global listener to make it possible
      // to move the cursor outside the element after mouseDown but before mouseUp
      document.addEventListener('mouseup', (mouseupEvent) => {
        this.handleMouseUpActions()
      }, { once: true })
    },
    handleDirItemMouseOver (event, item) {
      const someItemIsSelected = this.selectedDirItemsPaths.length !== 0
      const currentDirIsSelected = this.selectedDirItemsPaths.includes(this.currentDir.path)
      const isDraggingDirItem = this.dragDisplayDirItemDragOverlay
      const shouldHighlightDirItems = !this.dirItemDragOverlay &&
        this.inputState.shift &&
        someItemIsSelected &&
        !currentDirIsSelected &&
        !isDraggingDirItem
      if (shouldHighlightDirItems) {
        this.$store.dispatch('HIGHLIGHT_DIR_ITEM_RANGE', { hoveredItem: item })
      }
    },
    handleDirItemCheckboxMouseDown (event, item) {
      if (this.isDirItemSelected(item)) {
        this.$store.dispatch('DESELECT_DIR_ITEM', item)
      }
    }
  }
}
</script>

<style>
@keyframes outline-pulse-animation {
  0% { outline-color: rgb(159, 168, 218, 0.4); }
  50% { outline-color: rgb(255, 255, 255, 0.1); }
  100% { outline-color: rgb(159, 168, 218, 0.4); }
}

.dir-item-row-grid[type="directory"],
.dir-item-row-grid[type="directory-symlink"] {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  grid-auto-rows: 64px;
  gap: 24px;
}

.dir-item-row-grid[type="file"],
.dir-item-row-grid[type="file-symlink"]  {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  grid-auto-rows: 158px;
  gap: 24px;
}

/* OVERLAYS */
.dir-item-card__overlay-container {
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
  width: 100%;
  height: 100%;
}

.dir-item-card__overlay--selected {
  pointer-events: none;
  position: absolute;
  width: 100%;
  height: 100%;
  background-color: rgb(159, 168, 218, 0.05);
  z-index: 2;
  /* border: 2px dashed rgb(159, 168, 218, 0.3); */
  outline: 2px solid rgb(159, 168, 218, 0.2);
  outline-offset: -2px;
}

[data-layout="grid"][data-file-type="image"]
  .dir-item-card__overlay--selected {
    background-color: rgb(159, 168, 218, 0.2);
    outline: 2px solid rgb(159, 168, 218, 0.5);
  }

.dir-item-card__overlay--highlighted {
  pointer-events: none;
  position: absolute;
  width: 100%;
  height: 100%;
  background-color: rgb(159, 168, 218, 0.05);
  z-index: 1;
  /* border: 2px dashed rgb(159, 168, 218, 0.3); */
  outline: 2px solid rgb(159, 168, 218, 0.3);
  outline-offset: -2px;
  transition: all 0.3s;
  opacity: 0;
}

.dir-item-card__overlay--highlighted.is-visible {
  transition: all 0.3s;
  opacity: 1;
  animation: outline-pulse-animation 2s infinite;
}

.dir-item-card__overlay--hover {
  pointer-events: none;
  position: absolute;
  width: 100%;
  height: 100%;
  background-color: var(--highlight-color-5);
  z-index: 1;
  opacity: 0;
}

/* CARD CONTAINER */
[data-layout="list"][data-type="directory"].dir-item-card__container,
[data-layout="list"][data-type="directory-symlink"].dir-item-card__container {
  display: grid;
  grid-template-columns: 1fr;
  gap: 4px;
}

[data-layout="list"][data-type="file"].dir-item-card__container,
[data-layout="list"][data-type="file-symlink"].dir-item-card__container {
  display: grid;
  grid-template-columns: 1fr;
  gap: 4px;
}

[data-layout="grid"][data-type="directory"].dir-item-card__container,
[data-layout="grid"][data-type="directory-symlink"].dir-item-card__container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  grid-auto-rows: 64px;
  gap: 24px;
}

[data-layout="grid"][data-type="file"].dir-item-card__container,
[data-layout="grid"][data-type="file-symlink"].dir-item-card__container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  grid-auto-rows: 158px;
  gap: 24px;
}

/* CARD */
.dir-item-card * {
  user-select: none;
}

.dir-item-card {
  position: relative;
  z-index: 1;
  /* background-color: var(--bg-color-1); */
  background-color: var(--dir-item-card-bg);
  transition: all 0.3s ease-in-out; /* 'leave' transition */
}

.dir-item-card[cursor="pointer"] {
  cursor: pointer;
}

.dir-item--divider {
  z-index: 1;
  display: flex;
  align-items: center;
  height: 36px;
  margin: 0px !important;
  padding: 0px 0px;
}

.dir-item-card:hover {
  /* transition: all 0.05s; /* 'enter' transition */
  transition: all 0s; /* 'enter' transition */
  z-index: 2;
}

.dir-item-card:not([data-item-hover-is-paused])[data-hover-effect="scale"]:hover {
  box-shadow: 0px 4px 32px rgb(0, 0, 0, 0.3) !important;
  transform: scale(1.02);
}

.dir-item-card[data-hover-effect="highlight"]:hover
  .dir-item-card__overlay--hover {
    opacity: 1;
  }

[data-layout="list"][data-type="directory"] .dir-item-card,
[data-layout="list"][data-type="directory-symlink"] .dir-item-card,
[data-layout="list"][data-type="file"] .dir-item-card,
[data-layout="list"][data-type="file-symlink"] .dir-item-card {
  height: 48px;
  margin: 0px 36px;
  padding: 0px;
}

[data-layout="list"][data-type="directory"] .dir-item-card__content-container,
[data-layout="list"][data-type="directory-symlink"] .dir-item-card__content-container,
[data-layout="list"][data-type="file"] .dir-item-card__content-container,
[data-layout="list"][data-type="file-symlink"] .dir-item-card__content-container {
  display: grid;
  grid-template-columns: 48px 4fr 3fr 1fr 48px;
  gap: 16px;
  border-bottom: 1px solid var(--dir-item-card-border-3);
  /* Set the element height to 48px, otherwise
    the bottom border increases the height to 49px */
  height: 48px;
}

[data-layout="grid"][data-type="directory"] .dir-item-card,
[data-layout="grid"][data-type="directory-symlink"] .dir-item-card {
  height: 64px;
}

[data-layout="grid"][data-type="file"] .dir-item-card,
[data-layout="grid"][data-type="file-symlink"] .dir-item-card {
  height: 158px;
}

[data-layout="grid"][data-type="directory"] .dir-item-card__content-container,
[data-layout="grid"][data-type="directory-symlink"] .dir-item-card__content-container {
  display: grid;
  grid-template-columns: 64px 1fr 48px;
  gap: 0px;
  box-shadow: 0px 4px 16px rgb(0, 0, 0, 0.1);
  /* border: 1px solid var(--dir-item-card-border-3) !important; */
  border: 0px solid transparent !important;
  background-color: var(--bg-color-1);
  border-radius: 8px;
  height: 64px;
}

[data-layout="grid"][data-type="file"]
  .dir-item-card,
[data-layout="grid"][data-type="file-symlink"]
  .dir-item-card {
    border: 0px solid transparent !important;
    background-color: var(--bg-color-1);
  }

[data-layout="grid"][data-type="file"]
  .dir-item-card__content-container,
[data-layout="grid"][data-type="file-symlink"]
  .dir-item-card__content-container {
    box-shadow:
      -4px -4px 8px rgb(255, 255, 255, 0.05),
      0px 4px 12px rgb(0, 0, 0, 0.3);
    /* border: 1px solid var(--dir-item-card-border-3) !important; */
    height: 100%;
    width: 100%;
      /* #1D78A1
    #0E438B
    #ED519C
    #5A128D */
    /* background:
      linear-gradient(217deg, #4164a5, rgba(255,0,0,0) 70.71%),
      linear-gradient(127deg, #1D78A1, rgba(0,255,0,0) 70.71%),
      linear-gradient(336deg, #ED519C, rgba(0,0,255,0) 60.71%); */
  }

[data-layout="grid"][data-type="file"]:not([data-file-type="image"])
  .dir-item-card__content-container,
[data-layout="grid"][data-type="file-symlink"]:not([data-file-type="image"])
  .dir-item-card__content-container {
    /* box-shadow:
      -4px -4px 8px #4164a544,
      0px 4px 12px #ED519C44;
  */
  }

/* CARD CONTENT */
[data-layout="grid"][data-type="file"]
  .dir-item-card__bottom-container,
[data-layout="grid"][data-type="file-symlink"]
  .dir-item-card__bottom-container {
    /* display: block; */
    position: absolute;
    bottom: 0;
    left: 0;
    padding: 2px 16px;
    height: 52px;
    width: 100%;
    text-align: center;
    z-index: 3;
  }

[data-layout="grid"][data-type="file"][data-file-type="image"]
  .dir-item-card__overlay,
[data-layout="grid"][data-type="file-symlink"][data-file-type="image"]
  .dir-item-card__overlay {
    background-image: linear-gradient(0deg, rgb(0, 0, 0, 0.8) 0%, rgb(0, 0, 0, 0.0) 100%);
    position: absolute;
    bottom: 0;
    left: 0;
    height: 60%;
    width: 100%;
    z-index: 0;
  }

.dir-item-card__ext,
.dir-item-card__name,
.dir-item-card__item-count,
.dir-item-card__item-size,
.dir-item-card__date {
  color: var(--color-6) !important;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  font-size: 14px;
}

.dir-item-card__ext {
  font-size: 12px;
  width: 100%;
  padding: 0px 6px;
  text-align: center;
}

.dir-item-card__item-count,
.dir-item-card__date,
.dir-item-card__item-size {
  color: var(--color-7) !important;
}

[data-file-type="image"][data-layout="grid"][data-type="file"]
  .dir-item-card__name,
[data-file-type="image"][data-layout="grid"][data-type="file-symlink"]
  .dir-item-card__name {
    color: var(--color-4) !important;
  }

[data-layout="grid"][data-type="file"]
  .dir-item-card__name,
[data-layout="grid"][data-type="file-symlink"]
  .dir-item-card__name {
    color: var(--color-5) !important;
    line-height: 1.3;
  }

[data-layout="grid"][data-type="file"]
  .dir-item-card__item-count,
[data-layout="grid"][data-type="file-symlink"]
  .dir-item-card__item-count {
    font-size: 14px;
  }

[data-file-type="image"][data-layout="grid"][data-type="file"]
  .dir-item-card__item-count,
[data-file-type="image"][data-layout="grid"][data-type="file-symlink"]
  .dir-item-card__item-count {
    color: var(--color-6) !important;
  }

/* CARD THUMB */
[data-layout="list"]
  .dir-item-card__thumb-container {
    width: 48px;
    height: 48px;
    background-color: rgb(159, 168, 218, 0.1);
  }

[data-layout="grid"][data-type="directory"]
  .dir-item-card__thumb-container,
[data-layout="grid"][data-type="directory-symlink"]
  .dir-item-card__thumb-container {
    width: 64px;
    height: 64px;
  }

[data-layout="grid"][data-type="file"]
  .dir-item-card__thumb-container,
[data-layout="grid"][data-type="file-symlink"]
  .dir-item-card__thumb-container {
    width: 100%;
    height: 100%;
  }

.dir-item-card__thumb {
  width: 100%;
  height: 100%;
}

img.dir-item-card__thumb {
  object-fit: cover;
}

[data-layout="list"]
  .dir-item-card__name__line-2 {
    color: var(--color-7);
    font-size: 12px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

[data-layout="list"][data-type="file"]
  .dir-item-card__thumb,
[data-layout="list"][data-type="file-symlink"]
  .dir-item-card__thumb {
    background-color: var(--file-thumb-color);
    width: 48px;
  }

[data-layout="grid"][data-type="file"]
  .dir-item-card__thumb,
[data-layout="grid"][data-type="file-symlink"]
  .dir-item-card__thumb {
    background-color: var(--file-thumb-color);
    width: 100%;
  }

[data-type="directory"]
  .dir-item-card__thumb,
[data-type="directory-symlink"]
  .dir-item-card__thumb {
    background-color: var(--directory-thumb-color);
  }

[data-type="file"]
  .dir-item-card__icon,
[data-type="file-symlink"]
  .dir-item-card__icon {
    color: var(--dir-item-card-icon-color) !important;
  }

[data-type="directory"]
  .dir-item-card__icon,
[data-type="directory-symlink"]
  .dir-item-card__icon {
    color: var(--dir-item-card-icon-color) !important;
  }

[data-layout="grid"][data-type="file"]
  .dir-item-card__icon,
[data-layout="grid"][data-type="file-symlink"]
  .dir-item-card__icon {
    margin-top: -24px;
  }

.dir-item-card__actions {
  position: absolute;
  display: flex;
  align-items: center;
  justify-content: center;
  right: 48px;
  top: 50%;
  transform: translateY(-50%);
  z-index: 1;
  opacity: 0;
  transition: all 0.2s ease;
}

.dir-item-card:hover
  .dir-item-card__actions {
    opacity: 1;
    transition: all 0.3s;
  }

.dir-item-card__checkbox {
  /* position: absolute; */
  right: 0;
  justify-self: center;
  z-index: 3;
}

.dir-item-card__checkbox
  .v-icon {
    color: var(--dir-item-card-checkbox-color) !important;
  }

[data-layout="grid"][data-type="file"]
  .dir-item-card__checkbox,
[data-layout="grid"][data-type="file-symlink"]
  .dir-item-card__checkbox {
    position: absolute;
    bottom: 8px;
    right: 8px;
    padding: 0px;
    text-shadow: 0 1px 4px rgb(0, 0, 0, 0.5);
  }
</style>
