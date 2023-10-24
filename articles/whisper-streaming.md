---
title: "faster-whisper をリアルタイムストリーミング処理するプレプリントがあったのでColabで動かしてみた"
emoji: "🎤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["whisper", "googlecolaboratory", "python"]
published: true
---

OpenAI から [Whisper](https://github.com/openai/whisper) とかいう化け物ASRモデルが出たかと思えば，C++で書かれたCore MLをサポートした [whisper.cpp](https://github.com/ggerganov/whisper.cpp) が出たかと思えば，とても高速化された [faster-whisper](https://github.com/guillaumekln/faster-whisper) 出てきました．

https://github.com/openai/whisper

https://github.com/ggerganov/whisper.cpp

https://github.com/guillaumekln/faster-whisper

3つのモデルの性能比較については，以下の記事がよくまとまっています．

https://zenn.dev/piment/articles/24269616aa9c04


Whisperのリアルタイム処理について調べていると，以下のプレプリントを発見しました．

https://arxiv.org/abs/2307.14743

> Whisper is one of the recent state-of-the-art multilingual speech recognition and translation models, however, it is not designed for real time transcription. In this paper, we build on top of Whisper and create Whisper-Streaming, an implementation of real-time speech transcription and translation of Whisper-like models. Whisper-Streaming uses local agreement policy with self-adaptive latency to enable streaming transcription. We show that Whisper-Streaming achieves high quality and 3.3 seconds latency on unsegmented long-form speech transcription test set, and we demonstrate its robustness and practical usability as a component in live transcription service at a multilingual conference.

GitHub リポジトリはこちら↓

https://github.com/ufal/whisper_streaming

## この記事で行うこと

- whisper_streaming を Google Colab でASRサーバーとして動かす
- ローカルからマイク入力をサーバーへTCP通信で送る

すぐに試してみたい方は，以下の Colab をコピーして動かしてみてください．

https://colab.research.google.com/drive/1MMsCAnK6bc6nCvqLSZPUvRvNLBhHIFP5?usp=sharing

## Colab 実行環境の用意

ランタイムは任意のGPU環境を選択してください（もちろんA100が一番いい）．

## コードをクローン

```zsh
!git clone https://github.com/ufal/whisper_streaming
%cd whisper_streaming
```

## 依存関係のインストール

```zsh
!pip install librosa # 音声処理用
!pip install faster-whisper # ASRバックエンドとしてのfaster-whisper
!pip install opus-fast-mosestokenizer # 文章セグメンター（for English and more）
!pip install torch wtpsplit # 文章セグメンター（for Japanese and more）
```

## ngrokコマンドのインストール

```zsh
!wget https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-amd64.zip
!unzip ngrok-stable-linux-amd64.zip
```

## TCPエンドポイント設置

サーバーのTCPエンドポイントの外部公開のために ngrok を使います．エンドポイントを立てるために ngrok の Authtoken が必要なのでログインして，ダッシュボードから取得してください．
https://dashboard.ngrok.com/get-started/your-authtoken

```python
get_ipython().system_raw('./ngrok authtoken <YOUR NGROK AUTHTOKEN>')
get_ipython().system_raw('./ngrok tcp 43007 &')
```

```python
# check the endpoint
import json
import requests

# ngrokのAPIを呼び出し
response = requests.get('http://localhost:4040/api/tunnels')

# JSONからpublic_urlを取得
url = response.json()['tunnels'][0]['public_url']

# "tcp://"を取り除き、":"で分割
host, port = url.replace('tcp://', '').split(':')

print('Propagate the microphone voice using the following commands')
print('sox -d -e signed -b 16 -c 1 -r 16000 -t raw - | nc {} {}'.format(host, port))
```

## サーバー起動

```zsh
!python3 whisper_online_server.py --language ja --min-chunk-size 1
```

## [ローカル] マイクから音声入力

Linux:
```
arecord -f S16_LE -c1 -r 16000 -t raw -D default | nc <HOST> <PORT>
```

Others:
```
sox -d -e signed -b 16 -c 1 -r 16000 -t raw - | nc <HOST> <PORT>
```

## 動かしている様子

https://youtu.be/tukqhoNy-is

## まとめ

依然として，[Web Speech API](https://developer.mozilla.org/ja/docs/Web/API/Web_Speech_API)などに比べれば遅くはありますが，ある程度の遅延が許されるシーン（精度優先）ではユースケースがあるかもしれません．
