# au Qua Station (RTD1295) 移植版 — フォークについて

🇬🇧 **[English version here](README.md)**

このブランチ(`qua-station`)は、[symops/monarch-6.18](https://github.com/symops/monarch-6.18)
（同じRealtek RTD1295 SoCを使うWD My Cloud Home向け）を、au/KDDIの
**Qua Station**（LTE対応フォトストレージNAS、機種名HVD1）向けに移植したものです。

ベースとなる`monarch-6.18`のREADME（[UPSTREAM-README.md](UPSTREAM-README.md)）は
SoC/ボード共通部分の情報として引き続き有効です。このファイルはQua Station
固有の内容だけをまとめています。

## このボード固有の違い

Qua StationはWD My Cloud Homeと同じRTD1295 SoCを使っていますが、基板構成が異なります。
LTEモデム、11個のLED、物理ボタン、単独のUSB 2.0ソケット(EHCI/OHCI。WD側のDTには存在しない)、
GPIO/LEDの配線もWD側とは別物です。このボード固有のベンダーカーネルは
[tsukumijima/QuaStation-Kernel-BPi](https://github.com/tsukumijima/QuaStation-Kernel-BPi)
(4.9.119)で、このツリー＋稼働中4.9カーネルへのシリアルコンソール接続調査が、
今回のボード固有部分（DTS、LED/ボタンのマッピング、クロックゲートのビット割り当て）の
主な参照元になっています。

## 動作状況

内蔵SSD(SATA上のrootfs。USBだけでなく)から実機で最後まで起動できることを確認済みです。

- SATA(1TB SSD、rootfs)
- PCIe（Wi-Fi両スロット: RTL8192EE 2.4GHz + RTL8812AE 5GHz、同時リンクアップ）
- USB 2.0（dwc3ポート＋単独EHCI/OHCIソケット）・USB 3.0
- LED 11個＋ボタン（実機で物理配置を目視確認済み。COPYボタンをKEY_POWERとして流用）
- Bluetooth（RTL8761ATV、UART経由。ベンダーの`rtk_hciattach`を使用、serdev化はまだ）
- RTD129x用クロックゲートドライバ（57ゲート、ベンダーBSPの`cgc.c`を移植。4.9のDTBと機械照合済み）

既知の未解決事項:

- PCIeのMSI割り込みが実機で退行する（根本原因未解明）。現状は動作実績のあるINTxに固定
- systemdの`.link`によるMACアドレス固定がこのボードでは効かない（`wpa_supplicant@.service.d`で回避）
- 内蔵GMACはドライバのprobeには成功するが、このボードには物理的なRJ45ポートが無い（ベンダー4.9カーネルでも動いたことがない）

## 詳しい経緯（Qiita記事）

シリアルコンソール無しでの診断手法、SATA/PCIe/USB2.0の詳細な原因究明（レジスタレベル）、
ext4破損インシデントの復旧、Bluetooth SPP経由の緊急コンソール構築まで、
全5編のQiita記事にまとめています。

1. シリアルコンソール無しで、起動すらしない問題を切り分ける
2. SATA・PCIe・Wi-Fi・LEDの復活
3. rootfsを内蔵SSDへ移行する
4. Bluetooth SPP非常口コンソールの構築
5. ext4破損インシデントからの復旧、USB2.0/PCIe診断ログ洪水の解決

（公開後、ここにリンクを追記します）

## ビルド方法

x86_64の開発機からクロスビルドします。

```sh
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- rtd1295-qua-station_defconfig  # または同梱の.configを使用
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image dtbs modules
```

出力DTB: `arch/arm64/boot/dts/realtek/rtd1295-qua-station.dtb`

実機で動作確認済みのビルド済み`uImage`＋DTBは
[`qua-run41`リリース](../../releases/tag/qua-run41)に添付しています
（sha256はリリースノート参照）。**rootfsは配布していません**。
`ubuntu-base-*-arm64`（手順はQiita記事の(1)(3)参照）や他のarm64ディストリから
各自構築してください。本リポジトリが扱うのはkernel/DTB側だけです。

## ライセンス

上流のLinux・`monarch-6.18`と同じくGPL-2.0です。
