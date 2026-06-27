# EZKL 学習リポジトリ

PyTorch モデルのゼロ知識証明（ZKP）を EZKL で実装する学習記録。

## セットアップ

```bash
uv sync --python 3.12
uv run jupyter notebook
```

## EZKL フロー

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

### 入門

| ノートブック | 内容 |
|---|---|
| [ezkl_demo.ipynb](notebooks/ezkl_demo.ipynb) | Iris MLP → ZKP → Solidity Verifier → Sepolia デプロイまで一通り |
| [simple_demo.ipynb](notebooks/simple_demo.ipynb) | Fashion MNIST CNN、`input_visibility = "private"` で入力を隠した証明 |

デプロイ済みコントラクト（Sepolia）: `0xd2a5bC10698FD955D1Fe6cb468a17809A08fd005`

#### ステップ別（simple_demo を分解）

| ファイル | 内容 |
|---|---|
| [00_check_env.ipynb](notebooks/simple_demo/00_check_env.ipynb) | 環境確認 |
| [01_onnx.ipynb](notebooks/simple_demo/01_onnx.ipynb) | ONNX エクスポートと中身の確認 |
| [02_settings.ipynb](notebooks/simple_demo/02_settings.ipynb) | visibility 設定と settings.json |
| [03_calibrate.ipynb](notebooks/simple_demo/03_calibrate.ipynb) | 量子化・logrows |
| [04_compile.ipynb](notebooks/simple_demo/04_compile.ipynb) | ZK 回路コンパイル |
| [05_witness.ipynb](notebooks/simple_demo/05_witness.ipynb) | witness.json の生成と読み方 |
| [06_setup.ipynb](notebooks/simple_demo/06_setup.ipynb) | SRS + PK / VK 生成 |
| [07_prove_verify.ipynb](notebooks/simple_demo/07_prove_verify.ipynb) | 証明生成・検証・proof.json の読み方 |

### 応用

#### [set_membership.ipynb](notebooks/set_membership.ipynb)

「秘密の値が既知の集合に含まれているか」をゼロ知識で証明する Set Membership の実装（数値データ版）。

- **証明の仕組み**: `diff = (x - y)` の総乗（`torch.prod`）を計算。x が集合 y のいずれかと一致すれば差が 0 になり積が 0 になる
- **制約のポイント**: 出力を 0 に固定する制約を設けることで「メンバーである」ことを保証。制約がないと「正しく不合格になった証明」が作れてしまう
- **visibility**: 入力は `hashed`（Poseidon ハッシュ化して秘匿）、集合と結果は `fixed`
- **`ezkl.float_to_felt(e, 7)`**: 小数を有限体の元に変換。スケール $2^7 = 128$ 倍して固定小数点整数にする

#### [set_real_membership.ipynb](notebooks/set_real_membership.ipynb)

文字列・日付など現実的なデータで Set Membership ZKP を実現する実用版（`set_membership.ipynb` の続き）。

- **課題**: 文字列は float に変換できないため、SHA-256 → Poseidon の 2 段階でフィールド要素に変換する
- **`str_to_felt(s)`**: SHA-256 の 256 ビット出力を前後 128 ビットに分割し、ezkl の felt 形式（リトルエンディアン 64 桁 16 進数）に変換後、Poseidon ハッシュで 1 つのフィールド要素にまとめる
- **整数の扱い**: ハッシュ値（256 ビット）は float64 に収まらないため `>> BIT_SHIFT` で切り詰めてから `torch.int64` テンソルに渡す
- **visibility**: 個人データのハッシュ・メンバーリストともに `private`、出力のみ `public`

#### [proof_splitting.ipynb](notebooks/proof_splitting.ipynb)

大きなモデルをメモリ制約の中で ZKP 化するための Proof Splitting の実装。

- **分割方法**: `onnx.utils.extract_model` で中間ノードを指定して分割（分割点は Netron で確認）
- **接続方法**: `ezkl.swap_proof_commitments` で前段の出力と後段の入力が一致することを証明に埋め込む
- **メモリ戦略**: 各ステップの中間ファイルを都度ディスクに書き出して RAM を節約

### その他

#### [little_transformer.ipynb](notebooks/little_transformer.ipynb)

小さな Transformer の学習能力を多角的に検証（EZKL とは独立したテーマ）。

| タスク | 内容 | 検証ポイント |
|---|---|---|
| ReverseDataModule | 数列を逆順に並び替え | Positional Encoding と入力全体の把握 |
| AdditionDataModule | 2桁の足し算（桁ごとにトークン化） | 繰り上がりロジックの自己学習 |
| ParityDataModule | バイナリ列の累積パリティ計算 | 長期記憶・カウント能力（Transformer が苦手な領域） |
| WikipediaDataModule | enwik8 を使った文字レベル言語モデル | 自然言語の自律生成（LLM の基礎能力） |

## visibility 設定

```python
py_run_args.input_visibility  = "public"   # or "private" / "hashed"
py_run_args.output_visibility = "public"
py_run_args.param_visibility  = "fixed"    # or "private"
```

| 設定値 | 意味 |
|---|---|
| `public` | 証明に含まれ、検証者が確認できる |
| `private` | 証明者しか知らない（Witness に含まれる） |
| `fixed` | 回路に焼き込まれる（変えると証明が通らない） |
| `hashed` | Poseidon ハッシュ化して秘匿しつつ公開コミットを作る |

## 注意点

- ONNX エクスポートは `dynamo=False, opset_version=10` が必要（整数モデルは `opset_version=14`）
- `verify` が失敗する場合は `settings.json` の `logrows` を +1 して再実行
- Colab では `await ezkl.get_srs()` / ローカルでは `ezkl.get_srs(settings_path)`
