# esp-vocat 自定义补丁集合

在 78/xiaozhi-esp32 v2.2.6 (commit 49ac8a6) 基础上,为 EchoEar/ESP-VoCat (ESP32-S3 + ST77916 360x360 QSPI LCD + ES8311 + ES7210 + BQ27220 fuel gauge @ I2C 0x55) 加的若干能力。两组 patch 独立可选,顺序应用:

| Patch | 加什么 | 日期 |
|---|---|---|
| `0001-battery-and-wifi-display.patch` | 屏幕顶部时钟左边显示电量数字 + 充电 icon,右上(x=90,y=37)显示 WiFi 信号 icon | 2026-05-13 |
| `0002-meeting-mode-and-volume-control.patch` | 双击触屏 = MCP `meeting.toggle` 切会议模式;长按 1.5s = 音量循环 30/50/70/95/100;EmoteDisplay 加 SetMeetingMode hook 让"🎙 会议·静听"气泡常驻;默认音量从 70 → 95 + NVS<30 自动顶到 80 | 2026-05-17 |

## Patch 0001 — 电量 + WiFi 显示

## 修改的文件

- `main/boards/esp-vocat/esp_vocat.cc` — 重写 `Charge` class:实现 BQ27220 driver(读 V/I/SOC) + 推 `EMOTE_MGR_EVT_BAT` 事件 + 用 `emote_create_obj_by_type` 动态创建 `wifi_icon` widget 推 wifi 状态
- `managed_components/espressif2022__esp_emote_assets/360_360/layout.json` — **不改了**(我试过加 widget 失败,framework 有 name 白名单)
- `sdkconfig` — 5 项配置改动(见 `sdkconfig-changes.diff`)

## 4 个 hidden gotcha

1. **sdkconfig 默认 MESSAGE_STYLE 不是 emote**:v2.2.6 默认 `CONFIG_USE_DEFAULT_MESSAGE_STYLE=y`,需要切到 `CONFIG_USE_EMOTE_MESSAGE_STYLE=y`,否则 `display_` 是 `SpiLcdDisplay` 类型而不是 `EmoteDisplay`,推事件没用
2. **`CONFIG_MMAP_FILE_NAME_LENGTH=16` 太小**:`battery_charge.bin` (18 字符)、`icon_speaker.bin` (16 字符) 等长文件名被截断,asset lookup 失败。bump 到 32
3. **Charge task 起跑早于 display**:EspVocat 构造函数 `InitializeCharge()` → ... → `InitializeSt77916Display()`。Charge task 启动时 `display_=nullptr`,Board::GetInstance().GetDisplay() 返回 NoDisplay。**用 setter 显式注入**:`charge_->SetDisplay(static_cast<emote::EmoteDisplay*>(display_));` 在 InitializeSt77916Display 之后调
4. **layout JSON 的 widget name 有白名单**:emote_setup.c:210 `element_type_map` 只认 `eye_anim/emerg_dlg/status_icon/charge_icon/battery_label/clock_label/...` 这几个,自定义名(如 `wifi_icon`)在 layout.json 里加会报 `Unknown element`。要用 `emote_create_obj_by_type(h, "image", "wifi_icon")` **运行时**创建 + 手动 `gfx_obj_align(...)` 定位

## 复现步骤

```bash
# 1. clone 干净的 v2.2.6
git clone https://github.com/78/xiaozhi-esp32.git
cd xiaozhi-esp32
git checkout v2.2.6

# 2. set target + 默认 board 选 esp-vocat
source <path-to-esp-idf>/export.sh
idf.py set-target esp32s3
# (默认 sdkconfig 应该有 CONFIG_BOARD_TYPE_ESP_VOCAT=y;没有就 menuconfig 选)

# 3. 应用 sdkconfig 改动 (见 sdkconfig-changes.diff 里 sed 命令一行流)

# 4. 应用代码 patch
git apply /Users/orange001/Desktop/小智/esp-vocat-patches/0001-battery-and-wifi-display.patch

# 5. build (~5-10 min,因为切了 message style 要 full rebuild)
idf.py reconfigure  # 让 FLASH_EXPRESSION_ASSETS 自动 propagate
idf.py build

# 6. flash 完整 (NVS 在 0x9000 不动,wifi 配置保留)
esptool.py --port /dev/cu.usbmodem1101 --chip esp32s3 -b 460800 \
  --before default_reset --after hard_reset write_flash \
  --flash_mode dio --flash_size 16MB --flash_freq 80m \
  0x0       build/bootloader/bootloader.bin \
  0x8000    build/partition_table/partition-table.bin \
  0xd000    build/ota_data_initial.bin \
  0x20000   build/xiaozhi.bin \
  0x800000  build/expression_assets.bin
```

## Patch 0002 — 会议模式 + 音量控制(2026-05-17)

针对**没有侧面触摸滑条**的 EchoEar 单元(用户实测发现少了一个滑条),用触摸屏手势补回音量调节,同时加上一套"安静会议模式":

| 手势 | 行为 | 屏幕反馈 |
|---|---|---|
| 单击 | ToggleChatState(原行为) | 状态机:idle/connecting/listening |
| **双击** (600ms 内) | 发 MCP `meeting.toggle`,本地切换会议气泡 | 持久"🎙 会议·静听"或"✓ 退出会议" |
| **长按 ≥1.5s** | 音量循环 30/50/70/95/100 | "🔊 音量 XX" 1.5 秒通知 |

### 4 个关键设计决策

1. **第 2 次 tap 跳过 ToggleChatState** — 直觉是"每次 release 都调",但 ToggleChatState 在 listening 状态下会 `CloseAudioChannel`,导致 WS 撕掉,正在飞的 `meeting.toggle` 帧被丢(1006 close)。所以 double-tap 第 2 次只发 MCP + 切气泡,不动 chat state
2. **持久气泡靠 SetStatus/SetChatMessage 双 hook** — 会议气泡设过之后,异步状态机会发出 `SetStatus("连接中")` / `SetStatus("聆听中")` 覆盖。在 EmoteDisplay 里 hook 这两个方法,meeting_mode_ 为真时**每次都重写气泡**,这样后到的覆盖也压不掉
3. **长按消费 release** — TOUCH_HOLD 触发后,后续的 TOUCH_RELEASE 不再走 tap/double-tap,避免长按完又被识别成单击
4. **音量底线** — header 默认 70,Start() 改成"NVS<30 顶到 80"。原始默认 10 太轻用户以为设备没响应

### 服务端配合

需要 minion 插件(`openclaw-channel-minion-patched` repo)加 `meeting.toggle` MCP handler + 音量语音 intent + **去掉**老的 `auto-pushed listen` 推送(新固件 OnIncomingJson 不识别 listen 消息会 logW)。

### 应用

```bash
cd <xiaozhi-esp32-on-v2.2.6+0001-patch>
git apply 0002-meeting-mode-and-volume-control.patch
idf.py build && idf.py flash
```

## 应用现状(2026-05-17)

- Device `ac:a7:04:e4:5c:f0` (EchoEar #1) — **应用 0002**(会议+音量),刷过最新 build;0001(电量+WiFi)没装电池没装
- Device `90:e5:b1:cd:64:f0` (EchoEar #2) — **应用 0001**(电量+WiFi),屏幕显示正常;0002 还没装
- Stock 备份在 `/Users/orange001/Desktop/小智/esp-vocat-firmware/merged-binary.bin` (11MB,可一键 esptool write_flash 0x0 回滚)
- 5c:f0 完整 partitions 备份(NVS/ota_0/assets/etc) 在 `~/Desktop/小智/firmware-backup/echoear-90e5b1cc9994/`

## 已知限制

- BQ27220 寄存器读 `0x08` voltage 用 ESP_ERROR_CHECK 包的 `i2c_master_transmit_receive`,失败会 abort 设备。我们已经验证 0x55 上的芯片在响应,所以稳定(只适用 patch 0001)
- WiFi 图标 dedupe 后状态只在变化时更新,但屏幕缓存正常会渲染。开机首次连接前几秒可能空着(等 esp_wifi_sta_get_ap_info 返回稳定)(只适用 patch 0001)
- 这次没动 PROJECT_VER,xiaozhi.bin 的 esp_app_desc 还是 "2.2.6"。OTA 路径不能区分 stock vs 自编版本。要靠外部记账(本文档 + memory)知道哪台是哪个版本
- Patch 0002 双击逻辑跟 ToggleChatState 副作用绑死,如果上游改 state machine,需要重新验证
