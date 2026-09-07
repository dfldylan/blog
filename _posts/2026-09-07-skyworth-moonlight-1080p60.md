---
layout: default
title: 把一台 2017 年的创维电视调成可用的 1080p60 Moonlight 客户端
date: 2026-09-07 14:30:00 +0800
categories: [家庭影音, 技术排障]
tags: [创维电视, Moonlight, Sunshine, Android, 1080p60]
---

> 一次关于 Android 图形层、海思视频管线、Sunshine 可变帧率和老硬解码器的完整排障记录。

## 最终结果

这台创维 40E3500（8H31 机芯、Hi3751V320、Android 4.4.2）原本在 Android 层只报告 1280×720、约 25Hz。经过修改系统刷新率、排查海思视频管线，并调整 Sunshine/Moonlight 后，最终可以稳定播放带弹幕的 B 站 1080p60 视频。

最终配置：

| 位置 | 配置 |
|---|---|
| 电视系统 | `ro.config.frame_refresh_rate=60` |
| Sunshine | Intel QuickSync H.264、`minimum_fps_target=60`、`qsv_coder=cavlc` |
| Moonlight | 1920×1080、60fps、10Mbps、Balanced with FPS limit（配置值 `cap-fps`） |
| 网络 | 有线百兆、同一局域网 |

重启电视、Sunshine 和 Moonlight 后实测：

- 视频流：58.65fps
- 网络接收：58.56fps
- 渲染：58.56fps
- 网络丢帧：0.00%
- 局域网延迟：1ms

![最终 Moonlight 统计面板]({{ site.baseurl }}/assets/final-stats.png)

完整截图见：[final-1080p60.png]({{ site.baseurl }}/assets/final-1080p60.png)。Android 4.4 的 `screencap` 对视频层和图形层并非严格同步，完整截图可能出现横向拼接痕迹；统计面板数值和电视肉眼播放效果不受影响。

## 测试环境

### 电视端

| 项目 | 信息 |
|---|---|
| 型号 | 创维 40E3500 |
| 机芯 | 8H31 |
| SoC | HiSilicon Hi3751V320（系统设备名 `Hi3751V320`） |
| 面板 | FHD 1920×1080，支持 50/60Hz |
| Android | 4.4.2 / API 19 |
| 固件 | `017.010.260`，构建日期 2017-10-26 |
| Moonlight 解码器 | `OMX.hisi.video.decoder.avc` |
| 可用视频编码 | H.264 硬解；未发现可用的 HEVC/AV1 硬解入口 |

### 发送端

- Sunshine `2026.516.143833`
- Windows 主机，桌面 3840×2160@60Hz
- Intel Iris Xe QuickSync，编码器 `h264_qsv`
- Moonlight 请求 1920×1080@60fps

## 最初的现象

问题并不是简单的“电视只能跑 720p25”。更准确的说法是：

1. Android 图形合成层原本是 1280×720，SurfaceFlinger 约 25fps；
2. 物理面板和海思视频输出层实际支持 1920×1080@50/60Hz；
3. 修改系统刷新率后，菜单和桌面能到 60Hz，但 Moonlight 播放复杂视频仍会逐渐掉帧；
4. 暂停视频或切回 Windows 桌面操作，帧率会马上升高；重新播放复杂视频后，又会慢慢变卡；
5. Moonlight 叠加层把大量帧标成“网络丢失”，但局域网 ping、网卡错误计数和链路状态都正常。

这组现象很容易让人分别误判为：网络问题、电视节能模式、物理面板不支持 60Hz，或者 Sunshine 没有按 60fps 发送。实际原因是多个层次叠加。

## 先建立分层模型

排查这类电视串流问题，至少要把以下四层分开：

1. **Android 图形层**：SurfaceFlinger、应用 UI 分辨率和刷新率；
2. **视频解码层**：海思 OMX H.264 解码器；
3. **视频后处理和显示层**：VPSS、PQ、DISP、面板时序；
4. **串流协议层**：Sunshine 捕获/编码、网络传输、Moonlight 接收/帧调度。

“Moonlight 来源是 60fps”“Android 报告 60Hz”“面板输出 60Hz”和“最终渲染 60fps”是四件不同的事。只盯着其中一个数字，会不断得出互相矛盾的结论。

## 第一步：确认面板能力并修正 Android 刷新率

电视固件的 `/system/build.prop` 中原值为：

```properties
ro.config.frame_refresh_rate=25
```

先备份原文件，再改为：

```properties
ro.config.frame_refresh_rate=60
```

修改后重启，验证：

```sh
adb shell getprop ro.config.frame_refresh_rate
adb shell dumpsys SurfaceFlinger | grep -E 'refresh-rate|fps'
```

本机最终输出：

```text
60
refresh-rate : 60.000002 fps
```

### 推荐的安全修改流程

需要已获得 root shell。不同固件取得 root 的方式不同，不建议照搬其他机型的方法。

```sh
# 先把原文件备份到电脑
adb pull /system/build.prop build.prop.original

# 在电脑上复制并编辑，只修改目标键
cp build.prop.original build.prop.new
# 将 ro.config.frame_refresh_rate 的值改成 60

# 推到可写临时目录
adb push build.prop.new /data/local/tmp/build.prop.new
```

随后在电视的 root shell 中：

```sh
mount -o remount,rw /system
cp /system/build.prop /data/local/tmp/build.prop.25.bak
cp /data/local/tmp/build.prop.new /system/build.prop
chown 0:0 /system/build.prop
chmod 0644 /system/build.prop
mount -o remount,ro /system
reboot
```

务必确认新文件不是空文件、权限仍为 `0644`，并保留一份电视外部备份。错误的 `build.prop` 可能导致系统无法正常启动。

### 这一步证明了什么

- 面板和显示驱动确实能运行 1080p60；
- Android 原来的 25fps 是固件配置，不是面板硬限制；
- 但它只解决图形合成层和基础输出时序，不能保证复杂 H.264 视频最终也能稳定解码和渲染 60fps。

## 第二步：排除物理网络

电视使用有线百兆、全双工连接。测试结果：

- 电视到 Sunshine 主机的 1200 字节 ping：30 次、0% 丢包、平均约 2ms；
- 以太网错误计数为 0；
- 最终串流时 Moonlight 显示网络延迟 1ms；
- 降低码率时，Moonlight 的所谓“网络丢帧”会显著变化，但物理链路指标不变。

因此，叠加层中的“Frames dropped by network connection”不能在这个案例里直接等同于网线或交换机丢包。Moonlight 官方也提醒：统计值受客户端硬件、分辨率、帧率、码率和解码 API 限制影响；网络抖动类丢帧也可能由硬件或软件问题引起。

更可靠的判断方法是同时看：

- ping 和 jitter；
- 网卡错误/丢包计数；
- Sunshine 实际编码帧率和码率；
- Moonlight 的来源、接收、渲染和解码耗时；
- 电视端 OMX/VPSS 的实际处理速率。

## 第三步：海思 VPSS 的几个误导性线索

通过 root 读取：

```sh
cat /proc/hisi/msp/omxvdec
cat /proc/hisi/msp/vpss00
cat /proc/hisi/msp/pq
cat /proc/hisi/msp/disp1
```

曾看到：

```text
OMX: H264, 1920x1080, FrameRateLimit=0
VPSS: MaxFrameRate=30000
PQ: TIMING_1080P50
```

最初很自然地把 `MaxFrameRate=30000` 理解为“VPSS 被硬锁 30fps”，但最终状态推翻了这个解释：重启后该字段仍是 30000，而 VPSS 的 `GetSrcImg/Process OK` 已达到约 59fps，Moonlight 也稳定渲染 58.56fps。

**结论：这个字段至少不能脱离端口类型、时间基准和驱动内部路径按字面解释。它不是本案例的最终瓶颈。**

### `setrate` 试验

在公开的同系列海思 SDK 源码中，VPSS proc 节点支持：

```sh
echo 'setrate on 60' > /proc/hisi/msp/vpss00
echo 'setrate off' > /proc/hisi/msp/vpss00
```

我们一度错误地写入 `60000`。源码显示保存帧率的字段只有 8 位，因此 `60000 mod 256 = 96`，电视果然报告了约 96fps 的异常输入时序。改用正确单位 `60` 后，只改变了源帧率元数据，并没有解决实际处理速度。

这个失败试验说明：

- proc 参数的显示单位不一定等于写入单位；
- 找到同系列源码后仍需验证结构体位宽和本机输出；
- `setrate` 不是“强制解码器跑满 60fps”的总开关。

### `setbypass` 试验

```sh
echo 'setbypass on' > /proc/hisi/msp/vpss00
```

结果是画面冻结、VPSS 缓冲填满。关闭旁路后旧实例仍未恢复，需要退出并重新建立 Moonlight 会话。

它证明了 VPSS 是该电视 OMX 视频输出的必要环节，不能简单绕过来换性能。该试验没有写入固件，重建会话/重启后恢复。

### 强制 DISP 1080p60 试验

DISP proc 帮助中存在：

```text
misc 0 = disable
misc 1 = FHD50Hz
misc 2 = FHD60Hz
```

运行：

```sh
echo misc 2 > /proc/hisi/msp/disp1
```

确实能把底层显示输出固定为 1080p60，但 Moonlight 的实际渲染帧率没有提高，甚至更低。说明当时的瓶颈不在面板输出时序，而在送帧/解码链。最终用 `misc 0` 撤回。

这个试验很重要：**显示器在 60Hz 扫描，不代表每次扫描都有一张新视频帧。**

## 第四步：从“暂停就恢复”定位到可变帧率与解码复杂度

最关键的肉眼观察是：

- 播放复杂的 1080p60 视频时，帧率会逐渐降低；
- 暂停视频，或者切回 Windows 桌面操作，帧率马上升高；
- 再次播放，解码和渲染又慢慢积压。

Sunshine 日志同时显示：

```text
Requested frame rate [60/1 exactly 60 fps]
Minimum FPS target set to ~30fps (33.3333ms)
```

原来 Sunshine 的“最低 FPS 目标”为 `0`。官方配置文档明确说明：Sunshine 会在静止或低帧率内容时节省带宽；`minimum_fps_target=0` 表示最低目标约为串流 FPS 的一半。Moonlight 官方 FAQ 也说明 Sunshine 默认使用可变帧率，静态内容的实际帧率可能明显降低。

与此同时，老海思 H.264 解码器对复杂视频的承受能力明显弱于桌面画面。桌面容易压缩，单帧数据量和熵解码负担较低；复杂视频持续变化，解码队列更容易积压。两种机制叠加后，就出现了“标称来源 60fps，但复杂视频只渲染十几帧”的现象。

## 真正有效的 Sunshine 设置

在 Sunshine Web UI 中修改：

### Audio/Video

```text
最低 FPS 目标：60
最大比特率：0（继续采用 Moonlight 请求值）
```

等价配置：

```ini
minimum_fps_target = 60
```

### Intel QuickSync Encoder

```text
QSV 编码器预设：medium
QSV 编码器 (H264)：cavlc
```

等价配置：

```ini
qsv_preset = medium
qsv_coder = cavlc
```

Sunshine 官方文档将 CAVLC 描述为“faster decode”。它的压缩效率通常不如 CABAC，但对这类老硬解码器更友好。这里保留 medium 预设，是为了避免同时改变太多变量和明显牺牲画质。

保存并重启 Sunshine 后，日志必须能看到：

```text
config: 'minimum_fps_target' = 60
config: 'qsv_coder' = cavlc
Minimum FPS target set to ~60fps (16.6667ms)
```

注意：从其他电脑通过 `https://<SUNSHINE_IP>:47990` 访问时，Sunshine 可能因 CSRF 保护拒绝保存。最稳妥的办法是在发送端电脑本机打开：

```text
https://localhost:47990
```

## 最终 Moonlight 设置

```text
分辨率：1920×1080
帧率：60fps
码率：10Mbps
视频格式：自动（本机最终使用 H.264）
帧调度：Balanced with FPS limit / cap-fps
性能统计：开启（调试完成后可关闭）
```

为什么不是继续堆码率？因为客户端硬解码器的吞吐能力同样是上限。Moonlight 官方 FAQ 也明确指出，硬件解码器必须能够处理所选码率；网络带宽够，并不代表客户端解码器能吃下同样高的码流。

在这个案例中，10Mbps 是画质和稳定性的甜点位。更高码率曾出现播放一段时间后逐渐积压；更低码率虽然更容易稳定，但画质收益开始下降。

## 前后对比

以下数据来自同一台电视的 Moonlight 叠加层。内容并非完全逐帧一致，因此只用于展示量级变化，不应当当作严格基准测试。

| 阶段 | 来源 FPS | 接收 FPS | 渲染 FPS | “网络丢帧” | 解码统计 |
|---|---:|---:|---:|---:|---:|
| 原始自适应设置、复杂视频 | 约 60 | 约 16.9 | 约 12.7 | 约 75% | 约 388ms |
| `minimum_fps_target=60` + CAVLC，首次复测 | 58.65 | 56.67 | 55.69 | 12.78% | 270.5ms |
| 全部设备重启后，10Mbps 最终测试 | 58.65 | 58.56 | 58.56 | 0.00% | 245.28ms |

Moonlight 官方提醒，在某些 Android 设备上，Sunshine 可变帧率会让静态内容的解码延迟统计显得异常高。因此应优先结合接收/渲染 FPS、丢帧和肉眼稳定性判断，而不是只盯着一个延迟数字。

## 哪些修改会持久化

| 设置 | 重连 Moonlight | 重启电视 | 重启 Sunshine/电脑 |
|---|---:|---:|---:|
| `build.prop` 中的 60Hz | 保留 | 保留 | 不受影响 |
| Sunshine 最低 FPS 60 | 保留 | 不受影响 | 保留 |
| Sunshine QSV CAVLC | 保留 | 不受影响 | 保留 |
| Moonlight 1080p60/10Mbps | 保留 | 保留 | 不受影响 |
| root Telnet 诊断口 | 通常保留 | 消失 | 不受影响 |
| VPSS/OMX/DISP proc 调试命令 | 可能随实例消失 | 消失 | 不受影响 |

本案例最终没有保留任何 VPSS、OMX 或 DISP 临时覆盖；它们全部恢复默认。重启后真正需要的只有三处持久配置：电视系统 60Hz、Sunshine 最低 60fps+CAVLC、Moonlight 1080p60/10Mbps。

## 仍可能存在的问题

1. **这不是通用固件教程。** 8H31/Hi3751V320 的路径和 proc 节点不代表其他创维机芯也相同。
2. **旧 Android 的统计可能不完全可靠。** OMX 驱动、SurfaceFlinger 和 Moonlight 各自使用不同时间基准，单个数字要交叉验证。
3. **CAVLC 会牺牲部分压缩效率。** 在相同码率下可能略逊于 CABAC，但换来了明显更轻的解码负担。
4. **10Mbps 是本机甜点位，不是所有电视的标准答案。** 建议从 8Mbps 开始，每次增加 1–2Mbps，并用同一段高动态视频持续测试至少几分钟。
5. **Android 截图可能出现视频层撕裂/拼接。** 这不一定代表 HDMI/面板肉眼可见撕裂，需要用相机或采集设备复核。
6. **不要长期开放工厂 root Telnet。** 诊断结束后关闭服务或重启电视；即使只在可信局域网内，也不应把该端口暴露到外网。
7. **修改 `/system/build.prop` 有启动风险。** 必须保留外部备份和官方恢复固件，并确认恢复路径可用。

关闭诊断口后应从另一台设备验证端口确实拒绝连接。这个固件的守护接口曾返回“停止成功”，但实际 `skybusybox` telnetd 仍在监听；最终通过 `ps` 找到确切的 telnetd PID 并只结束该进程。不要使用模糊匹配或批量杀进程；不确定时，直接重启电视是更稳妥的收尾方式。

## 一套更高效的排查顺序

如果再遇到类似老电视串流问题，推荐按下面顺序操作：

1. 确认面板和底层 DISP 是否支持目标分辨率/刷新率；
2. 分别读取 Android、视频解码、VPSS/PQ/DISP 的状态，不要混为一谈；
3. 先用 ping、网卡错误和 jitter 排除真实网络问题；
4. 在 Sunshine 日志确认请求 FPS、实际编码器、实际码率和最低 FPS 目标；
5. 使用 Moonlight 叠加层同时记录来源、接收、渲染、丢帧和解码耗时；
6. 用同一段高动态视频，逐项改变一个变量；
7. 老客户端优先尝试更容易解码的 H.264 模式，再寻找码率甜点位；
8. 最后才考虑 proc 级旁路或修改驱动行为，并确保每一步都可撤回。

## 参考资料

- [Sunshine Configuration：`minimum_fps_target` 与 `qsv_coder`](https://docs.lizardbyte.dev/projects/sunshine/v2025.922.195653/md_docs_2configuration.html)
- [Moonlight FAQ：Sunshine 可变帧率、性能统计和 Android 帧调度](https://github.com/moonlight-stream/moonlight-docs/wiki/Frequently-Asked-Questions)
- [Moonlight Android `Game.java`：刷新率选择与 `cap-fps` 行为](https://github.com/moonlight-stream/moonlight-android/blob/master/app/src/main/java/com/limelight/Game.java)
- [同系列 HiSilicon SDK 的 VPSS proc 实现（非本机官方发布，仅用于交叉验证）](https://github.com/07bug/HiSTBLinuxV100R005C00SPC060/blob/a9f05973129d738e175416dd4a91b2c264ffcdd4/source/msp/drv/vpss/vpss_v4_0/vpss_info.c#L2416)

## 结语

这次排障最有价值的地方，不是把某个神秘开关从 0 改成 1，而是逐步证明了哪些直觉是对的、哪些字段不能按字面解释。

用户观察到“暂停就恢复、播放就逐渐变卡”，准确指出了问题与内容变化和解码负担有关；Sunshine 的可变帧率又进一步放大了旧客户端的异常表现。最终方案没有依赖危险的驱动旁路，而是让发送端稳定提供 60fps、选择更容易解码的 H.264 CAVLC，并把码率控制在老海思芯片能持续承受的范围内。

对于老电视来说，真正可用的 1080p60，不是某一层报告“60”，而是来源、接收、解码、渲染和物理输出能一起长期稳定地接近 60。
