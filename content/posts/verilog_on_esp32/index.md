---
title: "VerilogをC++に変換してESP32で動かす"
date: 2025-09-09T18:27:41+09:00
draft: false
toc: true
images:
tags:
  - verilog
  - ESP32
  - yosys
  - 8080 Emulator
---

## 背景

現在、8080 simulatorプロジェクトにて、Intel 8080 CPUの回路をVerilogで記述し、それをESP32上で実行して内部状態を可視化するシステムを開発しています。このプロジェクトでは、本来FPGAやASICで動作するはずのハードウェア回路を、マイクロコントローラー上でソフトウェアとして実行することで、CPUの動作を詳細に観察・デバッグできる環境を構築しています。

8080 simulatorの開発過程で、VerilogコードをESP32で動作させる手法について調査・実装を行ったため、その知見をまとめておきます。

## 概要

YosysのCXXRTL機能を使用して、VerilogコードをC++に変換し、ESP32上で実行する方法を紹介します。
今回は8ビットカウンタを例に、Verilogで記述したハードウェアをソフトウェアとして動作させる手順を説明します。この手法は8080 simulatorで使用している技術の基盤となるものです。

## Verilogコードの作成

まず、シンプルな8ビットカウンタのVerilogコードを作成します。

```verilog
module Counter8 (
  input wire reset,
  input wire clk,
  output reg [7:0] count
);

  always @(posedge clk or posedge reset) begin
    if (reset) begin
      count <= 8'b0;
    end else begin
      count <= count + 1;
    end
  end

endmodule
```

## YosysでC++に変換

VerilogコードをC++に変換するため、YosysのCXXRTL機能を使用します。

まず、Yosysスクリプトファイルを生成します：

```bash
# ディレクトリ作成
mkdir -p build

# Yosysスクリプトファイルを生成
cat > build/synthesize.ys << 'EOF'
read_verilog Counter8.v
hierarchy -top Counter8
proc; opt
write_cxxrtl -header -namespace counter8 -O6 -g4 Counter8.h
EOF

# Yosysでスクリプトを実行
yosys -s build/synthesize.ys
```

### 方法2: ワンライナーでの実行

より簡単に、以下のワンライナーでも実行できます：

```bash
yosys -p "read_verilog Counter8.v; hierarchy -top Counter8; proc; opt; write_cxxrtl -header -namespace counter8 -O6 -g4 Counter8.h"
```

### Yosysコマンドの説明

- `read_verilog Counter8.v`: Verilogファイルを読み込み
- `hierarchy -top Counter8`: トップモジュールを指定
- `proc; opt`: プロセス展開と最適化
- `write_cxxrtl`: CXXRTL形式でC++ヘッダーファイルを生成
  - `-header`: ヘッダーファイルとして出力
  - `-namespace counter8`: 名前空間を指定
  - `-O6`: 最適化レベル6
  - `-g4`: デバッグ情報レベル4

この処理により、Verilogモジュールが `Counter8.h` と `Counter8.cc` として生成されます。


## ESP32での実装

生成されたC++ファイルをESP32プロジェクトにコピーし、必要なCXXRTLランタイムライブラリを追加します。

### 必要なファイルの準備

1. **Yosysで生成されたファイル**: `Counter8.h`、`Counter8.cc`
2. **CXXRTLランタイム**: [YosysのGitHubリポジトリ](https://github.com/YosysHQ/yosys/tree/main/backends/cxxrtl/runtime)からダウンロード

### プロジェクト構成

以下のようなディレクトリ構成でESP32プロジェクトを作成します：

```
project_root/
├── src/
│   ├── main.cpp                    # メインのArduinoコード
│   ├── Counter8.h                  # Yosysで生成されたヘッダーファイル
│   ├── Counter8.cc                 # Yosysで生成された実装ファイル
│   └── cxxrtl/                     # CXXRTLランタイムライブラリ
│       ├── cxxrtl.h
│       ├── cxxrtl_vcd.h
│       ├── cxxrtl_time.h
│       ├── cxxrtl_replay.h
│       └── capi/
│           ├── cxxrtl_capi.h
│           ├── cxxrtl_capi.cc
│           ├── cxxrtl_capi_vcd.h
│           └── cxxrtl_capi_vcd.cc
└── platformio.ini                  # PlatformIO設定ファイル
```

### ヘッダーファイルの取り込み

ArduinoとCXXRTLのマクロが競合するため、適切な順序でインクルードする必要があります。

```cpp
// CXXRTL (Yosys) の必要なヘッダーファイルを先に読み込む
#include "Counter8.h" // yosys write_cxxrtl -header で生成されたファイル
#include "cxxrtl/cxxrtl.h"

// Arduinoマクロを無効化してから、Arduinoヘッダーを読み込む
#ifdef bit
#undef bit
#endif
#ifdef INPUT
#undef INPUT
#endif

#include <Arduino.h>
#include <Bounce2.h> // thomasfredericks/Bounce2

// 必要なArduinoマクロを再定義
#define INPUT 0x01
```

### ピン定義と初期化

ボタンのピン設定とVerilogモジュールのインスタンス化を行います。

```cpp
constexpr uint8_t RST_PIN = 9;
constexpr uint8_t CLK_PIN = 10;

// ボタン用のBounceオブジェクト
Bounce rst_button = Bounce();
Bounce clk_button = Bounce();

// CXXRTL Counter8インスタンス (生成クラス名は p_<module>)
cxxrtl_design::p_Counter8 dut;
```

### カウンタ値の表示関数

CXXRTLで生成されたカウンタの値をシリアルコンソールに出力

```cpp
// カウンタ値をシリアルに出力
void printCounterValue() {
  uint8_t counter_value = dut.p_count.get<uint8_t>();
  Serial.print("Counter value: ");
  Serial.print(counter_value);
  Serial.print(" (0x");
  Serial.print(counter_value, HEX);
  Serial.print(") [");

  // バイナリ表示
  for (int i = 7; i >= 0; i--) {
    Serial.print((counter_value >> i) & 1);
  }
  Serial.println("]");
}
```

### setup関数

ESP32の初期化とVerilogモジュールの初期化を行います。

```cpp
void setup() {
  Serial.begin(115200);
  Serial.println("Counter8 Test Program (CXXRTL)");
  Serial.println("==============================");

  // ボタンピンの初期化
  rst_button.attach(RST_PIN, INPUT_PULLUP);
  rst_button.interval(10);

  clk_button.attach(CLK_PIN, INPUT_PULLUP);
  clk_button.interval(10);

  // CXXRTL Counter8の初期化
  dut.p_clk.set<bool>(false);
  dut.p_reset.set<bool>(true);
  dut.step();

  // リセット解除
  dut.p_reset.set<bool>(false);
  dut.step();

  Serial.println("CXXRTL Counter8 initialized");
  Serial.println("RST: Reset counter to 0");
  Serial.println("CLK: Clock pulse (increment counter)");
  Serial.print("Initial ");
  printCounterValue();
}
```

### loop関数

メインループでボタン入力を監視し、Verilogモジュールのクロックとリセット信号を制御します。

```cpp
void loop() {
  rst_button.update();
  clk_button.update();

  if (rst_button.fell()) {
    // Counter8をリセット
    dut.p_reset.set<bool>(true);
    dut.step();

    // リセット解除
    dut.p_reset.set<bool>(false);
    dut.step();

    Serial.print("Reset: ");
    printCounterValue();
  }

  if (clk_button.fell()) {
    // クロック立ち上がりエッジ
    dut.p_clk.set<bool>(true);
    dut.step();

    dut.p_clk.set<bool>(false);
    dut.step();

    Serial.print("Clock: ");
    printCounterValue();
  }

  delay(10);
}
```

## まとめ

YosysのCXXRTL機能を使用することで、VerilogコードをC++に変換し、ESP32上でハードウェア設計をソフトウェアとして実行することができました。

特に、ESP32のような高性能なマイコンを使用することで、複雑なハードウェア設計もなかなか高速に実行できるため、そこそこ実用的に使用できることがわかりました。私達の8080コードでも、Clockボタンを押してほぼ遅延なしにLEDの表示を切り替えることができました。

回路規模と、動作速度の検証を今後行いたいです。
