<script setup>
const props = defineProps({
  url: {
    type: String,
    required: true,
  },
})

const video = ref(null)

const assetURL = props.url
const mimeCodec = 'video/mp4; codecs="avc1.42E01E, mp4a.40.2, avc1.64001f"'
let totalSegments
let segmentLength = 0
let segmentDuration = 0
let seeking = false
let bytesFetched = 0

const requestedSegments = []

let mediaSource = null

let sourceBuffer = null

async function sourceOpen() {
  sourceBuffer = mediaSource.addSourceBuffer(mimeCodec)
  sourceBuffer.mode = 'sequence'
  const fileLength = await getFileLength(assetURL)
  totalSegments = Math.ceil(fileLength / (1024 * 1024))
  for (let i = 0; i < totalSegments; ++i) requestedSegments[i] = false

  segmentLength = Math.round(fileLength / totalSegments)

  await fetchRange(assetURL, 0, segmentLength, appendSegment)
  requestedSegments[0] = true
  video.value.addEventListener('timeupdate', checkBuffer)
  video.value.addEventListener('canplay', async function () {
    segmentDuration = video.value.duration / totalSegments
    try {
      await video.value.play()
      console.log('Autoplay started!')
    } catch (error) {
      console.error('Autoplay was prevented.')
      // Show a "Play" button so that user can start playback.
    }
  })

  video.value.addEventListener('seeking', debounce(handleSeek, 500))
  video.value.addEventListener('seeked', onSeeked)
}

function debounce(func, wait, immediate) {
  let timeout
  return function () {
    const context = this
    const args = arguments
    const later = function () {
      timeout = null
      if (!immediate) func.apply(context, args)
    }
    const callNow = immediate && !timeout
    clearTimeout(timeout)
    timeout = setTimeout(later, wait)
    if (callNow) func.apply(context, args)
  }
}

function getFileLength(url) {
  return fetch(url, { method: 'HEAD' })
    .then(response => response.headers.get('content-length'))
    .catch(error => console.error(error))
}

function onSeeked() {
  console.log('seeked')
}

function fetchRange(url, start, end, cb) {
  return fetch(url, {
    headers: { Range: `bytes=${start}-${end}` },
  })
    .then(response => response.arrayBuffer())
    .then(chunk => {
      console.log('fetched bytes: ', start, end)
      bytesFetched += end - start + 1
      return cb(chunk)
    })
}

function appendSegment(chunk) {
  return sourceBuffer.appendBuffer(chunk)
}

async function checkBuffer() {
  console.log('check Buffer')
  if (seeking) {
    return true
  }
  const nextSegment = getNextSegment()
  if (nextSegment === totalSegments && haveAllSegments()) {
    mediaSource.endOfStream()
    video.value.removeEventListener('timeupdate', checkBuffer)
  } else if (shouldFetchNextSegment(nextSegment)) {
    requestedSegments[nextSegment] = true

    await fetchRange(
      assetURL,
      bytesFetched,
      bytesFetched + segmentLength,
      appendSegment
    )
  }
}

async function handleSeek() {
  console.log('seeking')
  seeking = true
  const nextSegment = getNextSegment()
  if (nextSegment === totalSegments && haveAllSegments()) {
    mediaSource.endOfStream()
    video.value.removeEventListener('timeupdate', checkBuffer)
  } else {
    for (let segment = 1; segment < nextSegment; segment++) {
      if (shouldFetchNextSegment(segment)) {
        requestedSegments[segment] = true

        await fetchRange(
          assetURL,
          bytesFetched,
          bytesFetched + segmentLength,
          appendSegment
        )
      }
    }
  }
  seeking = false
}

function seek() {
  if (mediaSource.readyState === 'open') {
    handleSeek()
  } else {
    console.log('seek but not open?')
    console.log(mediaSource.readyState)
  }
}

function getNextSegment() {
  return Math.floor(video.value.currentTime / segmentDuration) + 1
}

function haveAllSegments() {
  return requestedSegments.every(val => !!val)
}

function shouldFetchNextSegment(nextSegment) {
  return (
    video.value.currentTime > segmentDuration * nextSegment * 0.1 &&
    requestedSegments[nextSegment] === false
  )
}

onMounted(() => {
  if ('MediaSource' in window && MediaSource.isTypeSupported(mimeCodec)) {
    mediaSource = new MediaSource()
    video.value.src = URL.createObjectURL(mediaSource)
    mediaSource.addEventListener('sourceopen', sourceOpen)
  } else {
    console.error('Unsupported MIME type or codec: ', mimeCodec)
  }
})
</script>

<template>
  <div>
    <video ref="video" controls></video>
  </div>
</template>
