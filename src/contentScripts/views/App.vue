<script setup lang="ts">
/**
 * h/H: 后退
 * l/L: 前进
 * j: 减小音量
 * k: 增加音量
 * m: 静音 / 取消静音
 * z: 重置播放倍速
 * x: 减小播放倍速
 * c: 增加播放倍速
 *  (space): 暂停
 */
import 'uno.css'
import { Toast } from '../Toast'
import type { Plugin } from '../types'
import { PausePlugin, ProgressPlugin, ScreenPlugin, SpeedPlugin, VolumePlugin } from '../plugins'

const plugins: Plugin[] = [
  new PausePlugin(),
  new SpeedPlugin(),
  new ProgressPlugin(),
  new VolumePlugin(),
  new ScreenPlugin(),
]
const TAG = 'video-controller'
let toast: Toast
let observer: MutationObserver

const videoListDisplayed = ref(false)
const videoElements = ref<HTMLVideoElement[]>([])
const videoSelectedIndex = ref(0)
const videoListRef = ref(null)

const video = computed<HTMLVideoElement>(() => videoElements.value[videoSelectedIndex.value])

window.addEventListener('load', () => {
  init()
})

window.addEventListener('beforeunload', () => {
  observer && observer.disconnect()
})

function init() {
  toast = new Toast()
  listenVideoUpdate(() => {
    videoElements.value = getAllValidVideoElements()
    if (videoElements.value.length && !video.value)
      videoSelectedIndex.value = 0

    logInfo(videoElements.value)
    logInfo(video.value, videoSelectedIndex.value)
  })

  listenKeyEvent('keyup')
  listenKeyEvent('keydown')
}

function listenKeyEvent(type: 'keyup' | 'keydown') {
  const fnMap: Record<typeof type, keyof Plugin> = {
    keyup: 'onKeyUp',
    keydown: 'onKeyDown',
  }
  window.addEventListener(type, (event) => {
    if (!shouldMapKey(event))
      return

    let ret = false
    plugins.forEach((plugin) => {
      ret ||= plugin[fnMap[type]]({
        event, video: video.value, toast,
      })
    })

    if (ret) {
      // 避免和网站原本的快捷键功能冲突，优先使用我们自定义的
      event.stopImmediatePropagation()
      event.preventDefault()
    }
  },
  {
    // 在事件捕获阶段执行回调
    // 这样可以保证回调的优先级，有利于后续阻止其他 handle 执行
    capture: true,
  })
}

function listenVideoUpdate(onVideoUpdate: () => void) {
  observer = new MutationObserver((mutations) => {
    mutations.forEach((mutation) => {
      mutation.addedNodes?.forEach((node) => {
        if (!mayContainsVideoElement(node))
          return
        setTimeout(onVideoUpdate, 1000)
      })
      mutation.removedNodes?.forEach((node) => {
        if (!mayContainsVideoElement(node))
          return
        setTimeout(onVideoUpdate)
      })
    })
  })

  observer.observe(document.body, {
    subtree: true,
    childList: true,
  })

  setTimeout(onVideoUpdate)

  window.addEventListener('hashchange', onVideoUpdate)
}

function mayContainsVideoElement(node: Node) {
  if (node.nodeType !== Node.ELEMENT_NODE)
    return false

  if (node.nodeName === 'VIDEO')
    return true
  if (/video/i.test((node as HTMLElement).className))
    return true
  if ((node as HTMLElement).querySelector('iframe'))
    return true

  return false
}

function getAllValidVideoElements(frame: Window = window) {
  const elements: HTMLVideoElement[] = []

  try {
    if (frame.document) {
      elements.push(...Array.from(frame.document.querySelectorAll('video') ?? []).filter(
        video => video.src,
      ))
    }
  }
  catch (error) {
    // eslint-disable-next-line no-console
    console.info('Read frame error.', error)
  }

  if (frame.frames.length) {
    Array.from(frame.frames).forEach((f) => {
      elements.push(...getAllValidVideoElements(f))
    })
  }
  return elements
}

function shouldMapKey(event: KeyboardEvent) {
  if (
    ['input', 'textarea'].includes(
      (event.target as Element)?.tagName?.toLowerCase(),
    )
  )
    return false

  if (event.metaKey || event.altKey || event.ctrlKey)
    return false

  if (!video.value)
    return false

  return true
}

function selectVideo(index: number) {
  const video = videoElements.value[index]
  logInfo('mouseEnter', videoElements.value, index, video)
  if (video.parentElement?.firstElementChild?.className === 'video-controller-video-outline')
    return
  const div = document.createElement('div')
  div.className = 'video-controller-video-outline'
  Object.assign(div.style, {
    position: 'absolute',
    left: 0,
    right: 0,
    top: 0,
    bottom: 0,
    border: '6px solid red',
    zIndex: '10',
    boxSizing: 'border-box',
    width: video.offsetWidth,
    height: video.offsetHeight,
    margin: '0 auto',
  })
  video.parentElement?.prepend(div)
}

function unselectVideo(index: number) {
  const video = videoElements.value[index]
  logInfo('mouseLeave', videoElements.value, index, video)
  if (video.parentElement?.firstElementChild?.className !== 'video-controller-video-outline')
    return
  video.parentElement?.firstElementChild.remove()
}

function logInfo(message: any, ...optionalParams: any[]) {
  if (import.meta.env.DEV)
    // eslint-disable-next-line no-console
    console.log(`[${TAG}]`, message, ...optionalParams)
}
</script>

<template>
  <div
    v-if="videoElements.length"
    class="fixed right-0 bottom-0 m-20px z-100 flex items-end font-sans select-none leading-1em text-16px"
    @mouseenter="videoListDisplayed = true"
    @mouseleave="videoListDisplayed = false"
  >
    <ul
      ref="videoListRef"
      class="list-none m-0 p-0 bg-white text-gray-800 rounded-lg shadow w-max h-min"
      p="x-16px y-12px"
      m="y-auto r-8px"
      transition="opacity duration-300"
      :class="videoListDisplayed ? '' : 'hidden'"
    >
      <li
        v-for="index in videoElements.length"
        :key="index"
        @mouseenter="selectVideo(index - 1)"
        @mouseleave="unselectVideo(index - 1)"
      >
        <label class="flex items-center cursor-pointer py-4px">
          <input
            v-model="videoSelectedIndex"
            class="ml-0"
            type="radio"
            :value="index - 1"
          >
          Video {{ index }}
        </label>
      </li>
    </ul>
    <button
      class="flex w-40px h-40px rounded-full shadow cursor-pointer border-none"
      bg="teal-600 hover:teal-700"
    >
      <pixelarticons-power
        class="block m-auto text-white text-18px leading-28px"
      />
    </button>
  </div>
</template>
