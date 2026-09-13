# Xiaozhi ESP32 + ESP8266 智能家居紅外控制

用「小智」AI 語音玩具（ESP32-S3）配合 ESP8266 麵包板，實現**語音控制家中紅外（IR）家電**（電視等）嘅完整開源方案。

- 語音終端：小智 ESP32-S3（16MB Flash / 8MB PSRAM，板型 `xingzhi-cube-1.54tft-wifi`）
- IR 發射器：ESP8266 NodeMCU + HX1838 接收頭 + IR 發射 LED + OLED 顯示
- 已實測鏈路：講「幫我開電視」→ 電視開機 ✅

---

## 系統架構

```
用戶講嘢 (粵語/國語)
      │  Voice (WebSocket / MQTT)
      ▼
┌──────────────────┐
│   小智 ESP32-S3   │  語音終端：收音/喚醒/播放
│  (Xiaozhi 固件)   │
└────────┬─────────┘
         │  WebSocket (xiaozhi 協議)
         ▼
┌──────────────────┐
│   小智 Server     │  ASR(語音識別) → LLM(大模型) → TTS(合成)
│ (官方 or 自建)     │
└────────┬─────────┘
         │  MCP 工具調用: tv_ir.send(button, times)
         │  GET http://<8266-IP>/api/send?group=TV&name=電源
         ▼
┌──────────────────┐
│   ESP8266 NodeMCU │  麵包板：接收指令 → 38kHz IR 發射
│  (IR Blaster)     │
└────────┬─────────┘
         │  紅外線 (NEC/NIKAI 等協定)
         ▼
┌──────────────────┐
│   TCL 電視 / 家電  │
└──────────────────┘
```

**分工原則**：ESP32 只做「語音前端」（收音/播放）；所有 AI 邏輯（識別、對話、合成）喺 Server 側；設備控制由 MCP 工具橋接到 8266 嘅 HTTP API，8266 負責發射紅外線。

---

## 硬件清單

| 組件 | 型號/規格 | 用途 |
|---|---|---|
| ESP32-S3 | 小智 AI 玩具（16MB Flash、8MB PSRAM、1.54" TFT） | 語音終端 |
| ESP8266 | NodeMCU V3 (4MB) | IR Blaster 主控 |
| IR 接收頭 | HX1838 (VS1838B) | 學習原廠遙控器碼 |
| IR 發射 LED | 940nm 紅外發射管（+限流電阻） | 發射控制信號 |
| OLED | SSD1306 128×64 I2C | 顯示最後指令/狀態 |
| 麵包板 + 杜邦線 | — | 接線 |

---

## 8266 接線（已實測）

| 模組 | 接腳 | 8266 Pin |
|---|---|---|
| OLED VCC | → | 3V3 |
| OLED GND | → | GND |
| OLED SDA | → | D1 (GPIO5) |
| OLED SCL | → | D2 (GPIO4) |
| HX1838 VCC | → | 3V3 |
| HX1838 GND | → | GND |
| HX1838 OUT | → | D5 (GPIO14) |
| IR 發射 VCC | → | 3V3 |
| IR 發射 GND | → | GND |
| IR 發射 S(Data) | → | D6 (GPIO12) |

> 源碼對應：`irremote01/src/main.cpp` 第 38–43 行
>
> ```cpp
> #define OLED_SDA  5   // D1
> #define OLED_SCL  4   // D2
> #define IR_RX_PIN 14  // D5 (HX1838 OUT)
> #define IR_TX_PIN 12  // D6 (IR LED Data)
> #define IR_FREQ   38000
> ```

---

## 8266 固件功能（irremote01）

一個「IR 學習式遙控器」：

1. **內建網頁控制台**：`http://ir.local`（或 IP），按遙控器分組列出按鍵
2. **學習模式**：網頁點「學習」→ 12 秒內按原廠遙控器 → 自動存碼
   - 標準協定（NEC/Sony/RC5/三星/NIKAI…）→ 存「協定+數值」超省空間
   - 非標準/冷氣碼 → RLE 壓縮存原始碼，無損重播
3. **重播**：網頁/API 點按鍵 → D6 發射 → OLED 顯示最後指令+時間
4. **EEPROM 持久化**：支援多遙控器分組、新增/刪除按鍵，可無限擴展

### HTTP API

| 接口 | 說明 | 例子 |
|---|---|---|
| `GET /api/send?group=<組>&name=<按鈕>` | 發射某組某鍵（按鈕名需 URL 編碼） | `/api/send?group=TV&name=%E9%9B%BB%E6%BA%90` |
| `GET /api/status` | 狀態/已載入按鍵 | `/api/status` |
| `GET /api/scan` | 掃描 IR 接收頭（學習用） | `/api/scan` |
| `GET /api/learn?group=<組>&name=<按鈕>` | 進入學習模式（12 秒） | `/api/learn?group=TV&name=HDMI` |

### 預設 TV 按鍵（TCL NIKAI 協定，免學習）

| 按鍵 | 值 |
|---|---|
| 電源 | `0xD5F2A` |
| 頻道+ / 頻道- | `0xD2F2D` / `0xD3F2C` |
| 音量+ / 音量- | `0xD0F2F` / `0xD1F2E` |
| 訊號源(HDMI) | `0x5CFA3` |
| Enter(OK) | `0xBFF4` |
| 上 / 下 / 左 / 右 | `0xA6F59` / `0xA7F58` / `0xA9F56` / `0xA8F57` |

---

## ESP32 固件改造（MCP 工具）

喺小智固件嘅板文件 `main/boards/nologo/xingzhi-cube-1.54tft-wifi/xingzhi-cube-1.54tft-wifi.cc` 註冊一個 MCP 工具 `tv_ir.send`：

```cpp
McpServer::GetInstance().AddTool("tv_ir.send", "向 8266 IR Blaster 發射紅外信號", {
    {"button", "按鈕名（電源/頻道+/頻道-/音量+/音量-/HDMI/Enter/上/下/左/右）", true},
    {"times", "發射次數（預設 1）", false}
}, [](const JsonObject& args) -> String {
    const char* btn = args["button"];
    int times = args["times"] | 1;
    // GET http://<IR_BLASTER_HOST>/api/send?group=TV&name=<btn> ...
    return "{\"ok\":true}";
});
```

之後 AI 會自動學識：用戶講「**幫我熄電視**」→ Server 調用 `tv_ir.send(button="電源", times=1)` → 8266 發射 → 電視關機。

---

## 快速開始

1. **燒 8266**：用 Arduino IDE / PlatformIO 燒 `irremote01`，喺源碼填你嘅 WiFi SSID/密碼
2. **燒 ESP32**：小智固件（ESP-IDF v6.1 編譯，板型 `xingzhi-cube-1.54tft-wifi`），確認 `IR_BLASTER_HOST` 指向你 8266 嘅 IP
3. **配網**：ESP32 開機長按 BOOT 入配網模式 → 連接熱點 `Xiaozhi-XXXX` → 填 WiFi → 等佢連上 Server
4. **驗證**：喚醒小智，講「開電視」→ 睇 8266 OLED 顯示指令、電視有反應

> 提示：建議喺路由器為 8266 綁定靜態 IP（DHCP 分配會變），避免固件入面嘅 `IR_BLASTER_HOST` 失效。

---

## 實測經驗（坑位記錄）

- **發射次數**：連續發 5 次反而令電視「食唔到」信號——**改為 1 次即可穩定控制**
- **IR 發射供電**：發射 LED 用 3V3 已夠（若距離遠可試 5V + 三極管驅動增強）
- **接收頭閃爍 ≠ 發射成功**：接收頭閃只代表「收到環境紅外」，唔代表發射正常
- **TCL 電視**：P7K 系列用 NIKAI 協定已知碼，免學習直接用
- **喚醒詞**：官方固件喚醒詞係列表揀；自訂詞（如「小豬女」）需外接 ASR PRO 模塊，唔影響本方案

---

## 限制與擴展

- **2.4G RF 家電（如 TCL 風扇燈）**：用 2.4G 私有 RF 而唔係紅外，8266 驅唔到 → 建議改用 Tuya WiFi 接收器方案
- **藍牙家電**：ESP32-S3 支持 BLE，但小智官方固件未有 IR+BT 混合控制——需自建 Server 側加 MCP 工具
- **可擴展**：新增任何 IR 家電（冷氣/機頂盒/DVD）→ 學習模式收碼即可；加更多家電 → 加 MCP 工具同 Server 端 function_call 配置
- **語音自由**：自建 Server（xiaozhi-esp32-server）可換任意大模型（Ollama 本地 / DeepSeek / 豆包）、粵語 ASR（FunASR）、自訂克隆聲

---

## 安全提示

- 源碼入面嘅 WiFi 密碼係私人資料，**提交公開 repo 前請改為佔位符**
- 8266 HTTP API 冇鑑權，只建議喺可信家庭網絡使用
- 自建 Server 對外開放時，請開啟 `config.yaml` 嘅 `server.auth`

## License

MIT
