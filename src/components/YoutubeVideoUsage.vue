<template>
    <YoutubeVideo id="video" @ready="videoReady" @paused="videoPaused"/>
</template>

<script lang="coffee">
import YoutubeVideo from "./YoutubeVideo"

export default 
  components: 
    YoutubeVideo: YoutubeVideo
  methods:
    videoReady: (player) ->
      console.log "[videoReady] 비디오가 준비 되었습니다."
      player.playVideo()
    videoPaused: (player) ->
      console.log "[videoPaused] 비디오가 멈쳤습니다."
      setTimeout (=> player.playVideo()), 3000  

</script>

<style>

</style>
