---
title: M5Stack FIREを使ってIKEA風クロックを作ってみる
tags:
  - 時計
  - ArduinoIDE
  - ikea
  - M5stack
private: false
updated_at: '2023-04-08T21:44:50+09:00'
id: 1591724d8a2722951c7e
organization_url_name: null
slide: false
---

[M5Stack Advent Calendar 2019](https://qiita.com/advent-calendar/2019/m5stack) 8日目の記事です。
今日は、M5Stack社の製品のひとつである、**M5Stack FIRE**を使ったアプリについて紹介していきます。
かなり長めの記事になってしまいましたので、興が乗ったら読んでいただければ幸いです。
最終的なソースは最後の実装まとめに載せていますので、ソースだけ見たいという方はそちらをご覧ください。

# ◇はじめに
本記事では、先日購入したM5Stack FIREを使用して、作成したアプリケーションについて掲載しています。
当初、どんなアプリを作ろうと考えていましたが、

- IKEA社で販売している【[KLOCKIS クロッキス](https://www.ikea.com/jp/ja/catalog/products/00384828/)】と見た目がそっくり
- M5Stack FIREは3軸加速度センサを内蔵しているため、似たようなモノが作れそう（詳細は後述）  
ことから今回は、**KLOCKIS クロッキス**の挙動を確認しつつ、できるだけ近しいモノ（IKEA風クロック）を作ることにしました。

# ◇とりあえず最終成果物（できたもの）
まず、最終的にどんなもんができるのかを書いておきます。
時計の向きを変えると、機能が切り替わっていきます。
<blockquote class="twitter-tweet"><p lang="ja" dir="ltr"><a href="https://twitter.com/hashtag/Qiita?src=hash&amp;ref_src=twsrc%5Etfw">#Qiita</a> のM5Stack Advent Calendar記事投稿用<br>GIF化するよりこっちの方が動画埋め込むのらくそう。 <a href="https://t.co/l8MRaAz8kl">pic.twitter.com/l8MRaAz8kl</a></p>&mdash; yankee (@yankee_sns) <a href="https://twitter.com/yankee_sns/status/1201125638435823621?ref_src=twsrc%5Etfw">December 1, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 


なお、注意点として、

- バッテリーのもちは基本考慮しない。
-  温度表示は加速度センサに内蔵の温度センサの値を使用しており、外気温ではない  
ことをあらかじめご了承ください。

# ◇開発環境等
- OS : Windows 10 Home（バージョン1809）
- IDE : Arduino IDE 1.8.9 (Windows Store 1.8.21.0)
- ボードマネージャ：Arduino core for the ESP32（バージョン1.0.4）
- ライブラリ：M5Stack Library（バージョン0.2.9）
- デバイス : M5Stack FIRE  
- 内蔵加速度センサ：MPU9250

# ◇実装内容
### ㊟ 記事の記載順

以降、実際にコーディングした内容を機能ごとに記載していきますが、

- 今回参考にした、**KLOCKIS クロッキス**の挙動を調査した内容
- 調査した内容をベースにM5Stackに実装した内容  
の順に説明していきます。


## ①傾きに応じたモード切り替え
### **KLOCKIS クロッキス**の挙動チェック
>以下、IKEA社HPのKLOCKIS クロッキス商品説明からの引用です。
>>時刻/日付、 アラーム、温度、タイマーの4つの機能が、時計の向きを変えるだけで表示されます。場所を取らず、使い方も簡単。半分寝起きの頭にも分かりやすい表示です


上記[IKEAサイト](https://www.ikea.com/jp/ja/catalog/products/00384828/)にも記載があるように、**KLOCKIS クロッキス**は、時計の向きによって、機能が切り替わり画面表示も変わる仕様となっています。
また、実際に触ってみたところ、

- 時計を傾けていき、時計内部でカチッと音がなった（傾斜スイッチと思われる）直後に切り替わりが発生する
- 複数の傾斜スイッチが内蔵されていて、それぞれの機能切り替えタイミングの判断に用いる傾斜スイッチを変えることによって、境界値付近で切り替えが頻発することを防止している

ということがわかったため、この仕様をM5Stackでも実現していきます。

### M5Stackへの実装

M5Stack FIREでは傾斜スイッチはありませんが、代わりに3軸加速度センサを内蔵しているため、これをつかって時計の傾きを算出することが可能です。
そこで、まず傾斜角算出のための関数`getAngle()`を作成します。
加速度取得部分はスケッチ例の`M5StackFire_MPU9250.ino`ファイルを、
傾斜角算出部分は、以下のサイトを参考にしています。

_参考URL_：[加速度センサーから軸廻り角度への変換計算](https://garchiving.com/angle-from-acceleration/)
_参考URL_：[Arduino Reference math.h](https://www.arduino.cc/en/Reference/MathHeader)

なお、

- M5Stack FIREの加速度センサは何度かモデルチェンジされており、今回はMPU9250内蔵モデルを使用している
- （MPU9250以外の加速度センサでは不明だが）[以前の記事](https://qiita.com/yankee/items/35542dcf095317ac4d7e)で書いたように、起動時のおき方によっては加速度センサの初期化がうまくいかない  
といったことに注意してください。

```C++:clock.ino
// Acceleration sensor
MPU9250 IMU;

int32_t getAngle()
{
  int32_t angle = 0;
  if (IMU.readByte(MPU9250_ADDRESS, INT_STATUS) & 0x01)
  {
    IMU.readAccelData(IMU.accelCount);
    IMU.getAres();

    IMU.ax = (float)IMU.accelCount[0] * IMU.aRes; // - accelBias[0];
    IMU.ay = (float)IMU.accelCount[1] * IMU.aRes; // - accelBias[1];
    IMU.az = (float)IMU.accelCount[2] * IMU.aRes; // - accelBias[2];

    // 傾斜角算出
    double angle_XY_direction = 0.0;
    angle_XY_direction = atan2(IMU.ay, IMU.ax);
    double angle_XY_direction_deg = angle_XY_direction * 180.0 / (M_PI);
    angle = round(angle_XY_direction_deg) + 180 + 90;
    if (angle > 360)
    {
      angle -= 360;
    }
  }
  else
  {
    angle = -1;
  }
  return angle;
}
```

これで、M5Stackの傾きがとれるようになったので、つづいて取得した傾きに応じて機能を切り替える部分を実装していきます。
基本的には、それぞれの基準の角度±αの範囲内になったら、機能を切り替える処理になりますが、境界値付近で何度も切り替わりが発生するのを防ぐためにヒステリシスを持たせるような仕組みとしています。
イメージを下図に示しますが、M5Stack FIREが傾いて機能が変わったあとに元の機能に戻りくくなるようにしています。


![010.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/0d9e6639-04f2-1f17-99d3-99d621cf96cb.png)

その後、現在の機能に応じた関数を`switch-case`文で呼び出しています。
なお、画面の向きを変えた場合にスクリーン表示も追従するように、`M5.Lcd.setRotation()`関数を呼び出しています。
※`displayXXXScreen()`関数の詳細についてはそれぞれ後述します
実装したソースコードは以下のとおり。

```C++:clock.ino
boolean is_state_changed = true;
typedef enum
{
  APP_STATE_INDEFINITE = 0,
  APP_STATE_DATE_TIME_CLOCK = 1,
  APP_STATE_ALARM_CLOCK = 2,
  APP_STATE_COUNT_DOWN_TIMER = 3,
  APP_STATE_THERMOMETER = 4,
} app_state_e;

---略---

#define DATE_TIME_CLOCK_REF_ANGLE (0)
#define ALARM_CLOCK_REF_ANGLE (270)
#define COUNT_DOWN_TIMER_REF_ANGLE (180)
#define THERMOMETER_REF_ANGLE (90)

#define SWITCH_APP_ANGLE_RANGE (35)
#define CURRENT_APP_ANGLE_RANGE (75)

app_state_e calcCurrentAppState(int32_t angle)
{
  app_state_e e_current_state = APP_STATE_INDEFINITE;
  static app_state_e e_prev_state = APP_STATE_DATE_TIME_CLOCK;

  switch (e_prev_state)
  {
  case APP_STATE_INDEFINITE:
  case APP_STATE_DATE_TIME_CLOCK:
  default:
    if (((DATE_TIME_CLOCK_REF_ANGLE + CURRENT_APP_ANGLE_RANGE) <= angle) && (angle < (COUNT_DOWN_TIMER_REF_ANGLE - SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_ALARM_CLOCK;
      is_state_changed = true;
    }
    else if (((COUNT_DOWN_TIMER_REF_ANGLE - SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (COUNT_DOWN_TIMER_REF_ANGLE + SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_COUNT_DOWN_TIMER;
      is_state_changed = true;
    }
    else if (((COUNT_DOWN_TIMER_REF_ANGLE + SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (360 - CURRENT_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_THERMOMETER;
      is_state_changed = true;
    }
    else
    {
      e_current_state = APP_STATE_DATE_TIME_CLOCK;
    }
    break;

  case APP_STATE_THERMOMETER:
    if (((ALARM_CLOCK_REF_ANGLE + CURRENT_APP_ANGLE_RANGE) <= angle) || (angle < (THERMOMETER_REF_ANGLE - SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_DATE_TIME_CLOCK;
      is_state_changed = true;
    }
    else if (((THERMOMETER_REF_ANGLE - SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (THERMOMETER_REF_ANGLE + SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_ALARM_CLOCK;
      is_state_changed = true;
    }
    else if (((THERMOMETER_REF_ANGLE + SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (ALARM_CLOCK_REF_ANGLE - CURRENT_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_COUNT_DOWN_TIMER;
      is_state_changed = true;
    }
    else
    {
      e_current_state = APP_STATE_THERMOMETER;
    }
    break;

  case APP_STATE_COUNT_DOWN_TIMER:
    if (((COUNT_DOWN_TIMER_REF_ANGLE + CURRENT_APP_ANGLE_RANGE) <= angle) && (angle < (360 - SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_THERMOMETER;
      is_state_changed = true;
    }
    else if (((360 - SWITCH_APP_ANGLE_RANGE) <= angle) || (angle < (DATE_TIME_CLOCK_REF_ANGLE + SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_DATE_TIME_CLOCK;
      is_state_changed = true;
    }
    else if (((DATE_TIME_CLOCK_REF_ANGLE + SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (COUNT_DOWN_TIMER_REF_ANGLE - CURRENT_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_ALARM_CLOCK;
      is_state_changed = true;
    }
    else
    {
      e_current_state = APP_STATE_COUNT_DOWN_TIMER;
    }
    break;

  case APP_STATE_ALARM_CLOCK:
    if (((THERMOMETER_REF_ANGLE + CURRENT_APP_ANGLE_RANGE) <= angle) && (angle < (ALARM_CLOCK_REF_ANGLE - SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_COUNT_DOWN_TIMER;
      is_state_changed = true;
    }
    else if (((ALARM_CLOCK_REF_ANGLE - SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (ALARM_CLOCK_REF_ANGLE + SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_THERMOMETER;
      is_state_changed = true;
    }
    else if (((ALARM_CLOCK_REF_ANGLE + SWITCH_APP_ANGLE_RANGE) <= angle) || (angle < (THERMOMETER_REF_ANGLE - CURRENT_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_DATE_TIME_CLOCK;
      is_state_changed = true;
    }
    else
    {
      e_current_state = APP_STATE_ALARM_CLOCK;
    }
    break;
  }

  e_prev_state = e_current_state;
  return e_current_state;
}

void loop()
{
  static app_state_e e_app_state = APP_STATE_INDEFINITE;
  ---略---

  int32_t body_angle = getAngle();
  if (body_angle != -1)
  {
    e_app_state = calcCurrentAppState(body_angle);
  }

  switch (e_app_state)
  {
  case APP_STATE_ALARM_CLOCK:
    M5.Lcd.setRotation(0);
    displayAlarmScreen();
    break;

  case APP_STATE_DATE_TIME_CLOCK:
    M5.Lcd.setRotation(1);
    displayDateTimeScreen();
    break;

  case APP_STATE_THERMOMETER:
    M5.Lcd.setRotation(2);
    displayThermometerScreen();
    break;

  case APP_STATE_COUNT_DOWN_TIMER:
    M5.Lcd.setRotation(3);
    displayCountdownTimerScreen();
    break;

  case APP_STATE_INDEFINITE:
  default:
    break;
  }

  ---略---

}

```

## ②数字の7セグ風表示
### **KLOCKIS クロッキス**の挙動チェック
**KLOCKIS クロッキス**の時刻/日付、 アラーム、温度、タイマーそれぞれの画面の数字表示は、7セグメントLED風のデザインになっていることからM5Stackでもできるかぎり同様のデザインとなるようにしました。

### M5Stackへの実装
数字表示については関数化し、引数の数字に応じて、それぞれのセグメントのバー部分を描画するようにしました。
バーの描画は`M5.Lcd.fillRoundRect()`関数を使用し、コーナー半径を指定して長方形の角を丸めています。
どの数字に対してどのセグメント部分を描画するかは、それぞれのセグメントのON/OFFを0/1のビットで定義しています（`digits_norma[]`、`digits_V_long[]`の部分）。

実装内容は以下のとおりです。
`drawNumberVLong()`、`drawNumberNormal()`関数がありますが、`drawNumberVLong()`関数では、縦方向のセグメントのバーが倍になっています（数字の1を表示する場合は、セグメントのバーが縦に4つ並ぶイメージ）。
なので、VLongのほうは厳密には7セグ風ですらない・・・。

_参考URL_：[m5stack/m5-docs fillRoundRect()](https://github.com/m5stack/m5-docs/blob/master/docs/ja/api/lcd.md#fillroundrect)

```C++:clock.ino
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

const uint16_t digits_V_long[] =
    {
        0b0000001111111111, // 0
        0b0000001111000000, // 1
        0b0000011100111001, // 2
        0b0000011111100001, // 3
        0b0000011111000110, // 4
        0b0000010011100111, // 5
        0b0000010011111111, // 6
        0b0000001111000111, // 7
        0b0000011111111111, // 8
        0b0000011111100111, // 9
        0b0000000000000000, // off
};

void drawNumberVLong(uint8_t x_start, uint8_t y_start, uint8_t number, uint8_t bar_width, uint8_t bar_length, uint8_t bar_gap, uint8_t corner_radius, uint16_t color_value)
{
  if (number > 10)
  {
    number = 10;
  }
  // top
  if (digits_V_long[number] & 0b0000000000000001)
    M5.Lcd.fillRoundRect(x_start, y_start, bar_length, bar_width, corner_radius, color_value);
  // upper-left
  if (digits_V_long[number] & 0b0000000000000010)
    M5.Lcd.fillRoundRect((x_start - bar_gap * 2), (y_start + bar_gap), bar_width, bar_length, corner_radius, color_value);
  if (digits_V_long[number] & 0b0000000000000100)
    M5.Lcd.fillRoundRect((x_start - bar_gap * 2), (y_start + bar_gap + bar_length * 1), bar_width, bar_length, corner_radius, color_value);
  // under-left
  if (digits_V_long[number] & 0b0000000000001000)
    M5.Lcd.fillRoundRect((x_start - bar_gap * 2), (y_start + bar_gap + bar_length * 2), bar_width, bar_length, corner_radius, color_value);
  if (digits_V_long[number] & 0b0000000000010000)
    M5.Lcd.fillRoundRect((x_start - bar_gap * 2), (y_start + bar_gap + bar_length * 3), bar_width, bar_length, corner_radius, color_value);
  // bottom
  if (digits_V_long[number] & 0b0000000000100000)
    M5.Lcd.fillRoundRect(x_start, (y_start + bar_length * 4), bar_length, bar_width, corner_radius, color_value);
  // under-right
  if (digits_V_long[number] & 0b0000000001000000)
    M5.Lcd.fillRoundRect((x_start + bar_length), (y_start + bar_gap + bar_length * 3), bar_width, bar_length, corner_radius, color_value);
  if (digits_V_long[number] & 0b0000000010000000)
    M5.Lcd.fillRoundRect((x_start + bar_length), (y_start + bar_gap + bar_length * 2), bar_width, bar_length, corner_radius, color_value);
  // upper-right
  if (digits_V_long[number] & 0b0000000100000000)
    M5.Lcd.fillRoundRect((x_start + bar_length), (y_start + bar_gap + bar_length * 1), bar_width, bar_length, corner_radius, color_value);
  if (digits_V_long[number] & 0b0000001000000000)
    M5.Lcd.fillRoundRect((x_start + bar_length), (y_start + bar_gap), bar_width, bar_length, corner_radius, color_value);
  // center
  if (digits_V_long[number] & 0b0000010000000000)
    M5.Lcd.fillRoundRect(x_start, (y_start + bar_length * 2), bar_length, bar_width, corner_radius, color_value);
}

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
```

それぞれの関数を実行すると、以下のような数値表示がされます。

![IMG_2201_150.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/3d40cacd-1349-2441-41d5-fa2ff3299eb5.jpeg)
![IMG_2200_150.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/8f2a3168-a7a1-576e-d92d-8e668f405c7d.jpeg)

それぞれの機能では、これらの関数をつかって、数字表示を行っていきます。

## ③「温度」機能
### **KLOCKIS クロッキス**の挙動チェック
**KLOCKIS クロッキス**の温度機能を確認し、

- 背景色は青色
- 数字表示は縦長のセグメント風表示
- 左上に温度計アイコン

ということを確認できたので、これに合わせて実装していきます。

### M5Stackへの実装

#### ◆背景色設定
RGBそれぞれ0-255の範囲で指定した色を`M5.Lcd.color565()`関数をつかって変換した後に、`M5.Lcd.fillScreen()`関数を使って背景色を設定しています。
このとき、画面のチラつきをおさえるため、傾きが変わって機能が切り替わったときなどに限定して、`M5.Lcd.fillScreen()`関数を呼び出すようにしています。

※`if (is_state_changed == true)`といった書き方は無駄・冗長という考えもあるようですが、自分の癖なのでご了承ください笑

_参考URL_：[m5stack/m5-docs lcd.md](https://github.com/m5stack/m5-docs/blob/master/docs/ja/api/lcd.md)

#### ◆温度取得・表示
温度の取得については、今回はM5Stack FIREの加速度センサMPU9250に内蔵されている温度センサからデータをとっています。そのため、**室温の温度と一致せず、起動し続けると温度が上がっていく**ことに注意です。
温度取得の関数はサンプルプログラムをそのまま流用しています。
その後、取得した温度の値を10の位と1の位に分離して、それぞの数字に対して表示位置を指定して`drawNumberVLong()`関数を呼び出しています。

このとき、温度が変わった際にそのまま新しい数字を書きこむと前の数字と重なった状態になり、正しい表示がされなくなります。
そこで、温度が変化した場合は、**背景色**で数字を書きこんで数字部分の表示を一度クリアするような処理を入れています。

※`M5.Lcd.fillScreen()`関数を呼んでもクリアされますが、チラつきをおさえるためにこういった処理にしています

_参考URL_：[m5stack/M5Stack M5StackFire_MPU9250.ino](https://github.com/m5stack/M5Stack/blob/master/examples/Fire/M5StackFire_MPU9250/M5StackFire_MPU9250.ino)

#### ◆温度計アイコンの作成
アイコンについては、楕円（`M5.Lcd.drawEllipse()`、`M5.Lcd.fillEllipse()`）や円（`M5.Lcd.drawRoundRect()`、`M5.Lcd.fillRoundRect()`）作成関数を駆使してそれっぽいのをつくっています。
アイコン作成部分は`drawThermometerIcon()`関数内で実行しています。

「温度」機能部分のソースコードを以下に記載します。

```C++:clock.ino
// Thermometer Screen
#define LCD_THERMOMETER_ICON_DISP_Y_POS (80)
#define LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS (80)

void displayThermometerScreen()
{
  uint16_t bkground_color = M5.Lcd.color565(128, 128, 255);
  static uint8_t temperature_prev = 0;
  if (is_state_changed == true)
  {
    M5.Lcd.fillScreen(bkground_color);
    is_state_changed = false;
  }

  // draw thermometer icon
  drawThermometerIcon(40, LCD_THERMOMETER_ICON_DISP_Y_POS, bkground_color);

  // get temperature from MPU9250
  IMU.tempCount = IMU.readTempData(); // Read the adc values
  // Temperature in degrees Centigrade
  IMU.temperature = ((float)IMU.tempCount) / 333.87 + 21.0;
  uint8_t temperature = (int)IMU.temperature;
  uint8_t tens_place, ones_place;
  if (temperature < 10)
  {
    tens_place = 0;
    ones_place = temperature;
  }
  else if (temperature <= 99)
  {
    tens_place = temperature / 10;
    ones_place = temperature % 10;
  }
  else
  {
    tens_place = 9;
    ones_place = 9;
  }

  if (temperature != temperature_prev)
  {
    drawNumberVLong(60, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    drawNumberVLong(130, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    // drawNumberNormal(185, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    // drawNumberNormal(250, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberVLong(60, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, tens_place, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberVLong(130, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, ones_place, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

  M5.Lcd.drawEllipse(130 + LCD_LARGE_BAR_WIDTH + LCD_LARGE_BAR_LENGTH + LCD_LARGE_BAR_GAP, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, 3, 3, TFT_BLACK);
  M5.Lcd.drawChar(130 + LCD_LARGE_BAR_WIDTH + LCD_LARGE_BAR_LENGTH + LCD_LARGE_BAR_GAP + 10, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, 'C', TFT_BLACK, bkground_color, 4);

  temperature_prev = temperature;
}

void drawThermometerIcon(uint32_t base_x_pos, uint32_t base_y_pos, uint16_t bk_color)
{
  M5.Lcd.drawEllipse(base_x_pos, base_y_pos, 7, 7, TFT_BLACK);
  M5.Lcd.drawRoundRect(base_x_pos - 3, base_y_pos - 35, 8, 35, 4, TFT_BLACK);
  M5.Lcd.fillEllipse(base_x_pos, base_y_pos, 6, 6, bk_color);
  M5.Lcd.fillEllipse(base_x_pos, base_y_pos, 4, 4, TFT_BLACK);
  M5.Lcd.fillRoundRect(base_x_pos - 1, base_y_pos - 35 + 8, 4, 25, 2, TFT_BLACK);
}
```

表示される「温度」画面は以下のようになります。

![IMG_2202_640.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/f7c0e51f-9ac8-2b33-6d12-f670655d8c5f.jpeg)


## ④「時刻/日付」機能
### **KLOCKIS クロッキス**の挙動チェック
**KLOCKIS クロッキス**の時刻/日付機能では年月日、時刻、曜日を表示しています。

- 背景色は赤色
- 画面上側に年月日表示
- 中央に日時（HH:MM）表示
- 画面下側に曜日表示
- 日時（HH:MM）のコロン部分とその日の曜日部分が**点滅**表示する
- 右下に時計アイコン

これに合わせてM5Stackに実装していきます。

### M5Stackへの実装
背景色の部分は[先ほど](https://qiita.com/yankee/private/1591724d8a2722951c7e#%E8%83%8C%E6%99%AF%E8%89%B2%E8%A8%AD%E5%AE%9A)と同じなので省略。

#### ◆日時・曜日情報取得
時刻/日付機能表示では、最初にNTPサーバにアクセスして時刻情報の取得を行います。
ただし、その毎ループNTPサーバにアクセスするのはあまりよろしくないので、

- 「時刻/日付」機能切り替わりタイミング（起動時含む）
- 「時刻/日付」機能表示のまま規定時間経過したタイミング  
のみNTPサーバにアクセスするようにし、それ以外はソフトウェアタイマで経過時間を取得し、日時を算出しています。

_参考URL_：[espressif/arduino-esp32 SimpleTime.ino](https://github.com/espressif/arduino-esp32/blob/master/libraries/ESP32/examples/Time/SimpleTime/SimpleTime.ino)
_参考URL_：[NTPサーバから時刻を取得してM5Stackに表示する](https://kuracux.hatenablog.jp/entry/2018/10/08/160000)

#### ◆点滅表示
特定箇所のみ点滅させるため、ループ回数をカウントし、
`if ((count / LCD_DISP_BLINK_LOOP_CNT % 2) == 0)`といいった条件で表示のON（描画色を黒色にして、表示）、OFF（描画色を背景色にして、実質的に表示を消す）を切り替えています。

#### ◆時計アイコンの作成
アイコンについては、`M5.Lcd.drawPixel()`で時計の外周をプロットしていき、`M5.Lcd.drawLine()`関数をつかって、短針と長針つくっています。
アイコン作成部分は`drawClockIcon()`関数内で実行しています。

「時刻/日付」機能部分のソースコードを以下に記載します。

```C++:clock.ino
// NTP
const char *ntp_server_1st = "ntp.nict.jp";
const char *ntp_server_2nd = "time.google.com";
const long gmt_offset_sec = 9 * 3600; // 時差（秒換算）
const int daylight_offset_sec = 0;    // 夏時間

---略---

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

---略---

#define LCD_CLOCK_YMD_DISP_Y_POS (10)
#define LCD_CLOCK_MD_STR_DISP_Y_POS (45)
#define LCD_CLOCK_HM_DISP_Y_POS (100)
#define LCD_CLOCK_PM_STR_DISP_Y_POS (175)
#define LCD_CLOCK_ICON_DISP_Y_POS (200)
#define LCD_CLOCK_WEEK_STR_DISP_Y_POS (220)

void displayDateTimeScreen()
{
  // ntp_date_t t_date_now;
  // static ntp_date_t t_date_prev;
  static boolean ntp_access_flag = true;
  static uint32_t base_milli_time;
  uint32_t elasped_second = 0;
  uint32_t diff_milli_time = 0;

  uint16_t bkground_color = M5.Lcd.color565(200, 0, 0);
  if (is_state_changed == true)
  {
    M5.Lcd.fillScreen(bkground_color);
    ntp_access_flag = true;
    is_state_changed = false;
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
    // Serial.print("diff_milli_time:");
    // Serial.println(diff_milli_time);
    elasped_second = diff_milli_time / 1000;
    cl_system_clock.updateBySoftTimer(elasped_second);
  }

  M5.Lcd.setTextColor(TFT_BLACK);
  M5.Lcd.setTextSize(2);

  // Month
  if (cl_system_clock.month != cl_system_clock.prev_month)
  {
    drawNumberNormal(10, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(35, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(10, LCD_CLOCK_YMD_DISP_Y_POS, (cl_system_clock.month / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(35, LCD_CLOCK_YMD_DISP_Y_POS, (cl_system_clock.month % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);

  M5.Lcd.drawLine(60, LCD_SMALL_BAR_LENGTH * 2 + 10, 70, 10, TFT_BLACK);

  // Day
  if (cl_system_clock.day != cl_system_clock.prev_day)
  {
    drawNumberNormal(80, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(105, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(80, LCD_CLOCK_YMD_DISP_Y_POS, (cl_system_clock.day / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(105, LCD_CLOCK_YMD_DISP_Y_POS, (cl_system_clock.day % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);

  // Year
  if (cl_system_clock.year != cl_system_clock.prev_year)
  {
    drawNumberNormal(180, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(205, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(230, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(255, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(180, LCD_CLOCK_YMD_DISP_Y_POS, (cl_system_clock.year / 1000), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(205, LCD_CLOCK_YMD_DISP_Y_POS, ((cl_system_clock.year % 1000) / 100), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(230, LCD_CLOCK_YMD_DISP_Y_POS, (((cl_system_clock.year % 1000) % 100) / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(255, LCD_CLOCK_YMD_DISP_Y_POS, (((cl_system_clock.year % 1000) % 100) % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);

  // hour
  if (cl_system_clock.hour != cl_system_clock.prev_hour)
  {
    drawNumberNormal(30, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(95, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    M5.Lcd.setTextColor(bkground_color);
    M5.Lcd.drawString("PM", 20, LCD_CLOCK_PM_STR_DISP_Y_POS);
    M5.Lcd.setTextColor(TFT_BLACK);
  }
  drawNumberNormal(30, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(95, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
  // if (isPM == true)
  // {
  //   M5.Lcd.drawString("PM", 20, LCD_CLOCK_PM_STR_DISP_Y_POS);
  // }

  // Sec
  // if (cl_system_clock.second != cl_system_clock.prev_second)
  // {
  //   drawNumberNormal(180, LCD_CLOCK_YMD_DISP_Y_POS + 40, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  //   drawNumberNormal(205, LCD_CLOCK_YMD_DISP_Y_POS + 40, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  // }
  // drawNumberNormal(180, LCD_CLOCK_YMD_DISP_Y_POS + 40, (cl_system_clock.second / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  // drawNumberNormal(205, LCD_CLOCK_YMD_DISP_Y_POS + 40, (cl_system_clock.second % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);

  if (cl_blink_count.isHideDisplay() == false)
  {
    M5.Lcd.fillEllipse(150, 120, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(150, 150, 4, 4, TFT_BLACK);
  }
  else
  {
    M5.Lcd.fillEllipse(150, 120, 4, 4, bkground_color);
    M5.Lcd.fillEllipse(150, 150, 4, 4, bkground_color);
  }

  // minute
  if (cl_system_clock.minute != cl_system_clock.prev_minute)
  {
    drawNumberNormal(185, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(250, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(185, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(250, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

  // MONTH DATE
  const String s_month = "MONTH";
  const String s_date = "DATE";
  M5.Lcd.drawString(s_month, 10, LCD_CLOCK_MD_STR_DISP_Y_POS);
  M5.Lcd.drawString(s_date, 80, LCD_CLOCK_MD_STR_DISP_Y_POS);

  uint8_t week_count = 0;
  const char aweek[7][4] = {"SUN", "MON", "TUE", "WED", "THU", "FRI", "SAT"};
  uint8_t day_week_now = 0;
  // day_week_now = subZeller(t_date_now.npt_year, t_date_now.ntp_month, t_date_now.ntp_day);
  day_week_now = cl_system_clock.week_day;
  for (week_count = 0; week_count < 7; ++week_count)
  {
    if ((week_count == day_week_now) && (cl_blink_count.isHideDisplay() == true))
    {
      M5.Lcd.setTextColor(bkground_color);
      M5.Lcd.drawString(aweek[week_count], week_count * 45 + 10, LCD_CLOCK_WEEK_STR_DISP_Y_POS);
      M5.Lcd.setTextColor(TFT_BLACK);
    }
    else
    {
      M5.Lcd.drawString(aweek[week_count], week_count * 45 + 10, LCD_CLOCK_WEEK_STR_DISP_Y_POS);
    }
  }

  // clock icon
  drawClockIcon(300, LCD_CLOCK_ICON_DISP_Y_POS);

  // t_date_prev = t_date_now;
  cl_system_clock.backupCurrentTime();
}

---略---

void setup()
{
  ---略---

  WiFi.mode(WIFI_MODE_STA);
  WiFi.begin(ssid, password);
  while (WiFi.waitForConnectResult() != WL_CONNECTED)
  {
    delay(100);
  }

  ---略---

  // init time setting
  configTime(gmt_offset_sec, daylight_offset_sec, ntp_server_1st, ntp_server_2nd);
  struct tm time_info;
  if (!getLocalTime(&time_info))
  {
    M5.Lcd.fillScreen(TFT_RED);
    delay(3000);
  }
  Serial.println("hello world");
}
```

表示される「時刻/日付」画面は以下のようになります。

![IMG_2207_640.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/82340a41-887f-2057-931a-18461ea3235c.jpeg)



## ⑤「タイマー」機能
### **KLOCKIS クロッキス**の挙動チェック
**KLOCKIS クロッキス**のタイマー機能では**分**と**秒**を設定して、カウントダウンタイマーとして動作させることができます。仕様は以下のような感じです。

- 背景色は緑色
- ボタン①を押すと、タイマー（分）設定状態に切り替わり、**分**表示が点滅する
- この状態でボタン②を押していくと、**分**がインクリメントされる
- タイマー（分）設定状態でボタン①を押すと、タイマー（秒）設定状態に切り替わる（**秒**表示点滅）
- この状態でボタン①を押すと、設定したタイマーのカウントダウンが始まる（分、秒とも未設定の場合、初期状態に戻る）
- タイマーのカウントダウン中にボタン①を押すと、タイマーがクリアされる
- タイマーが0分0秒までカウントダウンされるとアラーム音が鳴る
- 右上に砂時計アイコン

なお、「タイマー」機能では、

- タイマーのカウントダウン中に時計を傾けて、別機能に切り替えるとカウントダウンが停止する
- もう一度「タイマー」機能に切り替えると、タイマーの初期設定状態からカウントダウンをやり直す（例：カウントダウンタイマー１分に設定し、残り１５秒までカウントダウンした状態で別機能に切り替え⏩「タイマー」機能に戻した場合、また１分のカウントダウンが始まる）

といった挙動になります（後述しますが、「アラーム」機能とはこの辺の動作が異なる）。

次に、これに合わせてM5Stackに実装していきます。


### M5Stackへの実装
背景色の部分は[先ほど](https://qiita.com/yankee/private/1591724d8a2722951c7e#%E8%83%8C%E6%99%AF%E8%89%B2%E8%A8%AD%E5%AE%9A)と同じなので省略。

#### ◆状態管理
「タイマー」機能では、タイマー設定状態やタイマーのカウントダウン状態などいくつかの状態を遷移する必要があったので、`e_cnt_timer_status`という変数でステータスを持たせて、その状態によって処理を変えていきます。
取り得る状態としては、

- 初期状態orタイマー未設定状態（`CNT_DOWN_TIMER_NOT_SET`）
- タイマー**分**設定状態（`CNT_DOWN_TIMER_SETTING_MINUTE`）
- タイマー**秒**設定状態（`CNT_DOWN_TIMER_SETTING_SECOND`）
- カウントダウン状態（`CNT_DOWN_TIMER_RUNNING`）
- アラーム音発生状態（`CNT_DOWN_TIMER_RINGING`）  
としています。  
・・・が、アラーム音については、M5Stackから出る音が思いのほか大きく、抵抗をかませたりしないと音量制御が難しそうだったため、音を出すかわりに「RINGING」という表示を出すことでお茶を濁しています。

#### ◆ボタンによるタイマー設定
ボタンによるステータス切り替えやタイマー時間設定に使用するボタンについてはM5Stack FIREのAボタンとCボタンを使用しました。
[挙動チェック](https://qiita.com/drafts/1591724d8a2722951c7e/edit#klockis-%E3%82%AF%E3%83%AD%E3%83%83%E3%82%AD%E3%82%B9%E3%81%AE%E6%8C%99%E5%8B%95%E3%83%81%E3%82%A7%E3%83%83%E3%82%AF-4)のところの書き方でいうと、Aボタンがボタン①、Cボタンがボタン②の役割になります。
`CNT_DOWN_TIMER_NOT_SET`状態からAボタンを押すたびに、
`CNT_DOWN_TIMER_SETTING_MINUTE`⏩`CNT_DOWN_TIMER_SETTING_SECOND`⏩`CNT_DOWN_TIMER_RUNNING`と状態を切り替えていきます。  
また、`CNT_DOWN_TIMER_SETTING_MINUTE`状態でCボタンを押すたびに分表示をインクリメント、`CNT_DOWN_TIMER_SETTING_SECOND`状態でCボタンを押すたびに秒表示をインクリメントしていきます（それぞれ、59の次は0に戻る）。
なお、ボタンを押したかの判断は`wasReleased()`関数を使用していますので、ボタンを押した後に一度ボタンを引かないとインクリメントがされないようにしています。

_参考URL_：[m5stack/m5-docs wasReleased()](https://github.com/m5stack/m5-docs/blob/master/docs/en/api/button.md#wasreleased)

#### ◆タイマーのカウントダウン処理
ステータスが`CNT_DOWN_TIMER_RUNNING`に切り替わると、設定されたタイマーの分と秒をチェックし、タイマー時間が0分0秒の場合は`CNT_DOWN_TIMER_NOT_SET`状態に戻ります。
分または秒どちらかでも0以外の値が設定されていた場合、カウントダウンしていきます。
最初に現在の時間を`millis()`関数で取得し、そこにタイマー設定した分と秒をミリ秒換算して加算したものを基準時間として`set_milli_time`変数に格納しておきます。
その後、`diff_milli_time = set_milli_time - millis()`で時間をカウントダウンしていき、0以下になったらタイマー時間に達したとして、`CNT_DOWN_TIMER_RINGING`状態に遷移します。

#### ◆カウントダウン終了処理
`CNT_DOWN_TIMER_RINGING`では下の画像のように、”RINGING”という文字を白文字で表示します。
この状態で、AボタンまたはCボタンを押すと、状態が遷移して、”RINGING”の文字は消えます。
※実際にアラーム音を出したい場合、ここにSpeaker関連の関数を呼び出す形になります。

![IMG_2211_320.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/4fce3f10-e78c-132e-94de-4c53533cb00f.jpeg)


#### ◆砂時計アイコンの作成
アイコンについては、`M5.Lcd.drawTriangle()`関数と`M5.Lcd.fillTriangle()`関数で三角形を4つ組み合わせて簡単な砂時計を作っています。
アイコン作成は`drawSandglassIcon()`関数内で処理しています。

「タイマー」機能部分のソースコードを以下に記載します。

```C++:clock.ino
// Timer Screen
#define LCD_SANDGLASS_ICON_DISP_Y_POS (30)
#define LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS (70)
#define LCD_CNT_DOWN_TIMER_STR_DISP_Y_POS (150)
#define LCD_CNT_DOWN_TIMER_ICON_DISP_Y_POS (200)

typedef enum
{
  CNT_DOWN_TIMER_NOT_SET = 0,
  CNT_DOWN_TIMER_RUNNING = 1,
  CNT_DOWN_TIMER_SETTING_MINUTE = 2,
  CNT_DOWN_TIMER_SETTING_SECOND = 3,
  CNT_DOWN_TIMER_RINGING = 4,
} count_down_timer_status_e;

void displayCountdownTimerScreen()
{
  static count_down_timer_status_e e_cnt_timer_status = CNT_DOWN_TIMER_NOT_SET;
  static uint8_t timer_minute = 0;
  static uint8_t timer_second = 0;
  static uint32_t base_milli_time = 0;
  static uint32_t set_milli_time = 0;
  static boolean is_count_down = false;

  uint16_t bkground_color = M5.Lcd.color565(0, 180, 0);
  if (is_state_changed == true)
  {
    M5.Lcd.fillScreen(bkground_color);
    is_state_changed = false;
    is_count_down = false;
    e_cnt_timer_status = CNT_DOWN_TIMER_RUNNING;
  }

  // draw sandglass icon
  if (cl_blink_count.isHideDisplay() == false)
  {
    drawSandglassIcon(240, LCD_SANDGLASS_ICON_DISP_Y_POS, TFT_BLACK);
  }
  else
  {
    drawSandglassIcon(240, LCD_SANDGLASS_ICON_DISP_Y_POS, bkground_color);
  }

  M5.Lcd.setTextColor(TFT_BLACK);
  M5.Lcd.setTextSize(2);

  M5.Lcd.drawString("MIN", 20, LCD_CNT_DOWN_TIMER_STR_DISP_Y_POS);
  M5.Lcd.drawString("SEC", 250, LCD_CNT_DOWN_TIMER_STR_DISP_Y_POS);

  switch (e_cnt_timer_status)
  {
  case CNT_DOWN_TIMER_NOT_SET:
  default:
    drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    if (M5.BtnA.wasReleased())
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_SETTING_MINUTE;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;

  case CNT_DOWN_TIMER_SETTING_MINUTE:
    if (cl_blink_count.isHideDisplay() == false)
    {
      drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
      drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    }
    else
    {
      drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
      drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    }
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    if (M5.BtnC.wasReleased())
    {
      ++timer_minute;
      if (timer_minute > 99)
      {
        timer_minute = 0;
      }
      M5.Lcd.fillScreen(bkground_color);
    }
    else if (M5.BtnA.wasReleased())
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_SETTING_SECOND;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;

  case CNT_DOWN_TIMER_SETTING_SECOND:
    drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    if (cl_blink_count.isHideDisplay() == false)
    {
      drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
      drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    }
    else
    {
      drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
      drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    }

    if (M5.BtnC.wasReleased())
    {
      ++timer_second;
      if (timer_second > 59)
      {
        timer_second = 0;
      }
      M5.Lcd.fillScreen(bkground_color);
    }
    else if (M5.BtnA.wasReleased())
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_RUNNING;
      M5.Lcd.fillScreen(bkground_color);
      cl_blink_count.resetCount();
    }
    break;

  case CNT_DOWN_TIMER_RUNNING:
    if ((timer_minute == 0) && (timer_second == 0))
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_NOT_SET;
    }
    else
    {
      uint8_t disp_minute = 0;
      uint8_t disp_second = 0;
      static uint8_t pre_disp_minute = 0;
      static uint8_t pre_disp_second = 0;
      if (is_count_down == false)
      {
        is_count_down = true;
        base_milli_time = millis();
        set_milli_time = base_milli_time + timer_second * 1000 + timer_minute * 60 * 1000;

        drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
        drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
        M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
        M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
        drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
        drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
        pre_disp_minute = timer_minute;
        pre_disp_second = timer_second;
      }
      else
      {
        int32_t diff_milli_time = set_milli_time - millis();
        if (diff_milli_time < 0)
        {
          e_cnt_timer_status = CNT_DOWN_TIMER_RINGING;
          is_count_down = false;
        }
        else
        {
          disp_minute = diff_milli_time / 1000 / 60;
          disp_second = diff_milli_time / 1000 % 60;

          if (disp_minute != pre_disp_minute)
          {
            drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (pre_disp_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
            drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (pre_disp_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
          }
          if (disp_second != pre_disp_second)
          {
            drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (pre_disp_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
            drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (pre_disp_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
          }

          drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (disp_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
          drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (disp_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
          M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
          M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
          drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (disp_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
          drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (disp_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

          pre_disp_minute = disp_minute;
          pre_disp_second = disp_second;
        }
      }

      if (M5.BtnA.wasReleased())
      {
        e_cnt_timer_status = CNT_DOWN_TIMER_SETTING_MINUTE;
        timer_minute = 0;
        timer_second = 0;
        is_count_down = false;
        M5.Lcd.fillScreen(bkground_color);
        cl_blink_count.resetCount();
      }
    }
    break;

  case CNT_DOWN_TIMER_RINGING:
    drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    M5.Lcd.setTextColor(TFT_WHITE);
    M5.Lcd.setTextSize(4);
    M5.Lcd.drawString("RINGING", 20, 180);

    if (M5.BtnA.wasReleased())
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_SETTING_MINUTE;
      timer_minute = 0;
      timer_second = 0;
      M5.Lcd.fillScreen(bkground_color);
      cl_blink_count.resetCount();
    }
    else if (M5.BtnC.wasReleased())
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_NOT_SET;
      timer_minute = 0;
      timer_second = 0;
      M5.Lcd.fillScreen(bkground_color);
      cl_blink_count.resetCount();
    }
    break;
  }
}

void drawSandglassIcon(uint32_t base_x_pos, uint32_t base_y_pos, uint16_t color)
{
  // 上三角
  M5.Lcd.drawTriangle(base_x_pos, base_y_pos, base_x_pos + 20, base_y_pos, base_x_pos + 10, base_y_pos + 14, color);
  // 下三角
  M5.Lcd.drawTriangle(base_x_pos, base_y_pos + 28, base_x_pos + 20, base_y_pos + 28, base_x_pos + 10, base_y_pos + 14, color);

  // 上砂
  M5.Lcd.fillTriangle(base_x_pos + 6, base_y_pos + 4, base_x_pos + 20 - 6, base_y_pos + 4, base_x_pos + 10, base_y_pos + 14 - 4, color);
  // 下砂
  M5.Lcd.fillTriangle(base_x_pos + 2, base_y_pos + 28 - 2, base_x_pos + 20 - 2, base_y_pos + 28 - 2, base_x_pos + 10, base_y_pos + 14 + 8, color);
}
```

表示される「タイマー」画面は以下のようになります。

![IMG_2213_640.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/3bc199cf-2b09-9d06-e8a6-d5881bccff6b.jpeg)

## ⑥「アラーム」機能
### **KLOCKIS クロッキス**の挙動チェック
**KLOCKIS クロッキス**の「アラーム」機能では**時**と**分**を設定して、アラーム（目覚まし時計）として動作します。仕様は以下のような感じです。

- 背景色は緑色
- ボタン①を押すと、タイマー（時）設定状態に切り替わり、**時**表示が点滅
- この状態でボタン②を押していくと、**時**がインクリメントされる
- タイマー（時）設定状態でボタン①を押すと、タイマー（分）設定状態に切り替え（**分**表示点滅）
- この状態でボタン①を押すと、アラーム設定がONになり、左上にベルアイコンが表示される
- この状態でボタン①を再度押すと、アラーム設定がキャンセルされ、ベルアイコンが消える
- 指定した日時になると、アラーム音が鳴る
- 左上に目覚まし時計アイコン

なお、「アラーム」機能では、「タイマー」機能と異なり、

- アラーム設定後に時計を傾けて、別機能に切り替えてもアラームは継続動作する
- 別機能に切り替わった状態でもアラーム時刻になるとアラーム音が発生

といった挙動になります。

この辺の仕様もふまえて、M5Stackに実装していきます。

### M5Stackへの実装
背景色の部分は[先ほど](https://qiita.com/yankee/private/1591724d8a2722951c7e#%E8%83%8C%E6%99%AF%E8%89%B2%E8%A8%AD%E5%AE%9A)と同じなので省略。

#### ◆状態管理
「アラーム」機能でも「タイマー」機能同様、状態によって処理を変えていきます。ステータスは`e_alarm_status`変数で管理します。
取り得る状態としては、

- 初期状態orアラーム未設定状態（`ALARM_NOT_SET`）
- アラーム**時**設定状態（`ALARM_SETTING_HOUR`）
- アラーム**分**設定状態（`ALARM_SETTING_MINUTE`）
- アラーム設定済み状態（`ALARM_RUNNING`）  
の４つです。  
なお、先ほど同様、アラーム時刻のアラーム音については、見送っています。

#### ◆ボタンによるアラーム時刻設定
基本的な処理は[「タイマー」設定](https://qiita.com/drafts/1591724d8a2722951c7e/edit#m5stack%E3%81%B8%E3%81%AE%E5%AE%9F%E8%A3%85-4)の時間設定と同じです。
異なる点としては、`ALARM_SETTING_MINUTE`状態でAボタンを押したときに`enableAlarm()`関数を呼び出してタイマーイベントを設定しています（詳細は後述）。

#### ◆アラーム設定済み状態表示＆ベルアイコンの作成
`ALARM_RUNNING`状態では設定されたアラーム時刻の表示とベルアイコンを表示するだけになります。
アイコンについては、`M5.Lcd.fillEllipse()`関数と`M5.Lcd.fillRect()`関数、`M5.Lcd.fillTriangle()`関数組み合わせてベルっぽいのを作っています。
※手間の関係で**KLOCKIS クロッキス**と比べて、ベルアイコンは少し形が変わってます・・

ベルアイコン作成は`drawBellIcon()`関数内で処理しています。

#### ◆アラーム時刻のタイマーイベント設定
「アラーム」機能では、M5Stackのタイマー系のライブラリである`M5Timer`を使用して、タイマーイベントを設定しています。
アラーム時刻設定時に`enableAlarm()`関数を呼び出し、その関数内で`setTimeout(timer_ms, wakeUpTime)`関数を実行しています。
`timer_ms`で指定した時間が経過すると、第2引数で指定した関数（`wakeUpTime()`）が呼び出されます。

タイマーイベントをキャンセルしたい場合、`deleteTimer(callback_slot_id)`を呼び出せばいいので、`ALARM_RUNNING`状態でAボタンを押した際に`deleteTimer()`関数を呼び出しています。

_参考URL_：[m5stack/m5-docs M5Timer.md](https://github.com/m5stack/m5-docs/blob/master/docs/en/api/M5Timer.md)

#### ◆タイマーイベント発生時処理
`setTimeout(timer_ms, wakeUpTime)`関数で指定した時間経過後に`wakeUpTime()`関数が呼び出されます。
`wakeUpTime()`関数では、音は鳴らさず、画面いっぱいに**WAKE UP**と表示しています。
この関数内では、`while (1)`でループさせ、A,B,Cボタンのいずれかが押されないと解除されない仕様にしています（時計を傾けても機能は切り替わらない）。

![IMG_2222_320.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/21d6a41f-7504-b0db-e379-4bda50db0799.jpeg)

#### ◆目覚まし時計アイコンの作成
アイコンについては、`M5.Lcd.drawEllipse()`関数と`M5.Lcd.drawLine()`、`M5.Lcd.fillRect()`、`M5.Lcd.fillTriangle()`関数を組み合わせて作成。
アイコン作成は`drawAlarmIcon()`関数内で処理しています。

「アラーム」機能部分のソースコードを以下に記載します。


```C++:clock.ino
M5Timer Task_Timer;

---略---

// Alarm Screen
#define LCD_ALARM_HM_DISP_Y_POS (150)
#define LCD_ALARM_STR_DISP_Y_POS (100)
#define LCD_ALARM_CLOCK_ICON_DISP_Y_POS (105)
#define LCD_ALARM_BELL_ICON_DISP_Y_POS (110)

#define LCD_ALARM_H_DISP_TENS_PLACE_X_POS (15)
#define LCD_ALARM_H_DISP_ONES_PLACE_X_POS (70)
#define LCD_ALARM_M_DISP_TENS_PLACE_X_POS (140)
#define LCD_ALARM_M_DISP_ONES_PLACE_X_POS (195)
#define LCD_ALARM_HM_DISP_COLON_X_POS (120)

typedef enum
{
  ALARM_NOT_SET = 0,
  ALARM_RUNNING = 1,
  ALARM_SETTING_HOUR = 2,
  ALARM_SETTING_MINUTE = 3,
} alarm_status_e;
alarm_status_e e_alarm_status = ALARM_NOT_SET;

void displayAlarmScreen()
{
  static uint8_t alarm_hour = 0;
  static uint8_t alarm_minute = 0;
  static int16_t callback_alarm_slot_id = -1;

  uint16_t bkground_color = M5.Lcd.color565(0, 180, 0);
  if (is_state_changed == true)
  {
    M5.Lcd.fillScreen(bkground_color);
    is_state_changed = false;
  }

  M5.Lcd.setTextColor(TFT_BLACK);
  M5.Lcd.setTextSize(2);

  M5.Lcd.drawString("ALARM", 80, LCD_ALARM_STR_DISP_Y_POS);

  // alarm icon
  drawAlarmIcon(55, LCD_ALARM_CLOCK_ICON_DISP_Y_POS);

  switch (e_alarm_status)
  {
  case ALARM_NOT_SET:
  default:
    drawNumberNormal(LCD_ALARM_H_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_H_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    if (M5.BtnA.wasReleased())
    {
      e_alarm_status = ALARM_SETTING_HOUR;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;

  case ALARM_SETTING_HOUR:
    if (cl_blink_count.isHideDisplay() == false)
    {
      drawNumberNormal(LCD_ALARM_H_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
      drawNumberNormal(LCD_ALARM_H_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    }
    else
    {
      drawNumberNormal(LCD_ALARM_H_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
      drawNumberNormal(LCD_ALARM_H_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    }
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    if (M5.BtnC.wasReleased())
    {
      ++alarm_hour;
      if (alarm_hour > 23)
      {
        alarm_hour = 0;
      }
      M5.Lcd.fillScreen(bkground_color);
    }
    else if (M5.BtnA.wasReleased())
    {
      e_alarm_status = ALARM_SETTING_MINUTE;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;

  case ALARM_SETTING_MINUTE:
    drawNumberNormal(LCD_ALARM_H_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_H_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    if (cl_blink_count.isHideDisplay() == false)
    {
      drawNumberNormal(LCD_ALARM_M_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
      drawNumberNormal(LCD_ALARM_M_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    }
    else
    {
      drawNumberNormal(LCD_ALARM_M_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
      drawNumberNormal(LCD_ALARM_M_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    }

    if (M5.BtnC.wasReleased())
    {
      ++alarm_minute;
      if (alarm_minute > 59)
      {
        alarm_minute = 0;
      }
      M5.Lcd.fillScreen(bkground_color);
    }
    else if (M5.BtnA.wasReleased())
    {
      callback_alarm_slot_id = enableAlarm(alarm_hour, alarm_minute);
      e_alarm_status = ALARM_RUNNING;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;

  case ALARM_RUNNING:
    drawNumberNormal(LCD_ALARM_H_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_H_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    // bell icon
    drawBellIcon(20, LCD_ALARM_BELL_ICON_DISP_Y_POS);

    if (M5.BtnA.wasReleased())
    {
      disableAlarm(callback_alarm_slot_id);
      callback_alarm_slot_id = -1;
      e_alarm_status = ALARM_NOT_SET;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;
  }
}

// Draw Icon API
void drawAlarmIcon(uint32_t base_x_pos, uint32_t base_y_pos)
{
  const int32_t offset = 14;

  M5.Lcd.drawEllipse(base_x_pos, base_y_pos, 7, 7, TFT_BLACK);
  // 短針
  M5.Lcd.drawLine(base_x_pos, base_y_pos, base_x_pos - 2, base_y_pos - 2, TFT_BLACK);
  M5.Lcd.drawLine(base_x_pos, base_y_pos + 1, base_x_pos - 2, base_y_pos - 1, TFT_BLACK);
  // 長針
  M5.Lcd.drawLine(base_x_pos, base_y_pos, base_x_pos + 4, base_y_pos - 3, TFT_BLACK);
  M5.Lcd.drawLine(base_x_pos, base_y_pos + 1, base_x_pos + 4, base_y_pos - 2, TFT_BLACK);

  // 左足
  M5.Lcd.drawLine(base_x_pos - 5, base_y_pos + 5, base_x_pos - 7, base_y_pos + 7, TFT_BLACK);
  M5.Lcd.drawLine(base_x_pos - 5, base_y_pos + 6, base_x_pos - 7, base_y_pos + 8, TFT_BLACK);
  // 右足
  M5.Lcd.drawLine(base_x_pos + 5, base_y_pos + 5, base_x_pos + 7, base_y_pos + 7, TFT_BLACK);
  M5.Lcd.drawLine(base_x_pos + 5, base_y_pos + 6, base_x_pos + 7, base_y_pos + 8, TFT_BLACK);

  // 角
  M5.Lcd.fillRect(base_x_pos - 3, base_y_pos - 11, 6, 2, TFT_BLACK);
  M5.Lcd.fillRect(base_x_pos - 2, base_y_pos - 9, 4, 1, TFT_BLACK);

  // 左角
  M5.Lcd.fillTriangle(base_x_pos - 8, base_y_pos - 5, base_x_pos - 5, base_y_pos - 8, base_x_pos - 7, base_y_pos - 8, TFT_BLACK);
  // 右角
  M5.Lcd.fillTriangle(base_x_pos + 8, base_y_pos - 5, base_x_pos + 5, base_y_pos - 8, base_x_pos + 7, base_y_pos - 8, TFT_BLACK);
}

void drawBellIcon(uint32_t base_x_pos, uint32_t base_y_pos)
{
  // 上丸
  M5.Lcd.fillEllipse(base_x_pos + 6, base_y_pos + 2, 2, 2, TFT_BLACK);

  // 胴体上
  M5.Lcd.fillRect(base_x_pos, base_y_pos + 3, 13, 10, TFT_BLACK);

  // 胴体下
  M5.Lcd.fillTriangle(base_x_pos, base_y_pos + 14, base_x_pos, base_y_pos + 17, base_x_pos - 4, base_y_pos + 17, TFT_BLACK);
  M5.Lcd.fillTriangle(base_x_pos + 13, base_y_pos + 14, base_x_pos + 13, base_y_pos + 17, base_x_pos + 17, base_y_pos + 17, TFT_BLACK);
  M5.Lcd.fillRect(base_x_pos, base_y_pos + 14, 13, 4, TFT_BLACK);

  // 下丸
  M5.Lcd.fillEllipse(base_x_pos + 6, base_y_pos + 20, 2, 2, TFT_BLACK);
}

void wakeUpTime()
{
  uint16_t bkground_color = M5.Lcd.color565(255, 255, 255);
  M5.Lcd.fillScreen(bkground_color);
  M5.Lcd.setTextColor(TFT_BLACK);
  M5.Lcd.setTextSize(6);
  M5.Lcd.drawString("WAKE", 50, 100);
  M5.Lcd.drawString(" UP ", 50, 170);
  while (1)
  {
    M5.update();
    if (M5.BtnA.wasReleased() || M5.BtnB.wasReleased() || M5.BtnC.wasReleased())
    {
      is_state_changed = true;
      e_alarm_status = ALARM_NOT_SET;
      break;
    }
    delay(100);
  }
}

int16_t enableAlarm(uint8_t hour, uint8_t minute)
{
  int32_t set_time_minute = hour * 60 + minute;
  int32_t now_time_minute = 0;
  int32_t timer_ms = 0;
  int16_t callback_slot_id = -1;

  struct tm time_info;
  if (!getLocalTime(&time_info))
  {
    M5.Lcd.drawString("ERR", 0, 190);
  }
  else
  {
    now_time_minute = time_info.tm_hour * 60 + time_info.tm_min;
    if (set_time_minute > now_time_minute)
    {
      timer_ms = ((set_time_minute - now_time_minute) * 60 - time_info.tm_sec) * 1000;
    }
    else
    {
      timer_ms = (((24 * 60) - (now_time_minute - set_time_minute)) * 60 - time_info.tm_sec) * 1000;
    }
    callback_slot_id = Task_Timer.setTimeout(timer_ms, wakeUpTime);
  }
  return callback_slot_id;
}

void disableAlarm(int16_t callback_slot_id)
{
  if (callback_slot_id != -1)
  {
    Task_Timer.deleteTimer(callback_slot_id);
  }
}
```

表示される「アラーム」画面は以下のようになります。

![IMG_2216_640.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371217/1163621a-1237-742e-2529-d12c8e0e1ff2.jpeg)

## ⑦実装まとめ
これで一通りコードができました。
<details><summary>最後に全体のソースコードを以下に記載します。
※ソースコードがかなり長くなってしまったため、折畳みにしてあります</summary><div>

```C++:clock.ino
#include <M5Stack.h>
#include <WiFi.h>
#include <math.h>
#include "utility/MPU9250.h"
#include "utility/M5Timer.h"
#include "time.h"

boolean is_state_changed = true;
typedef enum
{
  APP_STATE_INDEFINITE = 0,
  APP_STATE_DATE_TIME_CLOCK = 1,
  APP_STATE_ALARM_CLOCK = 2,
  APP_STATE_COUNT_DOWN_TIMER = 3,
  APP_STATE_THERMOMETER = 4,
} app_state_e;

// Wi-Fi
const char *ssid = "xxx";
const char *password = "yyy";

// NTP
const char *ntp_server_1st = "ntp.nict.jp";
const char *ntp_server_2nd = "time.google.com";
const long gmt_offset_sec = 9 * 3600; // 時差（秒換算）
const int daylight_offset_sec = 0;    // 夏時間

// Acceleration sensor
MPU9250 IMU;
M5Timer Task_Timer;

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

const uint16_t digits_V_long[] =
    {
        0b0000001111111111, // 0
        0b0000001111000000, // 1
        0b0000011100111001, // 2
        0b0000011111100001, // 3
        0b0000011111000110, // 4
        0b0000010011100111, // 5
        0b0000010011111111, // 6
        0b0000001111000111, // 7
        0b0000011111111111, // 8
        0b0000011111100111, // 9
        0b0000000000000000, // off
};

// スクリーンの解像度は 横320 x 高さ240 で、左上が原点(0, 0)です
void drawNumberVLong(uint8_t x_start, uint8_t y_start, uint8_t number, uint8_t bar_width, uint8_t bar_length, uint8_t bar_gap, uint8_t corner_radius, uint16_t color_value)
{
  if (number > 10)
  {
    number = 10;
  }
  // top
  if (digits_V_long[number] & 0b0000000000000001)
    M5.Lcd.fillRoundRect(x_start, y_start, bar_length, bar_width, corner_radius, color_value);
  // upper-left
  if (digits_V_long[number] & 0b0000000000000010)
    M5.Lcd.fillRoundRect((x_start - bar_gap * 2), (y_start + bar_gap), bar_width, bar_length, corner_radius, color_value);
  if (digits_V_long[number] & 0b0000000000000100)
    M5.Lcd.fillRoundRect((x_start - bar_gap * 2), (y_start + bar_gap + bar_length * 1), bar_width, bar_length, corner_radius, color_value);
  // under-left
  if (digits_V_long[number] & 0b0000000000001000)
    M5.Lcd.fillRoundRect((x_start - bar_gap * 2), (y_start + bar_gap + bar_length * 2), bar_width, bar_length, corner_radius, color_value);
  if (digits_V_long[number] & 0b0000000000010000)
    M5.Lcd.fillRoundRect((x_start - bar_gap * 2), (y_start + bar_gap + bar_length * 3), bar_width, bar_length, corner_radius, color_value);
  // bottom
  if (digits_V_long[number] & 0b0000000000100000)
    M5.Lcd.fillRoundRect(x_start, (y_start + bar_length * 4), bar_length, bar_width, corner_radius, color_value);
  // under-right
  if (digits_V_long[number] & 0b0000000001000000)
    M5.Lcd.fillRoundRect((x_start + bar_length), (y_start + bar_gap + bar_length * 3), bar_width, bar_length, corner_radius, color_value);
  if (digits_V_long[number] & 0b0000000010000000)
    M5.Lcd.fillRoundRect((x_start + bar_length), (y_start + bar_gap + bar_length * 2), bar_width, bar_length, corner_radius, color_value);
  // upper-right
  if (digits_V_long[number] & 0b0000000100000000)
    M5.Lcd.fillRoundRect((x_start + bar_length), (y_start + bar_gap + bar_length * 1), bar_width, bar_length, corner_radius, color_value);
  if (digits_V_long[number] & 0b0000001000000000)
    M5.Lcd.fillRoundRect((x_start + bar_length), (y_start + bar_gap), bar_width, bar_length, corner_radius, color_value);
  // center
  if (digits_V_long[number] & 0b0000010000000000)
    M5.Lcd.fillRoundRect(x_start, (y_start + bar_length * 2), bar_length, bar_width, corner_radius, color_value);
}

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
#define LCD_DISP_BLINK_LOOP_CNT (5)
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

// Alarm Screen
#define LCD_ALARM_HM_DISP_Y_POS (150)
#define LCD_ALARM_STR_DISP_Y_POS (100)
#define LCD_ALARM_CLOCK_ICON_DISP_Y_POS (105)
#define LCD_ALARM_BELL_ICON_DISP_Y_POS (110)

#define LCD_ALARM_H_DISP_TENS_PLACE_X_POS (15)
#define LCD_ALARM_H_DISP_ONES_PLACE_X_POS (70)
#define LCD_ALARM_M_DISP_TENS_PLACE_X_POS (140)
#define LCD_ALARM_M_DISP_ONES_PLACE_X_POS (195)
#define LCD_ALARM_HM_DISP_COLON_X_POS (120)

typedef enum
{
  ALARM_NOT_SET = 0,
  ALARM_RUNNING = 1,
  ALARM_SETTING_HOUR = 2,
  ALARM_SETTING_MINUTE = 3,
} alarm_status_e;
alarm_status_e e_alarm_status = ALARM_NOT_SET;

void displayAlarmScreen()
{
  static uint8_t alarm_hour = 0;
  static uint8_t alarm_minute = 0;
  static int16_t callback_alarm_slot_id = -1;

  uint16_t bkground_color = M5.Lcd.color565(0, 180, 0);
  if (is_state_changed == true)
  {
    M5.Lcd.fillScreen(bkground_color);
    is_state_changed = false;
  }

  M5.Lcd.setTextColor(TFT_BLACK);
  M5.Lcd.setTextSize(2);

  M5.Lcd.drawString("ALARM", 80, LCD_ALARM_STR_DISP_Y_POS);

  // alarm icon
  drawAlarmIcon(55, LCD_ALARM_CLOCK_ICON_DISP_Y_POS);

  switch (e_alarm_status)
  {
  case ALARM_NOT_SET:
  default:
    drawNumberNormal(LCD_ALARM_H_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_H_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    if (M5.BtnA.wasReleased())
    {
      e_alarm_status = ALARM_SETTING_HOUR;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;

  case ALARM_SETTING_HOUR:
    if (cl_blink_count.isHideDisplay() == false)
    {
      drawNumberNormal(LCD_ALARM_H_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
      drawNumberNormal(LCD_ALARM_H_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    }
    else
    {
      drawNumberNormal(LCD_ALARM_H_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
      drawNumberNormal(LCD_ALARM_H_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    }
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    if (M5.BtnC.wasReleased())
    {
      ++alarm_hour;
      if (alarm_hour > 23)
      {
        alarm_hour = 0;
      }
      M5.Lcd.fillScreen(bkground_color);
    }
    else if (M5.BtnA.wasReleased())
    {
      e_alarm_status = ALARM_SETTING_MINUTE;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;

  case ALARM_SETTING_MINUTE:
    drawNumberNormal(LCD_ALARM_H_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_H_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    if (cl_blink_count.isHideDisplay() == false)
    {
      drawNumberNormal(LCD_ALARM_M_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
      drawNumberNormal(LCD_ALARM_M_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    }
    else
    {
      drawNumberNormal(LCD_ALARM_M_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
      drawNumberNormal(LCD_ALARM_M_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    }

    if (M5.BtnC.wasReleased())
    {
      ++alarm_minute;
      if (alarm_minute > 59)
      {
        alarm_minute = 0;
      }
      M5.Lcd.fillScreen(bkground_color);
    }
    else if (M5.BtnA.wasReleased())
    {
      callback_alarm_slot_id = enableAlarm(alarm_hour, alarm_minute);
      e_alarm_status = ALARM_RUNNING;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;

  case ALARM_RUNNING:
    drawNumberNormal(LCD_ALARM_H_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_H_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(LCD_ALARM_HM_DISP_COLON_X_POS, LCD_ALARM_HM_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_TENS_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(LCD_ALARM_M_DISP_ONES_PLACE_X_POS, LCD_ALARM_HM_DISP_Y_POS, (alarm_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    // bell icon
    drawBellIcon(20, LCD_ALARM_BELL_ICON_DISP_Y_POS);

    if (M5.BtnA.wasReleased())
    {
      disableAlarm(callback_alarm_slot_id);
      callback_alarm_slot_id = -1;
      e_alarm_status = ALARM_NOT_SET;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;
  }
}

// Date-time Screen
// typedef struct
// {
//   uint32_t npt_year = 0;
//   uint32_t ntp_month = 0;
//   uint32_t ntp_day = 0;
//   uint32_t npt_hour = 0;
//   uint32_t npt_minute = 0;
// } ntp_date_t;

#define LCD_CLOCK_YMD_DISP_Y_POS (10)
#define LCD_CLOCK_MD_STR_DISP_Y_POS (45)
#define LCD_CLOCK_HM_DISP_Y_POS (100)
#define LCD_CLOCK_PM_STR_DISP_Y_POS (175)
#define LCD_CLOCK_ICON_DISP_Y_POS (200)
#define LCD_CLOCK_WEEK_STR_DISP_Y_POS (220)

void displayDateTimeScreen()
{
  // ntp_date_t t_date_now;
  // static ntp_date_t t_date_prev;
  static boolean ntp_access_flag = true;
  static uint32_t base_milli_time;
  uint32_t elasped_second = 0;
  uint32_t diff_milli_time = 0;

  uint16_t bkground_color = M5.Lcd.color565(200, 0, 0);
  if (is_state_changed == true)
  {
    M5.Lcd.fillScreen(bkground_color);
    ntp_access_flag = true;
    is_state_changed = false;
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
    // Serial.print("diff_milli_time:");
    // Serial.println(diff_milli_time);
    elasped_second = diff_milli_time / 1000;
    cl_system_clock.updateBySoftTimer(elasped_second);
  }

  M5.Lcd.setTextColor(TFT_BLACK);
  M5.Lcd.setTextSize(2);

  // Month
  if (cl_system_clock.month != cl_system_clock.prev_month)
  {
    drawNumberNormal(10, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(35, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(10, LCD_CLOCK_YMD_DISP_Y_POS, (cl_system_clock.month / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(35, LCD_CLOCK_YMD_DISP_Y_POS, (cl_system_clock.month % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);

  M5.Lcd.drawLine(60, LCD_SMALL_BAR_LENGTH * 2 + 10, 70, 10, TFT_BLACK);

  // Day
  if (cl_system_clock.day != cl_system_clock.prev_day)
  {
    drawNumberNormal(80, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(105, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(80, LCD_CLOCK_YMD_DISP_Y_POS, (cl_system_clock.day / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(105, LCD_CLOCK_YMD_DISP_Y_POS, (cl_system_clock.day % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);

  // Year
  if (cl_system_clock.year != cl_system_clock.prev_year)
  {
    drawNumberNormal(180, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(205, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(230, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(255, LCD_CLOCK_YMD_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(180, LCD_CLOCK_YMD_DISP_Y_POS, (cl_system_clock.year / 1000), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(205, LCD_CLOCK_YMD_DISP_Y_POS, ((cl_system_clock.year % 1000) / 100), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(230, LCD_CLOCK_YMD_DISP_Y_POS, (((cl_system_clock.year % 1000) % 100) / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(255, LCD_CLOCK_YMD_DISP_Y_POS, (((cl_system_clock.year % 1000) % 100) % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);

  // hour
  if (cl_system_clock.hour != cl_system_clock.prev_hour)
  {
    drawNumberNormal(30, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(95, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    M5.Lcd.setTextColor(bkground_color);
    M5.Lcd.drawString("PM", 20, LCD_CLOCK_PM_STR_DISP_Y_POS);
    M5.Lcd.setTextColor(TFT_BLACK);
  }
  drawNumberNormal(30, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.hour / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(95, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.hour % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
  // if (isPM == true)
  // {
  //   M5.Lcd.drawString("PM", 20, LCD_CLOCK_PM_STR_DISP_Y_POS);
  // }

  // Sec
  // if (cl_system_clock.second != cl_system_clock.prev_second)
  // {
  //   drawNumberNormal(180, LCD_CLOCK_YMD_DISP_Y_POS + 40, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  //   drawNumberNormal(205, LCD_CLOCK_YMD_DISP_Y_POS + 40, LCD_DIGITS_CLEAR_ELM_NO, LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, bkground_color);
  // }
  // drawNumberNormal(180, LCD_CLOCK_YMD_DISP_Y_POS + 40, (cl_system_clock.second / 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);
  // drawNumberNormal(205, LCD_CLOCK_YMD_DISP_Y_POS + 40, (cl_system_clock.second % 10), LCD_SMALL_BAR_WIDTH, LCD_SMALL_BAR_LENGTH, LCD_SMALL_BAR_GAP, LCD_SMALL_BAR_CORNER_RADIUS, TFT_BLACK);

  if (cl_blink_count.isHideDisplay() == false)
  {
    M5.Lcd.fillEllipse(150, 120, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(150, 150, 4, 4, TFT_BLACK);
  }
  else
  {
    M5.Lcd.fillEllipse(150, 120, 4, 4, bkground_color);
    M5.Lcd.fillEllipse(150, 150, 4, 4, bkground_color);
  }

  // minute
  if (cl_system_clock.minute != cl_system_clock.prev_minute)
  {
    drawNumberNormal(185, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    drawNumberNormal(250, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberNormal(185, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberNormal(250, LCD_CLOCK_HM_DISP_Y_POS, (cl_system_clock.minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

  // MONTH DATE
  const String s_month = "MONTH";
  const String s_date = "DATE";
  M5.Lcd.drawString(s_month, 10, LCD_CLOCK_MD_STR_DISP_Y_POS);
  M5.Lcd.drawString(s_date, 80, LCD_CLOCK_MD_STR_DISP_Y_POS);

  uint8_t week_count = 0;
  const char aweek[7][4] = {"SUN", "MON", "TUE", "WED", "THU", "FRI", "SAT"};
  uint8_t day_week_now = 0;
  // day_week_now = subZeller(t_date_now.npt_year, t_date_now.ntp_month, t_date_now.ntp_day);
  day_week_now = cl_system_clock.week_day;
  for (week_count = 0; week_count < 7; ++week_count)
  {
    if ((week_count == day_week_now) && (cl_blink_count.isHideDisplay() == true))
    {
      M5.Lcd.setTextColor(bkground_color);
      M5.Lcd.drawString(aweek[week_count], week_count * 45 + 10, LCD_CLOCK_WEEK_STR_DISP_Y_POS);
      M5.Lcd.setTextColor(TFT_BLACK);
    }
    else
    {
      M5.Lcd.drawString(aweek[week_count], week_count * 45 + 10, LCD_CLOCK_WEEK_STR_DISP_Y_POS);
    }
  }

  // clock icon
  drawClockIcon(300, LCD_CLOCK_ICON_DISP_Y_POS);

  // t_date_prev = t_date_now;
  cl_system_clock.backupCurrentTime();
}

// Thermometer Screen
#define LCD_THERMOMETER_ICON_DISP_Y_POS (80)
#define LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS (80)

void displayThermometerScreen()
{
  uint16_t bkground_color = M5.Lcd.color565(128, 128, 255);
  static uint8_t temperature_prev = 0;
  if (is_state_changed == true)
  {
    M5.Lcd.fillScreen(bkground_color);
    is_state_changed = false;
  }

  // draw thermometer icon
  drawThermometerIcon(40, LCD_THERMOMETER_ICON_DISP_Y_POS, bkground_color);

  // get temperature from MPU9250
  IMU.tempCount = IMU.readTempData(); // Read the adc values
  // Temperature in degrees Centigrade
  IMU.temperature = ((float)IMU.tempCount) / 333.87 + 21.0;
  uint8_t temperature = (int)IMU.temperature;
  uint8_t tens_place, ones_place;
  if (temperature < 10)
  {
    tens_place = 0;
    ones_place = temperature;
  }
  else if (temperature <= 99)
  {
    tens_place = temperature / 10;
    ones_place = temperature % 10;
  }
  else
  {
    tens_place = 9;
    ones_place = 9;
  }

  if (temperature != temperature_prev)
  {
    drawNumberVLong(60, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    drawNumberVLong(130, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    // drawNumberNormal(185, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    // drawNumberNormal(250, LCD_CLOCK_HM_DISP_Y_POS, LCD_DIGITS_CLEAR_ELM_NO, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
  }
  drawNumberVLong(60, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, tens_place, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
  drawNumberVLong(130, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, ones_place, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

  M5.Lcd.drawEllipse(130 + LCD_LARGE_BAR_WIDTH + LCD_LARGE_BAR_LENGTH + LCD_LARGE_BAR_GAP, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, 3, 3, TFT_BLACK);
  M5.Lcd.drawChar(130 + LCD_LARGE_BAR_WIDTH + LCD_LARGE_BAR_LENGTH + LCD_LARGE_BAR_GAP + 10, LCD_THERMOMETER_TEMPERATURE_DISP_Y_POS, 'C', TFT_BLACK, bkground_color, 4);

  temperature_prev = temperature;
}

// Timer Screen
#define LCD_SANDGLASS_ICON_DISP_Y_POS (30)
#define LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS (70)
#define LCD_CNT_DOWN_TIMER_STR_DISP_Y_POS (150)
#define LCD_CNT_DOWN_TIMER_ICON_DISP_Y_POS (200)

typedef enum
{
  CNT_DOWN_TIMER_NOT_SET = 0,
  CNT_DOWN_TIMER_RUNNING = 1,
  CNT_DOWN_TIMER_SETTING_MINUTE = 2,
  CNT_DOWN_TIMER_SETTING_SECOND = 3,
  CNT_DOWN_TIMER_RINGING = 4,
} count_down_timer_status_e;

void displayCountdownTimerScreen()
{
  static count_down_timer_status_e e_cnt_timer_status = CNT_DOWN_TIMER_NOT_SET;
  static uint8_t timer_minute = 0;
  static uint8_t timer_second = 0;
  static uint32_t base_milli_time = 0;
  static uint32_t set_milli_time = 0;
  static boolean is_count_down = false;

  uint16_t bkground_color = M5.Lcd.color565(0, 180, 0);
  if (is_state_changed == true)
  {
    M5.Lcd.fillScreen(bkground_color);
    is_state_changed = false;
    is_count_down = false;
    e_cnt_timer_status = CNT_DOWN_TIMER_RUNNING;
  }

  // draw sandglass icon
  if (cl_blink_count.isHideDisplay() == false)
  {
    drawSandglassIcon(240, LCD_SANDGLASS_ICON_DISP_Y_POS, TFT_BLACK);
  }
  else
  {
    drawSandglassIcon(240, LCD_SANDGLASS_ICON_DISP_Y_POS, bkground_color);
  }

  M5.Lcd.setTextColor(TFT_BLACK);
  M5.Lcd.setTextSize(2);

  M5.Lcd.drawString("MIN", 20, LCD_CNT_DOWN_TIMER_STR_DISP_Y_POS);
  M5.Lcd.drawString("SEC", 250, LCD_CNT_DOWN_TIMER_STR_DISP_Y_POS);

  switch (e_cnt_timer_status)
  {
  case CNT_DOWN_TIMER_NOT_SET:
  default:
    drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    if (M5.BtnA.wasReleased())
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_SETTING_MINUTE;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;

  case CNT_DOWN_TIMER_SETTING_MINUTE:
    if (cl_blink_count.isHideDisplay() == false)
    {
      drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
      drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    }
    else
    {
      drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
      drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    }
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    if (M5.BtnC.wasReleased())
    {
      ++timer_minute;
      if (timer_minute > 99)
      {
        timer_minute = 0;
      }
      M5.Lcd.fillScreen(bkground_color);
    }
    else if (M5.BtnA.wasReleased())
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_SETTING_SECOND;
      M5.Lcd.fillScreen(bkground_color);
    }
    break;

  case CNT_DOWN_TIMER_SETTING_SECOND:
    drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    if (cl_blink_count.isHideDisplay() == false)
    {
      drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
      drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    }
    else
    {
      drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
      drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
    }

    if (M5.BtnC.wasReleased())
    {
      ++timer_second;
      if (timer_second > 59)
      {
        timer_second = 0;
      }
      M5.Lcd.fillScreen(bkground_color);
    }
    else if (M5.BtnA.wasReleased())
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_RUNNING;
      M5.Lcd.fillScreen(bkground_color);
      cl_blink_count.resetCount();
    }
    break;

  case CNT_DOWN_TIMER_RUNNING:
    if ((timer_minute == 0) && (timer_second == 0))
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_NOT_SET;
    }
    else
    {
      uint8_t disp_minute = 0;
      uint8_t disp_second = 0;
      static uint8_t pre_disp_minute = 0;
      static uint8_t pre_disp_second = 0;
      if (is_count_down == false)
      {
        is_count_down = true;
        base_milli_time = millis();
        set_milli_time = base_milli_time + timer_second * 1000 + timer_minute * 60 * 1000;

        drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
        drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
        M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
        M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
        drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
        drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (timer_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
        pre_disp_minute = timer_minute;
        pre_disp_second = timer_second;
      }
      else
      {
        int32_t diff_milli_time = set_milli_time - millis();
        if (diff_milli_time < 0)
        {
          e_cnt_timer_status = CNT_DOWN_TIMER_RINGING;
          is_count_down = false;
        }
        else
        {
          disp_minute = diff_milli_time / 1000 / 60;
          disp_second = diff_milli_time / 1000 % 60;

          if (disp_minute != pre_disp_minute)
          {
            drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (pre_disp_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
            drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (pre_disp_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
          }
          if (disp_second != pre_disp_second)
          {
            drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (pre_disp_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
            drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (pre_disp_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, bkground_color);
          }

          drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (disp_minute / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
          drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (disp_minute % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
          M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
          M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
          drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (disp_second / 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
          drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, (disp_second % 10), LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

          pre_disp_minute = disp_minute;
          pre_disp_second = disp_second;
        }
      }

      if (M5.BtnA.wasReleased())
      {
        e_cnt_timer_status = CNT_DOWN_TIMER_SETTING_MINUTE;
        timer_minute = 0;
        timer_second = 0;
        is_count_down = false;
        M5.Lcd.fillScreen(bkground_color);
        cl_blink_count.resetCount();
      }
    }
    break;

  case CNT_DOWN_TIMER_RINGING:
    drawNumberNormal(30, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(95, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 20, 4, 4, TFT_BLACK);
    M5.Lcd.fillEllipse(150, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS + 50, 4, 4, TFT_BLACK);
    drawNumberNormal(185, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);
    drawNumberNormal(250, LCD_CNT_DOWN_TIMER_MS_DISP_Y_POS, 0, LCD_LARGE_BAR_WIDTH, LCD_LARGE_BAR_LENGTH, LCD_LARGE_BAR_GAP, LCD_LARGE_BAR_CORNER_RADIUS, TFT_BLACK);

    M5.Lcd.setTextColor(TFT_WHITE);
    M5.Lcd.setTextSize(4);
    M5.Lcd.drawString("RINGING", 20, 180);

    if (M5.BtnA.wasReleased())
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_SETTING_MINUTE;
      timer_minute = 0;
      timer_second = 0;
      M5.Lcd.fillScreen(bkground_color);
      cl_blink_count.resetCount();
    }
    else if (M5.BtnC.wasReleased())
    {
      e_cnt_timer_status = CNT_DOWN_TIMER_NOT_SET;
      timer_minute = 0;
      timer_second = 0;
      M5.Lcd.fillScreen(bkground_color);
      cl_blink_count.resetCount();
    }
    break;
  }
}

// Draw Icon API
void drawAlarmIcon(uint32_t base_x_pos, uint32_t base_y_pos)
{
  const int32_t offset = 14;

  M5.Lcd.drawEllipse(base_x_pos, base_y_pos, 7, 7, TFT_BLACK);
  // 短針
  M5.Lcd.drawLine(base_x_pos, base_y_pos, base_x_pos - 2, base_y_pos - 2, TFT_BLACK);
  M5.Lcd.drawLine(base_x_pos, base_y_pos + 1, base_x_pos - 2, base_y_pos - 1, TFT_BLACK);
  // 長針
  M5.Lcd.drawLine(base_x_pos, base_y_pos, base_x_pos + 4, base_y_pos - 3, TFT_BLACK);
  M5.Lcd.drawLine(base_x_pos, base_y_pos + 1, base_x_pos + 4, base_y_pos - 2, TFT_BLACK);

  // 左足
  M5.Lcd.drawLine(base_x_pos - 5, base_y_pos + 5, base_x_pos - 7, base_y_pos + 7, TFT_BLACK);
  M5.Lcd.drawLine(base_x_pos - 5, base_y_pos + 6, base_x_pos - 7, base_y_pos + 8, TFT_BLACK);
  // 右足
  M5.Lcd.drawLine(base_x_pos + 5, base_y_pos + 5, base_x_pos + 7, base_y_pos + 7, TFT_BLACK);
  M5.Lcd.drawLine(base_x_pos + 5, base_y_pos + 6, base_x_pos + 7, base_y_pos + 8, TFT_BLACK);

  // 角
  M5.Lcd.fillRect(base_x_pos - 3, base_y_pos - 11, 6, 2, TFT_BLACK);
  M5.Lcd.fillRect(base_x_pos - 2, base_y_pos - 9, 4, 1, TFT_BLACK);

  // 左角
  M5.Lcd.fillTriangle(base_x_pos - 8, base_y_pos - 5, base_x_pos - 5, base_y_pos - 8, base_x_pos - 7, base_y_pos - 8, TFT_BLACK);
  // 右角
  M5.Lcd.fillTriangle(base_x_pos + 8, base_y_pos - 5, base_x_pos + 5, base_y_pos - 8, base_x_pos + 7, base_y_pos - 8, TFT_BLACK);
}

void drawBellIcon(uint32_t base_x_pos, uint32_t base_y_pos)
{
  // 上丸
  M5.Lcd.fillEllipse(base_x_pos + 6, base_y_pos + 2, 2, 2, TFT_BLACK);

  // 胴体上
  M5.Lcd.fillRect(base_x_pos, base_y_pos + 3, 13, 10, TFT_BLACK);

  // 胴体下
  M5.Lcd.fillTriangle(base_x_pos, base_y_pos + 14, base_x_pos, base_y_pos + 17, base_x_pos - 4, base_y_pos + 17, TFT_BLACK);
  M5.Lcd.fillTriangle(base_x_pos + 13, base_y_pos + 14, base_x_pos + 13, base_y_pos + 17, base_x_pos + 17, base_y_pos + 17, TFT_BLACK);
  M5.Lcd.fillRect(base_x_pos, base_y_pos + 14, 13, 4, TFT_BLACK);

  // 下丸
  M5.Lcd.fillEllipse(base_x_pos + 6, base_y_pos + 20, 2, 2, TFT_BLACK);
}

void drawClockIcon(uint32_t base_x_pos, uint32_t base_y_pos)
{
  const int32_t offset = 14;
  //   const uint8_t dot_size = 2;

  M5.Lcd.drawPixel(base_x_pos, base_y_pos - offset, TFT_BLACK);
  M5.Lcd.drawPixel(base_x_pos - offset, base_y_pos, TFT_BLACK);
  M5.Lcd.drawPixel(base_x_pos, base_y_pos + offset, TFT_BLACK);
  M5.Lcd.drawPixel(base_x_pos + offset, base_y_pos, TFT_BLACK);

  M5.Lcd.drawPixel(base_x_pos - 7, base_y_pos + 12, TFT_BLACK);
  M5.Lcd.drawPixel(base_x_pos - 12, base_y_pos + 7, TFT_BLACK);
  M5.Lcd.drawPixel(base_x_pos - 7, base_y_pos - 12, TFT_BLACK);
  M5.Lcd.drawPixel(base_x_pos - 12, base_y_pos - 7, TFT_BLACK);

  M5.Lcd.drawPixel(base_x_pos + 7, base_y_pos + 12, TFT_BLACK);
  M5.Lcd.drawPixel(base_x_pos + 12, base_y_pos + 7, TFT_BLACK);
  M5.Lcd.drawPixel(base_x_pos + 7, base_y_pos - 12, TFT_BLACK);
  M5.Lcd.drawPixel(base_x_pos + 12, base_y_pos - 7, TFT_BLACK);

  // 短針
  M5.Lcd.drawLine(base_x_pos, base_y_pos, 296, 198, TFT_BLACK);
  M5.Lcd.drawLine(base_x_pos, base_y_pos + 1, 296, 199, TFT_BLACK);
  // 長針
  M5.Lcd.drawLine(base_x_pos, base_y_pos, 306, 195, TFT_BLACK);
  M5.Lcd.drawLine(base_x_pos, base_y_pos + 1, 306, 196, TFT_BLACK);
  // M5.Lcd.fillEllipse(160, 120, 3, 3, TFT_BLACK);
}

void drawThermometerIcon(uint32_t base_x_pos, uint32_t base_y_pos, uint16_t bk_color)
{
  M5.Lcd.drawEllipse(base_x_pos, base_y_pos, 7, 7, TFT_BLACK);
  M5.Lcd.drawRoundRect(base_x_pos - 3, base_y_pos - 35, 8, 35, 4, TFT_BLACK);
  M5.Lcd.fillEllipse(base_x_pos, base_y_pos, 6, 6, bk_color);
  M5.Lcd.fillEllipse(base_x_pos, base_y_pos, 4, 4, TFT_BLACK);
  M5.Lcd.fillRoundRect(base_x_pos - 1, base_y_pos - 35 + 8, 4, 25, 2, TFT_BLACK);
}

void drawSandglassIcon(uint32_t base_x_pos, uint32_t base_y_pos, uint16_t color)
{
  // 上三角
  M5.Lcd.drawTriangle(base_x_pos, base_y_pos, base_x_pos + 20, base_y_pos, base_x_pos + 10, base_y_pos + 14, color);
  // 下三角
  M5.Lcd.drawTriangle(base_x_pos, base_y_pos + 28, base_x_pos + 20, base_y_pos + 28, base_x_pos + 10, base_y_pos + 14, color);

  // 上砂
  M5.Lcd.fillTriangle(base_x_pos + 6, base_y_pos + 4, base_x_pos + 20 - 6, base_y_pos + 4, base_x_pos + 10, base_y_pos + 14 - 4, color);
  // 下砂
  M5.Lcd.fillTriangle(base_x_pos + 2, base_y_pos + 28 - 2, base_x_pos + 20 - 2, base_y_pos + 28 - 2, base_x_pos + 10, base_y_pos + 14 + 8, color);
}

void wakeUpTime()
{
  uint16_t bkground_color = M5.Lcd.color565(255, 255, 255);
  M5.Lcd.fillScreen(bkground_color);
  M5.Lcd.setTextColor(TFT_BLACK);
  M5.Lcd.setTextSize(6);
  M5.Lcd.drawString("WAKE", 50, 100);
  M5.Lcd.drawString(" UP ", 50, 170);
  while (1)
  {
    M5.update();
    if (M5.BtnA.wasReleased() || M5.BtnB.wasReleased() || M5.BtnC.wasReleased())
    {
      is_state_changed = true;
      e_alarm_status = ALARM_NOT_SET;
      break;
    }
    delay(100);
  }
}

// // 曜日を算出
// uint8_t subZeller(uint32_t year, uint32_t month, uint32_t day)
// {
//   if (month < 3)
//   {
//     --year;
//     month += 12;
//   }
//   return (year + (year / 4) - (year / 100) + (year / 400) + ((13 * month + 8) / 5) + day) % 7;
// }

int16_t enableAlarm(uint8_t hour, uint8_t minute)
{
  int32_t set_time_minute = hour * 60 + minute;
  int32_t now_time_minute = 0;
  int32_t timer_ms = 0;
  int16_t callback_slot_id = -1;

  struct tm time_info;
  if (!getLocalTime(&time_info))
  {
    M5.Lcd.drawString("ERR", 0, 190);
  }
  else
  {
    now_time_minute = time_info.tm_hour * 60 + time_info.tm_min;
    if (set_time_minute > now_time_minute)
    {
      timer_ms = ((set_time_minute - now_time_minute) * 60 - time_info.tm_sec) * 1000;
    }
    else
    {
      timer_ms = (((24 * 60) - (now_time_minute - set_time_minute)) * 60 - time_info.tm_sec) * 1000;
    }
    callback_slot_id = Task_Timer.setTimeout(timer_ms, wakeUpTime);
  }
  return callback_slot_id;
}

void disableAlarm(int16_t callback_slot_id)
{
  if (callback_slot_id != -1)
  {
    Task_Timer.deleteTimer(callback_slot_id);
  }
}

int32_t getAngle()
{
  int32_t angle = 0;
  if (IMU.readByte(MPU9250_ADDRESS, INT_STATUS) & 0x01)
  {
    IMU.readAccelData(IMU.accelCount);
    IMU.getAres();

    IMU.ax = (float)IMU.accelCount[0] * IMU.aRes; // - accelBias[0];
    IMU.ay = (float)IMU.accelCount[1] * IMU.aRes; // - accelBias[1];
    IMU.az = (float)IMU.accelCount[2] * IMU.aRes; // - accelBias[2];

    // 傾斜角算出
    double angle_XY_direction = 0.0;
    angle_XY_direction = atan2(IMU.ay, IMU.ax);
    double angle_XY_direction_deg = angle_XY_direction * 180.0 / (M_PI);
    angle = round(angle_XY_direction_deg) + 180 + 90;
    if (angle > 360)
    {
      angle -= 360;
    }
  }
  else
  {
    angle = -1;
  }
  return angle;
}

#define DATE_TIME_CLOCK_REF_ANGLE (0)
#define ALARM_CLOCK_REF_ANGLE (270)
#define COUNT_DOWN_TIMER_REF_ANGLE (180)
#define THERMOMETER_REF_ANGLE (90)

#define SWITCH_APP_ANGLE_RANGE (35)
#define CURRENT_APP_ANGLE_RANGE (75)

app_state_e calcCurrentAppState(int32_t angle)
{
  app_state_e e_current_state = APP_STATE_INDEFINITE;
  static app_state_e e_prev_state = APP_STATE_DATE_TIME_CLOCK;

  switch (e_prev_state)
  {
  case APP_STATE_INDEFINITE:
  case APP_STATE_DATE_TIME_CLOCK:
  default:
    if (((DATE_TIME_CLOCK_REF_ANGLE + CURRENT_APP_ANGLE_RANGE) <= angle) && (angle < (COUNT_DOWN_TIMER_REF_ANGLE - SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_ALARM_CLOCK;
      is_state_changed = true;
    }
    else if (((COUNT_DOWN_TIMER_REF_ANGLE - SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (COUNT_DOWN_TIMER_REF_ANGLE + SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_COUNT_DOWN_TIMER;
      is_state_changed = true;
    }
    else if (((COUNT_DOWN_TIMER_REF_ANGLE + SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (360 - CURRENT_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_THERMOMETER;
      is_state_changed = true;
    }
    else
    {
      e_current_state = APP_STATE_DATE_TIME_CLOCK;
    }
    break;

  case APP_STATE_THERMOMETER:
    if (((ALARM_CLOCK_REF_ANGLE + CURRENT_APP_ANGLE_RANGE) <= angle) || (angle < (THERMOMETER_REF_ANGLE - SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_DATE_TIME_CLOCK;
      is_state_changed = true;
    }
    else if (((THERMOMETER_REF_ANGLE - SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (THERMOMETER_REF_ANGLE + SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_ALARM_CLOCK;
      is_state_changed = true;
    }
    else if (((THERMOMETER_REF_ANGLE + SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (ALARM_CLOCK_REF_ANGLE - CURRENT_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_COUNT_DOWN_TIMER;
      is_state_changed = true;
    }
    else
    {
      e_current_state = APP_STATE_THERMOMETER;
    }
    break;

  case APP_STATE_COUNT_DOWN_TIMER:
    if (((COUNT_DOWN_TIMER_REF_ANGLE + CURRENT_APP_ANGLE_RANGE) <= angle) && (angle < (360 - SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_THERMOMETER;
      is_state_changed = true;
    }
    else if (((360 - SWITCH_APP_ANGLE_RANGE) <= angle) || (angle < (DATE_TIME_CLOCK_REF_ANGLE + SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_DATE_TIME_CLOCK;
      is_state_changed = true;
    }
    else if (((DATE_TIME_CLOCK_REF_ANGLE + SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (COUNT_DOWN_TIMER_REF_ANGLE - CURRENT_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_ALARM_CLOCK;
      is_state_changed = true;
    }
    else
    {
      e_current_state = APP_STATE_COUNT_DOWN_TIMER;
    }
    break;

  case APP_STATE_ALARM_CLOCK:
    if (((THERMOMETER_REF_ANGLE + CURRENT_APP_ANGLE_RANGE) <= angle) && (angle < (ALARM_CLOCK_REF_ANGLE - SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_COUNT_DOWN_TIMER;
      is_state_changed = true;
    }
    else if (((ALARM_CLOCK_REF_ANGLE - SWITCH_APP_ANGLE_RANGE) <= angle) && (angle < (ALARM_CLOCK_REF_ANGLE + SWITCH_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_THERMOMETER;
      is_state_changed = true;
    }
    else if (((ALARM_CLOCK_REF_ANGLE + SWITCH_APP_ANGLE_RANGE) <= angle) || (angle < (THERMOMETER_REF_ANGLE - CURRENT_APP_ANGLE_RANGE)))
    {
      e_current_state = APP_STATE_DATE_TIME_CLOCK;
      is_state_changed = true;
    }
    else
    {
      e_current_state = APP_STATE_ALARM_CLOCK;
    }
    break;
  }

  e_prev_state = e_current_state;
  return e_current_state;
}

// void displayAngleScreenTest()
// {
//   uint16_t bkground_color = M5.Lcd.color565(255, 255, 255);
//   if (is_state_changed == true)
//   {
//     M5.Lcd.fillScreen(bkground_color);
//     is_state_changed = false;
//   }

//   if (IMU.readByte(MPU9250_ADDRESS, INT_STATUS) & 0x01)
//   {
//     IMU.readAccelData(IMU.accelCount);
//     IMU.getAres();

//     IMU.ax = (float)IMU.accelCount[0] * IMU.aRes; // - accelBias[0];
//     IMU.ay = (float)IMU.accelCount[1] * IMU.aRes; // - accelBias[1];
//     IMU.az = (float)IMU.accelCount[2] * IMU.aRes; // - accelBias[2];

//     // 傾斜角算出
//     double angle_XY_direction = 0.0;
//     angle_XY_direction = atan2(IMU.ay, IMU.ax);
//     double angle_XY_direction_deg = angle_XY_direction * 180.0 / (M_PI);
//     int32_t angle = round(angle_XY_direction_deg) + 180 + 90;
//     if (angle > 360)
//     {
//       angle -= 360;
//     }
//     String deg_str = String(angle);
//     M5.Lcd.fillScreen(bkground_color);
//     M5.Lcd.setTextColor(TFT_BLACK);
//     M5.Lcd.setTextSize(6);
//     M5.Lcd.drawString(deg_str, 0, 100);
//   }
//   else
//   {
//     //
//   }
// }

void setup()
{
  M5.begin();
  Wire.begin();
  // Serial.begin(115200);
  M5.Lcd.setBrightness(100);
  // Serial.println("hello world");

  // 加速度センサ関係初期化
  IMU.initMPU9250();
  IMU.calibrateMPU9250(IMU.gyroBias, IMU.accelBias);

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
  Serial.println("hello world");
}

void loop()
{
  static app_state_e e_app_state = APP_STATE_INDEFINITE;
  M5.update();
  Task_Timer.run();

#if 0
  displayAngleScreenTest();
  delay(10);
#else
  int32_t body_angle = getAngle();
  if (body_angle != -1)
  {
    e_app_state = calcCurrentAppState(body_angle);
  }

  switch (e_app_state)
  {
  case APP_STATE_ALARM_CLOCK:
    M5.Lcd.setRotation(0);
    displayAlarmScreen();
    break;

  case APP_STATE_DATE_TIME_CLOCK:
    M5.Lcd.setRotation(1);
    displayDateTimeScreen();
    break;

  case APP_STATE_THERMOMETER:
    M5.Lcd.setRotation(2);
    displayThermometerScreen();
    break;

  case APP_STATE_COUNT_DOWN_TIMER:
    M5.Lcd.setRotation(3);
    displayCountdownTimerScreen();
    break;

  case APP_STATE_INDEFINITE:
  default:
    break;
  }
  cl_blink_count.incrementCount();
  delay(100);
#endif
}
```
</div></details>

ソースコード中のssid関係は各自の設定に合わせてください。
最後にあらためて最終成果物でこんなのができるよっていうのをあげときます。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr"><a href="https://twitter.com/hashtag/Qiita?src=hash&amp;ref_src=twsrc%5Etfw">#Qiita</a> のM5Stack Advent Calendar記事投稿用<br>GIF化するよりこっちの方が動画埋め込むのらくそう。 <a href="https://t.co/l8MRaAz8kl">pic.twitter.com/l8MRaAz8kl</a></p>&mdash; yankee (@yankee_sns) <a href="https://twitter.com/yankee_sns/status/1201125638435823621?ref_src=twsrc%5Etfw">December 1, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

# ◇おわりに
当初予定に比べて思いのほかコードが膨大になってしまいました。。
元々はM5Stackの使い方に慣れる目的で作り始めたアプリでしたが、やはり既存の製品と同じようなモノを作るというのは大変だなと感じた今日この頃・・。
もし、今回のアプリをご自分のM5Stackにも入れてみたいと思われた方は試していただければ幸いです。
この後も続く**M5Stack Advent Calendar**の投稿を楽しみにしつつ、次の方につなぎたいと思います。
