<template>
 <div id="app">>
    <div class="wrapper">
      <div class="header-div">
        <div class="logo-wrapper">
          <div class="logo-div">
            <div class="text-block-3">Type it</div>
          </div>
        </div>
      </div>
      <div class="content-wrapper">
        <div class="content">
          <div class="video w-embed w-video" style="padding-top: 56.17021276595745%;"><YoutubeVideo id="video" :videoId="videoId" @ready="videoReady" @playing="videoPlaying" @paused="videoPaused"/></div>
          <div class="progress-out">
            <div class="progress-in" :style="{width: `${percent}%`}"></div>
          </div>
          <div class="text-wrapper">
            <div class="texts">{{ text }}<span class="gray">{{hint}}</span></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="coffee">
import YoutubeVideo from "./components/YoutubeVideo"
import axios from 'axios'

class SubManager
  constructor: (@vid = 'ZIBWwdCyNro') ->
  getList: () =>
    url = "http://video.google.com/timedtext?type=list&v=#{@vid}"
    res = await axios.get url, {responseType : "document"}
    return res.data.childNodes[0]

  getSub: () =>
    url = "https://video.google.com/timedtext?lang=en&v=#{@vid}"
    res = await axios.get url, {responseType : "document"}
    xml = res.data.childNodes[0]
    htmlDecode = (input) =>
      e = document.createElement('div')
      e.innerHTML = input
      return if e.childNodes.length is 0 then "" else e.childNodes[0].nodeValue

    return subJson = ({
      text  : htmlDecode htmlDecode(node.textContent)
      start : 1 * node.attributes.start.nodeValue
      end   : 1 * node.attributes.start.nodeValue + 1 * node.attributes.dur.nodeValue
      dur   : 1 * node.attributes.dur.nodeValue 
    } for node in xml.childNodes)

class Nextable
  constructor: ({@getNth, @getSize, @onLast}) -> @idx = 0
  next: => 
    if @getSize() <= @idx + 1
      @idx = @getSize()
      @onLast() 
      return -1
    else 
      return @idx++
  
  getNext: => if @getNth(@idx + 1)? then @getNth(@idx + 1) else @getNth(0)
  getCurrent: => if @getNth(@idx)? then @getNth(@idx) else @getNth(0)
  reset: => @idx = 0
  getIdx: () => @idx
  setIdx: (idx) => @idx = idx
  isEnd: ()=> @idx >= @getSize()-1

class GameManager
  constructor: (@videoId, {@getText, @setText, @getHint, @setHint, @playVideo, @pauseVideo, @playSound,@getCurrentTime, @seekTo}) ->
    @sen = ""
    @mode = "NONE"
    @senCon = new Nextable
      getNth:  (n)=> @sen[n]
      getSize: () => @sen.length
      onLast:  () => 
        @subCon.next()
        @sen = @subCon.getCurrent().text
        @senCon.reset()
        @setMode("PLAY")
    @getSub()

  getSub: =>
    @subManager = new SubManager(@videoId)
    @sub = await @subManager.getSub()
    window.sub = @sub
    @subCon = new Nextable
      getNth:  (n)=> @sub[n]
      getSize: () => @sub.length
      onLast:  () => @onSubLast()
    
    @sen = @subCon.getCurrent().text
    @setMode("PLAY")

  setMode: (mode)=>
    switch mode
      when "INPUT"
        @pauseVideo()
        @skipNoneAlpha()
        @displayHint()
      when "PLAY"
        @playVideo()
        clearInterval intervalId 
        clearInterval textClearIntervelId
        textClearIntervelId = setTimeout (=>
          @setText ""
        ), 1000
        @setHint ""
        return if @subCon.isEnd()
        intervalId = setInterval (=>
          #console.log "#{@getCurrentTime()}, #{@subCon.getCurrent().end}"
          if @getCurrentTime() >= @subCon.getCurrent().end
            clearInterval intervalId 
            clearInterval textClearIntervelId
            @setText("") if @getText() is ""  
            @setMode("INPUT")
        ), 10
      else 
        return
      
    @mode = mode


  displayHint: => 
    @setHint @senCon.getCurrent()

  skipNoneAlpha: =>
    idx = 0
    while @senCon.getCurrent()?
      nextAlpha = @senCon.getCurrent().toLowerCase().charCodeAt()
      if nextAlpha not in ['a'.charCodeAt()..'z'.charCodeAt()]
        @setText @getText() + @senCon.getCurrent()
        idx = @senCon.next()
        return -1 if idx is -1
      else
        break
    return idx

  onKeyEvent: (key) => 
    if @mode is "INPUT"
      if key is @senCon.getCurrent().toLowerCase()
        @setText @getText() + @senCon.getCurrent()
        @playSound()
        if @senCon.next() isnt -1
          @displayHint() if @skipNoneAlpha() isnt -1
          
  
  onSubLast: =>
    @setMode "PLAY"
    
  onVideoPlaying: =>
    idx = 0
    for i, one of @sub
      idx++ if one.end < @getCurrentTime()

    if @subCon.getIdx() isnt idx
      @setHint ""
      @setText ""
      @subCon.setIdx idx
      @sen = @subCon.getCurrent().text
      @senCon.reset()
      @setMode "PLAY"
      
  onVideoPaused: =>
    # console.log @subCon.getCurrent().end
    # console.log @getCurrentTime()
    # console.log @subCon.getNext().start
    # console.log  @subCon.getCurrent().end <= @getCurrentTime() <= @subCon.getNext().start + 0.1
    unless @subCon.getCurrent().end <= @getCurrentTime() <= @subCon.getNext().start + 0.1
      @setHint ""
      @setText ""
     

export default 
  name: 'app'
  
  data: ->
    videoId: "wJnBTPUQS5A"
    text: ""
    hint: ""
    percent: 0
  
  components: 
    YoutubeVideo: YoutubeVideo
  
  beforeMount: ->
    @videoId = if "" isnt vid = @getUrlParameter("vid") then vid 

  mounted: ->
    setInterval (=>
      @percent = if @player? then 100 * @player.getCurrentTime() / @player.getDuration() else 0
    ), 100
  methods:
    getUrlParameter: (name) =>
        name = name.replace(/[\[]/, '\\[').replace(/[\]]/, '\\]')
        regex = new RegExp('[\\?&]' + name + '=([^&#]*)')
        results = regex.exec(location.search)
        return if results? then decodeURIComponent(results[1].replace(/\+/g, ' ')) else ''
    
    videoReady: (@player) ->
      @gm = new GameManager(@videoId,
        getText: () => @text
        setText: (val) => @text = val
        getHint: () => @hint
        setHint: (val) => @hint = val
        playVideo:  () => @player.playVideo()
        pauseVideo: () => @player.pauseVideo()
        playSound:  () => (new Audio('/static/type.wav')).play()
        getCurrentTime: () => @player.getCurrentTime()
        seekTo: (sec) => @player.seekTo(sec)
      )
      window.addEventListener 'keydown', (e)=> @gm.onKeyEvent(e.key.toLowerCase())
    videoPlaying: () ->
      @gm.onVideoPlaying()
    videoPaused: () ->
      @gm.onVideoPaused()

</script>

<style lang="sass">
@import url('./assets/flow.css')

.navbar
  position: fixed
  left: 0px
  top: 0px
  right: 0px
  padding-top: 12px
  padding-bottom: 12px
  background-color: hsla(0, 0%, 100%, 0.81)

.navlink
  transition: color 200ms ease
  font-family: 'Changa One', Impact, sans-serif
  color: #363636
  font-size: 16px
  font-weight: 400
  text-transform: uppercase
  &:hover
    color: #00a6c5
  &.w--current
    color: #00811a

.heroheading
  color: #fff

.herosubheading
  width: 60%
  margin-bottom: 51px
  color: #fff
  font-size: 24px
  font-weight: 700

.herobutton
  padding: 14px 40px
  border: 3px solid #0cf
  border-radius: 4px
  background-color: rgba(0, 204, 255, 0.61)
  transition: background-color 200ms ease
  text-align: center
  &:hover
    background-color: #0cf

.main-section
  padding-top: 70px
  padding-bottom: 70px
  &.blue
    background-color: #00c2ff
  &.purple
    background-color: #9600c5
  &.footer
    padding-top: 20px
    padding-bottom: 20px
    background-color: #252525

.heading
  font-family: Arial, 'Helvetica Neue', Helvetica, sans-serif
  font-size: 36px
  font-weight: 300
  text-align: center

.about-block
  padding: 20px
  text-align: center

.about-image
  width: 100px

.white
  color: #fff
  text-align: center
  text-transform: capitalize

.testimonial-slider
  background-color: transparent

.testimonial-block
  display: block
  width: 70%
  margin-right: auto
  margin-left: auto
  text-align: center

.testimonial-image
  display: block
  width: 200px
  max-width: 200px
  clear: none
  text-align: left

.testimonial
  margin-top: 7px
  margin-bottom: 7px
  font-family: Arial, 'Helvetica Neue', Helvetica, sans-serif
  color: #fff
  font-size: 22px
  line-height: 24px
  font-weight: 300
  &.author
    color: #00536d
    font-size: 15px
    line-height: 20px
    font-weight: 400

.subheading-text
  color: #fff
  font-size: 20px
  line-height: 25px
  font-weight: 300
  text-align: center

.form-wrapper
  display: block
  width: 60%
  margin-top: 40px
  margin-right: auto
  margin-left: auto

.text-field
  width: 75%
  height: 50px
  float: left
  border: 3px solid #00a3ff
  &:focus
    border-color: #d4d4d4
    color: #d11bff

.submit-button
  width: 25%
  height: 50px
  float: none
  background-color: #09f
  transition: background-color 200ms ease
  &:hover
    background-color: #818181

.footer-link
  padding: 8px 14px
  float: right
  transition: color 200ms ease
  color: #8d8d8d
  font-weight: 600
  text-decoration: none
  &:hover
    color: #fff

.container
  transition: opacity 200ms ease

.div-block
  display: flex
  overflow: hidden
  width: 100px
  height: 100px
  margin-right: auto
  margin-left: auto
  justify-content: center
  border-radius: 300px

.div-block-2
  position: fixed
  left: 0px
  top: 0px
  right: 0px
  bottom: 0px
  background-color: #a0bccf

.div-block-3
  position: fixed
  left: 0px
  top: 0px
  right: 0px
  bottom: 0px
  background-color: #437495

.brand
  bottom: 0px
  display: block

.div-block-4
  display: inline

.div-block-5
  display: inline-block
  float: left

.div-block-6
  width: 100%
  height: 100%

.div-block-7
  display: flex
  height: 80%
  flex-direction: row
  justify-content: space-between
  background-color: rgba(255, 0, 0, 0.1)

.div-block-8
  height: 20%
  border-color: rgba(0, 255, 239, 0.2)

.div-block-9
  display: flex
  width: 100%
  height: 100%

.div-block-10
  flex: 1

.div-block-11
  width: 20%
  padding-right: 10px
  padding-left: 10px

.heading-2
  margin-top: 0px
  font-size: 25px

.text-block
  overflow: auto
  height: 400px

.wrapper
  position: fixed
  left: 0px
  top: 0px
  right: 0px
  bottom: 0px
  display: flex
  flex-direction: column

.body
  display: flex
  flex: 1
  background-color: #747474

.header
  height: 70px
  background-color: #006bb4

.bottom
  display: flex
  height: 100px
  background-color: #25323a

.headercontainer
  display: flex
  width: 80%
  height: 100%
  margin-right: auto
  margin-left: auto
  justify-content: space-between

.logo
  height: 70px
  max-width: 200px
  padding-top: 22px
  color: #fff
  font-size: 50px
  font-weight: 700
  text-align: center
  text-transform: none

.button
  padding-top: 25px
  padding-bottom: 25px
  float: right
  background-color: transparent

.side
  display: flex
  width: 300px
  padding-right: 10px
  padding-left: 10px
  background-color: #fff

.codetextwrapper
  flex: 1

.resizewrapper
  display: flex
  width: 0px
  flex-direction: column
  justify-content: center

.resizediv
  display: block
  width: 50px
  height: 50px
  margin-left: -25px

.image
  display: inline-block
  width: 50px
  height: 50px

.paragraph
  margin-bottom: 0px
  flex: 1

.progresswrapper
  display: flex
  flex-direction: column
  justify-content: center
  flex: 1

.playbtn
  display: flex
  width: 100px
  flex-direction: row
  justify-content: center
  align-items: center

.progressout
  height: 10px
  margin-right: 17px
  margin-left: 29px
  border-radius: 100px
  background-color: #d1d1d1

.progressin
  width: 10%
  height: 100%
  border-radius: 100px
  background-color: #0062a7

.cir
  display: flex
  width: 500px
  height: 500px
  justify-content: center
  align-items: center
  border-radius: 500px
  background-color: #276792
  color: #fff

.text-block-2
  display: block
  flex-direction: row
  justify-content: center
  align-items: center
  font-size: 200px

.heading-3
  margin-top: 0px
  margin-bottom: 0px
  padding-top: 100px
  padding-bottom: 100px
  font-size: 200px

.paragraph-2
  font-size: 50px
  font-weight: 700

.div-block-13
  position: fixed
  left: 0px
  top: 0px
  right: 0px
  bottom: 0px
  display: flex
  justify-content: center
  align-items: center
  background-color: #292929

.div-block-14
  position: relative
  left: -256px
  z-index: -10
  width: 500px
  height: 500px
  background-color: #adadad

.asdf
  display: flex
  width: 500px
  height: 500px
  justify-content: center
  align-items: center
  border-radius: 500px
  background-color: #afafaf
  color: #fff

.header-div
  display: flex
  width: 100%
  height: 60px
  justify-content: space-between
  background-color: #1e75b0
  box-shadow: 2px 2px 3px 0 #bbb

.logo-wrapper
  display: flex
  width: 20%
  justify-content: flex-end

.logo-div
  display: flex
  align-items: center

.text-block-3
  font-family: 'Open Sans', sans-serif
  color: #fff
  font-size: 30px
  font-weight: 700

.fullsize-wrapper
  position: fixed
  left: 0px
  top: 0px
  right: 0px
  bottom: 0px
  z-index: -100
  overflow: visible

.content-wrapper
  display: flex
  justify-content: center

.content
  width: 65%
  margin-top: 27px
  flex: 0 auto
  background-color: #fff

.video
  box-shadow: 0 0 3px 1px #69b7ff

.body-2
  min-height: 800px

.text-wrapper
  padding: 15px
  border-bottom-left-radius: 3px
  border-bottom-right-radius: 3px
  box-shadow: 1px 1px 5px 0 #c5c5c5

.texts
  font-family: 'Open Sans', sans-serif
  color: #333
  font-size: 18px
  font-weight: 600
  min-height: 23px
  .gray
    color: #ddd

.progress-out
  display: flex
  height: 5px
  background-color: #cacaca

.progress-in
  width: 30%
  background-color: #0081ff

html.w-mod-js *[data-ix="load-hero"]
  opacity: 0
  transform: translate(-100px, 0px)

@media (max-width: 991px)
  .menu-button
    right: -1px
  .cir
    width: 300px
    height: 300px
  .heading-3
    padding-top: 68px
    padding-bottom: 68px
    font-size: 150px
    text-align: center
  .paragraph-2
    font-size: 30px
  .div-block-14
    left: -155px
    right: -17px
    width: 300px
    height: 300px
  .asdf
    width: 300px
    height: 300px
  .content
    width: 100%
    margin-top: 0px

@media (max-width: 767px)
  .logo-wrapper
    width: 25%

@media (max-width: 479px)
  .logo-wrapper
    width: 40%
    margin-left: 40px
    justify-content: flex-start

</style>
