---
title: ハードウェアとセットアップ
nav_order: 9
has_children: true
---

# ハードウェアとセットアップ

無線部は USB シリアルで接続する ESP32 ボードで、`firmware/esp32` を実行します。LDN、Pia、各ゲームはホスト上の
Python で動作し、root 権限も Wi-Fi ドライバーも不要です。すべてのタイトルがこの経路で交換を行います。Linux ホストでは
代わりに、AP に対応した Wi-Fi カードを直接操作できます。ただし、その方式は旧来のものです。

## ページ

- [ESP32 無線部](hardware_esp32.md): ボード、ファームウェア、シリアルプロトコル、測定値。
- [Switch キー](hardware_switch_keys.md): `prod.keys` を安全にインストールする方法。
- [アダプター](hardware_adapters.md): Linux 向けの Wi-Fi カード、その設定、障害の出方。
- [コントローラーボード](hardware_pad.md): 本体が有線コントローラーとして認識する ESP32-S3 と、
  Pro コントローラーとしてペアリングする従来型の ESP32 です。前者は Bluetooth LE、後者は USB シリアルで駆動します。
- [Raspberry Pi ホスト](hardware_raspberry_pi.md): Linux での配備と、監視付きの
  Mystery Gift ランナーについてです。
