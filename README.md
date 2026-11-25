# Windows UI Automation Helper (UIAHelper) 完全ガイド

Windows デスクトップアプリケーションの操作を PowerShell から直感的に自動化するためのヘルパーモジュール **UIAHelper.psm1** の解説ドキュメントです。本ドキュメントは、GitHub Pages（`README.md`）や Qiita などの技術情報共有プラットフォームでの公開に適した Markdown 形式で整理されています。

---

## 1. モジュール概要

`UIAHelper` は、.NET Framework の **System.Windows.Automation**（UI Automation: UIA）および Win32 API（`user32.dll`）をラップし、PowerShell スクリプトからデスクトップ UI 要素の探索・操作・同期を容易にする自動化ライブラリです。

### 主な特徴

* **直感的なコマンドレット群**: ウィンドウの捕捉、ボタン押下、テキスト入力、リスト選択などの頻出操作を専用コマンドレットで提供します。
* **効率的な要素探索メカニズム**: `ControlType` や `NameProperty` などの条件（PropertyCondition）を内部で生成し、キャッシュ管理します。
* **安定した入力待ち同期 (`_WaitForInputIdle`)**: UI 要素の描画やレスポンスを自動的に確認・待機し、タイミング依存による自動化エラーを防止します。
* **GUI ツリーエクスプローラー内蔵 (`Show-UiaElementTreeGui`)**: 画面上の UI 階層を対話的に視覚化し、プロパティ確認やテスト用コードの自動生成を可能にします。
* **スクリーンショットキャプチャ (`Save-UiaScreenCapture`)**: 画面全体、特定のウィンドウ、または各 UI 要素単位でのグラフィックキャプチャに対応します。

---

## 2. アーキテクチャと動作メカニズム

モジュール内部は、安定性と処理速度を両立させるため 3 層構造（Win32 API レイヤー、内部ヘルパーレイヤー、公開コマンドレットレイヤー）で設計されています。

```text
[公開コマンドレット (例: Push-UiaButton, Get-UiaWindow)]
         │
         ▼
[内部ヘルパー関数 (例: _GetButton, _FindFirst, _WaitForInputIdle)]
         │
         ├── モジュール広域変数 (例: $_TimeOut_Period, $_ControlTypeTable)
         ▼
[Windows UI Automation (UIA) & Win32 API (Win32IO.Helper)]
```

### 内部基盤と初期化メカニズム

* **アセンブリ動的読み込み (`_GetUiaRootElement`)**:  
  `UIAutomationClient`、`UIAutomationTypes`、`UIAutomationProvider` を自動読み込みし、`AutomationElement.RootElement`（デスクトップルート）を保持します。また、`UIAutomationClientSideProviders` のクライアント側プロバイダ登録処理を包含しています。
* **Win32 API 連携 (`Win32IO.Helper`)**:  
  `user32.dll` の `GetForegroundWindow`、`GetWindowText`、`SetForegroundWindow` を C# 型定義（`Add-Type`）経由で呼び出し、UIA だけでは制御が難しいフォアグラウンド処理やアクティブ化を確実に行います。
* **条件キャッシュ (`$_ControlTypeTable`)**:  
  各 `ControlType`（Button、Edit、Window 等）に対応する `PropertyCondition` をハッシュテーブルにキャッシュし、繰り返しの要素検索を高速化します。

---

## 3. クイックスタート & インストール

### モジュールの読み込み

スクリプトの先頭でモジュールファイル `.psm1` をインポートします。

```powershell
# UIAHelper モジュールのインポート
Import-Module .\UIAHelper.psm1

# 動作テスト：現在のデスクトップルート要素を取得
$root = Get-UiaRootElement
Write-Host "UIA Root Element Loaded: $($root.Current.Name)"
```

---

## 4. 主要コマンドレット リファレンス

### 4.1 ウィンドウ制御

| コマンドレット | 概要 | 主な引数 |
| :--- | :--- | :--- |
| `Get-UiaWindow` | 指定タイトルやプロセス ID に一致するウィンドウを取得 | `-WindowTitle`, `-ProcID`, `-TimeoutMS` |
| `Set-UiaWindowActive` | 指定ウィンドウを前面（フォアグラウンド）に移動しフォーカス設定 | `-WindowTitle`, `-ProcID` |
| `Set-UiaWindowVisualState`| ウィンドウの表示状態（Normal, Minimized, Maximized）を変更 | `-win`, `-Normal`, `-Minimized`, `-Maximized` |
| `Close-UiaWindow` | `WindowPattern.Close()` を呼び出してウィンドウを終了 | `-WindowTitle`, `-ProcID` |
| `Wait-UiaWindowClose` | 指定ウィンドウが閉じるまでループ待機 | `-WindowTitle`, `-TimeoutMS` |

#### コード例：ウィンドウの捕捉とアクティブ化

```powershell
# 電卓ウィンドウを取得し、最前面に移動
$win = Get-UiaWindow -WindowTitle "電卓"
Set-UiaWindowActive -WindowTitle "電卓"
```

---

### 4.2 コントロール操作

| コマンドレット | 概要 | 対応コントロール / パターン |
| :--- | :--- | :--- |
| `Push-UiaButton` | ボタンを検索してクリック | `ControlType.Button` / `InvokePattern` |
| `Set-UiaEditValue` | テキストボックス等に文字列を設定 | `ControlType.Edit` / `ValuePattern` |
| `Select-UiaRadioButton` | ラジオボタンを選択 | `ControlType.RadioButton` / `SelectionItemPattern` |
| `Set-UiaCheckBoxValue` | チェックボックスの状態（On / Off）を設定 | `ControlType.CheckBox` / `TogglePattern` |
| `Set-UiaComboBoxValue` | コンボボックスの値を設定または項目選択 | `ControlType.ComboBox` / `ValuePattern`, `SelectionItemPattern` |
| `Select-UiaListItem` | リスト内の指定項目を選択（スクロール移動対応） | `ControlType.ListItem` / `SelectionItemPattern`, `ScrollItemPattern` |

#### コード例：フォームへの一括入力

```powershell
# 1. 電卓での連続ボタン操作（パイプライン連携）
$win = Get-UiaWindow -WindowTitle "電卓"
$win | Push-UiaButton -Name "7"
$win | Push-UiaButton -Name "プラス"
$win | Push-UiaButton -Name "8"
$win | Push-UiaButton -Name "等号"

# 2. メモ帳へのテキスト入力
Set-UiaEditValue -Name "テキスト エディター" -Value "UI Automation Test Successful" -WindowTitle "無題 - メモ帳"
```

---

### 4.3 ツリー解析・キャプチャユーティリティ

| コマンドレット | 概要 |
| :--- | :--- |
| `Show-UiaElementTreeGui` | 視覚的な UI ツリーエクスプローラー画面（Windows Forms GUI）を起動 |
| `Get-UiaElementTree` | 指定要素配下の UI ツリーをテキストまたはカスタムオブジェクトで走査・取得 |
| `Save-UiaScreenCapture` | 指定要素やウィンドウのスクリーンショットを切り出し画像保存 |

#### コード例：キャプチャの保存

```powershell
# 電卓ウィンドウ全体を画像として出力
Save-UiaScreenCapture -Template "C:\Logs\Calculator_Test.png" -WindowTitle "電卓"
```

---

## 5. 視覚的解析ツール：Show-UiaElementTreeGui

要素の名前（`Name` プロパティ）や階層構造が不明な場合、GUI エクスプローラーを起動して直接調査できます。

```powershell
# GUI エクスプローラーの起動
Show-UiaElementTreeGui
```

### GUI エクスプローラーの機能と見方

1. **左パネル（UI ツリー）**:
   * アプリケーションのコントロール階層がツリー表示されます。
   * 操作可能なパターン（`InvokePattern` や `ValuePattern` 等）を保持しているコントロールは**青色・太字**でハイライトされます。
2. **右パネル（プロパティ・パターン表示）**:
   * 選択した要素の `AutomationId`、`ClassName`、`IsEnabled` などの基本情報や、サポートされている `Patterns` 一覧が表示されます。
3. **自動コード生成機能**:
   * ツリー上で要素を選択すると、その要素を操作するための PowerShell コード例（`$win = Get-UiaWindow ...`）が自動的に画面下部に生成されます。コピーしてテストスクリプトにそのまま貼り付け可能です。

---

## 6. GUI テスト自動化のベストプラクティス

### 1. タイミング依存エラーの回避
手動での `Start-Sleep` による固定時間待機は、環境負荷によってテストが不安定になる原因となります。本モジュールの `Push-UiaButton` や `Set-UiaEditValue` には、内部で `_WaitForInputIdle`（`WindowPattern.WaitForInputIdle`）が組み込まれているため、適切な待ち合わせが自動で行われます。

### 2. 要素のパイプライン処理
`Get-UiaWindow` で取得したウィンドウオブジェクトを変数に保持するか、パイプラインで後続の操作関数に渡すことで、毎回ウィンドウ全体を再検索するオーバーヘッドを削減できます。

```powershell
# 推奨：ウィンドウオブジェクトをパイプラインで渡す
Get-UiaWindow -WindowTitle "設定" | Select-UiaListItem -Name "システム" -ScrollIn
```

---

## 7. トラブルシューティング

* **要素が見つからない（Time Out）**:  
  対象アプリケーションが管理者権限で実行されている場合、PowerShell セッションも管理者権限で起動する必要があります。また、`Show-UiaElementTreeGui` で実際の `Name` や `ControlType` が意図通りか確認してください。
* **ボタンが押し込めない**:  
  コントロールの `IsEnabled` プロパティが `False` になっているか、別ウィンドウの背後に隠れている可能性があります。操作前に `Set-UiaWindowActive` を実行して対象ウィンドウをフォアグラウンドに移動させてください。

---

### ライセンス & 貢献

本モジュールは Windows Automation の標準アセンブリに依存して動作します。スクリプトの改善案や機能追加の要望はリポジトリの Issue / Pull Request にて受け付けています。
