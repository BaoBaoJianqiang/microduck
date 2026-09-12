# 解析：`audio/aic3104-init.sh`

## 这是什么

TLV320AIC3104 编解码器的 **ALSA 混音器初始化脚本**（42 行 bash）。内核编解码器驱动（见 `tlv320aic3x.c`/`tlv320aic3x-i2c.c` 的解析）负责 PLL、DAC、线路输出等通过设备树 overlay 配置的部分；本脚本只做驱动管不着的一层——设置默认混音电平和麦克风路由。

## 执行流程

```bash
sleep 2
```

先等 2 秒，随后进入重试循环：

```bash
for i in $(seq 1 15); do
    if amixer -c aic3104 info >/dev/null 2>&1; then
        ...
        break
    fi
    sleep 1
done
```

- `amixer -c aic3104 info` 探测名为 `aic3104` 的 ALSA 声卡是否已注册；
- **重试窗口共 15 次、每次间隔 1 秒**——注释解释了为什么窗口这么宽：在 Radxa 上声卡探测被推迟到 DKMS 编解码器模块自动加载之后，开机早期声卡可能还没出现；
- 探测成功则设置混音器并 `break`，失败则每秒再试一次。

## 混音器设置逐项解析

### 扬声器通路（`cset` 控制数值型控件）

| 命令 | 作用 |
|---|---|
| `PCM Playback Volume 127,127` | DAC 数字音量拉满（DAC 通路，-63.5~0 dB 刻度，127 即 0 dB） |
| `Line DAC Playback Volume 118,118` | DAC→线路输出（LOP）混音级的音量（0~118 刻度） |
| `Line Playback Switch on,on` | LOP 输出级静音开关打开 |
| `Line Playback Volume 9,9` | LOP 输出级增益拉满（0~9 dB，1 dB 步进） |

注释里有一个重要的踩坑记录：**`Line Playback Switch` 是 LOP 输出级的 MUTE，不是 line-in 旁路**。本脚本曾把它设为 `off`，直接把机器人静音了——当时树莓派上"看起来正常"只是因为 alsa-restore 恰好把一份旧保存状态（开关为 on）重新应用了回去。扬声器功放挂在编解码器的线路输出（`LEFT_LOP`/`RIGHT_LOP`）上，这条链路任何一级关掉都没有声音。

### 板载麦克风路由（`sset` 控制开关型控件）

```bash
amixer -c aic3104 sset 'Right PGA Mixer Mic3R'  on  >/dev/null
```

板载麦克风接在 **Mic3R**，路由为 **Mic3R → 右声道 PGA**；其余全部输入混音路由（左/右 PGA 的 Mic3L、Line1L/R、Line2R 共 9 路）显式 `off`——逐路关掉而不是依赖默认值，避免残留路由把噪声混进录音。

### 采集通路

| 命令 | 作用 |
|---|---|
| `PGA Capture Switch on,on` | 打开 PGA 采集开关 |
| `PGA Capture Volume 60,60` | PGA 采集增益（0~119 刻度，注释注明范围） |

所有设置输出都重定向到 `/dev/null`，成功后打印 `TLV320AIC3104 mixer levels set`。

## 在整个部署里的位置

`setup-board.sh` 装好 DKMS 驱动与 overlay → 声卡出现 → 本脚本（由守护进程启动流程在开机后调用）把混音器设到机器人需要的固定状态 → `robotd` 的 `[audio] device = "plughw:aic3104"` 从该设备播放语音、`"<device>,0"` 采集麦克风。若 15 次重试内声卡始终未出现，脚本静默退出，音频功能缺席但不阻塞启动。
