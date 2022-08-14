# PSGSoundDriver

## 概要

MSX用のPSGサウンドドライバです。  
z88dkのz80asmでコンパイルできる形にしています。  
サポートしている機能は以下となります。  
- 効果音の割り込み再生に対応
- 効果音の再生優先度（プライオリティ）を設定可能
- ノイズON/OFF、トーン、ノイズトーン、ボリューム、デチューンに対応
- lc2asm.pyでLovelyComposerで制作したデータから変換して使用可能（制約あり）

## 実行サンプル

以下のURLでサンプルプログラムを用いたドライバの動作確認ができます。  

https://webmsx.org/?MACHINE=MSX1J&ROM=https://github.com/aburi6800/msx-PSGSoundDriver/raw/master/dist/sample.rom&FAST_BOOT

## 使用方法

### 利用プログラムの事前準備

- プログラムソースの先頭で、以下の指定を行います。
```
EXTERN SOUNDDRV_INIT
EXTERN SOUNDDRV_EXEC
EXTERN SOUNDDRV_BGMPLAY
EXTERN SOUNDDRV_SFXPLAY
EXTERN SOUNDDRV_STOP
EXTERN SOUNDDRV_PAUSE
EXTERN SOUNDDRV_RESUME
EXTERN SOUNDDRV_STATUS
```
- プログラムの初期処理で、ドライバの初期化ルーチン(SOUNDDRV_INIT)をCALLします。
    - この初期化ルーチンの中でH.TIMIフックのコード（5byte）をバックアップし、ドライバを実行するように書き換えを行います。
    - アプリケーション側でもH.TIMIフックで実行する処理がある場合、必ずH.TIMIから5バイトのバックアップを取っておき、処理の最後でバックアップしたアドレスにJPしてください。
    - なお、アプリケーションでドライバの初期化ルーチンをCALLするのは、H.TIMIフックの書き替え前／後のどちらでも構いません。
```
    CALL SOUNDDRV_INIT              ; サウンドドライバ初期化
```
### 再生データの指定・再生・停止方法

- 次の「ドライバのAPI」を参照ください。

### ドライバAPI

上記の事前準備を行った後は、ユーザープログラムからは以下の手順で制御できます。

- `SOUNDDRV_BGMPLAY`
    - BGMの再生を開始する。
    - HLレジスタに再生するBGMデータのアドレスを指定する。
```
    LD HL,BGMDATA
    CALL SOUNDDRV_BGMPLAY
```
- `SOUNDDRV_SFXPLAY`
    - 効果音を再生する。既にプライオリティの高い効果音が再生中の場合は再生しない。
    - HLレジスタに再生する効果音データのアドレスを指定する。
```
    LD HL,SFXDATA
    CALL SOUNDDRV_BGMPLAY
```
- `SOUNDDRV_STOP`
    - BGM、効果音の再生を停止する。
```
    CALL SOUNDDRV_STOP
```
- `SOUNDDRV_PAUSE`
    - BGM、効果音の再生を一時停止する。
```
    CALL SOUNDDRV_PAUSE
```
- `SOUNDDRV_RESUME`
    - BGM、効果音の一時停止を解除する。
    - 再生中に呼び出した場合は、何も処理しない。
```
    CALL SOUNDDRV_RESUME
```
- `SOUNDDRV_STATUS`
    - ドライバのステータスを取得する。
    - bit 0は再生中であれば1、停止中であれば0となる。
    - bit 1は一時停止中であれば1、以外は0となる。
---
    LD A,(SOUNDDRV_STATUS)
---

## データ構造

このドライバで扱うデータ構造は、以下のイメージです。  
BGM、効果音共に同じ構成になります。  
```
[BGM/SFX Data]
  +- [Track 1 Data Address](2byte)
  +- [Track 2 Data Address](2byte)
  +- [Track 3 Data Address](2byte)
```
> すなわち、効果音として作成したデータをBGMとして鳴らすこともできますし、その逆も可能です。  
> また、上記のように、トラックデータの集まりを曲データとしているため、複数のBGM/効果音で同じトラックを使いまわすこともできます。  
> 

### 曲/効果音データ

BGM/効果音データの構成を定義します。  

- プライオリティ(1byte)：0～255（低～高）、効果音発声時のみ有効、BGMでは無視される。  
- トラック1のデータアドレス(2byte)：ない場合は`$0000`を設定する。  
- トラック2のデータアドレス(2byte)：ない場合は`$0000`を設定する。  
- トラック3のデータアドレス(2byte)：ない場合は`$0000`を設定する。  

### トラックデータ

トラックデータは、以下で構成されます。  

| コマンド | 意味 | 概要 |
|:---|:---|:---|
| 0〜95 | ノート(トーンテーブルのindex番号) | ノート番号の後に音長(n/60)を指定する。（音調についての詳細は後述） |
| 200〜215 | ボリューム | コマンドの値が0〜15に対応。(PSGレジスタ#8〜#10に設定する値と同じ)　値の設定は不要。 |
| 216 | ノイズトーン | 216,<値> の形式で指定。値に0〜31を指定。(PSGレジスタ#6に設定する値と同じ) |
| 217 | ミキシング | 217,<値> の形式で指定。値に0〜3を指定。(bit0=Tone/bit1=Noise、1=off/0=on) |
| 218 | デチューン | 218,<値> の形式で指定。値にノートに対する補正値を指定。 |
| 253 | ループ開始位置 | なし |
| 254 | データ終端(トラック先頭、またはループ開始位置に戻る) | なし |
| 255 | データ終端(再生終了) | なし |

### 音長について

音調は以下で求めることが可能です。  

- 1秒あたりの発音数：<テンポ>/60秒  
- 4分音符の音長：60/(1秒あたりの発音数)  
- 8分音符の音長：4分音符の音長/2  
- 16分音符の音長：8分音符の音長/2  

例)テンポ=120の場合 (=1分間に4分音符を120回鳴らす速度)  
- 1秒あたりの発音数：120 / 60秒 = 2  
- 4分音符の音長：60 / 2 = 30  
- 8分音符の音長：30 / 2 = 15  
- 16分音符の音長：15 / 2 = 7.5  
 
> データに設定する値は整数なので、端数が出る場合は、ノートの音長の合計で辻褄が合うように調整してください。  

### ビルド方法

（更新中）  

## LovelyComposerからのコンバートツール（lc2asm.py）について

### 概要

作曲ツール「LovelyComposer」で作成したデータを、当ドライバ用のデータに変換するツールを用意しています。（`src/python/lc2asm.py`）  
pythonが必要ですので、別途インストールしてください。  

なお、LovelyComposerは以下のサイトで購入できます。  
[BOOTH](https://booth.pm/ja/items/3006558)  
[itch.io](https://1oogames.itch.io/lovely-composer)  

### 利用方法

カレントディレクトリにLovelyComposerのユーザーデータ(.jsonl)をコピーし、以下で実行します。  
```
python src/python/lc2asm.py <jsonlファイル名>
```
作成された.asmファイルは、ソースに直接取り込むか、`INCLUDE`してください。  

### サポートしている機能

LovelyComposerの以下の機能に対応しています。

- 各ノートのボリューム指定に対応しています。
- ループ開始・終了の指定に対応しています。
- 1ページ単位のノート数の変更、テンポの変更に対応しています。

### 制約など

LovelyComposerの機能に対し、作成されるデータには以下の制約があります。

- 利用できる音色は、「SQUARE WAVE」「NOISE」の2つです。
- 利用できるチャンネルは、1〜3の3つです。(チャンネル4、コードチャンネルは無視されます)
- テンポは処理の都合上、完全に同一にはならず、近似値になります。（違和感はないはずです）
- パン指定は無視されます。

## 改訂履歴

2022/08/14  Version 1.5.0
- psgdriver.asm
    - 一時停止(SOUNDDRV_PAUSE)/再開(SOUNDDRV_RESUME)のAPIを追加
    - ドライバの状態を取得するAPI(SOUNDDRV_STATUS)を追加

2022/08/06  Version 1.4.1
- psgdriver.asm
    - includeのパス区切り文字のミスを修正

2022/07/24  Version 1.4.0
- psgdriver.asm
    - SFX再生時にBGMのプライオリティも判断するように修正  
      (プライオリティを最高に設定したBGMの演奏中はSFXが再生されなくなります)
- readme.md
    - lc2asm.pyに対する記述を修正

2022/05/29  Version 1.3.0
- バージョン表記をセマンティックバージョニングに合わせて修正
- psgdriver.asm
    - H.TIMIフックのコールチェインができるよう、バックアップを保存し処理の最後でCALLするように修正。
- readme.md
    - H.TIMIフックの書き換えについて追記。
    - lc2asmの制約等について追記。

2022/05/07  Version 1.20
- psgdriver.asm
    - ボリュームデータの構成変更、併せて他コマンドのデータ変更。1.1までと互換性がないのでご注意ください。
- lcsasm.py
    - ボリュームデータの構成変更、併せて他コマンドのデータ変更に対応。

2021/12/04  Version 1.11
- lc2asn.py
    - lc2asm.pyでページ単位のノート数・SPEED値の変更、全ページ数の変更に対応。

2021/11/28  Version 1.10
- psgdriver.asm
    - LovelyComposerのループ開始・終了位置の指定に対応
- lc2asn.py
    - ループ終了の指定があるときはループあり、指定がない場合はループ無しのデータを出力するように修正

2021/11/19  Version 1.00
- psgdriver.asm
    - 効果音の割り込み再生に対応
    - 効果音の再生優先度（プライオリティ）を設定可能
    - ノイズON/OFF、トーン、ノイズトーン、ボリューム、デチューンに対応
- lc2asn.py
    - LovelyComposerで制作したデータから変換して使用可能（制約あり）
