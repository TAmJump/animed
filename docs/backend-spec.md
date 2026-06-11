# Animed — バックエンド実装仕様（D1 / Worker / 権限 / API）

最終更新：2026-06-10　対象：JIN（Cloudflare 側で実装）。Claude はこの契約に沿って Worker / フロントを書き、GitHub へ push する。Cloudflare の D1 適用・Worker デプロイ・シークレット・DNS は JIN 担当。

前提：利用者は無料。収益は事業者（医療機関・薬局・企業・学校・保険・旅行・芸能/スポーツ）から。`audit_logs` は必須。UTC で保存し、表示時に JST 変換。相対月表現は使わず YYYY年M月 を保存。

---

## 1. D1 スキーマ（DDL）

```sql
-- ユーザーと本人確認
CREATE TABLE users(
  id TEXT PRIMARY KEY, email TEXT UNIQUE, phone TEXT,
  side TEXT NOT NULL,           -- 'user' | 'org'
  user_type TEXT,               -- 一般旅行者 / 留学生 ... or 事業者種別
  status TEXT DEFAULT '未登録',
  email_verified INTEGER DEFAULT 0, phone_verified INTEGER DEFAULT 0,
  created_at TEXT DEFAULT (datetime('now')), updated_at TEXT
);
CREATE TABLE profiles(
  user_id TEXT PRIMARY KEY REFERENCES users(id),
  name TEXT, passport_name TEXT, birthdate TEXT, sex TEXT, blood_type TEXT,
  nationality TEXT, language TEXT DEFAULT 'ja',
  emergency_contact TEXT, insurance_status TEXT, religion_note TEXT
);

-- 組織と所属
CREATE TABLE organizations(
  id TEXT PRIMARY KEY, type TEXT, name TEXT,
  billing_type TEXT, contact_person TEXT, contract_status TEXT DEFAULT 'pending',
  commission_rate REAL DEFAULT 0, created_at TEXT DEFAULT (datetime('now'))
);
CREATE TABLE organization_members(
  id TEXT PRIMARY KEY, organization_id TEXT REFERENCES organizations(id),
  user_id TEXT REFERENCES users(id),
  relation TEXT,                -- 学生/社員/タレント/選手/保護者/引率 等
  group_label TEXT,             -- 3年A組 / 営業部 等
  UNIQUE(organization_id,user_id)
);

-- 渡航・健康・薬剤
CREATE TABLE trips(
  id TEXT PRIMARY KEY, user_id TEXT REFERENCES users(id), organization_id TEXT,
  country TEXT, city TEXT, start_date TEXT, end_date TEXT, purpose TEXT,
  insurance_status TEXT, companions TEXT,
  grade TEXT,                   -- 渡航前チェック結果 A/B/C/D
  status TEXT, created_at TEXT DEFAULT (datetime('now'))
);
CREATE TABLE health_records(
  id TEXT PRIMARY KEY, user_id TEXT REFERENCES users(id),
  allergies TEXT, chronic_diseases TEXT, pregnancy_status TEXT,
  hospitalization TEXT, vaccine_notes TEXT, checkup_file_url TEXT, updated_at TEXT
);
CREATE TABLE medications(
  id TEXT PRIMARY KEY, user_id TEXT REFERENCES users(id),
  name_jp TEXT, name_en TEXT, ingredient TEXT, dose TEXT, frequency TEXT,
  quantity TEXT, days TEXT, prescribed_by TEXT, prescribed_at TEXT,
  category TEXT,                -- psycho/adhd/narcotic/injection/chronic/antibiotic/otc/unknown
  import_level TEXT             -- low/mid/high/ng
);

-- ルール DB（管理画面で更新、更新日・確認者を保持）
CREATE TABLE country_rules(
  id TEXT PRIMARY KEY, country TEXT, category TEXT,
  rule_text TEXT, required_documents TEXT, source_url TEXT,
  updated_at TEXT, verified_by TEXT
);
CREATE TABLE drug_rules(
  id TEXT PRIMARY KEY, country TEXT, ingredient TEXT,
  risk_level TEXT, required_action TEXT, note TEXT, updated_at TEXT
);
-- 薬剤変換 DB（日本商品名→一般名→国別）
CREATE TABLE drug_conversion(
  id TEXT PRIMARY KEY, name_jp TEXT, generic TEXT, ingredient TEXT, class TEXT,
  dose TEXT, form TEXT, name_en TEXT, country TEXT, otc_local INTEGER, rx_local INTEGER,
  cautions TEXT, contraindications TEXT, pregnancy_note TEXT, pediatric_note TEXT,
  import_level TEXT, alternatives TEXT, verified_at TEXT, verified_by TEXT
);

-- 相談・診療
CREATE TABLE appointments(
  id TEXT PRIMARY KEY, user_id TEXT, provider_id TEXT, type TEXT,
  scope TEXT,                   -- doctor/pharmacist/team
  interpreter INTEGER DEFAULT 0,
  starts_at TEXT, ends_at TEXT, status TEXT, video_url TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);
CREATE TABLE providers(
  id TEXT PRIMARY KEY, user_id TEXT, provider_type TEXT,  -- doctor/pharmacist/nurse/interpreter
  license_info TEXT, organization_id TEXT, languages TEXT, active INTEGER DEFAULT 1
);
CREATE TABLE consultation_notes(
  id TEXT PRIMARY KEY, appointment_id TEXT, summary TEXT, advice TEXT,
  emergency_flag INTEGER DEFAULT 0, followup_needed INTEGER DEFAULT 0, created_at TEXT
);

-- 日程調整（mやるゼ！流用）
CREATE TABLE schedule_polls(id TEXT PRIMARY KEY, trip_id TEXT, created_by TEXT, status TEXT DEFAULT 'open', confirmed_slot TEXT, call_url TEXT, deadline TEXT, created_at TEXT);
CREATE TABLE schedule_slots(id TEXT PRIMARY KEY, poll_id TEXT, slot_datetime TEXT, label TEXT DEFAULT '');
CREATE TABLE schedule_votes(id TEXT PRIMARY KEY, slot_id TEXT, voter_id TEXT, voter_type TEXT DEFAULT '', answer TEXT, voted_at TEXT, UNIQUE(slot_id,voter_id));

-- 書類
CREATE TABLE documents(
  id TEXT PRIMARY KEY, user_id TEXT, trip_id TEXT, document_type TEXT,
  signer_role TEXT,             -- 医師署名/薬剤師確認/本人入力
  status TEXT,                  -- none/prep/done
  file_url TEXT, signed_by TEXT, issued_at TEXT
);

-- 現地連携
CREATE TABLE partners(
  id TEXT PRIMARY KEY, country TEXT, city TEXT, partner_type TEXT,  -- hospital/pharmacy/transport
  name TEXT, address TEXT, phone TEXT, whatsapp TEXT, email TEXT,
  languages TEXT, hours_24 INTEGER, emergency INTEGER, pediatric INTEGER,
  female_doctor INTEGER, payment TEXT, insurance TEXT, lat REAL, lng REAL,
  contract_status TEXT, referral_ok INTEGER
);
CREATE TABLE referrals(id TEXT PRIMARY KEY, user_id TEXT, trip_id TEXT, partner_id TEXT, reason TEXT, status TEXT, fee_status TEXT, created_at TEXT);

-- 緊急
CREATE TABLE emergency_cases(
  id TEXT PRIMARY KEY, user_id TEXT, trip_id TEXT, lat REAL, lng REAL,
  symptom TEXT, triage TEXT,    -- red/orange/yellow/green
  action_taken TEXT, status TEXT, created_at TEXT DEFAULT (datetime('now'))
);

-- 健康診断（自社事業）
CREATE TABLE checkups(
  id TEXT PRIMARY KEY, user_id TEXT, taken_at TEXT,
  bp TEXT, glucose TEXT, hba1c TEXT, lipids TEXT, liver TEXT, renal TEXT,
  ecg TEXT, chest_xray TEXT, urine TEXT, bmi TEXT, doctor_comment TEXT,
  travel_flag TEXT             -- 渡航前注意/医師相談推奨/薬剤師相談推奨/保険推奨
);

-- 請求・監査・通知
CREATE TABLE billing_records(id TEXT PRIMARY KEY, payer_type TEXT, payer_id TEXT, user_id TEXT, service_type TEXT, gross_amount INTEGER, commission_rate REAL, net_amount INTEGER, status TEXT, created_at TEXT);
CREATE TABLE notifications(id TEXT PRIMARY KEY, user_id TEXT, kind TEXT, title TEXT, body TEXT, send_at TEXT, sent INTEGER DEFAULT 0);
CREATE TABLE audit_logs(id TEXT PRIMARY KEY, actor_id TEXT, action TEXT, target_table TEXT, target_id TEXT, ip TEXT, created_at TEXT DEFAULT (datetime('now')));
```

---

## 2. ロール権限マトリクス（閲覧/操作の最小権限）

| ロール | 本人の病名/薬 | 他者の病名/薬 | ステータスのみ | 書類発行 | 緊急対応 | 課金/監査 |
|---|---|---|---|---|---|---|
| 利用者 | ◯(自分) | × | 自分 | 依頼 | 起票 | × |
| 保護者 | ◯(子) | × | 子 | 依頼 | 起票 | × |
| 企業/学校 管理者 | × | × | ◯(所属) | × | 通知受領 | 自社請求 |
| 医師 | ◯(担当) | ◯(担当) | ◯ | ◯署名 | ◯ | × |
| 薬剤師 | ◯(担当) | ◯(担当) | ◯ | ◯確認 | ◯ | × |
| 看護師/オペレーター | 担当範囲 | 担当範囲 | ◯ | × | ◯ | × |
| 通訳 | 最小限 | 最小限 | × | × | 同席のみ | × |
| 現地医療機関/薬局 | 共有分のみ | × | × | 受領 | ◯ | × |
| TAmJ 管理者 | 設定のみ | × | ◯ | × | ◯ | ◯ |
| 監査管理者 | × | × | × | × | × | ◯(ログ) |

原則：企業・学校には**詳細病名を出さない**（ステータスのみ）。エピペン/吸入薬/アレルギー等の緊急情報は、許可担当者のみ閲覧。全アクセスを `audit_logs` に記録。

---

## 3. API コントラクト（Worker /api/*）

認証：Bearer（セッションJWT）。レスポンスは `{ok, data, error}`。全変更系で `audit_logs` 追記。

```
POST /api/auth/register        {email, phone, side, user_type} -> {user_id}
POST /api/auth/verify          {channel:'email'|'phone', code}
POST /api/auth/login           {email|phone, otp} -> {token}

GET/PUT /api/profile           本人プロフィール（医療パスポート元データ）
GET/PUT /api/health            health_records（自分のみ）
GET/POST/DELETE /api/medications

POST /api/precheck             {trip, health} -> {grade:'A|B|C|D', items[], docNeeds[], country_note}
                               ※サーバ側でも country_rules/drug_rules を参照して再判定（フロント判定は目安）
POST /api/drug-check           {meds[], country} -> [{name, category, import_level, action, docs[]}]

GET  /api/trips  POST /api/trips  PATCH /api/trips/:id
GET  /api/documents  PATCH /api/documents/:id   (status/file_url/signer)

POST /api/schedule/polls       候補枠提示
POST /api/schedule/votes       医師/薬剤師/通訳/利用者の○△×
POST /api/schedule/confirm     {poll_id, slot} -> {call_url}  全員○で確定
GET  /api/appointments  POST /api/appointments

POST /api/emergency            {symptom, lat, lng} -> {triage, emg_number, partners[]}
GET  /api/partners?country=&city=&type=&near=lat,lng  近い順

-- 事業者側
GET  /api/org/:id/members      ステータスのみ（病名なし）
GET  /api/org/:id/billing
-- 健康診断
POST /api/checkups  GET /api/checkups
```

決済：Square（mやるゼ！系と同方式）。利用者課金なし。事業者課金・手数料は `billing_records` に集約。医療広告/紹介規制・各国法規制は契約形態を弁護士確認。

---

## 4. ステータス遷移（users.status / trips.status）

`未登録 → 基本情報入力済 → 渡航情報入力済 → 薬剤確認中 → 医師確認中 → 薬剤師確認中 → 書類作成中 → 出発準備完了 → 渡航中 → 相談中 → 緊急対応中 → 帰国後フォロー → 完了`

フロント（app.html v2）は localStorage 試作で同じ語彙を使用済み。Worker 接続時はこのenumに合わせる。

---

## 5. 通知スケジュール（notifications）

出発30日前 / 14日前 / 7日前 / 前日 / 渡航中 / 帰国後。内容：薬剤確認・医師/薬剤師相談・書類未提出・英文診断書完了・保険確認・現地病院情報・緊急連絡先確認・帰国後フォロー。Cron Trigger で `send_at <= now & sent=0` を送信（Resend 等）。

---

## 6. 実装順（doc準拠）

1. MVP1：登録/本人確認 → 渡航前チェック(API再判定) → 医療パスポート → 薬剤登録 → 相談予約/通話 → 書類 → TAmJ管理。
2. MVP2：薬剤変換DB・各事業者管理画面（薬局/医療機関/企業/学校）。
3. MVP3：現地病院/薬局DB・緊急・GPS・通訳連携。
4. MVP4：保険/旅行会社連携・芸能/スポーツ案件・多言語・国別規制DB。

フロントは app.html v2 が MVP1 の利用者側UIを先取り済み。次は Worker をこの契約で実装（Cloudflare 操作は JIN）。

---

## 7. 認証と機微文書の保管（AWS 採用・JIN確認事項）

利用者の問いに対する設計判断：**認証と機微PIIはAWS、アプリ取引データはCloudflare**のハイブリッド。

- **認証**：AWS Cognito（User Pool）。app.html は試作でローカルSHA-256ハッシュだが、本番は Cognito のJWTに置換。`users.id` に Cognito sub を保存。
- **パスポート画像・本人確認書類**：**S3 + KMS暗号化（SSE-KMS）**。アップロードは Cognito Identity Pool 経由の **presigned URL**（Workerやクライアントから直接S3、画像は端末でリサイズ後送信）。DBには `s3_key` のみ保存し、画像本体は持たない。
- **MRZ/OCR**：本番は端末内 or サーバ側で OCR（Textract も可）。app.html は MRZ(TD3) パーサを実装済み（手入力・将来のOCR両対応）。

追加テーブル / フィールド：
```sql
ALTER TABLE profiles ADD COLUMN passport_no TEXT;       -- 暗号化して保存推奨
ALTER TABLE profiles ADD COLUMN passport_expiry TEXT;
ALTER TABLE profiles ADD COLUMN passport_country TEXT;
CREATE TABLE secure_documents(
  id TEXT PRIMARY KEY, user_id TEXT, kind TEXT,  -- passport/id/insurance
  s3_key TEXT, mime TEXT, uploaded_at TEXT, kms_key_id TEXT
);
```
追加API：
```
POST /api/secure-docs/presign   {kind, mime} -> {url, s3_key}   (S3 presigned PUT)
POST /api/secure-docs/commit    {kind, s3_key, mime}            (メタをDB登録)
GET  /api/secure-docs/:id/url    -> presigned GET（短時間）
```
注意：パスポート番号など識別子はアプリ内表示用に最小限に留め、保存時はアプリ層暗号化を推奨。閲覧は本人と、明示同意のある医療者のみ（`audit_logs` 必須）。

## 8. v2 で先取り実装済みのUX（フロント試作）
ログイン/登録 → 利用者種別 → 自動入力（渡航前チェック・医療パスポート）／パスポート・カメラ読み取り＋MRZ自動入力／**出発カウントダウン＋準備チェックリスト**／**オフライン緊急医療カード（QR＋現地語/英語フレーズ＋現地救急番号）**。これらはローカル試作。Cognito/S3/D1 接続で実データ化（Cloudflare/AWS操作は JIN）。

---

## 9. 追加ツールのデータ設計（v3）

**同意年齢の方針：自己決定は18歳以上**（民法改正で2022/04から成年=18歳。飲酒・喫煙等は20歳のまま）。18歳未満は保護者が共有を管理。**緊急時は同意設定に関わらず、緊急連絡先へ緊急医療カード＋現在地を共有可（命を守る例外）。**

```sql
CREATE TABLE share_consents(
  id TEXT PRIMARY KEY, user_id TEXT, enabled INTEGER DEFAULT 0,
  share_passport INTEGER, share_trip INTEGER, share_card INTEGER, share_location INTEGER,
  set_by TEXT,            -- self(18+) / guardian
  updated_at TEXT
);
CREATE TABLE family_links(
  id TEXT PRIMARY KEY, user_id TEXT, member_name TEXT, relation TEXT, contact TEXT,
  is_guardian INTEGER DEFAULT 0, is_emergency INTEGER DEFAULT 1
);
CREATE TABLE med_schedules(id TEXT PRIMARY KEY, user_id TEXT, medication_id TEXT, slot TEXT, home_time TEXT);
CREATE TABLE insurances(id TEXT PRIMARY KEY, user_id TEXT, company TEXT, policy_no TEXT, coverage TEXT, assist_phone TEXT);
CREATE TABLE wearable_links(id TEXT PRIMARY KEY, user_id TEXT, provider TEXT, share_with_doctor INTEGER, last_sync TEXT);
CREATE TABLE vitals(id TEXT PRIMARY KEY, user_id TEXT, taken_at TEXT, hr INTEGER, spo2 INTEGER, steps INTEGER, sleep_h REAL);
-- checkups は §1 に定義済み
```
- 共有は **本人(18+)のみ自己設定可**。サーバ側でも `getAge(birthdate)>=18` を検証し、未成年は guardian の `family_links.is_guardian=1` 経由のみ。緊急時オーバーライドは `emergency_cases` 起票時に `is_emergency=1` の連絡先へ通知（監査ログ必須）。
- 服薬スケジュールの時差変換はフロント計算（国→UTCオフセット）。用量変更は提案せず、医師・薬剤師相談へ誘導。
- ウェアラブルは本番で HealthKit / Google Fit / Fitbit API。`vitals` に正規化保存、相談時のみ医師へ提示（`wearable_links.share_with_doctor`）。
- 保険のアシスタンス番号は緊急医療カードに自動表示（フロント実装済み）。
