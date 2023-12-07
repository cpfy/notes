---
title: whisper使用
created: 2023-09-14 15:15:27, 星期四
modified: 2023-10-10 08:35:50, 星期二
---


openai 的语音模型，2022-09 开源
Github：[GitHub - openai/whisper: Robust Speech Recognition via Large-Scale Weak Supervision](https://github.com/openai/whisper)
Arxiv：[[2212.04356] Robust Speech Recognition via Large-Scale Weak Supervision](https://arxiv.org/abs/2212.04356)

### 安装

pip install -U openai-whisper

### 特点

- 西班牙语、意大利语、英文等少数语言很强
- 据说：whisper 的 WER 和 SER 指标比国内的科大讯飞 asr 和阿里 asr 要差很多

### model

#### 版本

五种可选：tiny、base、small、medium、large

前面就是很小、速度很快，但精度差一些。
据说 small 是句子分割，medium 能做到语义、停顿分割，large 应该效果更好

==**0917 更新**==：亲测 large-v2 好像会优先把句子断成短句，连贯性效果并不好（本 case 不如 medium）；medium.en 和 medium 效果差不多，仔细看了一些细节也比不出优劣

#### 手动下载

>自动太慢了，应该是没配 TUN mode 因此没有 proxy，几十 KB/s

whisper 运行时会自动下载，可选择手动下载替换到 `~/.cache/whisper` 文件夹中

各版本模型的下载链接在（注意不用旧版本，下载链接好像变过）：[https://github.com/openai/whisper/blob/main/whisper/\_\_init\_\_.py](https://github.com/openai/whisper/blob/main/whisper/__init__.py)

### 用法

#### 常用

在保存文件结果的文件夹运行：

```shell
whisper --language English --task translate --model medium.en ~/d4t/data/audios/0913/20230913_141253_resample.wav 
```

中文：

```shell
whisper --language Chinese --initial_prompt " 以下是普通话的句子。" --model large-v2 ~/data/audios/1009-ch/20231009_180142.m4a
```

#### 基本指令

直接使用 whisper 指令识别音频和视频文件为文本

```
whisper video.mp4
```

默认当前目录下生成 5 个文件，扩展名分别是：.json、.srt、.tsv、.txt、.vtt。包括普通文本、电影字幕、开发用 json 格式。

#### 检测语言

使用 `--language xxx` 指定

否则会自动使用前 xx 秒检测，还会不准

#### 简繁问题

讨论帖：[Simplified Chinese rather than traditional? · openai/whisper · Discussion #277 · GitHub](https://github.com/openai/whisper/discussions/277)

使用 `--language Chinese` 时，默认输出指定为繁体

转换成简体需要添加引导参数 `--initial_prompt "以下是普通话的句子。"`

```
$ whisper --language Chinese --model large audio.wav
[00:00.000 --> 00:08.000] 如果他们使用航空的方式运输货物在某些航线上可能要花几天的时间才能卸货和通关

$ whisper --language Chinese --model large audio.wav --initial_prompt "以下是普通話的句子。"  # traditional
[00:00.000 --> 00:08.000] 如果他們使用航空的方式運輸貨物,在某些航線上可能要花幾天的時間才能卸貨和通關。

$ whisper --language Chinese --model large audio.wav --initial_prompt "以下是普通话的句子。"  # simplified
[00:00.000 --> 00:08.000] 如果他们使用航空的方式运输货物,在某些航线上可能要花几天的时间才能卸货和通关。
```

### 其它

关于文件打包：发现一种简单的方法，选中几个文件，右键 `Compress`