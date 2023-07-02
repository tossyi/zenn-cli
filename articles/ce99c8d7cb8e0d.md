---
title: "XGenを試す"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python, AI, LLM]
published: false
---

# XGenとは
Salesforceが開発した7BのLLMです。特徴として、最大8Kシーケンス長・最大1.5Tトークンで訓練されています。

* 標準的なNLPベンチマークにおいて、XGenは同程度のモデルサイズのオープンソースLLM（MPT、Falcon、LLaMA、Redpajama、OpenLLaMAなど）と比較した場合に同等以上の結果を達成

https://blog.salesforceairesearch.com/xgen/


# XGenを動かす

HuggingFace Hubには、3種類のモデルがあります。

* [XGen-7B-4K-Base with support for 4K sequence length](https://huggingface.co/Salesforce/xgen-7b-4k-base)
* [XGen-7B-8K-Base with support for 8K sequence length](https://huggingface.co/Salesforce/xgen-7b-8k-base)
* [XGen-7B-8k-Inst with instruction-finetuning](https://huggingface.co/Salesforce/xgen-7b-8k-inst) ※ 研究開発目的に利用可能


https://github.com/salesforce/xGen

## xgenのリポジトリをクローン

xgenのリポジトリをクローンしてきます。今回はMacのローカルで動かします。
仮想環境はvenvで作成し、`source`コマンドでactivateします。

```zsh
python -m venv .venv
source .venv/bin/activate
git clone git@github.com:salesforce/xgen.git
```

続いて、`requirements.txt`が用意されているので、パッケージをインストールします。

```zsh
pip install -r requirements.txt 
```


## OpenAI Tittokenパッケージをインストール

```bash
pip install tiktoken
```

tittokenは、OpenAIのモデルで使用するための高速なBPEトークナイザーです。

https://github.com/openai/tiktoken



## 動かす

xgenのリポジトリをcloneすると、`sample.py`に同様のコードが用意されているので、そちらを利用すると良いです。

コードの中身は以下です。

```python
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM

tokenizer = AutoTokenizer.from_pretrained(
    "Salesforce/xgen-7b-8k-base",
    trust_remote_code=True)
model = AutoModelForCausalLM.from_pretrained(
    "Salesforce/xgen-7b-8k-base",
    torch_dtype=torch.bfloat16)
inputs = tokenizer("The world is", return_tensors="pt")
sample = model.generate(**inputs, max_length=128)
print(tokenizer.decode(sample[0]))
```

私の環境だと、PyTorchを入れる必要がありました。
```
pip3 install --pre torch torchvision torchaudio --index-url https://download.pytorch.org/whl/nightly/cpu
```

実行します。

```
python sample.py
```

ここできちんと動作すると、モデルなどのダウンロードが始まり、結果が表示されます。
なんかネガティブな結果がでました。

```
The world is full of people who are not happy with their lives. They are not happy with their jobs, their relationships, their families, their friends, their health, their finances, their appearance, their pasts, their futures, their choices, their lives. They are not happy with anything.

They are not happy with themselves.

They are not happy with the world.

They are not happy with the universe.

They are not happy with the cosmos.

They are not happy with the cosmos.

They are not happy with the cosmos.

They are not happy with the cosmos
```

他にも聞いてみます。tokenizerの部分を`Where is the capital of Japan?`にしました。
```
Tokyo is a major transportation hub, with a large number of airports, train stations,
# 東京は交通の要所であり、多くの空港や駅がある、
```


## エラーの切り分け

私の環境では、sample.pyを動かすときに、エラー`KeyError: 'llama'`がでました。
調査の結果、`pip install git+https://github.com/zphang/transformers@llama_push`を試すと動作しました。

```
pip install git+https://github.com/zphang/transformers@llama_push
```

https://huggingface.co/decapoda-research/llama-7b-hf/discussions/3#6425f70fb9dfed28cf6473c3

この結果、transformersのバージョンは`4.27.0.dev0`となりました。
切り分けでは`4.31.0.dev0`も試しましたが、動作しませんでした。