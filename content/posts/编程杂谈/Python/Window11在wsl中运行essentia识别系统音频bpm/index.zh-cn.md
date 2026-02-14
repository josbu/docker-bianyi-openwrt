---
date: 2026-02-14T17:03:12+08:00
lastmod: 2026-02-14T17:03:12+08:00
categories:
  - 编程杂谈
  - Python
title: Window11在wsl中运行essentia识别系统音频bpm
draft: "false"
tags:
  - wsl
  - anaconda
  - python
  - 音频识别
series: []
---
## 安装ubuntu24.04.1-lts
打开微软商店直接下载安装即可。

## 安装Anaconda
参考： https://gist.github.com/kauffmanes/5e74916617f9993bc3479f401dfec7da

在[这里](https://repo.anaconda.com/archive/)下载[Anaconda3-2025.12-2-Linux-x86_64.sh](https://repo.anaconda.com/archive/Anaconda3-2025.12-2-Linux-x86_64.sh)

wsl安装完成后，会自动出现在windows资源管理器左侧目录的Linux栏，把下载好的.sh文件放到home/user下

接着进入wsl终端，执行 
```
bash Anaconda3-2025.12-2-Linux-x86_64.sh
```



## 创建python环境

```shell
conda create -n essentia_env python=3.11
```

## 安装essentia
```bash
pip install essentia
```


## WSL连接windows音频
默认情况下wsl无法识别windows音频，需要建立桥梁。
https://chat.deepseek.com/share/9v02xm0s3yx5bwb9ql

直接采取方法一：利用 WSLg 自动音频重定向
WSLg（Windows Subsystem for Linux GUI）是微软官方为 WSL 提供的图形/音频支持。如果你已经安装或准备安装 WSLg，音频会自动通过 PulseAudio 桥接到 Windows。

### 1. 确认 WSLg 状态

在 PowerShell 中运行：

```bash
wsl --version
```


若输出包含 **WSLg** 版本号（例如 `1.0.0`），则说明已安装。若未安装，可执行 `wsl --update` 升级到最新版本，WSLg 会自动集成。

### 2. 在 WSL2 Ubuntu 中安装必要组件

即使 WSLg 已运行，仍需在 Ubuntu 内安装 PulseAudio 客户端和 PortAudio：


```bash
sudo apt update
sudo apt install pulseaudio libportaudio2

```

### 3. 设置环境变量（关键）

WSLg 启动时会在 Windows 后台运行 PulseAudio 服务器，WSL2 需要通过环境变量 `PULSE_SERVER` 连接到它。在 Ubuntu 内执行：

```bash
export PULSE_SERVER=unix:/mnt/wslg/PulseServer
```

将此行添加到 `~/.bashrc` 使永久生效：

```bash
echo 'export PULSE_SERVER=unix:/mnt/wslg/PulseServer' >> ~/.bashrc
source ~/.bashrc
```

### 4. 测试音频设备

```bash
# 测试系统音频播放
paplay /usr/share/sounds/alsa/Noise.wav   # 需要安装 pulseaudio-utils
# 或使用 speaker-test
speaker-test -t wav -c 2

```



## 测试bpm检测代码
```python
import sounddevice as sd  
import numpy as np  
import essentia.standard as es  
import time  
from collections import deque  
  
# ---------- 参数配置 ----------SAMPLE_RATE = 22050          # 采样率，22050 Hz 对 BPM 估算足够  
# SAMPLE_RATE = 48000  # 采样率，22050 Hz 对 BPM 估算足够  
WINDOW_SECONDS = 10         # 每次分析的时间窗口长度（秒）  
BLOCK_SIZE = 1024           # 音频块大小（采样点数）  
CHANNELS = 1                # 单声道（如果麦克风是多声道，回调中会混音）  
  
# 缓存：用 deque 存储最近 WINDOW_SECONDS 秒的音频块  
# 每个元素是 (timestamp, audio_block)audio_buffer = deque()  
buffer_lock = False         # 简易锁，避免分析过程中数据被修改  
  
# ---------- 回调函数：实时追加音频块 ----------def audio_callback(indata, frames, time_info, status):  
    """sounddevice 输入回调，每收到一个块就存入缓冲区"""  
    if status:  
        print(f"⚠️ 状态：{status}")  
    global buffer_lock  
    if not buffer_lock:  
        # 确保数据为 float32，值域 [-1, 1]        audio = indata.copy().astype(np.float32)  
        # 如果立体声，混音为单声道  
        if audio.shape[1] > 1:  
            audio = np.mean(audio, axis=1, keepdims=True)  
        audio_buffer.append((time.time(), audio.flatten()))  
  
# ---------- 使用 Essentia 估算 BPM ----------def estimate_bpm(audio_signal, sample_rate=SAMPLE_RATE):  
    """输入音频数组（1D float32），返回估算的 BPM"""    # 创建 PercivalBpmEstimator 算法实例  
    bpm_estimator = es.PercivalBpmEstimator(sampleRate=sample_rate)  
    try:  
        bpm = bpm_estimator(audio_signal)  
        return bpm  
    except Exception as e:  
        print(f"❌ BPM 估算失败：{e}")  
        return 0.0  
  
# ---------- 从缓冲区中提取最近 WINDOW_SECONDS 秒的音频 ----------def get_recent_audio():  
    global buffer_lock  
    buffer_lock = True      # 防止回调写入干扰  
  
    if len(audio_buffer) == 0:  
        buffer_lock = False  
        return None  
    now = time.time()  
    # 只保留最近 WINDOW_SECONDS 秒的数据  
    cutoff = now - WINDOW_SECONDS  
    chunks = []  
    for ts, block in audio_buffer:  
        if ts >= cutoff:  
            chunks.append(block)  
    # 如果数据长度不足，返回 None    if len(chunks) == 0:  
        buffer_lock = False  
        return None  
    # 拼接为一个大数组  
    full_audio = np.concatenate(chunks)  
    buffer_lock = False  
    return full_audio  
  
# ---------- 主循环 ----------def main():  
    print(f"🎤 启动录音，采样率 {SAMPLE_RATE} Hz，每 {WINDOW_SECONDS} 秒分析一次 BPM...")  
    # 启动输入流  
    stream = sd.InputStream(  
        samplerate=SAMPLE_RATE,  
        blocksize=BLOCK_SIZE,  
        channels=CHANNELS,  
        callback=audio_callback,  
        dtype='float32'  
    )  
    with stream:  
        try:  
            while True:  
                time.sleep(WINDOW_SECONDS)  # 每隔分析窗口时长处理一次  
                audio = get_recent_audio()  
                if audio is None or len(audio) < SAMPLE_RATE * 0.5:  # 至少 0.5 秒数据  
                    print("⏳ 等待足够音频数据...")  
                    continue  
  
                # 可选：对音频做简单归一化（Essentia 内部通常已处理）  
                # 但保证值域不过大是有益的  
                audio = np.clip(audio, -1.0, 1.0)  
  
                # 调用 Essentia 估算 BPM                bpm = estimate_bpm(audio, SAMPLE_RATE)  
                if bpm > 0:  
                    print(f"🎵 当前估计 BPM：{bpm:.1f}")  
                else:  
                    print("⚠️ 无法检测到稳定节拍")  
  
        except KeyboardInterrupt:  
            print("\n🛑 程序终止")  
  
if __name__ == "__main__":  
    main()
```


## 测试gif图片

先安装opencv-python
```
pip install opencv-python
```

```python
import cv2  
import time  
import numpy as np  
import os  
import threading  
from collections import deque  
import sounddevice as sd  
import essentia.standard as es  
  
# ---------- 全局共享变量 ----------current_bpm = 120.0          # 默认 BPMbpm_lock = threading.Lock()  # 保护 current_bpmrunning = True              # 控制线程退出  
  
# ---------- BPM 检测参数 ----------SAMPLE_RATE = 22050  
# SAMPLE_RATE = 48000  
WINDOW_SECONDS = 8  
BLOCK_SIZE = 1024  
CHANNELS = 1  
  
audio_buffer = deque()  
buffer_lock = threading.Lock()  
  
# ---------- 音频回调 ----------def audio_callback(indata, frames, time_info, status):  
    if status:  
        print(f"⚠️ 状态：{status}")  
    with buffer_lock:  
        audio = indata.copy().astype(np.float32)  
        if audio.shape[1] > 1:  
            audio = np.mean(audio, axis=1, keepdims=True)  
        audio_buffer.append((time.time(), audio.flatten()))  
  
# ---------- Essentia BPM 估算 ----------def estimate_bpm(audio_signal, sample_rate=SAMPLE_RATE):  
    bpm_estimator = es.PercivalBpmEstimator(sampleRate=sample_rate)  
    try:  
        return bpm_estimator(audio_signal)  
    except Exception as e:  
        print(f"❌ BPM 估算失败：{e}")  
        return 0.0  
  
# ---------- 获取最近窗口音频 ----------def get_recent_audio():  
    with buffer_lock:  
        if not audio_buffer:  
            return None  
        now = time.time()  
        cutoff = now - WINDOW_SECONDS  
        chunks = [block for ts, block in audio_buffer if ts >= cutoff]  
        if not chunks:  
            return None  
        full_audio = np.concatenate(chunks)  
        return full_audio  
  
# ---------- BPM 检测线程函数 ----------def bpm_detection_loop():  
    global current_bpm  
    print("🎤 启动 BPM 检测线程...")  
    stream = sd.InputStream(  
        samplerate=SAMPLE_RATE,  
        blocksize=BLOCK_SIZE,  
        channels=CHANNELS,  
        callback=audio_callback,  
        dtype='float32'  
    )  
    with stream:  
        while running:  
            time.sleep(WINDOW_SECONDS)  
            audio = get_recent_audio()  
            if audio is None or len(audio) < SAMPLE_RATE * 0.5:  
                print("⏳ 等待足够音频数据...")  
                continue  
            audio = np.clip(audio, -1.0, 1.0)  
            bpm = estimate_bpm(audio, SAMPLE_RATE)  
            if bpm > 0:  
                with bpm_lock:  
                    current_bpm = bpm  
                print(f"🎵 更新 BPM 为：{bpm:.1f}")  
            else:  
                print("⚠️ 无法检测稳定节拍")  
  
# ---------- 播放 GIF 函数（实时读取全局 BPM）----------  
def play_gif_with_beat_pattern(  
    gif_path,  
    beat_pattern=(1, 0, 1, 0),  
    frames_per_beat=6,  
):  
    global running  
    cap = cv2.VideoCapture(gif_path)  
    if not cap.isOpened():  
        print("❌ 无法打开 GIF")  
        return  
  
    gif_frame_count = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))  
    if gif_frame_count <= 0:  
        print("❌ 无效的 GIF 帧数")  
        return  
  
    pattern_len = len(beat_pattern)  
    start_time = time.perf_counter()  
  
    print(f"▶️ 开始播放 GIF，初始 BPM: {current_bpm:.1f}")  
    print(f"拍子模式: {beat_pattern}")  
    print(f"每拍帧数: {frames_per_beat}")  
  
    while running:  
        # 1. 获取当前最新的 BPM        with bpm_lock:  
            bpm = current_bpm  
  
        beat_interval = 60.0 / bpm  
  
        # 2. 时间计算  
        now = time.perf_counter()  
        elapsed = now - start_time  
  
        beat_index_global = int(elapsed / beat_interval)  
        beat_index = beat_index_global % pattern_len  
        is_strong = beat_pattern[beat_index] == 1  
  
        beat_phase = (elapsed % beat_interval) / beat_interval  
        frame_in_beat = int(beat_phase * frames_per_beat)  
        frame_in_beat = min(frame_in_beat, frames_per_beat - 1)  
  
        gif_frame_index = (  
            beat_index_global * frames_per_beat + frame_in_beat  
        ) % gif_frame_count  
  
        # 3. 读取并显示帧  
        cap.set(cv2.CAP_PROP_POS_FRAMES, gif_frame_index)  
        # ret, frame = cap.read()  
        ret, frame = cap.read()  
        if not ret:  
            continue  
        frame = cv2.resize(frame, (240,240))  
  
        # 显示当前 BPM 和拍信息  
        # cv2.putText(frame, f"BPM: {bpm:.1f}", (40, 30),  
        #             cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 255), 2)        # cv2.putText(frame, f"Beat frame: {frame_in_beat+1}/{frames_per_beat}",        #             (40, 70), cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255,255,255), 2)        # cv2.putText(frame, f"Pattern idx: {beat_index}",        #             (40, 110), cv2.FONT_HERSHEY_SIMPLEX, 0.6, (200,200,200), 2)  
        cv2.imshow("GIF Beat Sync (Live BPM)", frame)  
  
        # 4. 按 ESC 退出  
        if cv2.waitKey(1) == 27:  
            running = False  
            break  
    cap.release()  
    cv2.destroyAllWindows()  
  
# ---------- 主程序 ----------if __name__ == "__main__":  
    # gif_path = "resources/gif/dance4.gif"  
    gif_path = "resources/gif/common.gif"  
    # gif_path = os.path.join(application_path, "resources/gif/common.gif")  
  
    # 启动 BPM 检测线程（后台）  
    bpm_thread = threading.Thread(target=bpm_detection_loop, daemon=True)  
    bpm_thread.start()  
  
    # 主线程播放 GIF（阻塞）  
    play_gif_with_beat_pattern(  
        gif_path,  
        beat_pattern=(1, 0, 1, 0),  
        frames_per_beat=13,  
    )
```