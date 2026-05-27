# EmailCampaignDetail — Tài liệu chi tiết

> **Model:** `app/Models/EmailCampaignDetail.php`  
> **Service:** `app/Services/EmailCampaignDetail/EmailCampaignDetailServiceImp.php`  
> **Repository:** `app/Repositories/EmailCampaignDetail/EmailCampaignDetailRepositoryEloquent.php`  
> **Controller:** `app/Http/Controllers/API/EmailActivityController.php`

---

## Mục đích

`EmailCampaignDetail` (hay còn gọi là **activity**) là đơn vị cấu hình cho từng loại email automation trong một campaign. Mỗi shop có 4 activity mặc định tương ứng 4 loại email: Review Request → Review Reminder → Reward → Reward Reminder.

---

## Schema bảng `email_campaign_details`

| Column | Type | Nullable | Mô tả |
|--------|------|----------|-------|
| `id` | bigint unsigned | No | PK |
| `shop_id` | bigint unsigned | Yes | FK → shops (thêm 2025-03-28) |
| `email_campaign_id` | bigint unsigned | No | FK → email_campaigns |
| `email_type_id` | unsignedBigInt | No | FK → email_types (1–4) |
| `type` | string | Yes | Slug ngắn: `review-request`, `review-reminder`, `reward`, `reward-reminder` (thêm 2025-03-28) |
| `name` | string(255) | No | Tên hiển thị: "Review request", "Review reminder", v.v. |
| `email_data` | json | Yes | Metadata: `{ subject, previewText, is_resend, ... }` |
| `timing` | int | No | Số đơn vị thời gian (vd: 1, 3, 7) |
| `timing_unit` | enum | No | `Hours` hoặc `Days` |
| `timing_event` | enum | Yes | `Fulfillment` / `Paid` / `Archived` / `Delivered` (chỉ Review Request dùng) |
| `has_coupon` | boolean | No | default false |
| `reference_email_campaign_details_id` | bigint | Yes | FK self-reference: Reminder/Reward trỏ về Review Request; Reward Reminder trỏ về Reward |
| `status` | boolean | No | default false (on/off) |
| `email_template_id` | bigint | Yes | FK → email_templates (thêm 2022-02-17) |
| `thumbnail` | string | Yes | URL ảnh thumbnail (thêm 2022-02-25) |
| `data` | json | Yes | Cấu trúc block MJML của email (thêm 2024-03-18) |
| `coupon_setting_id` | bigint | Yes | FK → coupon_settings (thêm 2024-03-18) |
| `created_at` | timestamp | Yes | |
| `updated_at` | timestamp | Yes | |

### Indexes (thêm 2024-04 và 2025-03)
- `email_campaign_id` — tìm theo campaign
- `shop_id` — tìm theo shop trực tiếp (phục vụ `NewEmailCampaignService` query theo `shop_id + type`)

---

## 4 loại activity mặc định

| email_type_id | type | name | has_coupon | reference trỏ về |
|---|---|---|---|---|
| 1 | `review-request` | Review request | false | null |
| 2 | `review-reminder` | Review reminder | false | Review request |
| 3 | `reward` | Reward | true | Review request |
| 4 | `reward-reminder` | Reward reminder | false | Reward |

Cấu trúc cây phụ thuộc:
```
Review request (root)
├── Review reminder  (reference_id → Review request)
└── Reward           (reference_id → Review request)
    └── Reward reminder (reference_id → Reward)
```

---

## Khi nào EmailCampaignDetail được tạo

### 1. Khi cài app lần đầu — `AppInstalledJob`

**File:** `app/Jobs/AppInstalledJob.php` (line 112)

```php
$emailCampaignDetailService->createDefaultCampaignDetails($shopId);
```

Đây là trigger chính. Khi shop install app, `AppInstalledListener` dispatch `AppInstalledJob` vào queue `install`. Job này gọi `createDefaultCampaignDetails()` để tạo đủ 4 activities.

**Flow:**
```
AppInstalled event
  → AppInstalledListener::handle()
    → dispatch(AppInstalledJob)->onQueue('install')
      → AppInstalledJob::handle()
        → emailCampaignDetailService->createDefaultCampaignDetails($shopId)
```

---

### 2. Mỗi lần shop đăng nhập — `AppLoggedInListener`

**File:** `app/Listeners/AppLoggedInListener.php` (line 116)

```php
$this->emailCampaignDetailService->createDefaultCampaignDetails($shopId);
```

Mỗi khi shop mở admin panel (event `AppLogined`), listener cũng gọi `createDefaultCampaignDetails()`. **Tuy nhiên hàm này có idempotency check** — chỉ tạo mới nếu chưa có activity nào (kiểm tra `reviewRequest->count() <= 0`). Nếu đã có thì bỏ qua.

> **Mục đích:** Safety net — phòng trường hợp `AppInstalledJob` bị fail khi install, shop vẫn có activities sau lần login đầu.

---

### 3. Qua API thủ công — `EmailActivityController::store`

**File:** `app/Http/Controllers/API/EmailActivityController.php` (line 71)  
**Route:** `POST /api/email-campaigns/{campaignId}/activities`

```php
$activity = $emailCampaignDetailService->createActivity(
    $campaignId,
    $request->get('template_id'),
    $name,
    $request->get('reference_id')
);
```

Dùng để tạo thêm activity mới ngoài 4 mặc định, chủ yếu phục vụ **A/B testing** (nhiều variants của cùng loại email).

---

### 4. Artisan command debug — `php artisan email1`

**File:** `app/Console/Commands/createEmailCampaignDetailService.php`

```php
$emailCampaignDetailService->createDefaultCampaignDetails(1); // shopId hardcode = 1
```

Command này là dev/debug, chỉ chạy cho shop ID = 1.

---

## Logic bên trong `createDefaultCampaignDetails`

```php
public function createDefaultCampaignDetails(int $shopId)
{
    // 1. Tìm hoặc tạo EmailCampaign (automation type)
    $automationCampaign = getAutomationCampaign($shopId) ?? createDefaultCampaign($shopId);

    // 2. Idempotency check: chỉ tạo nếu chưa có Review Request nào
    $reviewRequest = $automationCampaign->campaignDetails->where('email_type_id', 1);
    if ($reviewRequest->count() > 0) return; // ← Đã có rồi, bỏ qua

    // 3. Tạo 4 activities theo thứ tự
    $reviewRequest = createActivity($shopId, $campaignId, 1, 'Review request');
    //   → Tạo ReviewReminder tham chiếu ReviewRequest
    createActivity($shopId, $campaignId, 2, 'Review reminder', $reviewRequest->id);
    //   → Tạo Reward tham chiếu ReviewRequest + tạo CouponSetting
    $reward = createActivity($shopId, $campaignId, 3, 'Reward', $reviewRequest->id);
    //   → Tạo RewardReminder tham chiếu Reward
    createActivity($shopId, $campaignId, 4, 'Reward reminder', $reward->id);
}
```

---

## `createActivity` — chi tiết từng bước

**File:** `EmailCampaignDetailServiceImp.php` (line 79)

```php
public function createActivity(int $shopId, int $campaignId, int $templateId, string $name, ?int $referenceId = null)
```

### Bước 1: Load EmailTemplate từ DB

```php
$emailTemplate = EmailTemplate::query()->find($templateId);
```

Template chứa `options` với default values: `subject`, `timing`, `timing_unit`, `timing_event`, `has_coupon`.

### Bước 2: Load block data từ file JSON

```php
$activity = $this->getDefaultActivityData($name);
// → Đọc app/Templates/activities.json
// → Tìm entry có data.name = $name ("Review request", "Reward", ...)
```

File `activities.json` chứa cấu trúc block MJML đầy đủ của từng loại email.

### Bước 3: Tạo CouponSetting (chỉ với Reward)

```php
if ($name === "Reward") {
    $couponSetting = $couponSettingService->createDefaultCouponSetting($shopId, 'email');
}
```

### Bước 4: Insert vào DB

```php
$this->_repository->createCampaignDetail([
    'shop_id'                              => $shopId,
    'email_campaign_id'                    => $campaignId,
    'email_type_id'                        => $emailTemplate->options['email_type_id'] ?? 5,
    'type'                                 => 'review-request' | 'review-reminder' | 'reward' | 'reward-reminder',
    'name'                                 => $name,
    'email_data'                           => ['subject' => $emailTemplate->options['subject'] ?? ''],
    'timing'                               => $emailTemplate->options['timing'] ?? 1,
    'timing_event'                         => $emailTemplate->options['timing_event'] ?? null,
    'timing_unit'                          => $emailTemplate->options['timing_unit'] ?? 'Days',
    'has_coupon'                           => $emailTemplate->options['has_coupon'] ?? false,
    'reference_email_campaign_details_id'  => $referenceId,
    'email_template_id'                    => $templateId,
    'thumbnail'                            => $emailTemplate->thumbnail ?? null,
    'data'                                 => json_encode($activity),     // block MJML JSON
    'coupon_setting_id'                    => $couponSetting->id ?? null  // chỉ Reward
]);
```

---

## Hai cách query EmailCampaignDetail

Có 2 pattern query trong codebase, tương ứng 2 thời kỳ phát triển:

### Cách cũ — qua `email_campaign_id`
Dùng trong các service cũ (`EmailCampaignDetailService`):
```php
EmailCampaignDetail::where('email_campaign_id', $campaignId)
    ->where('email_type_id', $typeId)
    ->first();
```

### Cách mới — qua `shop_id + type`
Dùng trong `NewEmailCampaignServiceImp` (thêm từ 2025-03-28):
```php
EmailCampaignDetail::where([
    'shop_id' => $shopId,
    'type'    => 'review-request' // hoặc 'review-reminder', 'reward', 'reward-reminder'
])->first();
```

> **Lý do thêm `shop_id` và `type`:** Tối ưu query — tránh phải join qua `email_campaigns` để lấy `shop_id`. Cột `type` là slug thay thế cho `email_type_id` giúp code dễ đọc hơn.

---

## Relationships

```php
// EmailCampaignDetail → lên trên
$detail->emailCampaign       // → EmailCampaign
$detail->emailType            // → EmailType
$detail->emailTemplate        // → EmailTemplate
$detail->couponSetting        // → CouponSetting (via coupon_setting_id)
$detail->reference / ->parent // → EmailCampaignDetail (cha)

// EmailCampaignDetail → xuống dưới
$detail->children             // → tất cả activities có reference trỏ về đây
$detail->reminders            // → children where email_type_id=2, status=true
$detail->rewards              // → children where email_type_id=3, status=true
$detail->sections             // → EmailSection[]
$detail->emails               // → Email[] (tất cả emails đã gửi từ activity này)
$detail->attributes           // → EmailAttribute[] (polymorphic: rate, use-coupon, ...)
```

---

## Helper methods trên Model

```php
$detail->isEmailReward()    // email_type slug == 'reward-email'
$detail->isCouponReminder() // có parent và parent.email_type_id == 3 (Reward)
$detail->isReviewRequest()  // email_type_id == 1
$detail->isReviewReminder() // có parent và email_type_id == 2
$detail->isActive()         // status == true
$detail->pageBlock()        // json_decode($this->data) — block MJML
```

---

## Update activity — `updateCampaignDetails`

**File:** `EmailCampaignDetailServiceImp.php` (line 135)

Chỉ cho phép update các fields:
```
name, email_data, timing_unit, timing, timing_event,
reference_email_campaign_details_id, status, thumbnail, data
```

**Logic đặc biệt khi update:**

1. **Review Request** → `reference_email_campaign_details_id` bị force về `null`
2. **Reward** → `timing` bị force về `0` (send immediately after review)
3. **Khi timing_event = Delivered** → gọi `checkUsedFeature('review_request_upon_delivery')`
4. **Khi timing_event thay đổi** → tự động tạo/cập nhật Shopify webhook tương ứng:
   - `Fulfillment` → subscribe `orders/fulfilled`
   - `Archived` / `Delivered` → subscribe `orders/updated`

---

## Delete activity — `deleteActivity`

**File:** `EmailCampaignDetailServiceImp.php` (line 384)

Cascade delete theo thứ tự:
1. Xóa `CouponSetting` của activity
2. Set `reference_email_campaign_details_id = null` cho tất cả activities con
3. Set `email_campaign_detail_id = null` cho tất cả emails đã gửi
4. Xóa `CouponSetting` của các section `discount`
5. Soft delete activity

---

## Sync template mặc định — `sync:default-template`

**File:** `app/Console/Commands/SyncDefaultTemplate.php`  
**Dùng khi:** Cần reset `data` (block MJML) của các activities về default template.

```bash
php artisan sync:default-template --shopName=myshop
```

Đọc lại `app/Templates/activities.json` và ghi đè `data` cho từng activity của shop.

---

## Vòng đời hoàn chỉnh

```
[Install app]
  → AppInstalledListener (sync)
    → AppInstalledJob (queue: install)
      → createDefaultCampaignDetails()
        → EmailCampaign (tìm hoặc tạo mới)
        → EmailCampaignDetail x4 (Review Request, Reminder, Reward, Reward Reminder)
        → CouponSetting x1 (cho Reward)

[Login lại]
  → AppLoggedInListener
    → createDefaultCampaignDetails() — idempotent, bỏ qua nếu đã có

[Admin cấu hình email]
  → PUT /api/email-campaigns/{id}/activities/{id}
    → updateCampaignDetails() → update DB + tạo webhook nếu cần

[Admin tạo A/B variant]
  → POST /api/email-campaigns/{id}/activities
    → createActivity() → EmailCampaignDetail mới cùng type

[Order webhook / review submitted]
  → ProcessWebhookOrderJob / ReviewSubmittedSideEffectJob
    → Đọc EmailCampaignDetail để lấy timing, template, coupon config
    → Tạo Email record → dispatch AutomationEmailCampaign job
```

---

*Tài liệu được tạo bởi Claude Cowork — 2026-05-27*
