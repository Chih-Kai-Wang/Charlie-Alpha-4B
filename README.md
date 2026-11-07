<p align="center">
  <img src="docs/assets/readme-hero.svg" alt="Charlie Alpha 4B project header" width="100%" />
</p>

# Charlie alpha

Charlie alpha（模型識別碼：`Charlie-Alpha-4B`）會依照你描述的分析問題，選出合適的統計
程序，並在本機對你的資料執行分析。它支援繁體中文、簡體中文與英文。這是衍生模型：以
Apache-2.0 授權的 [`Qwen3.5-4B`](https://huggingface.co/Qwen/Qwen3.5-4B) 為底模，用 Apple
MLX 做 4-bit QLoRA 微調。

這個版本是 **Experimental v0.3.0**。訓練方法 DGP-Regret 在封存 DGP 上降低了 regret，這組題目
由專案自己的模擬器計分。它沒有改善 P-Bench 或 StatQA，資訊不足的案例也明顯退步。請把它當成
程序選擇的研究成果，不要當成自動統計分析師。

[简体中文](README.zh-Hans.md) · [English](README.en.md) · [模型卡](MODEL_CARD.md) ·
[DGP-Regret 技術報告](docs/DGP_REGRET.md)

`main` 分支另有尚未發布的 v0.4 開發內容：受約束的統計編譯器，以及
[DGP-Evolve 自我迭代](docs/DGP_EVOLVE.md)。DGP-Evolve 只在候選版本通過明確的升級門檻時才採用
它，否則回退。這些開發內容不會改動下方 v0.3 的公開結果。

## 功能

執行時只載入一份 4-bit 底模，以及一個有 2,129,920 個參數的 adapter。

- 有資料附件或統計問題時走 `stats` 路由。其他問題走 `base` 路由，略過 adapter，使用同一份
  底模。
- adapter 從 28 個訓練程序中選擇方法，資料代理另外提供 7 個固定評測程序。生成的文字不會被
  當成程式執行。
- 每份分析方案都記錄估計目標、抽樣單位、研究設計、相依性、缺失機制、方法與診斷。
- CSV、TSV、JSON 與 Parquet 檔案留在本機。Python 與 R 工具在 macOS 沙箱中執行，不能連網，
  也不能讀取無關的使用者檔案。

API 仍接受 `adapter` 作為 `stats` 路由的相容別名。數值測試中，base 旁路與獨立載入的底模之間，
logits 最大誤差為 `0.0`；重新啟用 adapter 之後，誤差同樣是 `0.0`。

## 封存評測

三組使用相同提示、工具、temperature 0 與封存題目。normalized regret 越低越好。

| 模型 | Final DGP regret | 方法正確率 | 無效選法率 |
| --- | ---: | ---: | ---: |
| Qwen3.5-4B 底模 | 0.6727 | 20.83% | 63.33% |
| Hard-label 消融 | 0.7016 | 17.50% | 65.83% |
| DGP-Regret | **0.4437** | **45.00%** | **38.33%** |

DGP-Regret 相對底模降低 34.04% regret；配對 bootstrap 的平均絕對改善為 0.2290，95% CI
為 `[0.1199, 0.3361]`。無效選法率相對下降 39.47%。在 pilot dev 上，完整方法的 regret
比 hard-label 低 8.54%，通過預先設定的 5% 消融門檻。

| 評測 | 底模 | DGP-Regret | 結果 |
| --- | ---: | ---: | --- |
| 三語方法正確率，英文 | 16.67% | 43.33% | +26.67 點 |
| 三語方法正確率，繁中 | 16.67% | 26.67% | +10.00 點 |
| 三語方法正確率，簡中 | 30.00% | 40.00% | +10.00 點 |
| P-Bench Raw / Strict | 0% / 0% | 0% / 0% | 未改善 |
| StatQA exact | 1.00% | 1.00% | 未改善 |
| 資訊不足案例 | 43.33% | 0% | 退步 |
| 數學／程式／STEM／一般保留集 | 100% | 100% | 無變化 |

完整彙總、信賴區間與門檻判定見
[`reports/stats/evaluation.json`](reports/stats/evaluation.json)。改善只出現在 DGP 選法任務與
它的三語版本。P-Bench 與 StatQA 沒有變化，資訊不足案例從 43.33% 降到 0%。能力門檻因此沒有
全部通過，v0.3.0 只能以 Experimental 發布。

## 安裝與使用

需要 Apple Silicon Mac、Python 3.12、`uv`，以及專案鎖定的 Pixi Python／R 環境：

```bash
make setup
```

分析本機資料：

```bash
uv run charlie-alpha stats analyze \
  --data survey.csv \
  --question "比較 treatment 與 control 的平均結果，資料是獨立隨機分派。" \
  --language zh_Hant \
  --adapter-path f0909172434/Charlie-Alpha-4B-MLX-4bit
```

多輪對話：

```bash
uv run charlie-alpha stats chat \
  --data survey.csv \
  --adapter-path f0909172434/Charlie-Alpha-4B-MLX-4bit
```

本機 OpenAI-compatible API 預設只綁定 `127.0.0.1`：

```bash
uv run charlie-alpha stats serve \
  --adapter-path f0909172434/Charlie-Alpha-4B-MLX-4bit

curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model":"Charlie-Alpha-4B",
    "messages":[{"role":"user","content":"檢查 treatment 對 outcome 的效果"}],
    "charlie_files":["/absolute/path/survey.csv"],
    "charlie_route":"stats",
    "charlie_tools":true
  }'
```

API 會回傳路由、工具次數、隔離狀態與分析方案，不回傳隱藏推理。

## 已知限制

v0.3 的 DGP 引擎是半參數的 operating-characteristic emulator，回應函數都提交在 repo 裡。
它使用共同亂數與 128／256／512 次抽樣，但沒有在每次 replication 中把 28 個方法全部重新擬合
到原始表格。
Final DGP 分數衡量模型是否符合這個模擬器，不代表一般統計最優性。

資料代理在 P-Bench 上幾乎總是回傳 `needs_clarification`，因此沒有產生可計分的 p 值。在
保留的資訊不足測試集上，adapter 一次也沒有要求補充資料。使用者應明確提供估計目標、
抽樣單位、配對或群聚結構、分派機制與缺失假設，並檢查模型選法。

醫療、政策與財務分析需要合格統計人員審查。Qwen3.5 hybrid tensor 的相容問題在上游還沒有
可驗證的修正，所以 GGUF 暫不發布。MLX adapter 與融合版已在乾淨環境中通過載入測試。

完整可續跑流程見 [`docs/REPRODUCIBILITY.md`](docs/REPRODUCIBILITY.md)。來源 revision 與
授權見 [`configs/sources.lock.json`](configs/sources.lock.json) 和
[`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md)。訓練語料、快取、憑證與機器路徑不會
提交。v0.2 FORGE 的程式與成果保留在 Git tag `v0.2.0`。

專案程式碼與模型衍生物採 [Apache-2.0](LICENSE) 授權。上游資料仍遵循各自的授權。

[先檢視凍結結果，再選擇完整重現](docs/REPRODUCIBILITY.md)
