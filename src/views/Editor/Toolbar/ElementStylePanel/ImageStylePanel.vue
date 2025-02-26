<template>
  <div class="image-style-panel">
    <div 
      class="origin-image"
      :style="{ backgroundImage: `url(${handleElement.src})` }"
    ></div>

    <ButtonGroup class="row">
      <Button style="flex: 5;" @click="clipImage()"><IconTailoring class="btn-icon" /> 裁剪图片</Button>
      <Popover trigger="click" v-model:visible="clipPanelVisible">
        <template #content>
          <div class="clip">
            <div class="title">按形状：</div>
            <div class="shape-clip">
              <div 
                class="shape-clip-item" 
                v-for="(item, key) in shapeClipPathOptions" 
                :key="key"
                @click="presetImageClip(key)"
              >
                <div class="shape" :style="{ clipPath: item.style }"></div>
              </div>
            </div>

            <template v-for="type in ratioClipOptions" :key="type.label">
              <div class="title" v-if="type.label">按{{type.label}}：</div>
              <ButtonGroup class="row">
                <Button 
                  style="flex: 1;"
                  v-for="item in type.children"
                  :key="item.key"
                  @click="presetImageClip('rect', item.ratio)"
                >{{item.key}}</Button>
              </ButtonGroup>
            </template>
          </div>
        </template>
        <Button class="no-padding" style="flex: 1;"><IconDown /></Button>
      </Popover>
    </ButtonGroup>

    <Popover trigger="click">
      <template #content>
        <div class="filter">
          <div class="filter-item" v-for="filter in filterOptions" :key="filter.key">
            <div class="name">{{filter.label}}</div>
            <Slider
              class="filter-slider"
              :max="filter.max"
              :min="filter.min"
              :step="filter.step"
              :value="filter.value"
              @change="value => updateFilter(filter, value)"
            />
            <div class="value">{{filter.value}}</div>
          </div>
        </div>
      </template>
      <Button class="full-width-btn"><IconColorFilter class="btn-icon" /> 设置滤镜</Button>
    </Popover>
  
    <ElementFlip />
    <Divider />
    <ElementOutline />
    <Divider />
    <ElementShadow />
    <Divider />
    
    <FileInput @change="files => replaceImage(files)">
      <Button class="full-width-btn"><IconTransform class="btn-icon" /> 替换图片</Button>
    </FileInput>
    <Button class="full-width-btn" @click="resetImage()"><IconUndo class="btn-icon" /> 重置样式</Button>
    <Button class="full-width-btn" @click="setBackgroundImage()"><IconTheme class="btn-icon" /> 设为背景</Button>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, watch } from 'vue'
import { storeToRefs } from 'pinia'
import { useMainStore, useSlidesStore } from '@/store'
import { PPTImageElement, SlideBackground } from '@/types/slides'
import { CLIPPATHS } from '@/configs/imageClip'
import { getImageDataURL } from '@/utils/image'
import useHistorySnapshot from '@/hooks/useHistorySnapshot'

import ElementOutline from '../common/ElementOutline.vue'
import ElementShadow from '../common/ElementShadow.vue'
import ElementFlip from '../common/ElementFlip.vue'

interface FilterOption {
  label: string;
  key: string;
  default: number;
  value: number;
  unit: string;
  max: number;
  step: number;
}

const defaultFilters: FilterOption[] = [
  { label: '模糊', key: 'blur', default: 0, value: 0, unit: 'px', max: 10, step: 1 },
  { label: '亮度', key: 'brightness', default: 100, value: 100, unit: '%', max: 200, step: 5 },
  { label: '对比度', key: 'contrast', default: 100, value: 100, unit: '%', max: 200, step: 5 },
  { label: '灰度', key: 'grayscale', default: 0, value: 0, unit: '%', max: 100, step: 5 },
  { label: '饱和度', key: 'saturate', default: 100, value: 100, unit: '%', max: 200, step: 5 },
  { label: '色相', key: 'hue-rotate', default: 0, value: 0, unit: 'deg', max: 360, step: 10 },
  { label: '不透明度', key: 'opacity', default: 100, value: 100, unit: '%', max: 100, step: 5 },
]

const shapeClipPathOptions = CLIPPATHS
const ratioClipOptions = [
  {
    label: '纵横比（方形）',
    children: [
      { key: '1:1', ratio: 1 / 1 },
    ],
  },
  {
    label: '纵横比（纵向）',
    children: [
      { key: '2:3', ratio: 3 / 2 },
      { key: '3:4', ratio: 4 / 3 },
      { key: '3:5', ratio: 5 / 3 },
      { key: '4:5', ratio: 5 / 4 },
    ],
  },
  {
    label: '纵横比（横向）',
    children: [
      { key: '3:2', ratio: 2 / 3 },
      { key: '4:3', ratio: 3 / 4 },
      { key: '5:3', ratio: 3 / 5 },
      { key: '5:4', ratio: 4 / 5 },
    ],
  },
  {
    children: [
      { key: '16:9', ratio: 9 / 16 },
      { key: '16:10', ratio: 10 / 16 },
    ],
  },
]

export default defineComponent({
  name: 'image-style-panel',
  components: {
    ElementOutline,
    ElementShadow,
    ElementFlip,
  },
  setup() {
    const mainStore = useMainStore()
    const slidesStore = useSlidesStore()
    const { handleElement, handleElementId } = storeToRefs(mainStore)
    const { currentSlide } = storeToRefs(slidesStore)

    const clipPanelVisible = ref(false)

    const filterOptions = ref<FilterOption[]>(JSON.parse(JSON.stringify(defaultFilters)))

    watch(handleElement, () => {
      if (!handleElement.value || handleElement.value.type !== 'image') return
      
      const filters = handleElement.value.filters
      if (filters) {
        filterOptions.value = defaultFilters.map(item => {
          if (filters[item.key] !== undefined) return { ...item, value: parseInt(filters[item.key]) }
          return item
        })
      }
      else filterOptions.value = JSON.parse(JSON.stringify(defaultFilters))
    }, { deep: true, immediate: true })

    const { addHistorySnapshot } = useHistorySnapshot()

    // 设置滤镜
    const updateFilter = (filter: FilterOption, value: number) => {
      const _handleElement = handleElement.value as PPTImageElement
      
      const originFilters = _handleElement.filters || {}
      const filters = { ...originFilters, [filter.key]: `${value}${filter.unit}` }
      slidesStore.updateElement({ id: handleElementId.value, props: { filters } })
      addHistorySnapshot()
    }

    // 打开自由裁剪
    const clipImage = () => {
      mainStore.setClipingImageElementId(handleElementId.value)
      clipPanelVisible.value = false
    }

    // 获取原始图片的位置大小
    const getImageElementDataBeforeClip = () => {
      const _handleElement = handleElement.value as PPTImageElement

      // 图片当前的位置大小和裁剪范围
      const imgWidth = _handleElement.width
      const imgHeight = _handleElement.height
      const imgLeft = _handleElement.left
      const imgTop = _handleElement.top
      const originClipRange: [[number, number], [number, number]] = _handleElement.clip ? _handleElement.clip.range : [[0, 0], [100, 100]]

      const originWidth = imgWidth / ((originClipRange[1][0] - originClipRange[0][0]) / 100)
      const originHeight = imgHeight / ((originClipRange[1][1] - originClipRange[0][1]) / 100)
      const originLeft = imgLeft - originWidth * (originClipRange[0][0] / 100)
      const originTop = imgTop - originHeight * (originClipRange[0][1] / 100)

      return {
        originClipRange,
        originWidth,
        originHeight,
        originLeft,
        originTop,
      }
    }

    // 预设裁剪
    const presetImageClip = (shape: string, ratio = 0) => {
      const _handleElement = handleElement.value as PPTImageElement

      const {
        originClipRange,
        originWidth,
        originHeight,
        originLeft,
        originTop,
      } = getImageElementDataBeforeClip()
      
      // 纵横比裁剪（形状固定为矩形）
      if (ratio) {
        const imageRatio = originHeight / originWidth

        const min = 0
        const max = 100
        let range: [[number, number], [number, number]]

        if (imageRatio > ratio) {
          const distance = ((1 - ratio / imageRatio) / 2) * 100
          range = [[min, distance], [max, max - distance]]
        }
        else {
          const distance = ((1 - imageRatio / ratio) / 2) * 100
          range = [[distance, min], [max - distance, max]]
        }
        slidesStore.updateElement({
          id: handleElementId.value,
          props: {
            clip: { ..._handleElement.clip, shape, range },
            left: originLeft + originWidth * (range[0][0] / 100),
            top: originTop + originHeight * (range[0][1] / 100),
            width: originWidth * (range[1][0] - range[0][0]) / 100,
            height: originHeight * (range[1][1] - range[0][1]) / 100,
          },
        })
      }
      // 形状裁剪（保持当前裁剪范围）
      else {
        slidesStore.updateElement({
          id: handleElementId.value,
          props: {
            clip: { ..._handleElement.clip, shape, range: originClipRange }
          },
        })
      }
      clipImage()
      addHistorySnapshot()
    }

    // 替换图片（保持当前的样式）
    const replaceImage = (files: File[]) => {
      const imageFile = files[0]
      if (!imageFile) return
      getImageDataURL(imageFile).then(dataURL => {
        const props = { src: dataURL }
        slidesStore.updateElement({ id: handleElementId.value, props })
      })
      addHistorySnapshot()
    }

    // 重置图片：清除全部样式
    const resetImage = () => {
      const _handleElement = handleElement.value as PPTImageElement

      if (_handleElement.clip) {
        const {
          originWidth,
          originHeight,
          originLeft,
          originTop,
        } = getImageElementDataBeforeClip()

        slidesStore.updateElement({
          id: handleElementId.value,
          props: {
            left: originLeft,
            top: originTop,
            width: originWidth,
            height: originHeight,
          },
        })
      }

      slidesStore.removeElementProps({
        id: handleElementId.value,
        propName: ['clip', 'outline', 'flip', 'shadow', 'filters'],
      })
      addHistorySnapshot()
    }

    // 将图片设置为背景
    const setBackgroundImage = () => {
      const _handleElement = handleElement.value as PPTImageElement

      const background: SlideBackground = {
        ...currentSlide.value.background,
        type: 'image',
        image: _handleElement.src,
        imageSize: 'cover',
      }
      slidesStore.updateSlide({ background })
      addHistorySnapshot()
    }

    return {
      clipPanelVisible,
      shapeClipPathOptions,
      ratioClipOptions,
      filterOptions,
      handleElement,
      updateFilter,
      clipImage,
      presetImageClip,
      replaceImage,
      resetImage,
      setBackgroundImage,
    }
  },
})
</script>

<style lang="scss" scoped>
.row {
  width: 100%;
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}
.switch-wrapper {
  text-align: right;
}
.origin-image {
  height: 100px;
  background-size: contain;
  background-repeat: no-repeat;
  background-position: center;
  background-color: $lightGray;
  margin-bottom: 10px;
}
.full-width-btn {
  width: 100%;
  margin-bottom: 10px;
}
.btn-icon {
  margin-right: 3px;
}

.filter {
  width: 280px;
  font-size: 12px;
}
.filter-item {
  padding: 8px 5px;
  display: flex;
  justify-content: center;
  align-items: center;

  .name {
    width: 60px;
  }
  .filter-slider {
    flex: 1;
    margin: 0 6px;
  }
  .value {
    width: 40px;
    text-align: right;
  }
}

.clip {
  width: 260px;
  font-size: 12px;

  .title {
    margin-bottom: 5px;
  }
}
.shape-clip {
  margin-bottom: 10px;

  @include flex-grid-layout();
}
.shape-clip-item {
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;

  @include flex-grid-layout-children(5, 16%);

  &:hover .shape {
    background-color: #ccc;
  }

  .shape {
    width: 40px;
    height: 40px;
    background-color: #e1e1e1;
  }
}
</style>