<template>
  <div class="w-[65ch] mx-auto my-24">
    <h1 id="title" contenteditable="true" spellcheck="false" data-ph="Untitled" @blur="props.page.name=($event.target as HTMLElement).innerText.replace('\n', '')"
      class="px-4 sm:px-0 focus:outline-none focus-visible:outline-none text-5xl font-bold mb-12"
      :class="props.page.name ? '' : 'empty'">
      {{ props.page.name || '' }}
    </h1>
    <draggable tag="div" :list="props.page.blocks"
      v-bind="dragOptions" class="-ml-24 space-y-2 pb-4">
      <transition-group type="transition">
        <BlockComponent :block="block" v-for="block, i in props.page.blocks" :key="i"
          :ref="el => blockElements[i] = (el as unknown as typeof Block)"
          @deleteBlock="props.page.blocks.splice(i, 1)"
          @newBlock="insertBlock(i)"
          @moveToPrevChar="() => { if (blockElements[i-1]) blockElements[i-1].moveToEnd(); scrollIntoView(); }"
          @moveToNextChar="() => { if (blockElements[i+1]) blockElements[i+1].moveToStart(); scrollIntoView(); }"
          @moveToPrevLine="() => { if (blockElements[i-1]) blockElements[i-1].moveToLastLine(); scrollIntoView(); }"
          @moveToNextLine="() => { if (blockElements[i+1]) blockElements[i+1].moveToFirstLine(); scrollIntoView(); }"
          @merge="merge(i)"
          @split="split(i)"
          @setBlockType="type => setBlockType(i, type)"
          />
      </transition-group>
    </draggable>
  </div>
</template>

<style>
#title.empty[contenteditable='true']:before{
  content:attr(data-ph);
  color:#BBBBBB;
}
.ghost {
  opacity: 1;
  background: #F5F5F5;
}
</style>

<script setup lang="ts">
import { ref, onBeforeUpdate, PropType } from 'vue'
import { VueDraggableNext as draggable } from 'vue-draggable-next'
import { Block, BlockType } from '@/utils/types'
import BlockComponent from './Block.vue'

const props = defineProps({
  page: {
    type: Object as PropType<{ name:string, blocks:Block[] }>,
    required: true,
  }
})

const dragOptions = {
  animation: 150,
  group: 'blocks',
  disabled: false,
  ghostClass: 'ghost',
}


onBeforeUpdate(() => {
  blockElements.value = []
})

const blockElements = ref<typeof BlockComponent[]>([])

function scrollIntoView () {
  const selection = window.getSelection()
  if (selection?.anchorNode?.nodeType === Node.ELEMENT_NODE) {
    (selection?.anchorNode as HTMLElement).scrollIntoView({behavior: "smooth", block: "center", inline: "nearest"})
  } else {
    (selection?.anchorNode?.parentElement as HTMLElement).scrollIntoView({behavior: "smooth", block: "center", inline: "nearest"})
  }
}

function insertBlock (blockIdx: number) {
  props.page.blocks.splice(blockIdx + 1, 0, {
    type: BlockType.Text,
    details: {
      value: '',
    },
  })
  setTimeout(() => {
    blockElements.value[blockIdx+1].moveToStart()
    scrollIntoView()
  })
}

function setBlockType (blockIdx: number, type: BlockType) {
  if (props.page.blocks[blockIdx].type === BlockType.Text) {
    props.page.blocks[blockIdx].details.value = blockElements.value[blockIdx].getTextContent()
  }
  props.page.blocks[blockIdx].type = type
  if (type === BlockType.Divider) {
    props.page.blocks[blockIdx].details = {}
    insertBlock(blockIdx)
  } else setTimeout(() => blockElements.value[blockIdx].moveToEnd())
}

function merge (blockIdx: number) {
  if (props.page.blocks[blockIdx-1].type === BlockType.Text) {
    const prevBlockContentLength = blockElements.value[blockIdx-1].getTextContent().length
    props.page.blocks[blockIdx-1].details.value = ('<p>' + (props.page.blocks[blockIdx-1] as any).details.value.replace('<p>', '').replace('</p>', '') + blockElements.value[blockIdx].getHtmlContent().replace('<p>', '').replace('</p>', '') + '</p>').replace('</strong><strong>', '').replace('</em><em>', '')
    setTimeout(() => {
      blockElements.value[blockIdx-1].setCaretPos(prevBlockContentLength)
      props.page.blocks.splice(blockIdx, 1)
    })
  } else if ([BlockType.H1, BlockType.H2].includes(props.page.blocks[blockIdx-1].type)) {
    const prevBlockContentLength = (props.page.blocks[blockIdx-1] as any).details.value.length
    props.page.blocks[blockIdx-1].details.value += blockElements.value[blockIdx].getTextContent()
    setTimeout(() => {
      blockElements.value[blockIdx-1].setCaretPos(prevBlockContentLength)
      props.page.blocks.splice(blockIdx, 1)
    })
  } else {
    props.page.blocks.splice(blockIdx-1, 1)
    setTimeout(() => blockElements.value[blockIdx-1].moveToStart())
  }
}

function split (blockIdx: number) {
  const caretPos = blockElements.value[blockIdx].getCaretPos()
  insertBlock(blockIdx)
  props.page.blocks[blockIdx+1].details.value = (caretPos.tag ? `<p><${caretPos.tag}>` : '<p>') + props.page.blocks[blockIdx].details.value?.slice(caretPos.pos)
  if (props.page.blocks[blockIdx].type === BlockType.Text) {
    props.page.blocks[blockIdx].details.value = props.page.blocks[blockIdx].details.value?.slice(0, caretPos.pos) + (caretPos.tag ? `</${caretPos.tag}></p>` : '</p>')
  } else {
    props.page.blocks[blockIdx].details.value = props.page.blocks[blockIdx].details.value?.slice(0, caretPos.pos) + (caretPos.tag ? `</${caretPos.tag}></p>` : '')
  }
  setTimeout(() => blockElements.value[blockIdx+1].moveToStart())
}
</script>
