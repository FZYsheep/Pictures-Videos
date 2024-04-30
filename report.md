# AlphaRTC视频质量评估

组长：徐一茗
组员：冯之乐  俞俊喆  王致烨

## 实验准备

准备视频传输模拟所需的环境以及下载相关工具
### docker

从 [Github release](https://github.com/OpenNetLab/AlphaRTC/releases/latest/download/alphartc.tar.gz) 下载相应的Docker镜像准备实验环境。

```bash
iwr "https://github.com/OpenNetLab/AlphaRTC/releases/latest/download/alphartc.tar.gz" -OutFile "absolute_file_path_for_downloads"
docker load -i alphartc.tar.gz
```

### AlphaRTC

从 [Github](https://github.com/OpenNetLab/AlphaRTC) 上克隆实验所需的AlphaRTC仓库

```bash
git clone https://github.com/OpenNetLab/AlphaRTC.git
```

### ffmpeg & vmap
#### ffmpeg

ffmpeg 是一个处理媒体文件的命令行工具，我们使用 ffmpeg 进行不同媒体格式之间的转换。我们从 [Github release](https://github.com/BtbN/FFmpeg-Builds/releases) 下载ffmpeg，根据系统（Windows、Linux）选择配置合适的版本。
#### vmap

视频压缩前后的画质对比可以借助VMAF评分进行量化对比，从而确定视频压缩参数。
从 [Github release](https://github.com/Netflix/vmaf/releases) 直接下载可执行文件 `vmap.exe` ，用于视频传输质量的评分。

## 视频传输

传输实例：分辨率1280x720，50fps的原视频。
### 传输过程

为了实现 AlphaRTC 下的视频传输，我们使用 `peerconnection_serverless`——AlphaRTC附带的演示应用程序。它在不需要服务器的情况下与另一个对等点建立RTC通信。要运行AlphaRTC实例，将 json 格式的配置文件放在本地一个目录中，例如`config_files`，然后将其挂载到 `alphartc` 容器内。

使用 `BandwidthEstimator.py` 定义的 bandwidth estimator 运行AlphaRTC实例时，命令行如下：

```bash
docker run -d --rm -v `pwd`/examples/peerconnection/serverless/corpus:/app -w /app --name alphartc alphartc peerconnection_serverless receiver_pyinfer.json
docker exec alphartc peerconnection_serverless sender_pyinfer.json
```

执行上述命令后，我们可以得到传输后得到的视频、音频文件以及一个记录传输信息的 log 文件。根据传输后得到的视频文件和 log 记录文件，我们可以计算得到视频传输的相关指标。

## 传输指标计算

### loss

传输过程得到的 log 中的部分输出如下：

```text
(receive_statistics_proxy2.cc:495): WebRTC.Video.ReceiveStreamLifetimeInSeconds 19
Frames decoded 198
WebRTC.Video.DroppedFrames.Receiver 0
WebRTC.Video.ReceivedPacketsLostInPercent 0
WebRTC.Video.DecodedFramesPerSecond 10
WebRTC.Video.InterframeDelay95PercentileInMs 102
WebRTC.Video.InterframeDelay95PercentileInMs.S0 102
WebRTC.Video.MediaBitrateReceivedInKbps.S0 565
WebRTC.Video.MediaBitrateReceivedInKbps 589
```

`WebRTC.Video.ReceivedPacketsLostInPercent` 显示为0，该值的计算过程显示如下：

```c++
absl::optional<int> StreamStatisticianImpl::GetFractionLostInPercent() const {
  /* Code not shown ...*/
  int64_t expected_packets = 1 + received_seq_max_ - received_seq_first_;
  if (expected_packets <= 0) {
    return absl::nullopt;
  }
  if (cumulative_loss_ <= 0) {
    return 0;
  }
  return 100 * static_cast<int64_t>(cumulative_loss_) / expected_packets;
}
```

即为传输过程中对于接收方来说丢失的packets的数目除以接收方应接受到的packets的数目。
log 中该值始终显示为0。

### throughout

传输过程得到的 log 中的部分输出如下：

```text
(remote_estimator_proxy.cc:151): 
{...packetInfo":{"arrivalTimeMs":1714411982486,"header":{"headerLength":28,"paddingLength":0,"payloadType":125,"sendTimestamp":49037,"sequenceNumber":11881,"ssrc":3160138741},"lossRates":0.0,"payloadSize":1071}}
```

```c++
  // Save per-packet info locally on receiving
  auto res = stats_collect_.StatsCollect(
      pacing_rate, padding_rate, header.payloadType,
                              header.sequenceNumber, send_time_ms, header.ssrc,
                              header.paddingLength, header.headerLength,
                              arrival_time_ms, payload_size, 0);
  /* Code Not Shown ...*/
  RTC_LOG(LS_INFO) << out_data;
```

如上述代码显示，`remote_estimator_proxy.cc` 文件的151行在log中输出了packet相关信息，包括packet的达到时间 `arrival_time_ms` ，packet有效载荷大小 `payload_size`。根据这两个信息我们可以计算吞吐量：

```python
while(True):
	text_line = f.readline()
	if(text_line):
		if(text_line.startswith("(remote_estimator_proxy.cc:151):") == False):
			continue
		json_data = json.loads(text_line[33:])
		arrivalTimeMs = json_data["packetInfo"]["arrivalTimeMs"]
		payloadSize = json_data["packetInfo"]["payloadSize"]
		total_payload_size += int(payloadSize)
		if(flag == False):
			flag = True
			last_arrival_time = arrivalTimeMs
		total_time_ms += arrivalTimeMs - last_arrival_time
		last_arrival_time = arrivalTimeMs
	else:
		break
print(total_payload_size * 1000 / total_time_ms)
```

如上述代码，每次到达一个包记录有效载荷的大小和到达时间，由此记录在 `total_time_ms` 时间内到达的有效载荷的总字节数 `total_payload_size` ，最终计算出视频传输的吞吐量（单位：bytes/s）。

### vmap

首先需要使用 ffmpeg 将原视频和传输后得到的视频由原来的格式转码成 .y4m 格式，然后进行 vmap 的计算。
如下例，将 yuv 格式的原视频 `out_video.yuv` 转换生成一个 y4m 格式的视频 `video_output.y4m`

```bash
ffmpeg -i out_video.yuv video_output.y4m 
```

使用 vmaf 对进行评分，-d 后面为转码视频y4m文件路径，-r 后面为原视频y4m文件路径

```bash
.\vmaf -d video_input.y4m -r video_output.y4m
```

## 结果分析

### throughout

我们记录了传输过程中每1000ms的throughout变化，如下图所示

![[t_fig.png|450]]
![[8e7c44025e1788eeef915f5c8004f19.png|450]]
1280_720_xfps对应1280x720分辨率，不同帧率的输出设置。
### vmap
VMAF分值在0~100之间，越大越好，20分对应极差，100分对应极好。

![[vamp.jpg|500]]

### 其他

[用于传输的原视频](https://github.com/FZYsheep/Pictures-Videos/blob/main/1280_720_10fps_out_video.mp4)
[传输后得到的1280x720-10fps的输出视频](https://github.com/FZYsheep/Pictures-Videos/blob/main/origin_video.mp4)

可以看出传输后得到的视频的逐渐变得清晰，推测是在AlphaRTC传输过程中进行了动态编码参数调整过程。
