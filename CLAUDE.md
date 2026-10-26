# wechat-cli

用命令列查詢本機 WeChat 資料（SQLCipher 即時解密），預設輸出 JSON，給 LLM agent 呼叫用。

## 結構
- `wechat_cli/main.py`：click 進入點（`wechat-cli = wechat_cli.main:cli`）
- `wechat_cli/commands/`：各子命令（sessions、history、search、contacts、members、stats、export、favorites、unread、new-messages、init）
- `wechat_cli/core/`：設定、解密、DB 快取、訊息查詢
- `wechat_cli/keys/`：各平台的金鑰掃描（Windows / macOS / Linux）
- `npm/`：npm 發佈用的平台二進位包裝；`entry.py` 是 PyInstaller 進入點

## Build 與安裝
```bash
python -m pip wheel . --no-deps -w dist
python -m pip install --force-reinstall dist/wechat_cli-<version>-py3-none-any.whl
```
開發時可用 `pip install -e .`。

## 測試
```bash
python -m unittest discover tests
```

## 規則
- `*.json`、`*.db` 等使用者資料已被 `.gitignore` 排除，不可 commit。
