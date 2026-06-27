# EZKL 学習リポジトリ

PyTorch モデルのゼロ知識証明（ZKP）を EZKL で実装する学習記録。

## フロー概要

```
PyTorch モデル
    → ONNX エクスポート
    → gen_settings（visibility 設定）
    → calibrate_settings（量子化）
    → compile_circuit
    → gen_witness
    → get_srs + setup（PK / VK 生成）
    → prove（proof.json）
    → verify
    → create_evm_verifier（Verifier.sol）
```

## ノートブック

### notebooks/ezkl_demo.ipynb

Iris データセット（4 特徴量・3 クラス）を MLP で分類し、ZK 証明を生成。Solidity Verifier を生成して Sepolia testnet にデプロイするところまで通した。

- デプロイ済みコントラクト（Sepolia）: `0xd2a5bC10698FD955D1Fe6cb468a17809A08fd005`

### notebooks/simple_demo.ipynb

Fashion MNIST（28×28 画像・10 クラス）を CNN で分類。`input_visibility = "private"` にして入力を隠した証明を生成。

### little_transformer.ipynb

小さなTransformerモデルの学習能力を多角的に検証。4種類の難易度・性質の異なるタスクで、モデルの基礎能力を体系的にテストする。

| タスク | 内容 | 検証ポイント |
|---|---|---|
| ReverseDataModule | 数列を逆順に並び替え | 位置情報（Positional Encoding）と入力全体の把握 |
| AdditionDataModule | 2桁の足し算（桁ごとにトークン化） | 繰り上がりロジックの自己学習 |
| ParityDataModule | バイナリ列の累積パリティ計算 | 長期記憶・カウント能力（Transformerが苦手な領域） |
| WikipediaDataModule | enwik8 を使った文字レベル言語モデル | 自然言語の自律生成（LLMの基礎能力） |

### proof_splitting.ipynb

大きなモデルをメモリ制約の中でZKP化するための Proof Splitting の実装。モデルを複数のサブモデルに分割し、それぞれ独立して証明を生成して繋ぎ合わせる。

- **分割方法**: `onnx.utils.extract_model` で中間ノードを指定して分割（分割点は Netron で確認）
- **接続方法**: `ezkl.swap_proof_commitments` で前段の出力と後段の入力が一致することを証明に埋め込む
- **メモリ戦略**: 各ステップの中間ファイルを都度ディスクに書き出してRAMを節約

### notebooks/set_membership.ipynb

「秘密の値が既知の集合に含まれているか」をゼロ知識で証明する Set Membership の実装。

- **証明の仕組み**: `diff = (x - y)` の総乗（`torch.prod`）を計算。x が集合 y のいずれかと一致すれば差が 0 になり積が 0 になる
- **制約のポイント**: 出力を 0 に固定する制約を設けることで「メンバーである」ことを保証。制約がないと「正しく不合格になった証明」が作れてしまう
- **visibility 設定**: 入力は `hashed/private`（Poseidon ハッシュ化して秘匿）、集合と結果は `fixed`（公開）
- **`ezkl.float_to_felt(e, 7)`**: 小数を有限体の元に変換。スケール $2^7 = 128$ 倍して固定小数点整数にする

### notebooks/simple_demo/（ステップ別）

simple_demo を各ステップに分解した学習用ノートブック群。

| ファイル | 内容 |
|---|---|
| 00_check_env.ipynb | 環境確認 |
| 01_onnx.ipynb | ONNX エクスポートと中身の確認 |
| 02_settings.ipynb | visibility 設定と settings.json |
| 03_calibrate.ipynb | 量子化・logrows |
| 04_compile.ipynb | ZK 回路コンパイル |
| 05_witness.ipynb | witness.json の生成と読み方 |
| 06_setup.ipynb | SRS + PK / VK 生成 |
| 07_prove_verify.ipynb | 証明生成・検証・proof.json の読み方 |

## visibility 設定

```python
py_run_args.input_visibility  = "public"  # or "private"
py_run_args.output_visibility = "public"
py_run_args.param_visibility  = "fixed"   # or "private"
```

| 設定値 | 意味 |
|---|---|
| `public` | 証明に含まれ、検証者が確認できる |
| `private` | 証明者しか知らない |
| `fixed` | 回路に焼き込まれる（変えると証明が通らない） |

## 環境セットアップ

```bash
uv sync --python 3.12
uv run jupyter notebook
```

## 注意点

- ONNX エクスポートは `dynamo=False, opset_version=10` が必要
- 最終層に ReLU を付けると `gen_settings` が失敗する
- `verify` が失敗する場合は `settings.json` の `logrows` を +1 して再実行
- Colab では `await ezkl.get_srs()` / ローカルでは `ezkl.get_srs(settings_path)`
