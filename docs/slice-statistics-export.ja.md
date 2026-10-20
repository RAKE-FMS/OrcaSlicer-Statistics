# スライス統計の出力

FFF の G-code 保存時に、同じディレクトリへ `<最終 G-code ファイル名>.slice_statistics.json` を保存する。追加の設定は不要。CLI の `--slice` では各プレートの G-code 保存後に出力し、同じ JSON を `slice_statistics ` に続けて標準エラーへ出す。GUI では通常の G-code 保存と Bambu 向け G-code 保存に対応し、スライス成功時にもアプリのログへ出す。

スライスだけの GUI 操作では JSON ファイルを作らない。`.gcode.3mf` の保存とプリンタへのアップロードは対象外。既存の `result.json`、スライス設定、プロファイル形式は変更しない。

## 統計の意味

| 項目 | 出所・単位 |
| --- | --- |
| `plate_index`、`result_id` | 1 始まりのプレート番号と、アプリ実行中の G-code 処理結果 ID。結果 ID は永続的な識別子ではない。 |
| `roles[]` | 役割ごとのフィラメント長・重量と、押出移動の推定時間。Support と Support interface、Support transition は分けて保持する。 |
| `filaments[]` | フィラメントごとの Model、Support、Flushed、Tower、Total。Support は interface と transition も含む。フィラメント ID は 1 始まり。 |
| `total_filament`、`model_filament`、`cost` | `Print::print_statistics()` に基づく集計。Model は総量から Support・Flushed・Tower を差し引く。 |
| `time.normal`、`time.stealth` | 通常・静音モードの推定時間。`model_seconds` は総時間から準備時間を引いた値で、モデル押出専用の時間ではない。Travel は別集計。 |
| `details` | ツール変更、ワイプタワー、フィラメント別体積などの補足情報。 |

生値は mm・g・mm³・秒。比較や計算には生値を使う。表示文字列は参照元と同じ小数点以下 2 桁の m/g または in/oz を使い、CLI は metric、GUI は保存時の `use_inches` に従う。現行 GUI の重量表示は kg/t への切り替えや空白の使い方が異なるため、表示文字列の完全一致は保証しない。

`filaments[].total.volume_mm3` は Model + Support + Flushed + Tower の和。processor 自身の合計体積は `total.processor_volume_mm3` に別途保持する。役割・フィラメントの配列順と JSON の構成は参照元を維持しているが、オブジェクトのキー順には依存しないこと。

## 参照元と修正点

`../OrcaSlicer-DynaPin` のコミット `800cd32163`（元コミット `6eef0999a7`）と、説明を追加した `6cfff12fb4` を確認して移植した。`SliceStatisticsLog.cpp` の集計・保存処理は同じ実装を使用している。参照元の次の問題は引き継がない。

- GUI の JSON キャッシュが結果 ID しか見ておらず、同じ結果で単位設定やプレート番号が変わっても古い値を返す。出力時に現在の状態から生成するよう変更した。
- ログ済み ID を全プレートで一つだけ保持しており、A をスライス→B をスライス→既存の A を保存すると A が再度ログに出る。ログ済み ID を各プレートの実行時状態として保持するよう変更した。この状態は `.3mf` には保存しない。

## 保存失敗時の扱い

G-code の保存・検証後に JSON を書く。同じディレクトリの一時ファイルへ書き終えてから既存 JSON を置換し、失敗した一時ファイルは削除する。JSON 保存失敗は保存操作のエラーとして通知するが、保存済みの G-code は削除しない。電源断への永続化保証は行わない。

G-code または JSON の保存が失敗した場合、以前の同名 JSON が残ることがある。失敗した保存操作の JSON を新しい G-code の統計として扱わないこと。この失敗時の扱いも参照元と同じ。

## 検証方法

単体テストは `tests/libslic3r/test_slice_statistics_log.cpp`。役割別 Support、複数フィラメント、両時間モード、単位換算、ファイル名、既存 JSON の置換、保存失敗時の一時ファイル削除を確認する。

```sh
cmake --build build/arm64 --config Release --target libslic3r_tests OrcaSlicer --parallel 4
```

生成した `libslic3r_tests` に `'[SliceStatisticsLog]' --order rand --warn NoAssertions` を渡して実行する。macOS では実行ファイルは `build/arm64/tests/libslic3r/Release/libslic3r_tests.app/Contents/MacOS/libslic3r_tests`。

GUI の確認時は、通常保存・Bambu 向け保存・単位変更後の再保存・プレートの並べ替え後の保存を調べる。A/B/A の順でプレートを保存しても、同じ結果のログが重複しないことを確認する。CLI は複数プレートを `--slice 0 --outputdir <出力先>` で保存し、各 JSON のプレート番号と標準エラーの JSON が一致することを確認する。

2026-10-17 の macOS arm64 検証では、Release のコアライブラリと `libslic3r_tests` をビルドし、上記タグの 7 ケース・81 アサーションが成功した。`OrcaSlicer.cpp`、`BackgroundSlicingProcess.cpp`、`PartPlate.cpp`、`Plater.cpp` も GUI 有効の Release 構成でコンパイル成功。アプリ全体のリンク、CLI の実スライス、GUI の実操作による確認は未実施。
