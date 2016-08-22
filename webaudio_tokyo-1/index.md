title: WebAudioでなんちゃってWebRTCする
controls: false
--

# <a>WebAudio</a>で<br>なんちゃってWebRTC

## &nbsp;
## 2016/09/06 WebAudio.tokyo \#1

--

### はじめまして

- Yuji Sugiura / [@leader22](https://twitter.com/leader22)
- フロントエンド・エンジニア at PixelGrid Inc.
- 最近の仕事はパフォーマンスチューニング📈
- 趣味ではReactNative x Swiftで音楽アプリ

![leader22](./img/doseisan.jpg)

--

### WebAudioとわたし

- 前職（渋谷のDなんとか社）で、ソシャゲのBGM・効果音に使えないか検証したり
- [CodeGrid](http://www.codegrid.net/)に[記事](https://app.codegrid.net/series/2015-sound)書いたり
- いわゆる[音楽プレーヤー](https://github.com/leader22/mmss)作ってみたり

素人ではないが、玄人でもないレベル<br>
まさかWebAudioの勉強会が開催される日がくるとは・・💪

--

# <a>WebAudio</a>で<br>なんちゃってWebRTC

--

# （iOSだとWebRTCが使えないので<br>WebSocketと）<a>WebAudio</a>で<br>なんちゃってWebRTC<br>（という試みを半年前にやった話）

--

### WebRTCとは

ざっくり言うと、

- ブラウザでマイクやカメラのストリームを取得し
- ネットワークを越えてリアルタイム配信できるAPI
- "基本的"には通信はP2P
  - ただしP2Pだとスケールしない💨

SkypeがWebの技術でできるようなもの。<br>
ただ、このモバイル全盛期でもiOSのWebで使えない！！

--

### そこで

せめて、PC側での会話を聞くだけでも・・。

というわけで、

- WebRTCで音を拾うとこまではPCでやって
- それをWebSocketで送りつけて
- iOSでもWebAudioで再生できないか

・・・できた(∩´∀｀)∩

--

# デモ
##（WiFiがあれば）

![](https://raw.githubusercontent.com/leader22/audio-streaming-over-websocket/master/demo-img.jpg)

--

### やってること: Publisher

- getUserMedia()
- MediaStreamSourceNode
  - BiquadFilterNode
  - ScriptProcessorNode: 1ch
    - postMessage('audio', PCM&lt;ArrayBuffer&gt;)
- WebWorker
  - socket.emit('audio', payload)

--

### やってること: Subscriber

- WebWorker
  - socket.on('audio', payload)
- ctx.createBuffer.set(new Float32Array(payload))
- AudioBufferSourceNode
- source.start()

--

# WebAudioな部分の解説

--

### マイクの音をAudioNodeにする
```js
var crx = new AudioContext();

navigator.mediaDevices
  // 今回は音さえあればOK
  .getUserMedia({ audio: true })
  .then((stream) => {
    var micSrc = ctx.createMediaStreamSource(stream);

    // micSrc: AudioBufferSourceNode
    // コレをつないでいく
  })
```

localhostかhttps環境でお試しください！

--

### 雑音をフィルタする

```js
var filter = ctx.createBiquadFilter();
filter.type = 'bandpass';
// アナログ電話は300Hz ~ 3.4kHz / ひかり電話は100Hz ~ 7kHz
filter.frequency.value = (100 + 7000) / 2;
// 固定ならだいたい聴き良いのがこれくらい・・？
filter.Q.value = 0.25;

micSrc.connect(filter);
```

リアルタイムに周波数帯を検知してフィルタの領域も変動したかったけど面倒なのでやめた・・。

--

### 音源（INPUT）の様子をみる

```js
var analyser = ctx.createAnalyser();
analyser.smoothingTimeConstant = 0.4;
analyser.fftSize = BUFFER_SIZE;

var fbc = analyser.frequencyBinCount;

var freqs = new Uint8Array(fbc);
analyser.getByteFrequencyData(freqs);

// freqsの中身を、canvasにグラフとして描画
```

入力音によって波打つアレ。<br>
これがあるとそれっぽい感じに！

--

### 音を飛ばす
```js
// それなりに重そうなのでとりあえず1ch
var processor = ctx.createScriptProcessor(BUFFER_SIZE, 1, 1);
processor.onaudioprocess = _onAudioProcess;

function _onAudioProcess(ev) {
  var inputBuffer  = ev.inputBuffer;
  var outputBuffer = ev.outputBuffer;
  // 1chなのでインデックスは0
  var inputData  = inputBuffer.getChannelData(0);
  var outputData = outputBuffer.getChannelData(0);

  // Bypassしつつ飛ばさないとダメ
  outputData.set(inputData);
  // これをそのままWebSocketで流す
  worker.postMessage({
    type: 'AUDIO',
    data: outputData.buffer
  });
}
```

音質にこだわるなら、ここで巨大なバッファに2chつめこめばいいはず。

--

### 音を受け取る
```js
var ctx = new AudioContext();

// WebSocketで受けたバッファを処理する
function _handleAudioBuffer(buf) {
  var f32Audio = new Float32Array(buf);
  // 送られてきたのとあわせる
  var audioBuffer = ctx.createBuffer(1, BUFFER_SIZE, ctx.sampleRate);
  audioBuffer.getChannelData(0).set(f32Audio);

  // 後はいつも通り
  var source = ctx.createBufferSource();
  source.buffer = audioBuffer;
  source.connect(audio.gain);
}
```

後は鳴らすだけ！

--

### いいタイミングで再生する
```js
var _startTime = 0;
var currentTime = ctx.currentTime;

// つまってるので未来に再生
if (currentTime < _startTime) {
  source.start(_startTime);
  _startTime += audioBuffer.duration;
}
// すぐ再生
else {
  source.start(_startTime);
  _startTime = currentTime + audioBuffer.duration;
}
```

これが地味に重要で、こうしないとプツプツとぎれとぎれになります・・。

--

# さいごに実用性

--

### 気になる実用性

- リアルタイム性
  - ネットワークによってはじわじわ遅延する
  - つなぎなおせばLTEでも全然OK
- 音質
  - 今回の実装（1ch / 1024）だと音楽は辛い
  - ニュースや会話は問題ない

やってみた本人が一番驚いてるけど、意外にイケる😊

--

# Thank you!

<style>
:root {
  --bg-color: #fffdf8;
  --bar-color: #274a78;
  --em-color: #54b6d3;
}
\#slide-5 {
  font-size: 70%;
}
</style>
<link rel="stylesheet" href="../public/base.css">
<link rel="stylesheet" href="../public/timer.css">
<script src="../public/timer.js"></script>
<script src="../public/mobile-controls.js"></script>
