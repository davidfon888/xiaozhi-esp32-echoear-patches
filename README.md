# esp-vocat 屏幕显示电量 + WiFi 补丁

2026-05-13 自开发,在 78/xiaozhi-esp32 v2.2.6 (commit 49ac8a6) 基础上加 4 处修改,让 EchoEar/ESP-VoCat (ESP32-S3 + ST77916 360x360 QSPI LCD + ES8311 + ES7210 + BQ27220 fuel gauge @ I2C 0x55) 屏幕顶部时钟两侧显示:
- 时钟左边:电量数字 + 充电指示 icon
- 时钟右上 (x=90, y=37):WiFi 信号 icon(连接/断开切换)

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

## 应用现状

- Device `90:e5:b1:cd:64:f0` (Mac mini 上注册的 EchoEar #2) — **已应用**,屏幕显示正常
- Device `ac:a7:04:e4:5c:f0` (EchoEar #1) — 还是 stock v2.2.6,没装电池所以也没必要刷
- Stock 备份在 `/Users/orange001/Desktop/小智/esp-vocat-firmware/merged-binary.bin` (11MB,可一键 esptool write_flash 0x0 回滚)
- 对应原本 OEM v2.0.3.3 backup 在 `/Users/orange001/Desktop/小智/esp-box-3-backup-90e5b1cd64f0/`

## 已知限制

- BQ27220 寄存器读 `0x08` voltage 用 ESP_ERROR_CHECK 包的 `i2c_master_transmit_receive`,失败会 abort 设备。我们已经验证 0x55 上的芯片在响应,所以稳定
- WiFi 图标 dedupe 后状态只在变化时更新,但屏幕缓存正常会渲染。开机首次连接前几秒可能空着(等 esp_wifi_sta_get_ap_info 返回稳定)
- 这次没动 PROJECT_VER,xiaozhi.bin 的 esp_app_desc 还是 "2.2.6"。OTA 路径不能区分 stock vs 自编版本。要靠外部记账(本文档 + memory)知道哪台是哪个版本
