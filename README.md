# duplex-audio

Full-duplex audio front-end for Python: speaker/microphone I/O with acoustic
echo cancellation (AEC) and noise suppression (NS).

duplex-audio は、スピーカーで音を再生しながらマイクで集音する Python library です。
マイク入力を AEC、NS の順に処理し、再生音のエコーと周囲の雑音を除いた
クリーン音声を NumPy 配列で呼び出し側へ渡します。AEC の参照信号には、
duplex-audio が実際に再生した音を使います。

- 呼び出し側は、復号済みのモノラル PCM と sample rate を渡して音を再生できます。
- 呼び出し側は、AEC と NS の backend を選び、それぞれを有効・無効にできます。
- 呼び出し側は、再生 generation ごとに再生の保持停止、再開、無音化を要求できます。

## Install

Python 3.12 以上の環境へ、PyPI からインストールします。

```bash
pip install duplex-audio
```

LocalVQE を AEC backend として使う場合は、次の extra を指定します。

```bash
pip install "duplex-audio[localvqe]"
```

`localvqe` extra は Linux x86_64 でのみ `pylocalvqe` をインストールします。
`pylocalvqe` の wheel は native library を同梱しますが、model weight は別途取得します。

オーディオ I/O は、`pasimple` を使って PipeWire / PulseAudio の Pulse 互換 Unix socket
へ接続します。実行環境には libpulse のランタイムと、接続可能な Pulse 互換の
録音・再生環境が必要です。`pulse_server`、`source`、`sink` を `DuplexEngine` に渡すと、
接続先とマイク入力・再生出力を指定できます。

## Models

LocalVQE の model は、`pylocalvqe` の downloader で取得します。
downloader は固定した model repository revision から weight を取得し、
SHA-256 を検証します。戻り値は取得した model file の path です。

```python
from pylocalvqe.assets import download_model

model = download_model("v1.4-aec-200k", "models")
```

呼び出し側は、この path を `LocalVqeSettings.model_path` へ渡します。
`v1.4-aec-200k` は AEC-only variant なので、独立した RNNoise の NS と組み合わせられます。
AEC と NS をまとめて行う `v1.3` を使う場合は、独立 NS の二重掛けを避けるために
`ns_backend="none"` を指定します。

LocalVQE を使わない場合は、model の取得は不要です。
`DuplexEngine` の既定の AEC backend は、参考実装の NLMS です。

## Python usage

次の例は Linux x86_64 で `localvqe` extra を導入した環境を使います。
1 秒のテスト音を再生しながら 3 秒間集音し、LocalVQE による AEC と
RNNoise による NS を通した音を受け取ります。AEC と NS は既定では無効なので、
例では両方を明示的に有効にしています。

```python
import time

import numpy as np
from pylocalvqe.assets import download_model

from duplex_audio import DuplexEngine, LocalVqeSettings

model = download_model("v1.4-aec-200k", "models")
recorded = []

with DuplexEngine(
    aec_backend="localvqe",
    localvqe_settings=LocalVqeSettings(
        model_path=model,
        variant="v1.4-aec-200k",
        threads=4,
    ),
    aec_enabled=True,
    ns_backend="rnnoise",
    ns_enabled=True,
) as engine:
    sample_rate = 24_000
    t = np.arange(sample_rate) / sample_rate
    reference = (0.05 * np.sin(2 * np.pi * 440 * t)).astype(np.float32)
    generation = engine.create_playback_generation()
    engine.enqueue_playback(generation, reference, sample_rate=sample_rate)
    engine.finish_playback(generation)

    deadline = time.monotonic() + 3.0
    while time.monotonic() < deadline:
        clean = engine.read_clean(timeout=0.1)
        if clean is not None:
            recorded.append(clean)

clean = np.concatenate(recorded) if recorded else np.empty(0, dtype=np.float32)
print(clean.shape, clean.dtype)
```

`enqueue_playback()` は、1 次元の浮動小数点 PCM とその sample rate を受け取ります。
再生 sample rate の変換は library が行います。`finish_playback()` は generation の
末尾を確定します。`read_clean()` はモノラル 16 kHz の `float32` 配列を
block ごとに返し、指定した待ち時間内に音が届かなければ `None` を返します。
context manager は録音・再生を開始し、終了時に停止します。

呼び出し側は、`pause_playback()` と `resume_playback()` で再生キューを保持したまま
再生を止めて再開し、`quiesce_playback()` で generation の無音化を要求できます。
継続的に再生する場合は、`read_playback_event()` から再生 event を読み出し、
terminal になった generation を `release_playback_generation()` で解放します。
`stream_health` と `get_pipeline_stats()` は stream の稼働状態と処理の統計を返します。

## 複数のマイク

`capture_channels` で録音する ch 数を指定できます。再生は 1ch のままです。
既定の `microphone_mix="selected"` は、`selected_channel` (0 始まり、既定 0) の音を
1 本だけ選びます。`"sum"` は全 ch の和を返します。各 ch は、それぞれ 16 kHz に
変換されてから混ぜられます。

スピーカーを抑える固定の重みを使うには、稼働中の stream で calibration を測り、
全 ch の結果を明示して適用します。測定中は掃引音がスピーカーから再生されます。

```python
with DuplexEngine(capture_channels=4, selected_channel=0) as engine:
    result = engine.measure_calibration()
    if result.status != "success":
        raise RuntimeError(result.error or result.status)
    engine.apply_calibration(
        result.profile.bulk_delay,
        expected_stream_generation=result.stream_generation,
        channel_responses=result.channel_responses,
    )
    engine.set_microphone_mix("beamform")
    frame = engine.read_clean_frame(timeout=0.1)
    if frame is not None:
        print(frame.sample_offset, frame.microphone_mix,
              frame.speaker_match, frame.speaker_match_status)
        clean = frame.samples
```

`set_microphone_mix()` は処理を止めずに三つの出し方を切り替えます。切り替えは次の
処理 block から効き、AEC の状態をリセットします。`"beamform"` は複数 ch の結果を
適用した後に選べます。適用前の指定は `RuntimeError` になります。固定 FIR は出力に
5 ms の遅れを加え、処理した block をその都度渡します。

`CleanFrame` は、音、通し番号、sample の位置、その音を作った出し方、スピーカーとの
一致の値を持ちます。一致の値は直近 512 sample (16 kHz で 32 ms) の録音の観測であり、
AEC / NS の前の音から、出し方によらず計算されます。`speaker_match_status` は `ok`、
部屋の雑音の基準に対して小さい `quiet`、未適用の `uncalibrated`、1ch の構成の
`single_channel` を区別します。`ok` 以外では `speaker_match` は `None` です。

`read_clean()` と `read_clean_frame()` は同じ queue から一つずつ取り出します。
queue に残る古い block には、その block を作った時の出し方と値が付いています。
`read_capture_zero_run()` は全 ch が 0 ちょうどの長さを返し、
`read_capture_channel_zero_runs()` は ch の順に同じ観測を返します。

## License

duplex-audio の license は MIT です。
全文は [`LICENSE`](https://github.com/sync-dev-org/duplex-audio/blob/main/LICENSE) を参照してください。
