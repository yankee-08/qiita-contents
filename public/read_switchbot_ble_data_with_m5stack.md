---
title: SwitchBot 温湿度計のBLEデータをM5Stackで読み取って画面に表示する
tags:
  - BLE
  - ArduinoIDE
  - M5stack
  - SwitchBot
private: false
updated_at: '2023-06-02T23:43:28+09:00'
id: f1e1fd47a1a3e83501e4
organization_url_name: null
slide: false
ignorePublish: false
---

:::note info
2023/06/02追記
今更ですがバグを発見したため、一部コードを修正しています。
修正箇所には同様のコメントを入れてます。
また、`VSCode`＋`PlatformIO`＋`M5Unified`の構成で動作するプログラムをGitHubで公開しています。
よろしければ、そちらも是非ご活用ください。
:::

https://github.com/yankee-08/M5-Unified-SwitchBot-Meter-BLE

Advent Calendar 2019の最終日が過ぎてまだ間もないですが、もう一つM5Stackに関する記事を投稿します。

# ◇はじめに
本記事では、[SwitchBot 温湿度計](https://www.switchbot.jp/meter)とM5Stack間のBLE通信について紹介します。
**この記事を書いている間に、Beta版ですが、BLEの通信仕様も公開されたため、適宜そちらを参照ください。**

***参考URL***：[OpenWonderLabs/python-host Meter BLE open API](https://github.com/OpenWonderLabs/python-host/wiki/Meter-BLE-open-API)

なお、元々通信仕様が不明の状態で記事を書き始めているため、文章がつながっていない箇所があるかもしれません。
一応、追記部分についてはその旨表記するようにしています。

# ◇SwitchBot 温湿度計🌡とは？
無線機能を内蔵した温湿度計で、専用のアプリを使うことによって、温湿度のトレンドグラフを表示したりすることが可能です。
なお、このブランドでは、ワイヤレスのスイッチロボットSwitchBotや赤外線リモコン機能搭載でAmazonアレクサなどと連携できるSwitchBotハブミニなどもあり、これらをうまく組み合わせると、

- 温湿度計の温度が一定値以下になったら、リモコン機能でエアコンをONする
- スマートスピーカーの音声コントロール機能でSwitchBotを取り付けた部屋の照明スイッチを切る  
といったこともできるようです。

、、実はSwitchBot Hub Miniも手元にあるが、まだ箱に入ったまま・・・

# ◇どうせなら直接データを取りたい
アプリを使えばデータが取れることは分かったんですが、どうせならM5Stack（orラズパイ）で直接データを読み出したい衝動にかられました。
色々調べたところ、ラズパイでワイヤレススイッチロボットSwitchBotを制御するためのサンプルコードがあることが分かったので、多分温湿度計もデータ取れるだろうといった気持ちで挑戦してみました。

***参考URL***：[Github OpenWonderLabs/python-host
](https://github.com/OpenWonderLabs/python-host)

# ◇最終成果物（できたもの）
一応こんなのができました。
※一番重要な項目を載せるの忘れてた・・・。
<blockquote class="twitter-tweet"><p lang="ja" dir="ltr"><a href="https://twitter.com/hashtag/SwitchBot?src=hash&amp;ref_src=twsrc%5Etfw">#SwitchBot</a> 温湿度計データを <a href="https://twitter.com/hashtag/M5Stack?src=hash&amp;ref_src=twsrc%5Etfw">#M5Stack</a> で直接とって表示してみた🌡時刻はWi-FiでNTPサーバから取得。<br>タイミングによって取れたり取れなかったりなんで更新頻度はマチマチだがとりあえずいい感じー<br>公式にAPIが出てるわけじゃないんでファームウェアのバージョン変わったら使えなくなりそうだが･･ <a href="https://t.co/LODqRACPYH">pic.twitter.com/LODqRACPYH</a></p>&mdash; yankee (@yankee_sns) <a href="https://twitter.com/yankee_sns/status/1197491503846674432?ref_src=twsrc%5Etfw">November 21, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

# ◇開発環境等
- OS : Windows 10 Home
- IDE : Arduino IDE 1.8.9 (Windows Store 1.8.21.0)
- ボードマネージャ：Arduino core for the ESP32（バージョン1.0.4）
- ライブラリ：M5Stack Library（バージョン0.2.9）
- デバイス : M5Stack FIRE ＋ SwitchBot 温湿度計
- SwitchBot 温湿度計ファームウェアバージョン：V2.4（SwitchBotアプリを使用して確認）

# ◇実装
##SwitchBot温湿度計の温湿度データ取得
詳細は省きますが、いろいろ試した結果SwitchBot 温湿度計で表示されている温湿度データはBLEのアドバタイジングデータ（ADVERTISEMENT DATA内のService Data）に含まれていそうだということがわかりました。
BLEの通信の詳細については、以下のURLを参考にしました。

***参考URL***：[JELLYWARE 開発視点の超簡単BLE入門](http://jellyware.jp/kurage/bluejelly/ble_guide.html)
***参考URL***：[エンジニアの備忘録 LOW ENERGY : GAP 仕様](https://shizuk.sakura.ne.jp/bluetooth/ble/gap_spec.html)

👉予想どおりService dataにデータを載せていました（通信仕様公開後追記部分）

### ◆SwitchBot温湿度計デバイスの検索
アドバタイジングデータを受け取るには、まずM5Stack側でBLE Scanをしてデバイスを検索する必要があります。
その際、複数のデバイスが検索された場合にどれがSwitchBot温湿度計かわからないため、SwitchBot温湿度計のBDアドレスをあらかじめ調べておき、BDアドレスが一致したデバイスを温湿度計としています。
ソースコードは以下の通りです。
※BDアドレスは自分のデバイスにあったアドレスを記入してください

:::note info
2023/06/02追記
変更点：forループ内の変数を`i`から`iDevNo`に修正。
※その下のforループ内の変数でも`i`を使用していたため
:::

```c++:
const String TargetAddressStr = "xx:xx:xx:xx:xx:xx";
BLEScan *pBLEScan;

---略---

void setup()
{
  ---略---

  BLEDevice::init("hoge");

  pBLEScan = BLEDevice::getScan(); //create new scan
  pBLEScan->setInterval(1000);
  pBLEScan->setWindow(1000);
  pBLEScan->setActiveScan(true); //active scan uses more power, but get results faster

  delay(1000);
}

void loop()
{
  ---略---

  // Serial.println("Scan start!");
  BLEScanResults foundDevices = pBLEScan->start(1, false);
  uint32_t dev_count = foundDevices.getCount(); // 受信したデバイス数を取得
  // Serial.print("Devices found: ");
  // Serial.println(dev_count);

  for (int iDevNo = 0; iDevNo < dev_count; iDevNo++)
  { // 受信したデータに対して
    BLEAdvertisedDevice device = foundDevices.getDevice(iDevNo);
    BLEAddress address = device.getAddress();

    // アドレス一致検索
    if (TargetAddressStr.compareTo(address.toString().c_str()) != 0)
    {
      Serial.print(iDevNo);
      Serial.println(":coninue");
      continue;
    }
    else
    {
      ---略---
    }
}

```

### ◆SERVICE DATAの取得
デバイスが検出できたら、アドバタイジングデータの中身を取得します。
今回は、取得したデータを1バイトごとに区切って、数値に変換した後に配列に格納しています。

:::note info
2023/06/02追記
変更点：forループ内の変数`i`が未定義だったため、`int`の定義を追記
※過去コードでは、その上の変数`i`が使用されており、条件によっては無限ループ状態になっていました
:::

```c++:
  ---略---

      if (device.haveServiceData())
      {
        char *pHexService = BLEUtils::buildHexData(nullptr, (uint8_t *)device.getServiceData().data(), device.getServiceData().length());
        std::string service_data = pHexService;
        // Serial.print("service_data:");
        // Serial.println((service_data.c_str()));

        // 8bitごとに数値に変換する
        String full_str = service_data.c_str();
        String split_str[6] = {"", "", "", "", "", ""};
        uint32_t split_num[6] = {0, 0, 0, 0, 0, 0};
        char *endptr;
        for (int i = 0; i < 6; ++i)
        {
          split_str[i] = full_str.substring(i * 2, (i + 1) * 2);
          // Serial.println(split_str[i]);
          split_num[i] = strtoul(split_str[i].c_str(), &endptr, 16);
        }
        free(pHexService);

        ---略---
      }
      else
      {
        Serial.println("device NOT have ServiceData");
      }

  ---略---
```

### ◆温度、湿度、バッテリー量の算出
次に、取得したSERVICE DATAから温湿度等のデータを算出していきます。
SERVICE DATAの特定のビット範囲でマスクをかけて、その値を数値として変換していますが、この部分についてはあくまで**個人の推測**でマスクビットを指定しています。

なので、今回指定したマスクビットが正確ではないので、注意ください。

👉[こちらのページ](https://github.com/OpenWonderLabs/python-host/wiki/Meter-BLE-open-API#New_Broadcast_Message)を参考にマスクビットを変更しました（通信仕様公開後追記部分）

```c++:
  ---略---
        // 8bitごとに数値に変換する
        String full_str = service_data.c_str();
        String split_str[6] = {"", "", "", "", "", ""};
        uint32_t split_num[6] = {0, 0, 0, 0, 0, 0};
        char *endptr;
        for (int i = 0; i < 6; ++i)
        {
          split_str[i] = full_str.substring(i * 2, (i + 1) * 2);
          // Serial.println(split_str[i]);
          split_num[i] = strtoul(split_str[i].c_str(), &endptr, 16);
        }
        free(pHexService);

        // 温度算出
        const uint32_t mask_temp_sign = 0x80;    //128
        const uint32_t mask_temp_integer = 0x7F; //127
        const uint32_t mask_temp_decimal = 0x0F; //15
        integer_part_temperature = split_num[4] & mask_temp_integer;
        decimal_part_temperature = split_num[3] & mask_temp_decimal;
        temperature = (double)integer_part_temperature + (double)decimal_part_temperature / 10;
        if ((split_num[4] & mask_temp_sign) == 0)
        {
          temperature = temperature * -1;
        }
        Serial.print("温度：");
        Serial.print(integer_part_temperature);
        Serial.print(".");
        Serial.print(decimal_part_temperature);
        Serial.print("[℃] / ");
        Serial.print(temperature);
        Serial.println("[℃] / ");

        // 湿度算出
        const uint32_t mask_humi = 0x7F; //127
        humidity = split_num[5] & mask_humi;
        Serial.print("湿度：");
        Serial.print(humidity);
        Serial.print("[%RH] / ");

        // 電池残量
        const uint32_t mask_battery = 0x7F; //127
        battery_level = split_num[2] & mask_battery;
        Serial.print("バッテリー残量：");
        Serial.print(battery_level);
        Serial.println("[%]");
        Serial.flush();
  ---略---
```

### ◆温度、湿度の画面表示（数値に応じた文字色切り替え）
温度・湿度を算出したら、それらの数値をM5StackのLCDに表示します。このとき、現在温度、現在湿度に応じて文字色をグラデーションで切り替えています。

`calc_color()`関数内でRedとBlueの配分を決めており、
温度が上がるにつれて、青色～赤色の文字色
湿度が上がるについて、土色～青色の文字色
に変わっていきます。

温度や湿度の数字を描画する際に読んでいる`drawNumberNormal()`関数については、前回作成したアプリのコードを流用してるなので詳細は割愛します。
必要に応じて前回の記事を確認ください。

前回記事：[M5Stack FIREを使ってIKEA風クロックを作ってみる](https://qiita.com/yankee/items/1591724d8a2722951c7e)

なお、今回はバッテリー残量については画面表示から外しています。

```c++:
int32_t calc_color(int32_t lo_temp, int32_t hi_temp, int32_t lo_temp_color, int32_t hi_temp_color, int32_t now_temp)
{
  if (now_temp > hi_temp)
  {
    now_temp = hi_temp;
  }
  else if (now_temp < lo_temp)
  {
    now_temp = lo_temp;
  }

  int32_t color = 0;
  if (lo_temp != hi_temp)
  {
    double y = (double)(now_temp - lo_temp) * double(hi_temp_color - lo_temp_color) / double(hi_temp - lo_temp) + lo_temp_color;
    color = round(y);
  }
  else
  {
    color = lo_temp_color;
  }

  return color;
}

void loop()
{
  ---略---

  // 温度表示
  int32_t temp_red_value = calc_color(0, 30, 100, 250, (int)temperature);
  // Serial.print("RED[T]:");
  // Serial.println(temp_red_value);
  int32_t temp_blue_value = calc_color(0, 30, 250, 50, (int)temperature);
  // Serial.print("BLUE[T]:");
  // Serial.println(temp_blue_value);
  int32_t temp_green_value = 50;

  uint16_t temperature_font_color = M5.Lcd.color565(temp_red_value, temp_green_value, temp_blue_value);
  // 整数部
  drawNumberNormal(35, 20, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  drawNumberNormal(35 + 65, 20, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  if ((integer_part_temperature / 10) != 0)
  {
    drawNumberNormal(35, 20, (integer_part_temperature / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, temperature_font_color);
  }
  drawNumberNormal(35 + 65, 20, (integer_part_temperature % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, temperature_font_color);
  M5.Lcd.setTextColor(temperature_font_color);
  M5.Lcd.setTextSize(7);
  M5.Lcd.drawString(".", 135, 40);
  // 小数部
  drawNumberNormal(35 + 140, 20, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  drawNumberNormal(35 + 140, 20, decimal_part_temperature, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, temperature_font_color);
  M5.Lcd.setTextSize(6);
  M5.Lcd.drawString("C", 240, 50);
  M5.Lcd.fillEllipse(230, 55, 5, 5, temperature_font_color);

  M5.Lcd.fillRoundRect(10, 116, 300, 8, 4, TFT_WHITE);

  // 温度表示
  int32_t temp_red_value = calc_color(0, 30, 100, 250, (int)temperature);
  // Serial.print("RED[T]:");
  // Serial.println(temp_red_value);
  int32_t temp_blue_value = calc_color(0, 30, 250, 50, (int)temperature);
  // Serial.print("BLUE[T]:");
  // Serial.println(temp_blue_value);
  int32_t temp_green_value = 50;

  uint16_t temperature_font_color = M5.Lcd.color565(temp_red_value, temp_green_value, temp_blue_value);

  // 符号
  M5.Lcd.setTextColor(bkground_color);
  M5.Lcd.setTextSize(5);
  M5.Lcd.drawString("-", 10, 35);
  if (temperature < 0)
  {
    M5.Lcd.setTextColor(temperature_font_color);
    M5.Lcd.drawString("-", 10, 35);
  }
  // 整数部
  drawNumberNormal(55, 20, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  drawNumberNormal(55 + 65, 20, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  if ((integer_part_temperature / 10) != 0)
  {
    drawNumberNormal(55, 20, (integer_part_temperature / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, temperature_font_color);
  }
  drawNumberNormal(55 + 65, 20, (integer_part_temperature % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, temperature_font_color);
  M5.Lcd.setTextColor(temperature_font_color);
  M5.Lcd.setTextSize(7);
  M5.Lcd.drawString(".", 155, 40);
  // 小数部
  drawNumberNormal(55 + 140, 20, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  drawNumberNormal(55 + 140, 20, decimal_part_temperature, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, temperature_font_color);
  M5.Lcd.setTextSize(6);
  M5.Lcd.drawString("C", 260, 50);
  M5.Lcd.fillEllipse(250, 55, 5, 5, temperature_font_color);

  M5.Lcd.fillRoundRect(10, 116, 300, 8, 4, TFT_WHITE);

  // 湿度表示
  int32_t humi_red_value = calc_color(30, 100, 160, 100, humidity);
  // Serial.print("RED[H]:");
  // Serial.println(humi_red_value);
  int32_t humi_blue_value = calc_color(30, 100, 60, 240, humidity);
  // Serial.print("BLUE[H]:");
  // Serial.println(humi_blue_value);
  int32_t humi_green_value = 120;

  uint16_t humidity_font_color = M5.Lcd.color565(humi_red_value, humi_green_value, humi_blue_value);

  diff_milli_time = millis() - base_milli_time;
  elasped_second = diff_milli_time / 1000;
  drawNumberNormal(140, 150, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  drawNumberNormal(140 + 65, 150, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  if ((humidity / 10) != 0)
  {
    drawNumberNormal(140, 150, (humidity / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, humidity_font_color);
  }
  drawNumberNormal(140 + 65, 150, (humidity % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, humidity_font_color);
  M5.Lcd.setTextColor(humidity_font_color);
  M5.Lcd.setTextSize(6);
  M5.Lcd.drawString("%", 260, 180);

  ---略---
}
```

## 実装まとめ
表示の際、温湿度に加えて、現在日時も表示していますが、この部分も前回作成したアプリのコードを流用してるので詳細は割愛します。
必要に応じて前回の記事を確認ください。

前回記事：[M5Stack FIREを使ってIKEA風クロックを作ってみる](https://qiita.com/yankee/items/1591724d8a2722951c7e)

なお、今回のアプリでは、SwitchBot温湿度計の温度表示はセ氏(°C)表示の場合前提で作っていますので、その点注意ください。

<details><summary>ここまででひととおりアプリの説明ができたので、以下にソースコード全体を記載します。
※折畳みにしてあります</summary><div>


```c++:
#include <stdlib.h>
// #include <Math.h>
#include <M5Stack.h>
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEScan.h>
#include <BLEAdvertisedDevice.h>

#include <WiFi.h>
#include "utility/MPU9250.h"
#include "utility/M5Timer.h"
#include "time.h"

/*
/ 変数宣言
*/
static uint32_t iloopInterval = 5000;

// Wi-Fi
const char *ssid = "xxxxxx";
const char *password = "xxxxxx";

// NTP
const char *ntp_server_1st = "ntp.nict.jp";
const char *ntp_server_2nd = "time.google.com";
const long gmt_offset_sec = 9 * 3600; // 時差（秒換算）
const int daylight_offset_sec = 0;    // 夏時間

// The characteristic of the remote service we are interested in.
const String TargetAddressStr = "xx:xx:xx:xx:xx:xx";
BLEScan *pBLEScan;

boolean is_state_changed = true;

// LCD
#define LCD_LARGE_BAR_WIDTH (10)
#define LCD_LARGE_BAR_LENGTH (30)
#define LCD_LARGE_BAR_CORNER_RADIUS (6)
#define LCD_LARGE_BAR_GAP (LCD_LARGE_BAR_WIDTH >> 1)

#define LCD_SMALL_BAR_WIDTH (4)
#define LCD_SMALL_BAR_LENGTH (12)
#define LCD_SMALL_BAR_CORNER_RADIUS (3)
#define LCD_SMALL_BAR_GAP (LCD_SMALL_BAR_WIDTH >> 1)

#define LCD_DIGITS_CLEAR_ELM_NO (8)

const uint8_t digits_normal[] =
    {
        0b00111111, // 0
        0b00110000, // 1
        0b01101101, // 2
        0b01111001, // 3
        0b01110010, // 4
        0b01011011, // 5
        0b01011111, // 6
        0b00110011, // 7
        0b01111111, // 8
        0b01111011, // 9
        0b00000000, // off
};

void drawNumberNormal(uint8_t x_start, uint8_t y_start, uint8_t number, uint8_t bar_width, uint8_t bar_length, uint8_t bar_gap, uint8_t corner_radius, uint16_t color_value)
{
  if (number > 10)
  {
    number = 10;
  }
  // top
  if (digits_normal[number] & 0b0000000000000001)
    M5.Lcd.fillRoundRect(x_start, y_start, bar_length, bar_width, corner_radius, color_value);
  // upper-left
  if (digits_normal[number] & 0b0000000000000010)
    M5.Lcd.fillRoundRect((x_start - bar_gap * 2), (y_start + bar_gap), bar_width, bar_length, corner_radius, color_value);
  // under-left
  if (digits_normal[number] & 0b0000000000000100)
    M5.Lcd.fillRoundRect((x_start - bar_gap * 2), (y_start + bar_gap + bar_length * 1), bar_width, bar_length, corner_radius, color_value);
  // bottom
  if (digits_normal[number] & 0b0000000000001000)
    M5.Lcd.fillRoundRect(x_start, (y_start + bar_length * 2), bar_length, bar_width, corner_radius, color_value);
  // under-right
  if (digits_normal[number] & 0b0000000000010000)
    M5.Lcd.fillRoundRect((x_start + bar_length), (y_start + bar_gap + bar_length * 1), bar_width, bar_length, corner_radius, color_value);
  // upper-right
  if (digits_normal[number] & 0b0000000000100000)
    M5.Lcd.fillRoundRect((x_start + bar_length), (y_start + bar_gap), bar_width, bar_length, corner_radius, color_value);
  // center
  if (digits_normal[number] & 0b0000000001000000)
    M5.Lcd.fillRoundRect(x_start, (y_start + bar_length * 1), bar_length, bar_width, corner_radius, color_value);
}

// Blink
#define LCD_DISP_BLINK_LOOP_CNT (1)
class BlinkCount
{
private:
  uint32_t count;

public:
  void incrementCount(void);
  void resetCount(void);
  boolean isHideDisplay(void);
};

void BlinkCount::incrementCount(void)
{
  ++count;
}

void BlinkCount::resetCount(void)
{
  count = 0;
}

boolean BlinkCount::isHideDisplay(void)
{
  if ((count / LCD_DISP_BLINK_LOOP_CNT % 2) == 0)
  {
    return false;
  }
  else
  {
    return true;
  }
}

BlinkCount cl_blink_count;

// System Clock
class SystemClock
{
private:
  struct tm time_info;
  time_t timer;

public:
  uint32_t year = 0;
  uint32_t month = 0;
  uint32_t day = 0;
  uint32_t hour = 0;
  uint32_t minute = 0;
  uint32_t week_day = 0;
  uint32_t second = 0;
  uint32_t prev_year = 0;
  uint32_t prev_month = 0;
  uint32_t prev_day = 0;
  uint32_t prev_hour = 0;
  uint32_t prev_minute = 0;
  uint32_t prev_week_day = 0;
  uint32_t prev_second = 0;
  void backupCurrentTime(void);
  void updateByNtp(void);
  void updateBySoftTimer(uint32_t elasped_second);
};

void SystemClock::backupCurrentTime(void)
{
  prev_year = year;
  prev_month = month;
  prev_day = day;
  prev_hour = hour;
  prev_minute = minute;
  prev_week_day = week_day;
  prev_second = second;
}

void SystemClock::updateBySoftTimer(uint32_t elasped_second)
{
  struct tm *local_time;
  time_t timer_add = timer + elasped_second;
  local_time = localtime(&timer_add);
  year = local_time->tm_year + 1900;
  month = local_time->tm_mon + 1;
  day = local_time->tm_mday;
  hour = local_time->tm_hour;
  minute = local_time->tm_min;
  week_day = local_time->tm_wday;
  second = local_time->tm_sec;
}

void SystemClock::updateByNtp(void)
{
  Serial.println("---NTP ACCESS---");
  if (!getLocalTime(&time_info))
  {
    year = 0;
    month = 0;
    day = 0;
    hour = 0;
    minute = 0;
    week_day = 0;
    second = 0;
    timer = 0;
  }
  else
  {
    year = time_info.tm_year + 1900;
    month = time_info.tm_mon + 1;
    day = time_info.tm_mday;
    hour = time_info.tm_hour;
    minute = time_info.tm_min;
    week_day = time_info.tm_wday;
    second = time_info.tm_sec;
    timer = mktime(&time_info);
  }
}

SystemClock cl_system_clock;
#define NTP_ACCESS_MS_INTERVAL (300000)

#define LCD_CLOCK_YMD_DISP_Y_POS (10)
#define LCD_CLOCK_MD_STR_DISP_Y_POS (45)
#define LCD_CLOCK_HM_DISP_Y_POS (100)
#define LCD_CLOCK_PM_STR_DISP_Y_POS (175)
#define LCD_CLOCK_ICON_DISP_Y_POS (200)
#define LCD_CLOCK_WEEK_STR_DISP_Y_POS (220)

int32_t calc_color(int32_t lo_temp, int32_t hi_temp, int32_t lo_temp_color, int32_t hi_temp_color, int32_t now_temp)
{
  if (now_temp > hi_temp)
  {
    now_temp = hi_temp;
  }
  else if (now_temp < lo_temp)
  {
    now_temp = lo_temp;
  }

  int32_t color = 0;
  if (lo_temp != hi_temp)
  {
    double y = (double)(now_temp - lo_temp) * double(hi_temp_color - lo_temp_color) / double(hi_temp - lo_temp) + lo_temp_color;
    color = round(y);
  }
  else
  {
    color = lo_temp_color;
  }

  return color;
}

void setup()
{
  M5.begin();
  Wire.begin();
  M5.Lcd.setBrightness(100);

  WiFi.mode(WIFI_MODE_STA);
  WiFi.begin(ssid, password);
  while (WiFi.waitForConnectResult() != WL_CONNECTED)
  {
    delay(100);
  }

  M5.Lcd.fillScreen(TFT_WHITE);
  cl_blink_count.resetCount();
  delay(1000);

  // init time setting
  configTime(gmt_offset_sec, daylight_offset_sec, ntp_server_1st, ntp_server_2nd);
  struct tm time_info;
  if (!getLocalTime(&time_info))
  {
    M5.Lcd.fillScreen(TFT_RED);
    delay(3000);
  }
  WiFi.disconnect(true);

  Serial.println("Hello world");
  Serial.println("Starting BLE work!");

  BLEDevice::init("hoge");

  pBLEScan = BLEDevice::getScan(); //create new scan
  pBLEScan->setInterval(1000);
  pBLEScan->setWindow(1000);
  pBLEScan->setActiveScan(true); //active scan uses more power, but get results faster

  delay(1000);
}

#define LCD_TEMPERATURE_DISP_X_POS (35)
#define LCD_TEMPERATURE_DISP_Y_POS (20)
#define LCD_CLOCK_MD_STR_DISP_Y_POS (160)
#define LCD_CLOCK_HM_DISP_Y_POS (200)

void loop()
{
  static boolean is_state_changed = true;
  static boolean ntp_access_flag = true;
  static uint32_t base_milli_time;
  uint32_t elasped_second = 0;
  uint32_t diff_milli_time = 0;

  static int32_t integer_part_temperature = 0;
  static int32_t decimal_part_temperature = 0;
  static double temperature = 0.0;
  static uint32_t humidity = 0;
  static uint32_t battery_level = 0;
  delay(500);
  M5.update();

  // Serial.println("Scan start!");
  BLEScanResults foundDevices = pBLEScan->start(1, false);
  uint32_t dev_count = foundDevices.getCount(); // 受信したデバイス数を取得
  // Serial.print("Devices found: ");
  // Serial.println(dev_count);

  for (int iDevNo = 0; iDevNo < dev_count; iDevNo++)
  { // 受信したデータに対して
    BLEAdvertisedDevice device = foundDevices.getDevice(iDevNo);
    BLEAddress address = device.getAddress();
    Serial.println(device.toString().c_str());
    Serial.println(address.toString().c_str());

    // アドレス一致検索
    if (TargetAddressStr.compareTo(address.toString().c_str()) != 0)
    {
      Serial.print(iDevNo);
      Serial.println(":coninue");
      continue;
    }
    else
    {
      if (device.haveServiceData())
      {
        char *pHexService = BLEUtils::buildHexData(nullptr, (uint8_t *)device.getServiceData().data(), device.getServiceData().length());
        std::string service_data = pHexService;
        // Serial.print("service_data:");
        // Serial.println((service_data.c_str()));

        // 8bitごとに数値に変換する
        String full_str = service_data.c_str();
        String split_str[6] = {"", "", "", "", "", ""};
        uint32_t split_num[6] = {0, 0, 0, 0, 0, 0};
        char *endptr;
        for (int i = 0; i < 6; ++i)
        {
          split_str[i] = full_str.substring(i * 2, (i + 1) * 2);
          // Serial.println(split_str[i]);
          split_num[i] = strtoul(split_str[i].c_str(), &endptr, 16);
        }
        free(pHexService);

        // 温度算出
        const uint32_t mask_temp_sign = 0x80;    //128
        const uint32_t mask_temp_integer = 0x7F; //127
        const uint32_t mask_temp_decimal = 0x0F; //15
        integer_part_temperature = split_num[4] & mask_temp_integer;
        decimal_part_temperature = split_num[3] & mask_temp_decimal;
        temperature = (double)integer_part_temperature + (double)decimal_part_temperature / 10;
        if ((split_num[4] & mask_temp_sign) == 0)
        {
          temperature = temperature * -1;
        }
        Serial.print("温度：");
        Serial.print(integer_part_temperature);
        Serial.print(".");
        Serial.print(decimal_part_temperature);
        Serial.print("[℃] / ");
        Serial.print(temperature);
        Serial.println("[℃] / ");

        // 湿度算出
        const uint32_t mask_humi = 0x7F; //127
        humidity = split_num[5] & mask_humi;
        Serial.print("湿度：");
        Serial.print(humidity);
        Serial.print("[%RH] / ");

        // 電池残量
        const uint32_t mask_battery = 0x7F; //127
        battery_level = split_num[2] & mask_battery;
        Serial.print("バッテリー残量：");
        Serial.print(battery_level);
        Serial.println("[%]");
        Serial.flush();
      }
      else
      {
        Serial.println("device NOT have ServiceData");
      }
    }
  }
  Serial.println("Scan done!");
  pBLEScan->clearResults(); // delete results fromBLEScan buffer to release memory

  // 時刻取得
  WiFi.mode(WIFI_MODE_STA);
  WiFi.begin(ssid, password);
  while (WiFi.waitForConnectResult() != WL_CONNECTED)
  {
    delay(100);
  }
  if (ntp_access_flag == true)
  {
    base_milli_time = millis();
    Serial.print("base_milli_time:");
    Serial.println(base_milli_time);
    cl_system_clock.updateByNtp();
    ntp_access_flag = false;
  }
  else
  {
    diff_milli_time = millis() - base_milli_time;
    if (diff_milli_time > NTP_ACCESS_MS_INTERVAL)
    {
      ntp_access_flag = true;
    }
    elasped_second = diff_milli_time / 1000;
    cl_system_clock.updateBySoftTimer(elasped_second);
  }
  WiFi.disconnect(true);

  uint16_t bkground_color = M5.Lcd.color565(200, 200, 200);
  if (is_state_changed == true)
  {
    M5.Lcd.fillScreen(bkground_color);
    is_state_changed = false;
  }

  // 温度表示
  int32_t temp_red_value = calc_color(0, 30, 100, 250, (int)temperature);
  // Serial.print("RED[T]:");
  // Serial.println(temp_red_value);
  int32_t temp_blue_value = calc_color(0, 30, 250, 50, (int)temperature);
  // Serial.print("BLUE[T]:");
  // Serial.println(temp_blue_value);
  int32_t temp_green_value = 50;

  uint16_t temperature_font_color = M5.Lcd.color565(temp_red_value, temp_green_value, temp_blue_value);

  // 符号
  M5.Lcd.setTextColor(bkground_color);
  M5.Lcd.setTextSize(5);
  M5.Lcd.drawString("-", 10, 35);
  if (temperature < 0)
  {
    M5.Lcd.setTextColor(temperature_font_color);
    M5.Lcd.drawString("-", 10, 35);
  }
  // 整数部
  drawNumberNormal(55, 20, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  drawNumberNormal(55 + 65, 20, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  if ((integer_part_temperature / 10) != 0)
  {
    drawNumberNormal(55, 20, (integer_part_temperature / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, temperature_font_color);
  }
  drawNumberNormal(55 + 65, 20, (integer_part_temperature % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, temperature_font_color);
  M5.Lcd.setTextColor(temperature_font_color);
  M5.Lcd.setTextSize(7);
  M5.Lcd.drawString(".", 155, 40);
  // 小数部
  drawNumberNormal(55 + 140, 20, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  drawNumberNormal(55 + 140, 20, decimal_part_temperature, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, temperature_font_color);
  M5.Lcd.setTextSize(6);
  M5.Lcd.drawString("C", 260, 50);
  M5.Lcd.fillEllipse(250, 55, 5, 5, temperature_font_color);

  M5.Lcd.fillRoundRect(10, 116, 300, 8, 4, TFT_WHITE);

  // 湿度表示
  int32_t humi_red_value = calc_color(30, 100, 160, 100, humidity);
  // Serial.print("RED[H]:");
  // Serial.println(humi_red_value);
  int32_t humi_blue_value = calc_color(30, 100, 60, 240, humidity);
  // Serial.print("BLUE[H]:");
  // Serial.println(humi_blue_value);
  int32_t humi_green_value = 120;

  uint16_t humidity_font_color = M5.Lcd.color565(humi_red_value, humi_green_value, humi_blue_value);

  diff_milli_time = millis() - base_milli_time;
  elasped_second = diff_milli_time / 1000;
  drawNumberNormal(140, 150, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  drawNumberNormal(140 + 65, 150, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  if ((humidity / 10) != 0)
  {
    drawNumberNormal(140, 150, (humidity / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, humidity_font_color);
  }
  drawNumberNormal(140 + 65, 150, (humidity % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, humidity_font_color);
  M5.Lcd.setTextColor(humidity_font_color);
  M5.Lcd.setTextSize(6);
  M5.Lcd.drawString("%", 260, 180);

  // 時刻表示
  // Month
  if (cl_system_clock.month != cl_system_clock.prev_month)
  {
    drawNumberNormal(10, LCD_CLOCK_MD_STR_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(35, LCD_CLOCK_MD_STR_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(10, LCD_CLOCK_MD_STR_DISP_Y_POS, (cl_system_clock.month / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_DARKGREY);
  drawNumberNormal(35, LCD_CLOCK_MD_STR_DISP_Y_POS, (cl_system_clock.month % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_DARKGREY);
  M5.Lcd.drawLine(55, LCD_SMALL_BAR_LENGTH * 2 + LCD_CLOCK_MD_STR_DISP_Y_POS, 65, LCD_CLOCK_MD_STR_DISP_Y_POS, TFT_OLIVE);
  // Day
  if (cl_system_clock.day != cl_system_clock.prev_day)
  {
    drawNumberNormal(75, LCD_CLOCK_MD_STR_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(100, LCD_CLOCK_MD_STR_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(75, LCD_CLOCK_MD_STR_DISP_Y_POS, (cl_system_clock.day / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_DARKGREY);
  drawNumberNormal(100, LCD_CLOCK_MD_STR_DISP_Y_POS, (cl_system_clock.day % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_DARKGREY);
  // Hour
  if (cl_system_clock.hour != cl_system_clock.prev_hour)
  {
    drawNumberNormal(10, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(35, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(10, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.hour / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_DARKGREY);
  drawNumberNormal(35, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.hour % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_DARKGREY);
  // Sec
  if (cl_system_clock.minute != cl_system_clock.prev_minute)
  {
    drawNumberNormal(75, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(100, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(75, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.minute / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_DARKGREY);
  drawNumberNormal(100, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.minute % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_DARKGREY);
  if (cl_blink_count.isHideDisplay() == false)
  {
    M5.Lcd.fillEllipse(60, LCD_CLOCK_HM_DISP_Y_POS + 8, 2, 2, TFT_DARKGREY);
    M5.Lcd.fillEllipse(60, LCD_CLOCK_HM_DISP_Y_POS + 20, 2, 2, TFT_DARKGREY);
  }
  else
  {
    M5.Lcd.fillEllipse(60, LCD_CLOCK_HM_DISP_Y_POS + 8, 2, 2, bkground_color);
    M5.Lcd.fillEllipse(60, LCD_CLOCK_HM_DISP_Y_POS + 20, 2, 2, bkground_color);
  }

  cl_blink_count.incrementCount();
  // delay(3000);
}

```
</div></details>

ソースコード中のssidやBDアドレスは各自の設定に合わせてください。


# ◇おわりに
当初、新年一発目の記事として予定していましたが、SwitchBot温湿度計のBLEの通信仕様も公開されたことから記事の鮮度的にも早めに書いたほうがいいかなと思い、急ぎ仕上げました。
コーディングミスもあるかもしれませんが、それについては後日修正できればと思います。
今年最後の投稿になると思いますが、また来年もいくつか記事をかけたらいーなと思います。
