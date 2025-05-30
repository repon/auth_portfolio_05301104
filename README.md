# 認証・認可アーキテクチャ設計ポートフォリオ

本リポジトリは **認証・認可を中心に据えた疎結合 Web アーキテクチャ** の構想・設計ドキュメントです。
実装済みコードや動作デモは別リポジトリ（例: `auth-portfolio/`）で管理しています。

---

## 目的

- 認証方式（OAuth2 / 自前パスワード / Magic Link / APIキーなど）を一元的に理解・設計する
- Next.js (App Router) + Rails API サーバによる分離構成を通して、**クライアント・サーバ構成全体を体系的に再現**
- 再利用可能・拡張可能なアーキテクチャ設計力をポートフォリオとして提示
- `CSR+SSR 混在 / 別ドメイン API` 構成を採用することで、実務でも頻出するセキュリティ対策や状態同期の課題に向き合うことができ、汎用性の高い実装力を身につけられる

---

## 採用アーキテクチャ

| コンポーネント     | 技術 / 役割                                                                             |
| ------------------ | --------------------------------------------------------------------------------------- |
| **バックエンド**   | Rails API / JWT 発行・Cookie 管理 / メール送信 (Magic Link, パスリセット) / PostgreSQL  |
| **フロントエンド** | Next.js (App Router) ‑ CSR+SSR 混在 / `useAuth()` と `middleware.ts` で認証 UI 切り替え |
| **通信**           | Cookie または Bearer トークン / CORS 越境対応                                           |

※ フロントエンドと API サーバは **別ドメイン** で運用します。

---

## 対応予定の認証方式

| 認証方式                     | ステータス | 備考                     |
| ---------------------------- | ---------- | ------------------------ |
| 自前パスワード               | 実装中     | JWT + Cookie 構成        |
| OAuth2(PKCE)                 | 計画       | Google 連携など          |
| Magic Link                   | 計画       | メールのワンタイムリンク |
| SNS ログイン (LINE, GitHub…) | 計画       | OAuth2 の派生            |
| 外部 IdP (SAML / OIDC)       | 設計のみ   | 図解のみ                 |
| API キー / Basic 認証        | 設計のみ   | CLI・外部連携向け        |

---

## クライアント / サーバ構成と認証方式の組み合わせ比較

> 本リポジトリのデフォルト構成 (CSR+SSR 混在 / 別ドメイン API) を実装すると、他の構成で必要となる課題を **ほぼ網羅的** に体験できます。

### CSR+SSR 混在 & 別ドメイン API (本構成)との比較

| #   | 認証方式             | 主な攻撃面                                  | 実装ポイント                                                                                              | 他構成との違い                                                     | 備考                                |
| --- | -------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ----------------------------------- |
| 1   | パスワード           | セッション固定 / CSRF / CSR↔SSR 状態不整合 | JWT を `HttpOnly; Secure; SameSite=None` で Cookie 保存。SSR では `Set-Cookie`、CSR では `/api/me` で同期 | MPA は `SameSite=Lax` でも機能しやすい。SPA は CSRF トークン必須。 | **SSR と CSR 間の状態共有が核心。** |
| 2   | OAuth2(PKCE)         | Code 詐取 / リダイレクト改ざん / 状態不整合 | PKCE でトークン取得 → JWT → Cookie。SSR 用 `Set-Cookie` + CSR 用 `/api/me`                                | BFF や MPA では PKCE 省略可。SPA は必須。                          | **`state` 整合が肝。**              |
| 3   | Magic Link           | URL 再利用 / トークン盗用                   | ワンタイムリンク → JWT → Cookie。CSR 側はクエリ処理と `/api/me` 同期                                      | SPA では URL 露出リスク増。                                        | **遷移ギャップ対策が必要。**        |
| 4   | SNS ログイン         | OAuth2と同                                  | 同上                                                                                                      | 同上                                                               |                                     |
| 5   | 外部 IdP (SAML/OIDC) | Assertion 改ざん / RelayState 固定          | 署名検証と POST/Redirect Binding (設計のみ)                                                               | MPA/BFF はサーバ完結しやすい。SPA は難易度高。                     | **設計力を示せるポイント。**        |
| 6   | API キー / Basic     | キー漏洩 / 不正利用                         | HTTPS + Authorization ヘッダ。キーは環境変数等で注入。                                                    | CLI 用途に適するが SPA は保存が課題。                              | **CORS & HTTPS を前提に設計。**     |

---

## クライアント / サーバ構成一覧

| #   | 構成名                            | 略称   | 説明                                                                     | 実装例                    | 特徴                                |
| --- | --------------------------------- | ------ | ------------------------------------------------------------------------ | ------------------------- | ----------------------------------- |
| 1   | **CSR+SSR 混在 / 別ドメイン API** | Mixed  | 本リポジトリのメイン構成。Cookie/CORS/状態同期の難所を実務レベルで扱う。 | Next.js (App) + Rails API | セキュリティ要件高。                |
| 2   | CSR + BFF (同ドメイン)            | BFF    | SPA が `/api` を同 origin で叩く。CORS & PKCE 不要。                     | Nuxt, Next (Pages)        | 状態管理容易。                      |
| 3   | SSR + BFF                         | SSR    | App Router SSR 中にサーバ側で API 呼び出し。                             | Next.js (App)             | CORS 不要、`state/nonce` 管理必要。 |
| 4   | テンプレートレンダリング (MPA)    | MPA    | Rails/Laravel など伝統的 MPA。                                           | Rails ERB                 | サーバで状態完結。                  |
| 5   | 完全分離 SPA                      | SPA    | フロント完全分離。CORS 必須、PKCE ほぼ必須。                             | Vite + Rails API          | 状態保持が最難関。                  |
| 6   | モバイル + API                    | Mobile | React Native / Flutter。常に Bearer トークン。                           | Expo + Rails API          | PKCE・認証制限が鍵。                |
| 7   | CLI + API                         | CLI    | CLI から直接 API。API キーや Basic 認証が主。                            | Node CLI + fetch          | キー管理 & 監査が重要。             |
| 8   | モノリシック Web アプリ           | Mono   | WordPress / Rails 単体。1 HTML で完結。                                  | Rails only, WordPress     | スケールしづらい。                  |

---

## システム構成図

![システム構成図](./assets/architecture.png)

---

## 進行状況

### 実装中 (Rails + Next.js)

- [ ] 自前パスワードログイン

  - `/auth/register`, `/auth/login`, `/auth/logout`
  - Cookie 属性: `Secure`, `HttpOnly`, `SameSite=None`
  - `/api/me` で認証状態取得

### 今後予定

- [ ] OAuth2(PKCE) ログイン
- [ ] Magic Link ログイン
- [ ] SNS ログイン (GitHub / LINE …)
- [ ] パスワードリセット & メール認証
- [ ] API キー / Basic 認証 (CLI)
- [ ] RBAC, 通知, ファイルアップロード, アクティビティログ

---

## 今後の展望

- 開発案件・チーム開発・アーキテクトとしての活動に向け、継続的にアップデートしていきます。
