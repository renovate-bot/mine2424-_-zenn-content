---
title: "Jetson Orin Nano Superでローカルイルを動かす — セットアップから日本語LLM運用まで"
emoji: "🤖"
type: "tech"
topics: ["llm", "jetson", "ollama", "ai", "edge"]
published: false
---

## なぜJetsonでローカルLLMなのか

ChatGPTやClaudeのようなクラウドAPIは便利だが、ネットワーク越しの処理にはレイテンシが伴い、APIキーの管理やコストも発生する。個人開発やプロトタイプの段階では「すぐ手元で動くLLM」があると格段に試行錯誤しやすい。

Jetson Orin Nano Superはまさにその答えの一つだ。NVIDIAのAmpere GPU（1024コア・32 Tensor Core）と8GB LPDDR5メモリを搭載しながら、消費電力は最大25W。手のひらに乗るサイズでGPU推論ができる。

この記事では、Jetson Orin Nano Super上にローカルLLM環境を構築した実践記録をまとめる。

## ハードウェア概要

まず使っているマシンのスペックを整理しておく。

| 項目 | スペック |
|---|---|
| GPU | 1024-core NVIDIA Ampere / 32 Tensor Cores |
| RAM | 8GB LPDDR5（CPU/GPU共有） |
| Storage | 256GB NVMe SSD |
| OS | JetPack 6.x (Ubuntu 22.04) |
| 消費電力 | 7〜25W |

重要なのは「CPU/GPU共有メモリ」という点だ。通常のデスクトップPCでは CPUのシステムRAMとGPUのVRAMは別になっているが、Jetsonは統合メモリアーキテクチャ（UMA）を採用している。つまり8GBをOSとGPU推論で共有する。実効的に使えるのは約5〜6GBになるため、モデルサイズの選定に直結する。

## セットアップアーキテクチャ

推論フレームワークとして**Ollama**をメインに採用した。Ollamaを選んだ理由は以下の3点だ。

**1. Docker経由で環境が汚れない**
JetPackはNVIDIAのCUDAドライバと密に結びついており、ネイティブインストールで依存関係が壊れるリスクがある。Dockerコンテナとして分離することで、Jetsonのシステム環境を汚さずにOllamaを動かせる。

**2. OpenAI互換APIが標準で使える**
OllamaはOpenAI互換のREST APIを `/v1/chat/completions` で提供する。既存のPython/TypeScriptコードのAPIエンドポイントをJetsonのIPに向けるだけで動く。移行コストがゼロに近い。

**3. モデル管理が楽**
`ollama pull qwen2.5:7b` のような直感的なコマンドでモデルをダウンロード・管理できる。モデルのロード・アンロードもAPIで制御できる。

サブフレームワークとして**llama.cpp**も用意した。GPU layerの細かい制御（`-ngl`パラメータ）でメモリと速度のトレードオフを調整したいときに使う。

## セットアップ手順

リポジトリ（[mine2424/jetson-local-llm](https://github.com/mine2424/jetson-local-llm)）にワンショットのセットアップスクリプトを用意した。

```bash
# ワンショットセットアップ（推奨）
bash install.sh
```

このスクリプト1本で、環境確認→Docker Ollama起動→スターターモデルのダウンロードまで完了する。

個別に実行する場合は以下の順番になる。

```bash
# 1. JetPack・CUDA環境の確認
bash setup/00_jetpack_check.sh

# 2. Docker Ollama のセットアップ
bash setup/05_setup_docker_ollama.sh

# 3. 推奨モデルの一括ダウンロード
bash setup/03_pull_models.sh
```

`00_jetpack_check.sh` は地味に重要で、JetPackのバージョンやCUDAが正常に認識されているかを事前確認する。Jetsonのセットアップでは「CUDAが使えていると思っていたが実は使えていなかった」というケースが多い。

## モデル選定：8GBの壁とどう戦うか

Jetsonの8GB共有メモリは制約であると同時に、モデル選定を強制するフィルタでもある。「動けばいい」ではなく「快適に動く」サイズを選ぶことが重要だ。

現在の推奨構成はこうなっている。

| モデル | サイズ(Q4量子化) | 日本語 | 用途 |
|---|---|---|---|
| LFM-2.5 3B | ~2.0GB | △ | バランス型・高速 |
| LFM-2.5 7B | ~4.5GB | △ | 高品質 |
| Qwen2.5 7B | ~4.5GB | ◎ | **日本語メイン推奨** |
| Phi-3.5 Mini | ~2.4GB | △ | コード生成 |
| Gemma 2 2B | ~1.8GB | △ | 超軽量 |

日本語で使うなら**Qwen2.5 7B**が現時点で最もバランスが良い。Alibabaが開発したモデルで、日本語コーパスの品質が高く、7BサイズでもGPT-3.5に近い応答品質が出る場面がある。

**LFM-2.5**（Liquid Foundation Model）は注目のアーキテクチャだ。Transformerではなく、SSM（State Space Model）ベースのハイブリッド構造を採用しており、同サイズのTransformerモデルより少ないメモリで長いコンテキストを処理できる。Jetsonのメモリ制約を考えると、今後のメインプレイヤーになり得る。

注意点として、**複数モデルを同時にロードしない**こと。`ollama ps` で現在ロード中のモデルを確認し、不要なモデルは `ollama stop` でアンロードしてメモリを解放する習慣が必要だ。

## OpenAI互換APIとして活用する

OllamaのAPIエンドポイントはデフォルトで `http://localhost:11434` だ。外部マシンからアクセスするには、systemdの設定でホストをオープンにする。

```bash
# /etc/systemd/system/ollama.service.d/jetson.conf
Environment="OLLAMA_HOST=0.0.0.0:11434"

sudo systemctl daemon-reload && sudo systemctl restart ollama
```

これでLAN内の他マシン（開発用MacBookなど）からJetsonのLLMを呼び出せる。

PythonのOpenAIクライアントからは以下のように接続できる。

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://192.168.x.x:11434/v1",  # JetsonのIPに変更
    api_key="ollama",  # 任意の文字列でOK
)

response = client.chat.completions.create(
    model="qwen2.5:7b",
    messages=[
        {"role": "user", "content": "日本語で説明してください"},
    ],
)
```

`base_url` と `api_key` の2箇所だけ変えれば、既存のOpenAI向けコードがそのまま動く。これがOllamaを選ぶ最大の理由の一つだ。

## ローカルLLM運用で得られたもの

実際に運用してみて感じたことを率直に書いておく。

**良かった点**

*レイテンシの予測可能性*が上がった。クラウドAPIはネットワーク状態や混雑具合で応答時間がブレる。ローカルであればハードウェアのスペックが安定した下限になる。プロトタイプ開発中に「遅いのはネットワークかモデルか」という切り分けが不要になった。

*コスト感覚が変わった*。APIコールの都度課金がないため、「もう少し試してみる」という気軽さが格段に上がった。試行錯誤のループが速くなる。

**難しい点**

*8GBのメモリ制約は厳しい*。Qwen2.5 7BをQ4量子化で動かすと約4.5GBを使い、残りのヘッドルームが少ない。複雑なプロンプトや長いコンテキストでスワップが発生すると推論速度が落ちる。用途によっては3Bモデルで妥協する判断が必要になる。

*日本語精度はまだクラウドに及ばない*。Qwen2.5 7Bは日本語性能が高いとはいえ、GPT-4やClaude 3.5 Sonnetと比較すると微妙なニュアンスや長文の一貫性で差がある。「プライバシーを保ちながらある程度の品質でいい」というユースケースに最も向いている。

## 今後の展開

まだベンチマーク実測値が取れていない（`benchmark/results/` は空の状態）。実際の tokens/sec と TTFT（Time to First Token）を計測して公開する予定だ。

また、LFM-2.5の日本語fine-tuneモデルの調査も進めている。SSMアーキテクチャの省メモリ特性と日本語精度が組み合わされば、Jetsonのメモリ制約下でも実用的な日本語LLMが動く可能性がある。

エッジデバイスでのLLM活用はまだ発展途上だが、ハードウェアとモデルの進化は速い。Jetson Orin Nano Superのような手頃な価格帯のエッジAIデバイスが、個人開発者にとってより現実的な選択肢になりつつある。

---

セットアップスクリプトやモデル一覧の詳細は [mine2424/jetson-local-llm](https://github.com/mine2424/jetson-local-llm) を参照してください。
