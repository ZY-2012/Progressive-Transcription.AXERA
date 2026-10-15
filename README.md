# Progressive-Transcription.AXERA

AX650N 上的实时**渐进式**会议转写 demo：一边说话，一边以固定节拍先出流式字幕并不断修订，随后用非流式识别 + 在线说话人分离把每一句对账成**说话人 + 时间 + 文字**的权威稿。音频链路（采集/APM/NPU 降噪/VAD/ASR/说话人分离/归档）全部为板端原生 C++，另带一个网页实时演示页，可直观看到真实端到端时延。

- [x] 流式字幕（先出字、后修订，节拍可配）
- [x] 在线说话人分离（会话增量模式，声纹表持久化）
- [x] finals 权威稿与流式文字同屏对账（每条 final 唯一归属，不重复）
- [x] WAV 文件模拟麦克风（file-realtime）或 ALSA 实时采集
- [x] MP3/WAV 归档 + 带时间戳转写文本

## 1. 支持平台

- AX650N（VNPU `BIG_LITTLE` 2+1）

## 2. 功能

| 模块 | 说明 |
|------|------|
| 音频前端 | ALSA 采集（或 16k WAV 文件按实时节拍喂入）+ WebRTC APM（回声/增益/高通） |
| NPU 降噪 | FastEnhance（NPU1），`axmodel` 48k 降噪 |
| VAD | TEN-VAD（NPU1），输出语音段；段长/停顿切分/尾静音均可配 |
| ASR | SenseVoice（NPU2）：流式切片渐进出字 + 段级非流式权威稿 |
| 说话人分离 | pyannote community-1（NPU2），在线会话模式 + 说话人表（speakers.json）跨会话匹配 |
| 输出 | `transcript.txt`（说话人@时间 文字）、`transcript_streaming.txt`（流式）、MP3/WAV、`events.jsonl`（演示页事件） |
| 演示页 | 流式字幕→finals 对账、说话人标签、与音频同步播放，真实时延可见 |

## 3. 目录结构

```
Progressive-Transcription.AXERA/
|-- config/
|   |-- meeting-transcribe.yaml   # 示例配置（60s 双人会议 WAV）
|   `-- default-config.yaml      # 默认配置（ALSA 麦克风输入）
|-- driver/                       # 运行库与引擎源码
|   |-- apm/                     # WebRTC APM 预编译库
|   |-- ns/                       # FastEnhance 降噪（源码）
|   |-- vad/                      # TEN-VAD 预编译库
|   |-- asr/sensevoice/           # SenseVoice AX650 适配库
|   |-- alsa_briage/              # run-alsa-apm 音频前端（源码）
|   `-- pyannote/                 # 说话人分离引擎 + diar-worker（源码）
|-- framework/
|   |-- audio_pipeline.cpp       # 主管线（单文件）
|   `-- configui/                 # yaml 配置解析
|-- samples/                      # 2P_meeting_60s.wav + reference.rttm（参考标注）
|-- scripts/                      # build.sh / download_toolchains.sh / run-target.sh
|-- tools/live-demo/              # 网页实时演示（python3 标准库，板端零依赖安装）
`-- LICENSE
```

## 4. 相关模型及下载地址

| 模块 | 模型 | 下载 |
|------|------|------|
| ASR | SenseVoice | [AXERA-TECH/models-for-asr-chain](https://huggingface.co/AXERA-TECH/models-for-asr-chain) → `asr/sensevoice` |
| VAD | TEN-VAD | 同上 → `vad/ten-vad` |
| 降噪 | FastEnhance | 同上 → `ns/fastenhance` |
| 说话人分离 | pyannote community-1（NPU2 版） | [HY-2012/pyannote-speaker-diarization.axera](https://huggingface.co/HY-2012/pyannote-speaker-diarization.axera) → `models_npu2/` |

```bash
pip install -U "huggingface_hub[cli]"

hf download AXERA-TECH/models-for-asr-chain \
    --include "asr/*" "vad/*" "ns/*" \
    --local-dir models-for-asr-chain

hf download HY-2012/pyannote-speaker-diarization.axera \
    --include "models_npu2/*" \
    --local-dir diar-tmp
mkdir -p models-for-asr-chain/pyannote
cp -r diar-tmp/models_npu2/. models-for-asr-chain/pyannote/
rm -rf diar-tmp
```

完成后应存在 `models-for-asr-chain/asr/sensevoice/sensevoice.axmodel`、
`models-for-asr-chain/vad/ten-vad/ten-vad.axmodel`、
`models-for-asr-chain/ns/fastenhance/fastenhance_48k.axmodel` 与
`models-for-asr-chain/pyannote/embedding_convs.axmodel` 等。

## 5. 板端部署（C++）

### 5.1 工具链、BSP 与 ALSA 头文件

```bash
bash scripts/download_toolchains.sh
```

脚本下载三样东西到 `toolchains/`：Arm GNU 交叉编译器 9.2-2019.12、
AX650N BSP SDK（含 `msp/out` 的 ax_engine/ax_sys 头文件与库）、
ALSA 头文件（编译 `run-alsa-apm` 需要；链接库用板端自带的
`/usr/lib/aarch64-linux-gnu/libasound.so.2`，见 5.3）。

### 5.2 交叉编译

```bash
export CXX=toolchains/gcc-arm-9.2-2019.12-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-g++
export AX650_SDK_ROOT=toolchains/ax650n_bsp_sdk/msp/out
export ALSA_INCLUDE_DIR=toolchains/alsa
export ALSA_LIBRARY=<板端 libasound.so.2 拷出来的路径>   # 也可省略，见下
bash scripts/build.sh
```

产物在 `build/`：`audio-pipeline`（主管线）、`run-alsa-apm`（音频前端）、
`diar-worker`（说话人分离常驻 worker）、`libfastenhance.so`。

> `ALSA_LIBRARY` 需要是 ARM64 的 `libasound.so`。最简单的做法：从板端拷
> `/usr/lib/aarch64-linux-gnu/libasound.so.2.0.0` 到开发机，建一个
> `libasound.so` 软链后把路径传给 `ALSA_LIBRARY`。若不设置，`build.sh`
> 会跳过 `run-alsa-apm`（此时可复用板端已有的同名可执行文件）。

### 5.3 板端运行

把整个工程目录（含 `build/`、`models-for-asr-chain/`）拷到板端，在**工程根目录**执行：

```bash
# WAV 文件模拟麦克风（file-realtime，与真实采集同一时序）
CONFIG=config/meeting-transcribe.yaml ./scripts/run-target.sh

# ALSA 实时采集（按板子实际声卡改 config/default-config.yaml 的 audio.capture.device）
CONFIG=config/default-config.yaml ./scripts/run-target.sh
```

或启动网页实时演示（python3 标准库，无需 pip 安装）：

```bash
python3 tools/live-demo/server.py 8090
# 浏览器打开 http://<板端IP>:8090，点击 Start
```

关键流式参数（`config/*.yaml` 的 `vad:` 段）：`stream_tick_seconds`（流式节拍，
0 为关闭）、`preview_seconds`（前缀预览）、`max_segment_seconds`、
`trailing_silence_seconds`；说话人参数见 `diarization:` 段。

## 6. 结果

AX650N 实测（`samples/2P_meeting_60s.wav`，双人会议，`stream_tick_seconds: 0.8`）：

| 指标 | 数值 |
|------|------|
| 首个流式文字（stext） | ~1.0 s |
| 流式节拍 | 0.8 s/次（可配 0.5–1.2 s） |
| 流式→finals 修订滞后 | ~0.7 s |
| 首条权威稿（final） | ~11 s（VAD 段落定后） |
| NPU 占用（降噪 / VAD） | ~5% / ~2.5% |
| finals 与非流式基线 | 15/15 逐条一致（说话人 + 时间 + 文字） |

<!-- 录制动图后取消注释并替换路径
<p align="center">
  <img src="docs/demo.gif" alt="realtime demo">
</p>
-->

## 7. 参考

- [SenseVoice](https://github.com/FunAudioLLM/SenseVoice)
- [pyannote/audio](https://github.com/pyannote/pyannote-audio)、[diart](https://github.com/serca-umh/diart)
- [TEN-VAD](https://github.com/TEN-framework/ten-vad)
- [pyannote-speaker-diarization.axera](https://github.com/ZY-2012/pyannote-speaker-diarization.axera)（说话人分离板端引擎与本工程 `driver/pyannote` 的来源）
- [AXERA-TECH/asr-llm-tts-chain](https://github.com/AXERA-TECH/asr-llm-tts-chain)（本工程音频链路的裁剪来源）

## 8. 技术讨论

- GitHub issues
- QQ 群: 139953715
