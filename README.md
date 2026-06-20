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
