---
title: Firebase無料枠停止への解決策
---

# Firebase無料枠停止への解決策

現在、Firebase の無料版制限に到達してゲームが止まっています。このゲームは GitHub Pages 上の静的フロントエンドから Firebase Realtime Database を直接読み書きしており、株価、会社、ユーザー、ニュース、提携、M&A などの状態を Firebase に集めています。

公式情報では、Firebase Spark プランは無料枠を超えると、そのプロダクトの利用がその月の残り期間停止されます。Realtime Database の Spark 枠は、同時接続100、保存1GB、ダウンロード10GB/月です。Firebase Hosting も Spark ではデータ転送360MB/日です。Firestore も無料枠はありますが、読み取り50,000/日、書き込み20,000/日、保存1GiBなどの上限があります。  
参考: [Firebase Pricing](https://firebase.google.com/pricing), [Firebase pricing plans](https://firebase.google.com/docs/projects/billing/firebase-pricing-plans), [Cloud Firestore pricing](https://firebase.google.com/docs/firestore/pricing)

結論として、このゲームに最適なのは「Firebaseを完全に捨てる」よりも、まずは Firebase を認証と重要データだけに絞り、重いリアルタイム株価更新を Firebase から逃がすことです。

## なぜ止まったか

主な原因は、Realtime Database に高頻度で接続・購読・書き込みをしていることです。

| 負荷源 | 今の挙動 | 問題 |
|---|---|---|
| 共通株価 | `market/state` を全員が購読 | 全員に大きなJSONが配信される |
| 株価更新 | ブラウザのリーダーが定期更新 | 書き込み回数とダウンロードが増える |
| ニュース | イベントごとにDB更新 | 全員に配信される |
| ユーザーデータ | 売買、税金、会社操作で保存 | プレイ人数が増えるほど増える |
| 管理機能 | 全プレイヤーや全会社を読む | 管理画面を開くと大きく読む |

Realtime Database は便利ですが、「全員が同じ市場をリアルタイム購読するゲーム」には無料枠だと厳しいです。

## 解決策A: 最短復旧プラン

今すぐ再発を減らすなら、この案です。

| 変更 | 内容 |
|---|---|
| 株価更新間隔を伸ばす | 2秒ごとではなく30秒から60秒ごと |
| `market/state`を軽量化 | 全履歴を毎回保存せず、現在価格だけ中心にする |
| 履歴はローカル生成 | チャート用履歴は各ブラウザで補完 |
| 管理画面読み込みを手動化 | 開いた瞬間に全員分読まない |
| ニュース履歴は最大20件 | 古いニュースを保存しない |

メリット:
- 実装が一番早い
- Firebase をそのまま使える
- 既存コードの修正が少ない

デメリット:
- 根本解決ではない
- プレイヤーが増えるとまた止まる
- Spark の月間10GBダウンロード制限にはまだ弱い

おすすめ度: 短期なら高い、長期なら低い。

## 解決策B: Firebase節約型ハイブリッド

このゲームに一番現実的な案です。

Firebase には以下だけ残します。

| Firebaseに残す | 理由 |
|---|---|
| Auth | ログインは便利で無料枠も比較的使いやすい |
| ユーザー残高 | プレイヤーごとの重要データ |
| 保有株 | 売買結果 |
| 会社データ | 自社、提携、M&A |
| 管理者データ | 権限管理 |

Firebase から外すもの:

| Firebaseから外す | 移動先 |
|---|---|
| 高頻度株価更新 | ブラウザ内の決定論的シミュレーション |
| 株価履歴 | localStorage |
| ニュース演出 | ブラウザ内生成 + 必要時だけ保存 |
| 市場全体のリアルタイム購読 | 定期的な手動同期 |

決定論的シミュレーションとは、全員が同じ seed と時刻を使って同じ株価を計算する方式です。

例:

```js
price = basePrice * formula(stockId, currentMinute, marketSeed)
```

こうすると、Firebase に毎秒株価を書かなくても、全員がほぼ同じ価格を見られます。

メリット:
- Firebase の読み書きが激減する
- GitHub Pages のまま続けられる
- 無料枠でもかなり粘れる
- サーバー不要

デメリット:
- 完全なリアルタイム市場ではなくなる
- チート耐性は強くない
- 売買時に価格検証をどうするか考える必要がある

おすすめ度: 一番おすすめ。

## 解決策C: GitHub Actions市場スナップショット方式

GitHub Actions で数分ごとに市場価格JSONを生成し、GitHub Pages に置く方式です。GitHub Actions は標準のGitHubホストランナーなら public repository で無料利用できます。  
参考: [GitHub Actions billing and usage](https://docs.github.com/en/actions/learn-github-actions/usage-limits-billing-and-administration)

構成:

```txt
GitHub Actions
  5分ごとに market.json を生成
  ↓
GitHub Pages
  /market.json を配信
  ↓
プレイヤー
  market.json を読む
```

Firebase はユーザー保存だけに使います。

メリット:
- Firebase の市場配信負荷がほぼゼロ
- 価格が全員共通になる
- GitHub Pages と相性がいい

デメリット:
- 価格更新は数分単位になる
- GitHub Actions の cron は厳密な秒単位更新には向かない
- GitHub に自動コミットする設計は履歴が増えやすい

おすすめ度: 市場更新が数分ごとでよければ高い。

## 解決策D: Supabase移行

Firebase の代わりに Supabase を使う案です。

使い方:

| 機能 | Supabase |
|---|---|
| ログイン | Supabase Auth |
| DB | Postgres |
| リアルタイム | Supabase Realtime |
| 管理 | SQLで集計しやすい |

メリット:
- SQLでランキング、税金、履歴が作りやすい
- データ構造を整理しやすい
- Firebaseより分析しやすい

デメリット:
- 移行コストが大きい
- 無料枠には別の上限がある
- 今のFirebase Authユーザーをどう移すかが問題

おすすめ度: 中期的にはあり。ただし今すぐ復旧には重い。

## 解決策E: Cloudflare Workers + D1 / KV

Firebase をやめて、Cloudflare Workers をAPIにし、D1やKVに保存する案です。

構成:

```txt
GitHub Pages または Cloudflare Pages
  ↓
Cloudflare Worker API
  ↓
D1 / KV
```

メリット:
- 軽いAPIを作れる
- キャッシュが強い
- 市場価格JSON配信に向いている
- GitHub Pagesからでも呼べる

デメリット:
- 認証を作り直す必要がある
- WorkerやD1の学習コストがある
- 無料枠を超える可能性は残る

おすすめ度: 技術的にはかなり良いが、実装量は多い。

## 解決策F: Blazeに上げて上限監視

Firebase を使い続けたいなら、Blaze に上げる案もあります。Blaze は無料枠を超えた分が従量課金になります。公式ドキュメントでは、Blaze では無料枠超過分に課金され、予算アラートを設定できる一方、アラートは課金停止の上限ではないと説明されています。  
参考: [Firebase pricing plans](https://firebase.google.com/docs/projects/billing/firebase-pricing-plans)

やるなら必須:

| 対策 | 内容 |
|---|---|
| 予算アラート | 100円、500円、1,000円など低め |
| 使用量監視 | Realtime Databaseのダウンロード量を見る |
| Kill switch | 管理画面から市場更新を止める |
| 読み書き制限 | 1ユーザーあたりの操作制限 |
| App Check | 不正アクセスを減らす |

メリット:
- すぐ復旧しやすい
- 既存コードがほぼそのまま
- Cloud Functions なども使える

デメリット:
- 予想外の課金リスクがある
- 今の設計のままだと費用が増えやすい
- 未成年や学校利用だと運用しづらい可能性がある

おすすめ度: 管理者が少額課金を許容できるなら短期復旧向き。完全無料前提なら非推奨。

## このゲームへの最適解

一番よいのは、段階的に次の形へ移ることです。

```txt
GitHub Pages
  画面配信

Firebase Auth
  ログインだけ担当

Firebase Realtime Database
  ユーザー、保有株、会社、提携、M&Aだけ保存

市場価格
  ブラウザ内の決定論的計算
  または GitHub Actions が生成する market.json
```

つまり、Firebase を「全部のリアルタイム同期」ではなく、「プレイヤーのセーブデータ置き場」に格下げします。

## 推奨ロードマップ

### Phase 1: 今日やる応急処置

| 作業 | 目的 |
|---|---|
| 株価更新を60秒ごとにする | 書き込み削減 |
| `market/state/history`を短くする | ダウンロード削減 |
| ニュース履歴を保存しすぎない | DB肥大化防止 |
| 管理画面の全件取得をボタン式にする | 読み取り削減 |
| 市場停止スイッチを作る | 緊急停止 |

### Phase 2: 今週やる本命対応

| 作業 | 目的 |
|---|---|
| 市場価格を決定論的計算に変更 | Firebase依存を激減 |
| 売買時だけ価格を保存 | 重要データだけDBへ |
| チャート履歴をlocalStorageへ | DB通信削減 |
| ユーザーデータ保存をdebounce | 連打書き込み削減 |
| 支援・提携・M&Aにクールダウン | 書き込み削減 |

### Phase 3: 余裕があれば

| 作業 | 目的 |
|---|---|
| GitHub Actionsでmarket.json生成 | 全員共通価格を無料配信 |
| ランキングを静的JSON化 | DB集計を減らす |
| Cloudflare Workers移行検討 | よりゲーム向きのAPI化 |
| Blaze + 予算監視 | 無料にこだわらない場合の安定化 |

## 具体的な実装案

### 案1: 決定論的株価

Firebase に保存するのはこれだけです。

```json
{
  "marketConfig": {
    "seed": "2026-05-season-1",
    "epoch": 1770000000000,
    "volatility": 0.02
  }
}
```

各ブラウザは現在時刻から価格を計算します。

```js
const minute = Math.floor((Date.now() - epoch) / 60000);
const price = calculatePrice(stockId, basePrice, seed, minute);
```

これなら、価格表示だけではFirebase通信が発生しません。

### 案2: market.json

GitHub Pages に以下を置きます。

```json
{
  "updatedAt": 1770000000000,
  "prices": {
    "AAPL": 26800,
    "NVDA": 18400
  }
}
```

ブラウザは1分から5分ごとにこのJSONを読むだけです。

Firebase Realtime Database よりも、GitHub Pages の静的配信の方がこの用途には向いています。

### 案3: Firebaseを使う部分を絞る

Firebaseに残すデータ:

| データ | 理由 |
|---|---|
| `stock_users/{uid}` | セーブデータ |
| `companies/{id}` | プレイヤー企業 |
| `admins/{uid}` | 管理者権限 |
| `warnings/{uid}` | 警告 |

Firebaseから消す・縮小するデータ:

| データ | 対応 |
|---|---|
| `market/state/history` | localStorageへ |
| `market/state/prices` | market.jsonか計算式へ |
| `market/leader` | 廃止 |
| 高頻度ニュース | 最大20件だけ |

## 結論

完全無料で安定させたいなら、最適解は次です。

1. Firebaseはログインとセーブデータだけにする
2. 株価はFirebaseでリアルタイム同期しない
3. 価格は決定論的計算、またはGitHub Pagesの`market.json`で配信する
4. Firebaseへの書き込みは「売買確定」「会社操作」「提携/M&A」だけにする
5. 管理画面やランキングは必要な時だけ読み込む

一番おすすめは「Phase 1で応急処置しつつ、Phase 2で決定論的株価に移行」です。これならゲーム性をあまり落とさず、Firebase無料枠にかなり強くできます。
