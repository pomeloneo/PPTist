<template>
  <div class="common-element-operate">
    <BorderLine 
      class="operate-border-line"
      v-for="line in borderLines" 
      :key="line.type" 
      :type="line.type" 
      :style="line.style"
    />
    <template v-if="handlerVisible">
      <ResizeHandler
        class="operate-resize-handler" 
        v-for="point in resizeHandlers"
        :key="point.direction"
        :type="point.direction"
        :rotate="elementInfo.rotate"
        :style="point.style"
        @mousedown.stop="$event => scaleElement($event, elementInfo, point.direction)"
      />
      <RotateHandler
        class="operate-rotate-handler" 
        :style="{ left: scaleWidth / 2 + 'px' }"
        @mousedown.stop="rotateElement(elementInfo)"
      />
    </template>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, PropType } from 'vue'
import { storeToRefs } from 'pinia'
import { useMainStore } from '@/store'
import { PPTShapeElement, PPTVideoElement, PPTLatexElement } from '@/types/slides'
import { OperateResizeHandler } from '@/types/edit'
import useCommonOperate from '../hooks/useCommonOperate'

import RotateHandler from './RotateHandler.vue'
import ResizeHandler from './ResizeHandler.vue'
import BorderLine from './BorderLine.vue'

type PPTElement = PPTShapeElement | PPTVideoElement | PPTLatexElement

export default defineComponent({
  name: 'common-element-operate',
  inheritAttrs: false,
  components: {
    RotateHandler,
    ResizeHandler,
    BorderLine,
  },
  props: {
    elementInfo: {
      type: Object as PropType<PPTElement>,
      required: true,
    },
    handlerVisible: {
      type: Boolean,
      required: true,
    },
    rotateElement: {
      type: Function as PropType<(element: PPTElement) => void>,
      required: true,
    },
    scaleElement: {
      type: Function as PropType<(e: MouseEvent, element: PPTElement, command: OperateResizeHandler) => void>,
      required: true,
    },
  },
  setup(props) {
    const { canvasScale } = storeToRefs(useMainStore())

    const scaleWidth = computed(() => props.elementInfo.width * canvasScale.value)
    const scaleHeight = computed(() => props.elementInfo.height * canvasScale.value)
    const { resizeHandlers, borderLines } = useCommonOperate(scaleWidth, scaleHeight)

    return {
      scaleWidth,
      resizeHandlers,
      borderLines,
    }
  },
})
</script>