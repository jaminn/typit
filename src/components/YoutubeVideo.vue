<template>
  <div :id="id"></div>
</template>

<script lang="coffee">
import YouTubeIframeLoader from 'youtube-iframe'

export default 
  props: 
    id: String
    height: {type: Number, default: 360}
    width:  {type: Number, default: 640}
    videoId: {type: String, default: 'M7lc1UVf-VE'}

  mounted: ->
    YouTubeIframeLoader.load (YT) => @onIframeAPIReady(YT)
  
  methods:
    onIframeAPIReady: (YT) ->
      @player = new YT.Player(@id, 
        height:  @height
        width:   @width
        videoId: @videoId
        events: 
          onReady: @onReady
          onStateChange: @onStateChange
      )

    onReady: ->
      @$emit('ready', @player)

    onStateChange: (event) ->
      switch event.data
        when YT.PlayerState.PLAYING then @$emit('playing')
        when YT.PlayerState.ENDED   then @$emit('ended')
        when YT.PlayerState.PAUSED  then @$emit('paused') 
        when YT.PlayerState.BUFFERING then @$emit('buffering') 
        when YT.PlayerState.CUED then @$emit('cued')
        
</script>

<style scoped>

</style>
