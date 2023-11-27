---
title: M5Stack FIRE内蔵の加速度センサ使用時（Calibrationをライブラリ関数で実行する場合）の注意点
tags:
  - サンプルコード
  - ArduinoIDE
  - 加速度センサー
  - mpu9250
  - M5stack
private: false
updated_at: '2023-11-28T00:55:05+09:00'
id: 35542dcf095317ac4d7e
organization_url_name: null
slide: false
ignorePublish: false
---
# ◇はじめに

先日開催されていた、「スイッチサイエンスGW直前セール」で[M5Stack FIRE](https://docs.m5stack.com/#/ja/core/fire)を衝動買いしてしまったので、これから色々さわって遊んでみたいと思います。  
当初、
__開封～初期環境設定～簡単なアプリ開発__
までを一つの記事にしようと思っていたのですが、加速度センサの使用で一点ハマった部分があったため、その対策を備忘録代わりに載せておきます。

# ◇前提条件

この記事は、以下の条件でM5Stack FIREを動かしたい場合の注意点でそれ以外の方には該当しませんのでご注意ください。

+ M5Stack FIREをArduino IDEで使用する
+ M5Stack FIREに内蔵されている加速度センサ（MPU9250）を使用する
+ 加速度センサ（MPU9250）のキャリブレーションをM5Stackのライブラリ内にある`calibrateMPU9250()`関数を使って行う、もしくはスケッチ例の`M5StackFire_MPU9250.ino`をそのまま実行する

# ◇とりあえず結論

`calibrateMPU9250()`関数を実行するタイミング（`setup()`内で呼んでる場合は起動時）では、M5Stack FIREを

+ 液晶画面を上に向けた状態（液晶画面を上から覗き込む状態）で平らなところに置く  
or
+ 液晶画面を下に向けた状態（液晶画面が見えない状態）で平らなところに置く  

のどちらかにしておく必要があります。

※結論としてはこれで終わりですが、以降に少し調べた内容を記載します。興味がある方は読み進めてください

# ◇開発環境

+ OS : Windows 10 Home（バージョン1809）
+ IDE : Arduino IDE 1.8.9 (Windows Store 1.8.21.0)
+ デバイス : M5Stack FIRE
+ 内蔵加速度センサ：MPU9250

# ◇初期設定等々

開発環境のインストールからボード情報・ライブラリのインストール、サンプルプログラムの実行まで基本的にM5Stackの製品ドキュメントどおりに進めていけば特に問題なくできました。
ただし、今回はM5Stack FIREを使用しているため、［ツール］-［ボード］設定は`M5Stack-Core-ESP32`ではなく、`M5Stack-Fire`を選択しています。

_参考URL_：[M5Core クイックスタート - Arduino Win ](https://docs.m5stack.com/#/ja/quick_start/m5core/m5stack_core_get_started_Arduino_Windows)

# ◇サンプルスケッチの実行（ハマった箇所）

初期設定完了後、MPU9250（3軸加速度センサ＋3軸ジャイロセンサ＋3軸磁力センサ）のデータを定期的に取得して液晶画面に表示するサンプルプログラム（`M5StackFire_MPU9250.ino`）をM5Stack FIREに書き込んで動作確認を行いました。

このとき、M5Stack FIREは液晶画面を横に向けた状態で起動しましたが、起動後のZ軸方向の加速度は約１Ｇが表示され、その後に液晶画面を上向きになるように倒すと、Z軸方向の加速度が約２Ｇという表示になってしまいました。

# ◇原因解析（ソースコードチェック）

その後、液晶画面を上向きの状態でM5Stack FIREを起動すると加速度の表示が問題ないことが判明したため、ライブラリにあるキャリブレーションの関数を調べてみました。

MPU9250のキャリブレーション関数（`calibrateMPU9250()`）は、`MPU9250.cpp`ファイル内で定義されています。
なお、インストール先のフォルダを特に指定していなければ、このファイルは
`Arduino`フォルダ下の`libraries\M5Stack\src\utility\`内にあるはずです。

以下は、`calibrateMPU9250()`関数のコードになります。
ここでは、部分的に抜粋しているため、ファイル全体のソースコードを確認したい場合、M5Stackの[GitHub](https://github.com/m5stack/M5Stack/blob/master/src/utility/MPU9250.cpp)をご覧ください。

```C++:MPU9250.cpp
void MPU9250::calibrateMPU9250(float * gyroBias, float * accelBias) {
  uint8_t data[12]; // data array to hold accelerometer and gyro x, y, z, data
  uint16_t ii, packet_count, fifo_count;
  int32_t gyro_bias[3] = { 0, 0, 0 }, accel_bias[3] = { 0, 0, 0 };

  ---略---

  uint16_t  gyrosensitivity  = 131;   // = 131 LSB/degrees/sec
  uint16_t  accelsensitivity = 16384;  // = 16384 LSB/g

  // Configure FIFO to capture accelerometer and gyro data for bias calculation
  writeByte(MPU9250_ADDRESS, USER_CTRL, 0x40);   // Enable FIFO
  writeByte(MPU9250_ADDRESS, FIFO_EN, 0x78);   // Enable gyro and accelerometer sensors for FIFO  (max size 512 bytes in MPU-9150)
  delay(40); // accumulate 40 samples in 40 milliseconds = 480 bytes

  // At end of sample accumulation, turn off FIFO sensor read
  writeByte(MPU9250_ADDRESS, FIFO_EN, 0x00);  // Disable gyro and accelerometer sensors for FIFO
  readBytes(MPU9250_ADDRESS, FIFO_COUNTH, 2, &data[0]); // read FIFO sample count
  fifo_count = ((uint16_t)data[0] << 8) | data[1];
  packet_count = fifo_count / 12; // How many sets of full gyro and accelerometer data for averaging

  for (ii = 0; ii < packet_count; ii++) {
    int16_t accel_temp[3] = { 0, 0, 0 }, gyro_temp[3] = { 0, 0, 0 };
    readBytes(MPU9250_ADDRESS, FIFO_R_W, 12, &data[0]); // read data for averaging
    accel_temp[0] = (int16_t) (((int16_t)data[0] << 8) | data[1]  );  // Form signed 16-bit integer for each sample in FIFO
    accel_temp[1] = (int16_t) (((int16_t)data[2] << 8) | data[3]  );
    accel_temp[2] = (int16_t) (((int16_t)data[4] << 8) | data[5]  );
    gyro_temp[0]  = (int16_t) (((int16_t)data[6] << 8) | data[7]  );
    gyro_temp[1]  = (int16_t) (((int16_t)data[8] << 8) | data[9]  );
    gyro_temp[2]  = (int16_t) (((int16_t)data[10] << 8) | data[11]);

    accel_bias[0] += (int32_t) accel_temp[0]; // Sum individual signed 16-bit biases to get accumulated signed 32-bit biases
    accel_bias[1] += (int32_t) accel_temp[1];
    accel_bias[2] += (int32_t) accel_temp[2];
    gyro_bias[0]  += (int32_t) gyro_temp[0];
    gyro_bias[1]  += (int32_t) gyro_temp[1];
    gyro_bias[2]  += (int32_t) gyro_temp[2];
  }
  accel_bias[0] /= (int32_t) packet_count; // Normalize sums to get average count biases
  accel_bias[1] /= (int32_t) packet_count;
  accel_bias[2] /= (int32_t) packet_count;
  gyro_bias[0]  /= (int32_t) packet_count;
  gyro_bias[1]  /= (int32_t) packet_count;
  gyro_bias[2]  /= (int32_t) packet_count;

  if(accel_bias[2] > 0L) {
    // Remove gravity from the z-axis accelerometer bias calculation
    accel_bias[2] -= (int32_t)accelsensitivity;
  }else {
    accel_bias[2] += (int32_t) accelsensitivity;
  }

  ---略---

  // Output scaled accelerometer biases for display in the main program
  accelBias[0] = (float)accel_bias[0] / (float)accelsensitivity;
  accelBias[1] = (float)accel_bias[1] / (float)accelsensitivity;
  accelBias[2] = (float)accel_bias[2] / (float)accelsensitivity;
}
```

具体的な処理内容は理解しきれていないため、割愛します。。
ここで、注目したいのが、
`// Remove gravity from the z-axis accelerometer bias calculation`
のコメント前後の処理です。

コード（コメント）からわかるように、重力がZ軸方向にかかる前提でその分のキャンセリングを行っているため、キャリブレーション時にはZ軸が上下になるように（液晶画面が上向きか下向きになるように）する必要がありそうです。
なお、

+ Z軸の値がプラス（＋）の場合、１Ｇ分を引き、
+ Z軸の値がマイナス（－）の場合、１Ｇ分を足す  
という処理になっているため、液晶画面を上向き、下向きどちらにしても問題はなさそうです。

# ◇あらためて結論

`calibrateMPU9250()`関数を実行するタイミング（`setup()`内で呼んでる場合は起動時）で、M5Stack FIREを以下のどちらかの状態にする。

+ 液晶画面を上に向けた状態（液晶画面を上から覗き込む状態）で平らなところに置く  
or
+ 液晶画面を下に向けた状態（液晶画面が見えない状態）で平らなところに置く  

なお、ライブラリでは、Z軸方向（`accel_bias[2]`）固定で重力（１Ｇ）分キャンセリングしていますが、例えば、

+ それぞれの軸（`accel_bias[i]`）の数値の絶対値を算出
+ 絶対値が一番大きい軸（要素）の値に対して重力（１Ｇ）分をキャンセリングする  

といった処理にすると、液晶画面を横にした状態でもキャリブレーションが正しく行われるんじゃなかなと思ってみたりしましたが、まだ試せてはいません。

※もちろん、M5Stackを斜めにした状態や振りながらキャリブレーションした場合はそれでもダメですが・・・

# ◇最後に

とりあえず、加速度センサが正しく動かせるようになったので、次回はM5Stack FIREを使ったガジェットを作ってみたいと思います。
