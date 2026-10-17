# LINE会話をMac miniのローカルスクリプトに取り込む、ユーザー側の正当な手段(2026年10月時点)

注記: LINE公式ヘルプ(help.line.me)は検索/取得ツールから本文を取得できなかった。以下の「公式」表記は terms.line.me の規約本文のみ。機能面の記述は第三者記事ベースで、公式ヘルプでの再確認が必要。

## 1. LINE公式のトーク履歴機能(テキスト送信/エクスポート/バックアップ/PC版)と自動化可否

### Takeaway
公式に用意されているのは「トークごとの手動テキスト保存/送信」のみで、時間単位の自動エクスポートを行う公式手段は確認できなかった。PC/Mac版は単体でバックアップ機能を持たず、API的な読み出し手段も公式には存在しない。

### Cited Findings
- スマホアプリの「トーク履歴を送信」でトークルーム単位に.txtを作成しメール等で送れる。画像・動画・スタンプはラベルのみで、全トーク一括は不可、テキスト書き出しからトークルームへの復元は不可 — [appllio](https://appllio.com/line-send-talk-history-from-friend)
- PC版LINEはトークを開き「…」→「トークを保存」で .txt または .html を選んでフォルダ保存でき、写真・動画・スタンプは含まれない。PC版に直接のバックアップ機能はない、との記述あり(第三者記事) — [minto.tech](https://minto.tech/line-chat-history-export-pc/)、検索結果要約より。PC版は初回ログイン前のメッセージを表示しないという記述もある(同系統の第三者記事、一次確認なし)
- 第三者記事「LINE Mac版 バックアップ」の検索では、MacでのバックアップがOS標準で不可との明確な公式記述は得られなかった — 検索結果は gori.me の画像ページのみで内容未確認
- 外部検証済み: LINEメッセンジャーはDBを暗号化しており、鍵生成要素はサーバー通信で受け取る、とする学術論文がある(対象OSはスニペットでは不明) — [MDPI Electronics 13(7):1325](https://www.mdpi.com/2079-9292/13/7/1325)
- 非公式OSS「line-cua-mcp」は、MacのLINEをMCP経由で操作しローカルのDBスナップショットを読むと自称している。コミュニティ製で公式サポートではなく、後述の規約リスクがあり、本ノートでは推奨しない — [remoteopenclaw](https://www.remoteopenclaw.com/mcp/andrew54068/line-cua-mcp)

### Inferences
- 暗号化DBを公式に読む手段が無い以上、「ローカルDBを直接読む」案は非サポート。本調査では暗号の解除手順は扱わない。
- 公式のエクスポートは人手のUI操作(PC版の「トークを保存」)を前提とする。iOSショートカットでLINEの「トーク履歴を送信」を自動起動できるという公式情報は見つからなかった(確認できず、可能性は低いと推測)。
- 保存した.txtには送信者名と時刻が含まれる形式のはずだが、形式の一次情報は未確認。

### Gaps
- LINE公式ヘルプのPC/Mac版仕様(保存期間、保存メニュー、Keep)を取得できなかった。
- Keepで他メンバーの発言を自動取得する方法は調査できなかった。
- Google Drive/iCloudへの自動バックアップはトーク履歴バックアップ(LINEアプリ内機能)だが、これはLINE専用形式で外部から読めない可能性が高く、一次確認できていない。

## 2. 通知ベースの取得

### Takeaway
通知本文に内容が出る設定にしておけば、iPhone通知のMac転送やmacOSの通知DBから「新着の断片」は拾えるが、Sequoia以降は通知DBが保護領域に移動しており、切り詰め・要約・グループ名のみ表示などで網羅的な会話ログには向かない。

### Cited Findings
- LINEの通知設定には「メッセージ内容を表示しない」があり、オンにすると「新着メッセージがあります」だけが表示され、名前も内容も出ない — [ASCII.jp](https://weekly.ascii.jp/elem/000/002/635/2635749/)、[appllio](https://appllio.com/line-notification-hidden)
- macOS 15 Sequoiaは通知センターのDBをTCC保護されたグループコンテナへ移し、従来のsqlite3一発での読み出しが不可になった。以前は /private/var/folders 配下 — [mjtsai.com](https://mjtsai.com/blog/2024/07/15/sequoia-finally-addresses-notification-center-privacy)、[heise](https://www.heise.de/en/news/Security-risk-notifications-macOS-15-seals-Mac-better-9802404.html?view=print)
- 同記事のコメントでは、回避策としてFull Disk AccessやOCRが提案されているが、Apple公式の推奨ではない — [mjtsai.com](https://mjtsai.com/blog/2024/07/15/sequoia-finally-addresses-notification-center-privacy)
- iPhone通知はSequoia+iOS 18(同一Apple ID、Apple siliconまたはT2搭載Mac)でMacの通知センターに表示でき、アプリ単位でオンオフ可能。通知クリックでiPhone Mirroringが起動する — [Tom's Guide](https://tomsguide.com/computing/macos/macos-sequoia-7-things-you-need-to-know-about-iphone-mirroring)、[MacRumors](https://macrumors.com/guide/iphone-mirroring/0)
- 初期のEUでは iPhone Mirroring が未提供(DMA)との報道。現状は未確認 — [MacRumors](https://macrumors.com/guide/iphone-mirroring/0)

### Inferences
- Mac版LINEをMac mini上で動かして通知を出させる案もあるが、公式のMac版を常時ログインさせても、通知DBはTCC領域で、読むにはFull Disk Accessの許可が必要(ユーザーが明示許可する形)。
- 通知はグループ名と送信者名の表示は設定次第で、複数メッセージの統合・省略があり得る。取りこぼしがあるため「今日の全メッセージ」の保証はできない。
- Androidでは通知リスナーアプリで本文を取れるが、同様に切り詰め・取りこぼしがあり、スマホの常時稼働が要る(一次情報は未確認)。

### Gaps
- 通知の最大文字数、グループ通知の形式(「グループ名: 送信者: 本文」か否か)の一次情報は未取得。
- Sequoia後の通知DBの正確なパスとスキーマは未確認(意図的に深掘りしていない)。

## 3. 画面ベース(Accessibility/OCR/GUIエージェント)

### Takeaway
Mac版LINEのウィンドウをApple標準のAccessibility API/Vision OCR/GUIエージェントで読む方式は技術的に成立しうるが、規約上のグレーさとUI変更への脆さがあり、一次裏付けは取れていない。

### Cited Findings
- LINE規約15(5)は「BOT、チートツールその他の技術的手段を利用して当社サービスを不正に操作する行為」を禁止し、15(6)は不当な目的/態様のリバースエンジニアリングを禁止、ソフトウェア規則3(6)は当社が不適当とみなした方法でのソフトウェア利用を禁止 — [LINE利用規約](https://terms.line.me/line_terms/?lang=ja)
- 規約は「非公式クライアント」を明示していない — 同上

### Inferences
- 画面を読むだけの個人利用が「不正な操作」に当たるかは規約文言上は断定できない。ただし自動操作(スクロール、クリック)はBOT的と解釈される余地があり、アカウント制限リスクは「低くはない」と見るのが妥当。
- 読み取り専用(ユーザー自身が開いたウィンドウをOCR)より、スクロール/クリックを伴う方がリスクが高い。
- Hermesのcomputer_useはGUI操作なので同様。ウィンドウ配置・ダークモード・アプリ更新で壊れやすい。
- 技術的前提: Screen Recording/Accessibility許可が要る(macOSの一般的仕様、本調査では未引用)。

### Gaps
- LINE Mac版がAccessibilityツリーにメッセージ本文を公開するか、実機検証していない。

## 4. HermesエージェントのDockerからの到達性(高水準)

### Takeaway
どの方式でも、取得処理はホスト(Mac mini)側で行い、出力ファイルをDockerのボリュームマウントでHermesに渡すのが現実的。コンテナ内からmacOSのGUI/通知/Accessibilityには到達できない。

### Inferences(一次情報なし)
- ホストのlaunchdで毎時スクリプトを走らせ、`~/line_export/` のようなフォルダに.txt/JSONを書き、それを読み取り専用でコンテナにマウントする。
- computer_useツールセットがコンテナ内で動く場合、対象は仮想環境のデスクトップであり、ホストのMac版LINEには届かない可能性が高い(Hermesの実装は未確認)。

### Gaps
- Hermes側の仕様(computer_use、マウント設定)は本調査では未確認。

## 5. グループのメンバー一覧と発言時刻

### Takeaway
公式エクスポート(.txt)には発言時刻と送信者名が含まれる見込みだが、メンバー一覧は含まれず、「Aさんがグループにいない」判定は公式エクスポート/通知のみでは不確実。

### Inferences
- 「トークを保存」のテキストは日付・時刻・名前付き行を持つとされる(一次確認なし)が、退会者/参加履歴は「〇〇が退出しました」等のシステムメッセージで推測できる程度。
- メンバー一覧はアプリのUI(グループ詳細画面)にあり、画面ベースでしか取れない。通知にはメンバー情報が含まれない。
- 発言時刻は通知到着時刻で代替できるが、遅延する。

### Gaps
- メンバー一覧を公式機能でエクスポートする手段は見つからなかった。

## 6. 日本での私的利用・プライバシーの簡易メモ

### Takeaway
個人が私的にメッセージを自分用に処理するだけなら、一般にはリスクは小さいが、他メンバーの個人情報を収集・蓄積・提供する行為はLINE規約と個人情報保護法の観点で注意が必要。法的助言ではない。

### Cited Findings
- LINE規約15(8)は第三者に関する個人情報等を「不正に」収集・蓄積・開示・提供する行為を禁止。第14条はサービスやそれを構成するデータの提供目的を超えた利用を禁止 — [LINE利用規約](https://terms.line.me/line_terms/?lang=ja)
- 個人情報保護委員会のページは法令・ガイドラインの目次のみで、私的利用の扱いは確認できなかった — [PPC](https://www.ppc.go.jp/personalinfo/legal/)

### Inferences
- 個人情報保護法は事業として個人情報データベースを扱う者(個人情報取扱事業者)が対象。私的・家庭内の利用は通常対象外と考えられるが、一次確認はできていない(要確認)。
- 外部クラウドLLMに他メンバーの発言を送る場合は、メンバーの同意/プライバシー上の配慮が望ましい。ローカル処理に限定すれば心理的・法的リスクは小さい。

### Gaps
- 個人情報保護法の私的利用に関する条文とガイドライン(通則編)を直接確認できていない。
