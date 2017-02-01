# vst.js
## What Is It?
**vst.js** is a NodeJS module that can be used to launch VST3 plugins (including GUI interface) in a separate process. Audio data is sent to the child process via a [grpc](http://www.grpc.io/) service call and the processed audio is sent back.
 

## WARNING:
This library is in an extremely experimental state. Large portions of functionally have yet to be implemented and it is currently only buildable for OSX devices, although since the plugin host is built using the [JUCE audio framework](http://juce.com), multiplatform builds shouldn't be too difficult.

I am currently focused on building out functionality and haven't tested the build on any device other than my MacBook Pro so it would be a small miracle if you were able to run it without any trouble.

Additionally, I'm learning C++ by developing this project. If you have any criticisms on my approach or code quality I'd love to hear them.

## Requirements
### Steinberg VST3 SDK
Due to licensing concerns I am currently not bundling the VST3 SDK along with this project. You will need to download the SDK from [Steinbergs's Website](http://www.steinberg.net/en/company/developers.html) and place it at `~/SDKs/VST3` (This may be automated or at least pulled from an environment variable in the future).

## Usage Examples
The example below will play back an audio file via [node-web-audio-api](https://github.com/sebpiq/node-web-audio-api), and manipulate the audio via a VST3 plugin
```javascript
const PluginHost = require('vst-js').PluginHost

const AudioContext = require('web-audio-api').AudioContext
const AudioBuffer = require('web-audio-api').AudioBuffer
const Speaker = require('speaker')
const fs = require('fs')
const path = require('path')

const bufferSize = 512
const numChannels = 2
const pluginPath = '/Library/Audio/Plug-Ins/VST3/PrimeEQ.vst3'
const hostAddress = '0.0.0.0:50051'

// launch plugin host process
const pluginHost = new PluginHost(pluginPath, hostAddress)
pluginHost.launchProcess()

// setup webaudio stuff
const audioContext = new AudioContext()
const sourceNode = audioContext.createBufferSource()
const scriptNode = audioContext.createScriptProcessor(bufferSize, numChannels, numChannels)

audioContext.outStream = new Speaker({
  channels: audioContext.format.numberOfChannels,
  bitDepth: audioContext.format.bitDepth,
  sampleRate: audioContext.sampleRate,
})

sourceNode.connect(scriptNode)
scriptNode.connect(audioContext.destination)

scriptNode.onaudioprocess = function onaudioprocess(audioProcessingEvent) {
  const inputBuffer = audioProcessingEvent.inputBuffer
  const channels = [...Array(numChannels).keys()]
    .map(i => audioProcessingEvent.inputBuffer.getChannelData(i))

  // process audio block via pluginHost
  const output = pluginHost.processAudioBlock(channels)

  const outputBuffer = AudioBuffer.fromArray(output, inputBuffer.sampleRate)
  audioProcessingEvent.outputBuffer = outputBuffer // eslint-disable-line no-param-reassign
}

fs.readFile(path.resolve(__dirname, './test.wav'), (err, fileBuf) => {
  console.log('reading file..')
  if (err) throw err
  audioContext.decodeAudioData(fileBuf, (audioBuffer) => {
    sourceNode.buffer = audioBuffer
    sourceNode.start(0)
  }, (e) => { throw e })
})

pluginHost.on('info', (data) => {
  console.log(`stdout: ${data}`)
})

pluginHost.on('error', (data) => {
  console.log(`stderr: ${data}`)
})

pluginHost.on('close', (code) => {
  console.log(`child process exited with code ${code}`)
})
```
