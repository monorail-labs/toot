<!-- File: README.md -->
<div align="center">

<a id="top"></a>

# TOOT

**Take Out Of Twelve**

A mobile pre-ordering and pickup system for school festivals and other events.

[![Version](https://img.shields.io/badge/version-v1.0.0-000000?style=flat-square)](#)
[![React](https://img.shields.io/badge/React-19-61DAFB?style=flat-square&logo=react&logoColor=white)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-6.0-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Rust](https://img.shields.io/badge/Rust-%E2%89%A51.77.2-000000?style=flat-square&logo=rust&logoColor=white)](https://www.rust-lang.org/)
[![Tauri](https://img.shields.io/badge/Tauri-v2-24C8DB?style=flat-square&logo=tauri&logoColor=white)](https://v2.tauri.app/)
[![SQLite](https://img.shields.io/badge/SQLite-local%20replica-003B57?style=flat-square&logo=sqlite&logoColor=white)](https://www.sqlite.org/)
[![Supabase](https://img.shields.io/badge/Supabase-backend-3FCF8E?style=flat-square&logo=supabase&logoColor=white)](https://supabase.com/)

</div>

> [!NOTE]
> **日本語のドキュメントはこちら / Japanese documentation is available below** → [日本語ドキュメントへ移動 / Jump to Japanese](#japanese)

---

<a id="english"></a>

## English Documentation

### Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [System Architecture](#system-architecture)
- [Tech Stack & Tooling](#tech-stack--tooling)
- [Quick Start](#quick-start)
- [Legal & Governance](#legal--governance)
- [Roadmap](#roadmap)
- [Team](#team)

---

### Overview

TOOT lets festival visitors order food from their phone, pick a 15-minute pickup slot, and collect the order at the counter by showing a 6-character code. Staff run a desktop app that keeps a local SQLite copy of the orders, so counter lookups keep working when the venue Wi-Fi drops.

It is made of two applications built from one Vite project:

| App | Entry | Runs on | Purpose |
| --- | --- | --- | --- |
| **Customer web app** | `index.html` → `src/main.tsx` | Phone browser | Menu → pickup slot → ticket with pickup code |
| **Staff desktop app** | `staff.html` → `src/staff.tsx` | Tauri v2 window | Pickup counter mode and kitchen mode |

Design principles:

1. **Zero-PII** — No accounts, no names, phone numbers or email addresses. An order is identified only by its pickup code.
2. **Local replica at the counter** — The staff app reads from a local SQLite database, so lookup and redemption do not wait on the network.
3. **Capacity-bounded intake** — Each pickup slot accepts a fixed number of orders, and each item has a per-order quantity cap.

---

### How It Works

#### 1. Customer flow

1. **Menu** (`/`) — Choose quantities. Each menu item is capped at **3 per order** (`MAX_ITEM_QUANTITY`). Remaining stock per item is shown as `maxQuantity` (currently a placeholder of 200) minus the total already ordered, read from the Supabase view `product_quantity_totals`. Pressing "受け取り時間を選ぶ" (choose a pickup time) counts as agreeing to the Terms of Service and Privacy Policy.
2. **Pickup slot** (`/pickup-time`) — Slots run every **15 minutes from 10:00 to 15:00** (20 slots). Availability is computed from the number of non-cancelled orders in each slot against a capacity of **5 orders per slot**:

   | Fill ratio | Mark | Meaning |
   | --- | --- | --- |
   | < 50% | ◎ | Available |
   | ≥ 50% | ○ | Filling up |
   | ≥ 80% | △ | Few left |
   | 100% | × | Full |
   | — | 締切 | Closed: ordering for a slot stops 15 minutes before it starts |
   | — | 終了 | Past: the slot has ended |

   The grid refreshes every 60 seconds and is re-checked right before submitting. If availability cannot be fetched (e.g. offline), every slot is shown as closed.
3. **Submit** — The client counts the slot's current orders, and if the slot is not full, `INSERT`s the order into the Supabase `orders` table (`slot_id`, `items`, `total_price`, `status: "pending"`). The **pickup code is issued on the Supabase side** and returned from the insert.
4. **Ticket** (`/ticket`) — The code, pickup time and items are saved to `localStorage` and shown as a ticket. While a ticket is stored, opening the menu redirects to the ticket. "次の注文を始める" (start next order) clears it after a confirmation.

#### 2. Pickup codes (Crockford's Base32)

- Codes are 6 characters from the [Crockford's Base32](https://www.crockford.com/base32.html) alphabet `0123456789ABCDEFGHJKMNPQRSTVWXYZ`, which omits the easily confused `I`, `L`, `O` and `U`. The keyspace is $32^6 = 1{,}073{,}741{,}824$.
- Codes are generated **server-side** on purpose, so a client cannot choose its own code (for example `AAAAAA` via DevTools). The Supabase table and its code generation are defined outside this repository.
- In the staff app, `auth_code` is `UNIQUE` across the whole local database, and lookup is by code alone.
- The counter input is upper-cased and validated against `^[0-9A-HJKMNP-TV-Z]{6}$` before lookup.
- `src/lib/crockfordBase32.ts` contains a client-side generator (`generateAuthCode`, using `crypto.getRandomValues`) that the current order flow does not use.

#### 3. Staff desktop app

The staff app has two modes, switched from the settings (gear) menu and remembered in `localStorage`.

- **Pickup counter mode (受取窓口)** — Enter the customer's code to look it up in local SQLite (indexed on `auth_code`). The matched order shows items, slot and total. "受け渡しを完了する" (complete handover) marks the order `completed` in SQLite first, then sends the same update to Supabase. If the Supabase update fails, it is logged and **not retried**.
- **Kitchen mode (厨房キュー)** — Groups all non-completed orders by `slot_id` and sums quantities per item. The earliest slot with open orders is shown as "いま作る枠" (cook now) with a per-item cooked counter; the next two slots are shown as "次の準備" (up next). The cooked counters live in memory only and are reset when a slot leaves the list.

#### 4. Sync between Supabase and SQLite

`src/hooks/useOrderSync.ts` (run by the kitchen view) keeps the local database up to date:

- **Initial sync** — On start, fetches every order from Supabase and upserts it into SQLite with the `sync_orders` Tauri command.
- **Realtime** — Subscribes to Supabase Realtime `INSERT` and `UPDATE` events on `orders` and upserts each one. Disconnects are retried with backoff (1 s, 2 s, 5 s, 10 s, 30 s).
- **Upsert rule** — Upserts never overwrite a local `completed` status, so a redemption made offline is not reverted by later sync.
- **Refresh** — Bursts of changes are debounced (250 ms) before the kitchen view re-reads SQLite.

The SQLite file is `culture_festival.db`, created in the app's working directory. Debug builds insert three sample orders into an empty database.

#### 5. Privacy by design (Zero-PII)

- No account creation or sign-in for customers.
- The order tables hold only slot, items, total, status, timestamps and the pickup code — no names, phone numbers or email addresses.
- The customer's ticket is stored only in their own browser's `localStorage`.

---

### System Architecture

~~~text
+-----------------------------+                       +-------------------------------+
|  Customer Web App           |  count slot orders    |  Supabase                     |
|  React 19 + React Router    | --------------------> |  orders table                 |
|  (index.html)               |  INSERT order         |   - issues auth_code          |
|                             | --------------------> |  product_quantity_totals view |
|  - Menu (3 per item)        | <-------------------- |  Realtime (postgres_changes)  |
|  - 15-min slots, 5 / slot   |  auth_code            |                               |
|  - Ticket in localStorage   |                       |                               |
+-----------------------------+                       +---------------+---------------+
                                                         ^            |
                                    UPDATE status =      |            | initial fetch +
                                    'completed'          |            | Realtime INSERT/UPDATE
                                    (best effort)        |            v
                                                      +--+----------------------------+
                                                      |  Staff Desktop App (Tauri v2) |
                                                      |  staff.html + Rust commands   |
                                                      |                               |
                                                      |  +-------------------------+  |
                                                      |  | SQLite (rusqlite)       |  |
                                                      |  | culture_festival.db     |  |
                                                      |  +------------+------------+  |
                                                      |               |               |
                                                      |      +--------+--------+      |
                                                      |      |                 |      |
                                                      |  [Pickup Counter] [Kitchen]   |
                                                      |   lookup_order     get_orders |
                                                      |   complete_order   per-slot   |
                                                      |                    totals     |
                                                      +-------------------------------+

 Offline: lookup and redemption use SQLite only. New orders arrive once Realtime reconnects.
 Completion sent while offline is applied locally but not re-sent to Supabase.
~~~

Tauri commands (`src-tauri/src/lib.rs`): `lookup_order`, `complete_order`, `get_orders`, `sync_orders`.

---

### Tech Stack & Tooling

| Layer | Technology |
| --- | --- |
| **Frontend** | React 19, React Router 7, TypeScript ~6.0, Vite 8, React Compiler (via `@rolldown/plugin-babel`) |
| **Desktop** | Tauri v2, Rust (edition 2021, MSRV 1.77.2), SQLite via `rusqlite` (bundled), `tauri-plugin-log` (debug builds) |
| **Backend** | Supabase — PostgreSQL, Realtime (`@supabase/supabase-js` v2) |
| **Tooling** | Oxlint |

---

### Quick Start

#### Prerequisites

- Node.js
- Rust 1.77.2 or later (`rustup`)
- [Tauri v2 system prerequisites](https://v2.tauri.app/start/prerequisites/) for your OS
- A Supabase project with an `orders` table (which issues `auth_code`) and a `product_quantity_totals` view

#### 1. Clone the Repository

~~~bash
git clone https://github.com/monorail-labs/toot.git
cd toot
npm install
~~~

#### 2. Environment Variables

Create `.env` from the template:

~~~bash
cp .env.sample .env
~~~

~~~dotenv
VITE_SUPABASE_URL=https://<your-project-ref>.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=<your-publishable-key>
~~~

Both apps throw at startup if either value is missing.

> [!WARNING]
> **Never commit real credentials.** `.env` is excluded via `.gitignore`. Never expose a secret / `service_role` key to the client.

#### 3. Customer Web App

~~~bash
npm run dev
~~~

Opens on `http://localhost:5173`. The staff UI is also served at `/staff.html`, but its Tauri commands only work inside the desktop app.

#### 4. Staff Desktop App

~~~bash
npm run tauri dev     # development
npm run tauri build   # production bundle
~~~

#### 5. Verification

~~~bash
npm run lint    # oxlint
npm run build   # tsc -b && vite build
~~~

---

### Legal & Governance

| Document | Scope |
| --- | --- |
| [`public/terms.html`](./public/terms.html) | **Terms of Service.** The disclaimer is a partial exemption: liability for intent or gross negligence is kept, and liability for slight negligence is limited to direct, ordinary damages. The menu page states that proceeding to pickup-time selection counts as agreement. |
| [`public/privacy.html`](./public/privacy.html) | **Privacy Policy.** Zero-PII declaration and the three kinds of data handled: order data in Supabase, the ticket in `localStorage`, and infrastructure logs. |

---

### Roadmap

- [ ] **Atomic slot reservation** — The capacity check and the `INSERT` are currently two separate requests, so concurrent orders can exceed 5 per slot. Move both into one PostgreSQL function (RPC) that runs in a single transaction.
- [ ] **Retry for completion sync** — Queue completions that fail to reach Supabase and resend them on reconnect.
- [ ] **Distributed code issuance** — Move away from server-only code issuance while keeping protection against client-chosen codes.

~~~sql
-- Conceptual sketch
CREATE OR REPLACE FUNCTION reserve_order(p_slot_id text, p_items jsonb, p_total_price integer)
RETURNS text
LANGUAGE plpgsql
AS $$
DECLARE
  v_code text;
BEGIN
  PERFORM pg_advisory_xact_lock(hashtext(p_slot_id));

  IF (SELECT count(*) FROM orders
       WHERE slot_id = p_slot_id AND status <> 'cancelled') >= 5 THEN
    RETURN NULL;
  END IF;

  INSERT INTO orders (slot_id, items, total_price, status)
  VALUES (p_slot_id, p_items, p_total_price, 'pending')
  RETURNING auth_code INTO v_code;

  RETURN v_code;
END;
$$;
~~~

---

### Team

Developed by **Team MONO Rail**.

<p align="right"><a href="#top">Back to top ↑</a></p>

---
---

<a id="japanese"></a>

## 日本語ドキュメント

### 目次

- [概要](#jp-overview)
- [仕組み](#jp-how)
- [システム構成図](#jp-diagram)
- [技術スタックとツール](#jp-stack)
- [クイックスタート](#jp-quickstart)
- [法務・ガバナンス](#jp-legal)
- [ロードマップ](#jp-roadmap)
- [チーム](#jp-team)

---

<a id="jp-overview"></a>

### 概要

TOOT は、文化祭などのイベント向けのモバイル事前注文・受け取りシステムです。来場者はスマートフォンで商品を選び、15 分刻みの受け取り枠を予約し、6 桁のコードを窓口で見せて商品を受け取ります。スタッフ側のデスクトップアプリは注文をローカルの SQLite に複製して持つため、会場の Wi-Fi が切れても窓口での照合は止まりません。

一つの Vite プロジェクトから、二つのアプリをビルドします。

| アプリ | エントリ | 動作環境 | 役割 |
| --- | --- | --- | --- |
| **客側 Web アプリ** | `index.html` → `src/main.tsx` | スマートフォンのブラウザ | メニュー → 受け取り枠 → 受け取りコード付きチケット |
| **スタッフ用デスクトップアプリ** | `staff.html` → `src/staff.tsx` | Tauri v2 のウィンドウ | 受取窓口モードと厨房モード |

設計方針:

1. **Zero-PII** — アカウント・氏名・電話番号・メールアドレスは不要です。注文は受け取りコードのみで識別します。
2. **窓口側のローカル複製** — スタッフアプリはローカルの SQLite を参照するため、照合と消し込みがネットワークを待ちません。
3. **定員による受付制限** — 受け取り枠ごとに受け付ける注文数を固定し、1 注文あたりの各商品の個数にも上限を設けています。

---

<a id="jp-how"></a>

### 仕組み

#### 1. 客側の流れ

1. **メニュー**（`/`）— 個数を選びます。各商品は **1 注文あたり 3 個まで**（`MAX_ITEM_QUANTITY`）です。各商品の残数は、`maxQuantity`（現在は仮の値 200）から、Supabase のビュー `product_quantity_totals` で得た累計注文数を引いて表示します。「受け取り時間を選ぶ」を押すと、利用規約とプライバシーポリシーに同意したものとみなされます。
2. **受け取り枠**（`/pickup-time`）— 枠は **10:00〜15:00 の 15 分刻み**（20 枠）です。各枠のキャンセル以外の注文数を、**1 枠 5 件**の定員と比べて空き状況を出します。

   | 埋まり具合 | 表示 | 意味 |
   | --- | --- | --- |
   | 50% 未満 | ◎ | 空きあり |
   | 50% 以上 | ○ | やや混雑 |
   | 80% 以上 | △ | 残りわずか |
   | 100% | × | 満枠 |
   | — | 締切 | 受付終了。枠の開始 15 分前に締め切る |
   | — | 終了 | 枠の時間が過ぎた |

   枠の一覧は 60 秒ごとに更新され、確定の直前にも再取得します。空き状況を取得できないとき（オフラインなど）は、すべての枠を「締切」として表示します。
3. **確定** — クライアントがその枠の現在の注文数を数え、満枠でなければ Supabase の `orders` テーブルに注文を `INSERT` します（`slot_id`・`items`・`total_price`・`status: "pending"`）。**受け取りコードは Supabase 側で発行**され、`INSERT` の戻り値として受け取ります。
4. **チケット**（`/ticket`）— コード・受け取り時間・注文内容を `localStorage` に保存し、チケットとして表示します。チケットが保存されている間は、メニューを開いてもチケット画面へ戻ります。「次の注文を始める」を押すと、確認のあとチケットを消去します。

#### 2. 受け取りコード（Crockford's Base32）

- コードは [Crockford's Base32](https://www.crockford.com/base32.html) の文字 `0123456789ABCDEFGHJKMNPQRSTVWXYZ` による 6 桁です。紛らわしい `I`・`L`・`O`・`U` を含みません。符号空間は $32^6 = 1{,}073{,}741{,}824$ 通りです。
- コードは意図的に**サーバー側で発行**しています。クライアントが自分でコードを決められる状態（DevTools で `AAAAAA` を送るなど）を防ぐためです。Supabase のテーブル定義とコード生成は、このリポジトリの外で管理しています。
- スタッフアプリでは、`auth_code` はローカル DB 全体で `UNIQUE` であり、照合はコードのみで行います。
- 窓口の入力は大文字に変換し、`^[0-9A-HJKMNP-TV-Z]{6}$` で形式を確認してから照合します。
- `src/lib/crockfordBase32.ts` にはクライアント側の生成関数（`generateAuthCode`、`crypto.getRandomValues` を使用）がありますが、現在の注文処理では使っていません。

#### 3. スタッフ用デスクトップアプリ

設定（歯車）メニューで二つのモードを切り替えます。選んだモードは `localStorage` に保存されます。

- **受取窓口モード** — お客様のコードを入力し、ローカル SQLite から照合します（`auth_code` にインデックスあり）。見つかった注文の品目・受取枠・合計金額を表示します。「受け渡しを完了する」を押すと、まず SQLite 上で `completed` にし、続けて Supabase にも同じ更新を送ります。Supabase への更新に失敗した場合はログに残すだけで、**再送はしません**。
- **厨房モード（厨房キュー）** — 未完了の注文を `slot_id` ごとにまとめ、商品ごとの個数を合計します。未完了の注文がある最も早い枠を「いま作る枠」として、商品ごとの調理済みカウンターとともに表示し、続く 2 枠を「次の準備」として表示します。調理済みカウンターはメモリ上だけに持ち、その枠が一覧から消えるとリセットされます。

#### 4. Supabase と SQLite の同期

`src/hooks/useOrderSync.ts`（厨房モードで動作）がローカル DB を最新に保ちます。

- **初回同期** — 起動時に Supabase から全注文を取得し、Tauri コマンド `sync_orders` で SQLite に upsert します。
- **リアルタイム** — Supabase Realtime で `orders` の `INSERT`・`UPDATE` を購読し、1 件ずつ upsert します。切断時は間隔を広げながら再接続します（1 秒・2 秒・5 秒・10 秒・30 秒）。
- **upsert の規則** — ローカルで `completed` になった注文の状態は上書きしません。オフライン中に消し込んだ注文が、後の同期で元に戻ることはありません。
- **画面の更新** — 短時間に続いた変更は 250 ms まとめてから、厨房画面が SQLite を読み直します。

SQLite のファイルは `culture_festival.db` で、アプリの作業ディレクトリに作られます。デバッグビルドでは、DB が空のときにサンプル注文を 3 件入れます。

#### 5. プライバシー・バイ・デザイン（Zero-PII）

- 客にアカウント作成やログインを求めません。
- 注文テーブルが持つのは、受取枠・品目・合計金額・状態・日時・受け取りコードだけです。氏名・電話番号・メールアドレスは持ちません。
- 客のチケットは、その人のブラウザの `localStorage` にだけ保存されます。

---

<a id="jp-diagram"></a>

### システム構成図

~~~text
+-----------------------------+                       +-------------------------------+
|  Customer Web App           |  count slot orders    |  Supabase                     |
|  React 19 + React Router    | --------------------> |  orders table                 |
|  (index.html)               |  INSERT order         |   - issues auth_code          |
|                             | --------------------> |  product_quantity_totals view |
|  - Menu (3 per item)        | <-------------------- |  Realtime (postgres_changes)  |
|  - 15-min slots, 5 / slot   |  auth_code            |                               |
|  - Ticket in localStorage   |                       |                               |
+-----------------------------+                       +---------------+---------------+
                                                         ^            |
                                    UPDATE status =      |            | initial fetch +
                                    'completed'          |            | Realtime INSERT/UPDATE
                                    (best effort)        |            v
                                                      +--+----------------------------+
                                                      |  Staff Desktop App (Tauri v2) |
                                                      |  staff.html + Rust commands   |
                                                      |                               |
                                                      |  +-------------------------+  |
                                                      |  | SQLite (rusqlite)       |  |
                                                      |  | culture_festival.db     |  |
                                                      |  +------------+------------+  |
                                                      |               |               |
                                                      |      +--------+--------+      |
                                                      |      |                 |      |
                                                      |  [Pickup Counter] [Kitchen]   |
                                                      |   lookup_order     get_orders |
                                                      |   complete_order   per-slot   |
                                                      |                    totals     |
                                                      +-------------------------------+
~~~

- **オフライン時:** 照合と消し込みは SQLite だけで完結します。新しい注文は Realtime の再接続後に届きます。
- **オフライン中の消し込み:** ローカルには反映されますが、Supabase へは再送されません。

Tauri コマンド（`src-tauri/src/lib.rs`）: `lookup_order`・`complete_order`・`get_orders`・`sync_orders`

---

<a id="jp-stack"></a>

### 技術スタックとツール

| レイヤー | 技術 |
| --- | --- |
| **フロントエンド** | React 19、React Router 7、TypeScript ~6.0、Vite 8、React Compiler（`@rolldown/plugin-babel` 経由） |
| **デスクトップ** | Tauri v2、Rust（edition 2021、MSRV 1.77.2）、SQLite（`rusqlite`、bundled）、`tauri-plugin-log`（デバッグビルドのみ） |
| **バックエンド** | Supabase（PostgreSQL、Realtime、`@supabase/supabase-js` v2） |
| **ツール** | Oxlint |

---

<a id="jp-quickstart"></a>

### クイックスタート

#### 前提条件

- Node.js
- Rust 1.77.2 以降（`rustup` で導入）
- 各 OS 向けの [Tauri v2 前提環境](https://v2.tauri.app/start/prerequisites/)
- `orders` テーブル（`auth_code` を発行するもの）と `product_quantity_totals` ビューを持つ Supabase プロジェクト

#### 1. リポジトリの取得

~~~bash
git clone https://github.com/monorail-labs/toot.git
cd toot
npm install
~~~

#### 2. 環境変数

テンプレートから `.env` を作成します。

~~~bash
cp .env.sample .env
~~~

~~~dotenv
VITE_SUPABASE_URL=https://<your-project-ref>.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=<your-publishable-key>
~~~

どちらかが未設定だと、両アプリとも起動時にエラーになります。

> [!WARNING]
> **実際の認証情報は絶対にコミットしないでください。** `.env` は `.gitignore` で除外されています。secret / `service_role` キーはクライアントに渡さないでください。

#### 3. 客側 Web アプリ

~~~bash
npm run dev
~~~

`http://localhost:5173` で開きます。スタッフ画面も `/staff.html` で表示されますが、Tauri コマンドはデスクトップアプリの中でしか動きません。

#### 4. スタッフ用デスクトップアプリ

~~~bash
npm run tauri dev     # 開発
npm run tauri build   # 本番ビルド
~~~

#### 5. 検証

~~~bash
npm run lint    # oxlint
npm run build   # tsc -b && vite build
~~~

---

<a id="jp-legal"></a>

### 法務・ガバナンス

| ドキュメント | 内容 |
| --- | --- |
| [`public/terms.html`](./public/terms.html) | **利用規約。** 免責条項は一部免責です。故意・重過失による損害は責任を負い、軽過失による損害は直接かつ通常の損害に限ります。メニュー画面で、受け取り時間の選択へ進むことをもって同意とみなす旨を表示しています。 |
| [`public/privacy.html`](./public/privacy.html) | **プライバシーポリシー。** Zero-PII 宣言と、取り扱う 3 種類のデータ（Supabase 上の注文データ、`localStorage` のチケット、インフラのログ）を定めています。 |

---

<a id="jp-roadmap"></a>

### ロードマップ

- [ ] **枠予約の原子化** — 現在は定員の確認と `INSERT` が別々のリクエストのため、同時に注文が来ると 1 枠 5 件を超えることがあります。両方を一つのトランザクションで実行する PostgreSQL 関数（RPC）に移します。
- [ ] **完了同期の再送** — Supabase に届かなかった消し込みをキューに残し、再接続時に送り直します。
- [ ] **コード発行の分散化** — クライアントによるコードの指定を防ぐ仕組みを保ったまま、サーバーだけに頼る発行から移行します。

~~~sql
-- 概念スケッチ
CREATE OR REPLACE FUNCTION reserve_order(p_slot_id text, p_items jsonb, p_total_price integer)
RETURNS text
LANGUAGE plpgsql
AS $$
DECLARE
  v_code text;
BEGIN
  PERFORM pg_advisory_xact_lock(hashtext(p_slot_id));

  IF (SELECT count(*) FROM orders
       WHERE slot_id = p_slot_id AND status <> 'cancelled') >= 5 THEN
    RETURN NULL;
  END IF;

  INSERT INTO orders (slot_id, items, total_price, status)
  VALUES (p_slot_id, p_items, p_total_price, 'pending')
  RETURNING auth_code INTO v_code;

  RETURN v_code;
END;
$$;
~~~

---

<a id="jp-team"></a>

### チーム

開発: **Team MONO Rail**

<p align="right"><a href="#top">ページ先頭へ戻る ↑</a></p>
