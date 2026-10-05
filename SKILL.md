---
name: video-to-chinese-transcript
description: 把本地视频/音频转写成中文文字稿（faster-whisper），并处理「HuggingFace 下载被墙/超时」的环境。触发：视频转文字、语音转写、ASR、视频生成学案/逐句分析、课程视频提取知识点。
---

# 视频/音频 → 中文转写稿（faster-whisper）

在 Windows + 中国网络环境下，把视频转成中文文字稿的可复用流程。核心坑：HuggingFace Hub（`huggingface_hub.snapshot_download`）在沙箱/墙内环境会 **ConnectTimeout**，必须用 modelscope 镜像手动下载模型权重。

## 环境判定
- ffmpeg 是否可用：`where ffmpeg` / `ffmpeg -version`
- faster-whisper 是否已装：`python -c "import importlib.util as u; print(u.find_spec('faster_whisper'))"`
- 若 `faster_whisper`、`ctranslate2`、`onnxruntime`、`av` 已装但 `soundfile` 缺：`pip install soundfile`

## 步骤 1：抽音频
```bash
ffmpeg -y -i "视频.mp4" -ar 16000 -ac 1 -c:a pcm_s16le audio.wav
```

## 步骤 2：下载 faster-whisper 模型（绕过 HF 被墙）
HF 直连超时，改从 modelscope 拉取。以 `medium` 为例（中文够用；要更准用 `large-v3`，但更大更慢）：
```bash
mkdir -p models/medium && cd models/medium
BASE="https://modelscope.cn/models/Systran/faster-whisper-medium/resolve/master"
for f in config.json configuration.json tokenizer.json vocabulary.txt; do
  curl -sL -o "$f" "$BASE/$f"
done
# model.bin 较大（~1.5GB），放后台 + 关沙箱跑，否则网络被拦
curl -sL -o model.bin "$BASE/model.bin"
```
> 沙箱会阻断外网下载：下载 model.bin 时务必 `dangerouslyDisableSandbox: true`。
> modelscope CDN 返回 206 即代表可达；hf-mirror.com 在本环境实际超时，不可用。
> faster-whisper 只需这 4 个文件：`config.json / model.bin / tokenizer.json / vocabulary.txt`。

## 步骤 3：转写（加载本地模型路径）
```python
from faster_whisper import WhisperModel
model = WhisperModel("models/medium", device="cpu", compute_type="int8")
segs, info = model.transcribe("audio.wav", language="zh", beam_size=5,
                              vad_filter=True,
                              vad_parameters=dict(min_silence_duration_ms=500))
for s in segs:
    print(round(s.start,1), round(s.end,1), s.text.strip())
```
- `device="cpu", compute_type="int8"`：纯 CPU 也能跑，13 分钟音频约 17 分钟。
- 长音频放后台（`run_in_background`）避免超时。
- 导出 `transcript.json`（segments + 时间戳）和 `transcript.txt`。

## 步骤 4：切句 + 类型标注
- 按 `。！？!?；;…\n` 切句，时间戳在段内线性插值。
- 启发式分类（提问互动/举例/方法指导/强调结论/诊断评估/过渡/陈述），用于「逐句分析」表格。

## 步骤 5：ASR 纠错（中文同音字必错）
medium 模型对教学术语/同音字误识严重，务必做语义纠错后再交付。高频错误映射（长匹配优先）：
```
出生高→初升高  弯道抽折→弯道超车  背单子→背单词
五维整段→五维诊断  长男句/长男剧→长难句  位语→谓语  非位语→非谓语
现实→限时（现实检测→限时检测、组合现实训练→组合限时训练）
证券率→正确率  作题/座题→做题  声词→生词  熟词生意/俗词声音→熟词生义
防写→仿写  方文→范文  试剑→试卷  剧法→句法  单子→单词
必抓100奖→必刷100篇  搞点3500词→搞定3500词  机器主页→七七主页
心底中→新题中（心底中证据→新题中举一反三）  准够二→准高二  不漏洞→补漏洞
扩塑范围→扩大范围  文章大力→文章大意  两个语调→两个选项  七卷五→七选五
笼哄→笼统  并明→病名
```
> 纠错词典要按「长串优先」顺序替换，否则短串会先误伤。

## 产出建议
- `xuean.html`：结构化学案（概览 + 方法论 + 诊断表 + 行动计划 + 每日时间表 + 逐句精讲 + 纠错说明）。
- `transcript_corrected.txt`：带时间轴的修正版纯文本稿。
- 逐句表用颜色区分类型，便于快速定位「方法/诊断/结论」句。
