<template>
  <div :class="ui.wrapper" v-bind="attrs">
    <slot name="prev" :on-click="onClickPrev">
      <UButton
        v-if="prevButton"
        :size="size"
        :disabled="!canGoPrev"
        :class="[ui.base, ui.rounded]"
        v-bind="{ ...ui.default.prevButton, ...prevButton }"
        :ui="{ rounded: '' }"
        aria-label="Prev"
        @click="onClickPrev"
      />
    </slot>

    <UButton
      v-for="(page, index) of displayedPages"
      :key="`${page}-${index}`"
      :size="size"
      :label="`${page}`"
      v-bind="page === currentPage ? { ...ui.default.activeButton, ...activeButton } : { ...ui.default.inactiveButton, ...inactiveButton }"
      :class="[{ 'pointer-events-none': typeof page === 'string', 'z-[1]': page === currentPage }, ui.base, ui.rounded]"
      :ui="{ rounded: '' }"
      @click="() => onClickPage(page)"
    />

    <slot name="next" :on-click="onClickNext">
      <UButton
        v-if="nextButton"
        :size="size"
        :disabled="!canGoNext"
        :class="[ui.base, ui.rounded]"
        v-bind="{ ...ui.default.nextButton, ...nextButton }"
        :ui="{ rounded: '' }"
        aria-label="Next"
        @click="onClickNext"
      />
    </slot>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent } from 'vue'
import type { PropType } from 'vue'
import UButton from '../elements/Button.vue'
import { useUI } from '../../composables/useUI'
import { mergeConfig } from '../../utils'
import type { Button, Strategy } from '../../types'
// @ts-expect-error
import appConfig from '#build/app.config'
import { pagination, button } from '#ui/ui.config'

const config = mergeConfig<typeof pagination>(appConfig.ui.strategy, appConfig.ui.pagination, pagination)

const buttonConfig = mergeConfig<typeof button>(appConfig.ui.strategy, appConfig.ui.button, button)

export default defineComponent({
  components: {
    UButton
  },
  inheritAttrs: false,
  props: {
    modelValue: {
      type: Number,
      required: true
    },
    pageCount: {
      type: Number,
      default: 10
    },
    total: {
      type: Number,
      required: true
    },
    max: {
      type: Number,
      default: 7,
      validate (value) {
        return value >= 7 && value < Number.MAX_VALUE
      }
    },
    size: {
      type: String as PropType<keyof typeof buttonConfig.size>,
      default: () => config.default.size,
      validator (value: string) {
        return Object.keys(buttonConfig.size).includes(value)
      }
    },
    activeButton: {
      type: Object as PropType<Button>,
      default: () => config.default.activeButton as Button
    },
    inactiveButton: {
      type: Object as PropType<Button>,
      default: () => config.default.inactiveButton as Button
    },
    prevButton: {
      type: Object as PropType<Button>,
      default: () => config.default.prevButton as Button
    },
    nextButton: {
      type: Object as PropType<Button>,
      default: () => config.default.nextButton as Button
    },
    divider: {
      type: String,
      default: '…'
    },
    ui: {
      type: Object as PropType<Partial<typeof config & { strategy?: Strategy }>>,
      default: undefined
    }
  },
  emits: ['update:modelValue'],
  setup (props, { emit }) {
    const { ui, attrs } = useUI('pagination', props.ui, config, { mergeWrapper: true })

    const currentPage = computed({
      get () {
        return props.modelValue
      },
      set (value) {
        emit('update:modelValue', value)
      }
    })

    const pages = computed(() => Array.from({ length: Math.ceil(props.total / props.pageCount) }, (_, i) => i + 1))

    const displayedPages = computed(() => {
      if (!props.max || pages.value.length <= 5) {
        return pages.value
      } else {
        const current = currentPage.value
        const max = pages.value.length
        const r = Math.floor((Math.min(props.max, max) - 5) / 2)
        const r1 = current - r
        const r2 = current + r
        const beforeWrapped = r1 - 1 > 1
        const afterWrapped = r2 + 1 < max
        const items: Array<number | string> = [1]

        if (beforeWrapped) items.push(props.divider)

        if (!afterWrapped) {
          const addedItems = (current + r + 2) - max
          for (let i = current - r - addedItems; i <= current - r - 1; i++) {
            items.push(i)
          }
        }

        for (let i = r1 > 2 ? (r1) : 2; i <= Math.min(max, r2); i++) {
          items.push(i)
        }

        if (!beforeWrapped) {
          const addedItems = 1 - (current - r - 2)
          for (let i = current + r + 1; i <= current + r + addedItems; i++) {
            items.push(i)
          }
        }

        if (afterWrapped) items.push(props.divider)

        if (r2 < max) items.push(max)

        // Replace divider by number on start edge case [1, '…', 3, ...]
        if (items.length >= 3 && items[1] === props.divider && items[2] === 3) {
          items[1] = 2
        }
        // Replace divider by number on end edge case [..., 48, '…', 50]
        if (items.length >= 3 && items[items.length - 2] === props.divider && items[items.length - 1] === items.length) {
          items[items.length - 2] = items.length - 1
        }

        return items
      }
    })

    const canGoPrev = computed(() => currentPage.value > 1)
    const canGoNext = computed(() => currentPage.value < pages.value.length)

    function onClickPage (page: number | string) {
      if (typeof page === 'string') {
        return
      }

      currentPage.value = page
    }

    function onClickPrev () {
      if (!canGoPrev.value) {
        return
      }

      currentPage.value--
    }

    function onClickNext () {
      if (!canGoNext.value) {
        return
      }

      currentPage.value++
    }

    return {
      // eslint-disable-next-line vue/no-dupe-keys
      ui,
      attrs,
      currentPage,
      pages,
      displayedPages,
      canGoPrev,
      canGoNext,
      onClickPrev,
      onClickNext,
      onClickPage
    }
  }
})
</script>
