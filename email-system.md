# Hệ thống Email - LAI Product Reviews

> **Cập nhật lần cuối:** 2026-05-26 | Version 7.0
> 
> Tài liệu này mô tả toàn bộ hệ thống email của ứng dụng LAI Product Reviews, bao gồm cấu trúc database, luồng xử lý, setup ban đầu và kiến trúc kỹ thuật.

---

## Mục lục

1. [Tổng quan hệ thống](#1-tổng-quan-hệ-thống)
2. [Database - Các bảng liên quan Email](#2-database---các-bảng-liên-quan-email)
3. [Models và quan hệ](#3-models-và-quan-hệ)
4. [Dữ liệu Setup ban đầu (Seed)](#4-dữ-liệu-setup-ban-đầu-seed)
5. [Luồng xử lý Webhook Order → Email](#5-luồng-xử-lý-webhook-order--email)
6. [Kiến trúc kỹ thuật chi tiết](#6-kiến-trúc-kỹ-thuật-chi-tiết)
7. [Tracking & Unsubscribe](#7-tracking--unsubscribe)
8. [Custom Email Sender](#8-custom-email-sender)
9. [A/B Testing](#9-ab-testing)
10. [API Endpoints](#10-api-endpoints)
11. [Các file quan trọng](#11-các-file-quan-trọng)

---

## 1. Tổng quan hệ thống

Hệ thống email của LAI Product Reviews là một **automation email pipeline** giúp shop chủ tự động gửi email cho khách hàng sau khi đặt hàng, nhằm thu thập đánh giá sản phẩm. 

### Các loại email chính

| Loại | Mục đích | Trigger |
|------|----------|---------|
| **Review Request** | Yêu cầu khách hàng viết review | Sau khi order fulfilled/paid/archived |
| **Review Reminder** | Nhắc nhở nếu khách chưa review | N ngày sau Review Request |
| **Reward** | Tặng coupon sau khi khách đã review | Sau khi khách submit review |
| **Reward Reminder** | Nhắc sử dụng coupon | N ngày sau Reward |
| **Rejection Review** | Thông báo review bị từ chối | Khi admin reject review |
| **Custom/Promotion** | Email marketing thủ công | Manual dispatch |

### Công nghệ sử dụng

- **Mail driver:** AWS SES (SMTP) - cấu hình qua `config/mail.php`
- **Custom sender:** SendGrid SMTP với DKIM verification cho domain riêng
- **Queue:** Laravel Queue (`emails` queue cho gửi email, `webhook` queue cho xử lý webhook)
- **Template format:** MJML blocks dạng JSON (new editor) + HTML sections (legacy)
- **Tracking:** Pixel tracking (open), redirect URL (click)

---

## 2. Database - Các bảng liên quan Email

### 2.1 Bảng `email_types` — Danh mục loại email

```sql
CREATE TABLE email_types (
    id        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name      VARCHAR(255),
    slug      VARCHAR(255),
    status    TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**Dữ liệu seed mặc định:**

| id | name | slug | status |
|----|------|------|--------|
| 1 | Request Email | request-email | 1 |
| 2 | Reminder Email | reminder-email | 1 |
| 3 | Reward Email | reward-email | 1 |
| 4 | Custom Email | custom-email | 1 |
| 5 | Promotion Email | promotion-email | 1 |
| 6 | Rejection Email | rejection-email | 1 |

> Constants trong `EmailType` model: `REJECTION_REVIEW = 6`

---

### 2.2 Bảng `email_campaigns` — Campaign container

```sql
CREATE TABLE email_campaigns (
    id                    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shop_id               BIGINT UNSIGNED NOT NULL,
    name                  VARCHAR(255) NOT NULL,
    description           TEXT NULL,
    status                TINYINT(1) DEFAULT 0,
    type                  VARCHAR(50) NULL,          -- 'automation' | 'manual'
    ab_test_campaign_id   BIGINT UNSIGNED NULL,       -- FK ab_test_campaigns
    created_at            TIMESTAMP,
    updated_at            TIMESTAMP,
    FOREIGN KEY (shop_id) REFERENCES shops(id),
    FOREIGN KEY (ab_test_campaign_id) REFERENCES ab_test_campaigns(id)
);
```

**Mô tả:**
- Mỗi shop có **đúng 1 automation campaign** (type = 'automation')
- Là container chứa nhiều `EmailCampaignDetail` (activities)
- `status = false` = campaign đang tắt, không gửi email

---

### 2.3 Bảng `email_campaign_details` — Activity trong campaign

Đây là bảng **trung tâm** của hệ thống, mỗi record đại diện cho 1 activity (Review Request, Reminder, Reward...).

```sql
CREATE TABLE email_campaign_details (
    id                                    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shop_id                               BIGINT UNSIGNED NULL,        -- thêm 2025
    email_campaign_id                     BIGINT UNSIGNED NOT NULL,
    email_type_id                         BIGINT UNSIGNED NOT NULL,
    type                                  VARCHAR(100) NULL,           -- thêm 2025
    -- 'review-request' | 'review-reminder' | 'reward' | 'reward-reminder'
    name                                  VARCHAR(255),
    email_data                            JSON NULL,                   -- legacy template data
    timing_unit                           ENUM('Hours','Days'),
    timing                                INT,
    timing_event                          ENUM('Fulfillment','Paid','Archived','Delivered') NULL,
    has_coupon                            TINYINT(1) DEFAULT 0,
    reference_email_campaign_details_id   BIGINT UNSIGNED NULL,        -- self-FK (parent)
    status                                TINYINT(1) DEFAULT 0,
    email_template_id                     BIGINT UNSIGNED NULL,        -- thêm 2022
    thumbnail                             VARCHAR(255) NULL,           -- thêm 2022
    data                                  JSON NULL,                   -- thêm 2024 (MJML blocks)
    coupon_setting_id                     BIGINT UNSIGNED NULL,        -- thêm 2024
    created_at                            TIMESTAMP,
    updated_at                            TIMESTAMP,
    FOREIGN KEY (email_campaign_id)  REFERENCES email_campaigns(id),
    FOREIGN KEY (email_type_id)      REFERENCES email_types(id),
    FOREIGN KEY (reference_email_campaign_details_id) REFERENCES email_campaign_details(id)
);
```

**Quan hệ self-referential (parent-child):**
```
Review Request (id: 1, ref: null)
  ├── Review Reminder (id: 2, ref: 1)
  └── Reward         (id: 3, ref: 1)
        └── Reward Reminder (id: 4, ref: 3)
```

**Scopes quan trọng trong model:**
- `emailRequest` → `email_type_id = 1` (Request Email)
- `emailReminder` → `email_type_id = 2` (Reminder Email)
- `emailReward` → `email_type_id = 3` (Reward Email)
- `emailRequestCoupon` → `has_coupon = true` và type Request

---

### 2.4 Bảng `emails` — Từng email được tạo/gửi

Đây là bảng **tracking** toàn bộ email cụ thể cho từng khách hàng.

```sql
CREATE TABLE emails (
    id                        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shop_id                   BIGINT UNSIGNED NOT NULL,
    order_id                  BIGINT UNSIGNED NOT NULL,
    email_campaign_detail_id  BIGINT UNSIGNED NOT NULL,
    customer_id               BIGINT UNSIGNED NOT NULL,
    email_data                JSON,                                    -- rendered email data tại thời điểm gửi
    status                    ENUM(
                                'pending','sent','opened','clicked',
                                'rated','failed','cancel','reach-out','skipped'
                              ) DEFAULT 'pending',
    email_trigger_id          BIGINT UNSIGNED NULL,                    -- parent email (self-FK)
    scheduled_at              TIMESTAMP NULL,
    category                  ENUM('automatic','manual') DEFAULT 'automatic',
    ab_test_campaign_id       BIGINT UNSIGNED NULL,
    created_at                TIMESTAMP,
    updated_at                TIMESTAMP,
    FOREIGN KEY (shop_id)                  REFERENCES shops(id),
    FOREIGN KEY (order_id)                 REFERENCES orders(id),
    FOREIGN KEY (email_campaign_detail_id) REFERENCES email_campaign_details(id),
    FOREIGN KEY (customer_id)             REFERENCES customers(id),
    FOREIGN KEY (email_trigger_id)        REFERENCES emails(id)
);
```

**Vòng đời status:**
```
pending → sent → opened → clicked → rated
                                  ↘ cancel (nếu đã review trước khi gửi)
                                  ↘ skipped (excluded email/product)
                                  ↘ reach-out (shop tắt campaign)
                                  ↘ failed (lỗi gửi)
```

---

### 2.5 Bảng `email_attributes` — Event tracking (append-only)

```sql
CREATE TABLE email_attributes (
    id               BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    attributeable_id   BIGINT UNSIGNED NOT NULL,
    attributeable_type VARCHAR(255) NOT NULL,   -- polymorphic: Email | EmailCampaignDetail
    is_done            TINYINT(1) DEFAULT 0,
    attribute_name     VARCHAR(255) NULL,
    -- 'pending' | 'sent' | 'rate' | 'opened' | 'reach-out' | 'cancel' | 'clicked' | 'coupon' | 'skipped'
    created_at         TIMESTAMP,
    updated_at         TIMESTAMP
);
```

**Mục đích:** Ghi lại mọi sự kiện theo kiểu append-only (không update, chỉ insert). Dùng để:
- Detect "đã review" → cancel reminder
- Detect "đã mở email" → update status
- Detect "đã dùng coupon" → cancel reward reminder

---

### 2.6 Bảng `email_templates` — Template library

```sql
CREATE TABLE email_templates (
    id                     BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shop_id                BIGINT UNSIGNED NULL,           -- NULL = global template
    email_reference_type_id BIGINT UNSIGNED NOT NULL,     -- FK email_types
    name                   VARCHAR(255),
    description            TEXT NULL,
    thumbnail              VARCHAR(255) NULL,
    default_sections       JSON NULL,                      -- legacy sections
    options                JSON NULL,
    required_sections      JSON NULL,
    exclude_sections       JSON NULL,
    created_at             TIMESTAMP,
    updated_at             TIMESTAMP
);
```

**Constants trong model:**
```php
const REVIEW_REQUEST   = 1;
const REVIEW_REMINDER  = 2;
const REWARD           = 3;
const REWARD_REMINDER  = 4;
const REJECTION_REVIEW = 5;
```

---

### 2.7 Bảng `email_sections` — Block cấu trúc template

```sql
CREATE TABLE email_sections (
    id                        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    email_campaign_detail_id  BIGINT UNSIGNED NOT NULL,   -- cascade delete
    coupon_setting_id         BIGINT UNSIGNED NULL,        -- cascade delete
    index                     INT,                         -- thứ tự hiển thị
    type                      VARCHAR(100),
    -- 'header' | 'footer' | 'text' | 'button' | 'image'
    -- 'divider' | 'discount' | 'productReview'
    contents                  JSON,
    style                     JSON,
    created_at                TIMESTAMP,
    updated_at                TIMESTAMP
);
```

> **Global scope:** `orderBy('index')` — sections luôn được sắp xếp theo index

---

### 2.8 Bảng `email_settings` — Cài đặt per-shop

```sql
CREATE TABLE email_settings (
    id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shop_id             BIGINT UNSIGNED NOT NULL,
    exclude_emails      JSON NULL,       -- danh sách email bị loại trừ
    exclude_product_ids JSON NULL,       -- danh sách product_id bị loại trừ
    created_at          TIMESTAMP,
    updated_at          TIMESTAMP
);
```

> **Lưu ý:** Bảng này đã được rebuild hoàn toàn vào 2023. Phiên bản cũ (2019) có nhiều fields hơn như `timing`, `timing_unit`, `template`, `email_subject`, v.v. — tất cả đã bị xoá và chuyển sang `email_campaign_details`.

---

### 2.9 Bảng `custom_email_senders` — Sender domain tuỳ chỉnh

```sql
CREATE TABLE custom_email_senders (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shop_id     BIGINT UNSIGNED NOT NULL,   -- cascade delete
    from_email  VARCHAR(255) NOT NULL,
    from_name   VARCHAR(255) NULL,
    account     VARCHAR(255) NOT NULL,      -- SendGrid account
    verified    TINYINT(1) DEFAULT 0,
    domain_id   BIGINT NULL,
    domain_dns  JSON NULL,                  -- DNS records để verify DKIM
    country     VARCHAR(100) NULL,
    city        VARCHAR(100) NULL,
    created_at  TIMESTAMP,
    updated_at  TIMESTAMP
);
```

---

### 2.10 Bảng `ab_test_campaigns` — A/B Testing

```sql
CREATE TABLE ab_test_campaigns (
    id                                    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shop_id                               BIGINT UNSIGNED NOT NULL,
    status                                ENUM('active','stop','draft','completed') DEFAULT 'draft',
    stopping_rule                         ENUM('duration','quantity','winner') DEFAULT 'winner',
    start_date                            TIMESTAMP NULL,
    end_date                              TIMESTAMP NULL,
    quantity_to_stop                      INT NULL,
    winner_activity_percentage_to_stop    INT DEFAULT 30,
    active_activities                     JSON NULL,      -- array of EmailCampaignDetail IDs
    created_at                            TIMESTAMP,
    updated_at                            TIMESTAMP
);
```

---

### 2.11 Bảng `coupon_settings` — Cài đặt coupon cho Reward email

```sql
CREATE TABLE coupon_settings (
    id                        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shop_id                   BIGINT UNSIGNED NOT NULL,
    email_campaign_detail_id  BIGINT UNSIGNED NOT NULL,
    discount_value            FLOAT NOT NULL,
    discount_type             ENUM('fixed_amount','percentage') NOT NULL,
    expires_after             SMALLINT,                    -- số ngày hết hạn
    shopify_price_rule_id     INT UNSIGNED,
    in_use                    TINYINT(1),
    code                      VARCHAR(255),
    review_setting_id         BIGINT UNSIGNED NULL,
    code_type                 VARCHAR(100) NULL,
    type                      VARCHAR(100) NULL,
    created_at                TIMESTAMP,
    updated_at                TIMESTAMP
);
```

---

### 2.12 Bảng `customer_coupons` — Coupon đã tạo cho từng khách

```sql
CREATE TABLE customer_coupons (
    id                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shop_id           BIGINT UNSIGNED NOT NULL,
    coupon_setting_id BIGINT UNSIGNED NOT NULL,
    coupon            VARCHAR(255) NOT NULL,    -- unique coupon code
    expired_at        TIMESTAMP NULL,
    deleted_at        TIMESTAMP NULL,           -- soft delete
    created_at        TIMESTAMP,
    updated_at        TIMESTAMP
);
```

---

### 2.13 Bảng `orders` — Đơn hàng từ Shopify

```sql
CREATE TABLE orders (
    id               BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    shop_id          BIGINT UNSIGNED NOT NULL,
    order_id         BIGINT UNSIGNED NOT NULL,  -- Shopify order ID
    email            VARCHAR(255) NULL,
    customer         TEXT NULL,                 -- JSON customer data
    number           INT,
    country_code     VARCHAR(10),
    country_name     VARCHAR(100),
    line_items       TEXT,                      -- JSON line items
    token            VARCHAR(255),
    state            ENUM('imported','created','fulfilled','paided','closed'),
    created          TIMESTAMP NULL,
    updated          TIMESTAMP NULL,
    closed           TIMESTAMP NULL,
    mail_action      ENUM('waiting','sent','open','rated') NULL,  -- legacy
    note_attributes  TEXT NULL,
    is_test          TINYINT(1) DEFAULT 0,
    created_at       TIMESTAMP,
    updated_at       TIMESTAMP
);
```

---

### 2.14 Bảng `customers` — Khách hàng

```sql
CREATE TABLE customers (
    id             BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name     VARCHAR(255) NULL,
    last_name      VARCHAR(255) NULL,
    email          VARCHAR(255) NOT NULL UNIQUE,
    verified_email TINYINT(1) DEFAULT 0,
    created_at     TIMESTAMP,
    updated_at     TIMESTAMP
);
```

---

## 3. Models và quan hệ

### 3.1 Sơ đồ quan hệ

```
Shop (1) ──────────────────────────── (n) EmailCampaign
                                              │
                                              │ hasMany
                                              ▼
                                    EmailCampaignDetail (n) ──────── (1) EmailType
                                              │                              
                    ┌─────────────────────────┤ self-referential            
                    │ children                │ (parent-child)              
                    ▼                         │                             
         EmailCampaignDetail                  │                             
         (Reminder/Reward)                    │                             
                                              │ hasMany                     
                                              ▼                             
Shop (1) ── Order (1) ── Customer (1) ──── Email (n)
                                              │
                              ┌───────────────┴──────────────┐
                              │ morphMany                     │
                              ▼                               ▼
                      EmailAttribute                    CustomerCoupon
```

### 3.2 Chi tiết Model Email

**`App\Models\Email`**
```php
// Relationships
belongsTo(EmailCampaignDetail::class)
belongsTo(Shop::class)
belongsTo(Customer::class)
belongsTo(Order::class)
hasMany(Review::class)           // reviews được submit từ email này
morphToMany(CustomerCoupon::class) // coupon đính kèm
belongsTo(Email::class, 'email_trigger_id')  // parent email
hasMany(Email::class, 'email_trigger_id')    // child emails (reminders/rewards)

// Scopes
scopePending()     // status = 'pending'
scopeSent()        // status = 'sent'
scopeOpened()      // status = 'opened'
scopeRated()       // status = 'rated'
```

**`App\Models\EmailCampaignDetail`**
```php
// Relationships
belongsTo(EmailCampaign::class)
belongsTo(EmailType::class)
belongsTo(EmailCampaignDetail::class, 'reference_email_campaign_details_id')  // parent
hasMany(EmailCampaignDetail::class, 'reference_email_campaign_details_id')   // children
hasMany(Email::class)
hasOne(CouponSetting::class)
belongsTo(EmailTemplate::class)
morphMany(EmailAttribute::class)

// Scopes
scopeEmailRequest()        // email_type_id = 1
scopeEmailReminder()       // email_type_id = 2
scopeEmailReward()         // email_type_id = 3
scopeEmailRequestCoupon()  // has_coupon = true
scopeActive()              // status = true
```

---

## 4. Dữ liệu Setup ban đầu (Seed)

### 4.1 EmailTypeSeeder

File: `database/seeds/EmailTypeSeeder.php`

Tạo 4 (sau này 6) rows trong bảng `email_types`:

```php
$types = [
    ['id' => 1, 'name' => 'Request Email',   'slug' => 'request-email'],
    ['id' => 2, 'name' => 'Reminder Email',  'slug' => 'reminder-email'],
    ['id' => 3, 'name' => 'Reward Email',    'slug' => 'reward-email'],
    ['id' => 4, 'name' => 'Custom Email',    'slug' => 'custom-email'],
    // Thêm sau:
    ['id' => 5, 'name' => 'Promotion Email', 'slug' => 'promotion-email'],
    ['id' => 6, 'name' => 'Rejection Email', 'slug' => 'rejection-email'],
];
```

### 4.2 AutomationTemplateSeeder

File: `database/seeds/AutomationTemplateSeeder.php`  
Data source: `app/Templates/automation.json`

Tạo 5 global `EmailTemplate` records:

| id | name | email_type_id | Mô tả |
|----|------|---------------|-------|
| 1 | Review request | 1 | Subject: "Please let us know what do you think" |
| 2 | Review reminder | 2 | Subject: "Your product still await a review" |
| 3 | Reward | 3 | Subject: "You have a gift from (Shop name)" |
| 4 | Reward reminder | 2 | Subject: "Don't forget your (coupon value) discount code!" |
| 5 | Rejection review | 6 | Subject: "Your review was unfortunately rejected!" |

Cũng tạo `EmailCategory` với slug `automation` và link templates vào category này.

### 4.3 Setup khi Shop install (createDefaultCampaignDetails)

Được gọi trong `AppInstalledJob` hoặc `ChangePlanJob` khi shop cài đặt app.

**Bước 1:** Tạo `EmailCampaign`:
```php
EmailCampaign::create([
    'shop_id'     => $shop->id,
    'name'        => 'Automation',
    'type'        => 'automation',
    'status'      => false,   // tắt mặc định, shop phải tự bật
]);
```

**Bước 2:** Tạo 4 `EmailCampaignDetail`:

```
1. Review Request
   - email_type_id: 1
   - timing: 1, timing_unit: 'Days'
   - timing_event: 'Fulfillment'
   - type: 'review-request'
   - status: false

2. Review Reminder  (reference: Review Request)
   - email_type_id: 2
   - timing: 3, timing_unit: 'Days'
   - type: 'review-reminder'
   - status: false

3. Reward  (reference: Review Request)
   - email_type_id: 3
   - timing: 1, timing_unit: 'Days'
   - has_coupon: true
   - type: 'reward'
   - status: false

4. Reward Reminder  (reference: Reward)
   - email_type_id: 2
   - timing: 3, timing_unit: 'Days'
   - type: 'reward-reminder'
   - status: false
```

**Bước 3:** Tạo default `CouponSetting` cho Reward:
```php
CouponSetting::create([
    'email_campaign_detail_id' => $rewardActivity->id,
    'discount_value'  => 10,
    'discount_type'   => 'percentage',
    'expires_after'   => null,    // không hết hạn
]);
```

**Bước 4:** Copy default sections từ global template vào từng activity.

---

## 5. Luồng xử lý Webhook Order → Email

### 5.1 Tổng quan flow

```
Shopify Order Event
        │
        ▼
  [POST /webhook/orders/{event}]
  OrderWebHookController
        │
        ▼ dispatch (queue: webhook)
  ProcessWebhookOrderJob  (hoặc PreHandleWebhookOrderUpdatedJob)
        │
        ├─ saveOrder()
        ├─ saveCustomer()
        ├─ Klaviyo/Omnisend integration
        ├─ filterAutomationEmailToSend()
        ├─ createEmailToSend()  → status: pending, scheduled_at = now + timing
        │
        ▼ dispatch (queue: emails, delay: scheduled_at)
  AutomationEmailCampaign Job
        │
        ├─ emailInterceptionHandler() — kiểm tra điều kiện gửi
        ├─ shouldStopSending() — nếu đã review thì cancel
        ├─ createCoupon() — tạo coupon nếu là reward email
        ├─ emailService->send() — gửi email thực sự
        │
        ▼ fire event
  AutomationEmailSend Event
        │
        ▼ listen
  AutomationEmailSent Listener
        │
        ▼ schedule child emails
  Review Reminder / Reward Reminder
```

### 5.2 Webhook Routes

File: `routes/webhook.php` (hoặc `routes/web.php`)  
Middleware: `VerifyShopifyWebhook` (xác thực HMAC signature)

```
POST /webhook/orders/fulfilled    → OrderWebHookController@fulfilled
POST /webhook/orders/paid         → OrderWebHookController@paid
POST /webhook/orders/archived     → OrderWebHookController@archived
POST /webhook/v2/orders/fulfilled → OrderWebHookController@fulfilledv2
POST /webhook/v2/orders/paid      → OrderWebHookController@paidv2
POST /webhook/v2/orders/archived  → OrderWebHookController@archivedv2
```

### 5.3 OrderWebHookController

File: `app/Http/Controllers/OrderWebHookController.php`

```php
// fulfilled / fulfilledv2
public function fulfilled(Request $request, $shopName)
{
    ProcessWebhookOrderJob::dispatch($shopName, $payload, Order::FULFILLMENT)
        ->onQueue('webhook');
    return response()->json(['success' => true]);
}

// paid / paidv2
public function paid(Request $request, $shopName)
{
    ProcessWebhookOrderJob::dispatch($shopName, $payload, Order::PAID)
        ->onQueue('webhook');
}

// archived (phức tạp hơn — cần kiểm tra Delivered vs Archived)
public function archived(Request $request, $shopName)
{
    PreHandleWebhookOrderUpdatedJob::dispatch($shopName, $payload)
        ->onQueue('webhook');
}
```

### 5.4 PreHandleWebhookOrderUpdatedJob

File: `app/Jobs/PreHandleWebhookOrderUpdatedJob.php`

Phân tích payload để xác định loại event:

```php
// Nếu timing_event của shop là 'Archived' và order có closed_at
if ($timingEvent === 'Archived' && $payload['closed_at']) {
    ProcessWebhookOrderJob::dispatch($shopName, $payload, Order::ARCHIVED);
}

// Nếu shipment_status là 'delivered'
if ($payload['fulfillments'][0]['shipment_status'] === 'delivered') {
    ProcessWebhookOrderJob::dispatch($shopName, $payload, Order::DELIVERED);
}
```

### 5.5 ProcessWebhookOrderJob (Core Logic)

File: `app/Jobs/ProcessWebhookOrderJob.php`

```
1. Lấy shop từ shopName
2. Kiểm tra feature 'email_review_request' có enabled không
   └─ Nếu không → return (bỏ qua)
3. Nếu event = DELIVERED, kiểm tra feature 'review_request_upon_delivery'
4. Trích xuất customer email từ order payload
5. Kiểm tra order đã tồn tại chưa
   └─ Nếu có → kiểm tra isContinueSending() (tránh duplicate)
6. orderService->saveOrder() → lưu/cập nhật bảng `orders`
7. customerService->saveCustomer() → upsert bảng `customers`
8. [Integrations]
   ├─ Klaviyo: nếu enabled → gửi event ORDER_PAID hoặc ORDER_FULFILLED
   └─ Omnisend: nếu enabled và có phone → gửi trigger event
9. Lấy active Request Email activities phù hợp với orderEvent:
   └─ emailCampaignDetailService->getActiveRequestEmailCampaignDetails($shop, $event)
10. filterAutomationEmailToSend():
    ├─ Nếu A/B test active → chọn ngẫu nhiên 1 activity
    └─ Nếu không → lấy activity đầu tiên
11. Kiểm tra email đã được queue chưa (tránh duplicate)
12. emailService->createEmailToSend():
    └─ INSERT INTO emails (status='pending', scheduled_at = now() + timing)
13. sendEmailService->automationSend():
    └─ AutomationEmailCampaign::dispatch($email)->delay($scheduledAt)
         onQueue('emails')
```

### 5.6 AutomationEmailCampaign Job

File: `app/Jobs/AutomationEmailCampaign.php`  
Queue: `emails`

```php
public function handle()
{
    // 1. Load Email với đầy đủ relations
    $email = Email::with(['emailCampaignDetail', 'customer', 'order', 'shop'])->find($this->emailId);

    // 2. Kiểm tra điều kiện gửi (emailInterceptionHandler)
    $checks = [
        'email_status_valid'      => email->status không phải cancel/skipped,
        'shop_installed'          => shop chưa uninstall,
        'email_volume_quota'      => còn quota email tháng này,
        'reward_feature_enabled'  => nếu là reward email, shop phải có feature email_coupon,
    ];
    if (any check fails) → cancel email, return;

    // 3. shouldStopSending() — dành cho Reminder và Reward
    if ($detail->isReminder() || $detail->isReward()) {
        $parentEmail = $email->parentEmail;
        $parentAttributes = $parentEmail->attributes;
        if ($parentAttributes->contains('attribute_name', 'rate')) {
            // Khách đã review rồi → cancel reminder
            $email->update(['status' => 'cancel']);
            return;
        }
        if ($detail->isRewardReminder()) {
            // Kiểm tra thêm coupon đã được dùng chưa
            if ($parentAttributes->contains('attribute_name', 'coupon')) {
                $email->update(['status' => 'cancel']);
                return;
            }
        }
    }

    // 4. createCoupon() — chỉ cho Reward email
    if ($detail->has_coupon) {
        $coupon = couponService->createCustomerCoupon($shop, $couponSetting, $customer);
        // Tạo Shopify Price Rule → Discount Code
        // Lưu vào customer_coupons
    }

    // 5. Gửi email thực sự
    $mailable = new AutomationCampaign($email, $coupon);
    Mail::to($customer->email)->send($mailable);
    
    // 6. Cập nhật status → 'sent'
    $email->update(['status' => 'sent']);

    // 7. Gắn coupon vào email record (nếu có)
    
    // 8. Fire event để schedule children
    event(new AutomationEmailSend($email));
}
```

### 5.7 AutomationEmailSent Listener

File: `app/Listeners/AutomationEmailSent.php`

```php
public function handle(AutomationEmailSend $event)
{
    $sentEmail = $event->email;
    $detail    = $sentEmail->emailCampaignDetail;
    
    // Lấy tất cả child activities (reminder/reward)
    $children = $detail->children()->active()->get();
    
    foreach ($children as $childDetail) {
        // Tạo Email record mới cho child
        $childEmail = emailService->createEmailToSend(
            shop: $shop,
            order: $order,
            customer: $customer,
            detail: $childDetail,
            parentEmailId: $sentEmail->id,  // link về parent
            scheduledAt: now()->add($childDetail->timing, $childDetail->timing_unit)
        );
        
        // Schedule AutomationEmailCampaign job với delay
        AutomationEmailCampaign::dispatch($childEmail)->delay($childEmail->scheduled_at);
    }
}
```

### 5.8 EmailService - Xây dựng và gửi email

File: `app/Services/Email/EmailService.php`

```php
public function send(Email $email, ?CustomerCoupon $coupon = null): void
{
    $detail   = $email->emailCampaignDetail;
    $customer = $email->customer;
    $shop     = $email->shop;

    // Build mailable
    $mailable = new AutomationCampaign($email);
    
    // Set subject (với variable substitution)
    $subject = $this->resolveSubject($detail, $shop, $customer);
    // Variables: {shop_name}, {customer_name}, {coupon_value}, etc.
    
    // Xác định sender
    $customSender = $shop->customEmailSender;
    if ($customSender && $customSender->verified) {
        // Dùng SendGrid SMTP với from_email của shop
        $mailable->from($customSender->from_email, $customSender->from_name);
    } else {
        // Dùng AWS SES với MAIL_FROM_ADDRESS
        $mailable->from(config('mail.from.address'), $shop->name);
        $mailable->replyTo($shop->customer_email);
    }
    
    // Build HTML từ MJML blocks JSON
    $html = (new AutomationCampaignAdapter($detail, $shop, $customer, $coupon))->render();
    $mailable->html($html);
    
    // Gửi
    Mail::to($customer->email)->send($mailable);
}
```

---

## 6. Kiến trúc kỹ thuật chi tiết

### 6.1 Queue Configuration

```
Queue: webhook   → Xử lý webhook events từ Shopify (ProcessWebhookOrderJob)
Queue: emails    → Gửi email thực sự (AutomationEmailCampaign)
Queue: default   → Các job khác
```

Cấu hình trong `config/horizon.php` (Laravel Horizon):
```php
'environments' => [
    'production' => [
        'emails-supervisor' => [
            'connection' => 'redis',
            'queue'      => ['emails'],
            'processes'  => 5,
            'tries'      => 3,
        ],
        'webhook-supervisor' => [
            'connection' => 'redis', 
            'queue'      => ['webhook'],
            'processes'  => 3,
        ],
    ],
],
```

### 6.2 Template System (New Editor — 2024+)

Từ 2024, template email được lưu dưới dạng MJML blocks JSON trong `email_campaign_details.data`:

```json
{
  "pageBlock": {
    "type": "page",
    "data": {
      "value": {
        "subject": "Please let us know what do you think",
        "previewText": "...",
        "backgroundColor": "#ffffff"
      }
    },
    "children": [
      {
        "type": "section",
        "children": [
          {
            "type": "text",
            "data": { "value": { "content": "Hi {customer_name}..." } }
          },
          {
            "type": "button",
            "data": { "value": { "content": "Write a review", "href": "{review_url}" } }
          }
        ]
      }
    ]
  }
}
```

Variables có thể dùng trong template: `{shop_name}`, `{customer_name}`, `{order_number}`, `{product_name}`, `{coupon_code}`, `{coupon_value}`, `{review_url}`, `{unsubscribe_url}`

### 6.3 New API (2025 Refactor)

Từ 2025, có 2 API song song:

**API cũ** (legacy — dùng campaign ID):
```
GET  /email-campaigns/{campaignId}/activities
PUT  /email-campaigns/{campaignId}/activities/{detailId}
```

**API mới** (dùng `type` field + `shop_id`):
```
GET  /email/getReviewRequestSetting     → lấy activity type='review-request'
GET  /email/getReviewReminderSetting    → lấy activity type='review-reminder'  
GET  /email/getRewardSetting            → lấy activity type='reward'
GET  /email/getRewardReminderSetting    → lấy activity type='reward-reminder'
PUT  /email/updateReviewRequestSetting  → update theo type
```

API mới query thẳng theo `shop_id + type` thay vì đi qua campaign ID, giúp đơn giản hoá frontend.

---

## 7. Tracking & Unsubscribe

### 7.1 Track Open (Pixel)

```
GET /email/track-open?email_attribute={emailId}&store={shopName}
→ Client\EmailController@trackOpenEmail
```

Logic:
1. Tìm Email record theo `emailId`
2. Insert EmailAttribute: `attribute_name = 'opened'`
3. Update Email: `status = 'opened'`
4. Trả về 1x1 transparent pixel GIF

### 7.2 Track Click (Redirect)

```
GET /email/track-click?email_attribute={emailId}&store={shopName}&redirect_url={url}
→ Client\EmailController@trackClickEmail
```

Logic:
1. Insert EmailAttribute: `attribute_name = 'clicked'`
2. Update Email: `status = 'clicked'`
3. Redirect đến `redirect_url`

### 7.3 Unsubscribe

```
GET  /email/unsubscribe?store={shopName}&email={customerEmail}
POST /email/unsubscribe
```

Logic:
1. Thêm `customerEmail` vào `email_settings.exclude_emails` JSON array
2. Cancel tất cả pending emails của customer đó

---

## 8. Custom Email Sender

### Quy trình verify domain

1. Shop nhập `from_email` (vd: `hello@myshop.com`) trong settings
2. Hệ thống tạo record trong `custom_email_senders` với `verified = false`
3. Tạo domain trên SendGrid API → lấy về `domain_id` + `domain_dns` (các DNS records cần add)
4. Shop thêm DNS records vào domain của họ (DKIM, SPF)
5. Shop click "Verify" → hệ thống gọi SendGrid API kiểm tra
6. Nếu pass → `verified = true`, email từ đó sẽ dùng `from_email` custom thay vì AWS SES

---

## 9. A/B Testing

### Cách hoạt động

1. Shop tạo `AbTestCampaign` với `stopping_rule` (duration/quantity/winner)
2. Chọn các `EmailCampaignDetail` ID vào `active_activities`
3. Khi `ProcessWebhookOrderJob` chạy:
   - Nếu AB test `status = 'active'` → chọn ngẫu nhiên 1 activity từ `active_activities`
   - Mỗi email được ghi nhận `ab_test_campaign_id`
4. Hệ thống theo dõi open rate / click rate / review rate của từng activity
5. Dừng theo rule:
   - `duration`: dừng khi đến `end_date`
   - `quantity`: dừng khi tổng email đạt `quantity_to_stop`
   - `winner`: dừng khi 1 variant dẫn trước `winner_activity_percentage_to_stop`%

---

## 10. API Endpoints

### Email Campaign

```
GET    /email-campaigns                          → Lấy danh sách campaigns
POST   /email-campaigns                          → Tạo campaign mới
GET    /email-campaigns/{id}                     → Chi tiết campaign
PUT    /email-campaigns/{id}                     → Cập nhật campaign
DELETE /email-campaigns/{id}                     → Xoá campaign
PUT    /email-campaigns/{id}/status              → Bật/tắt campaign

GET    /email-campaigns/{id}/activities          → Lấy activities
POST   /email-campaigns/{id}/activities          → Thêm activity
PUT    /email-campaigns/{id}/activities/{detailId} → Cập nhật activity
DELETE /email-campaigns/{id}/activities/{detailId} → Xoá activity
```

### New Email API (2025)

```
GET    /email/getReviewRequestSetting
GET    /email/getReviewReminderSetting
GET    /email/getRewardSetting
GET    /email/getRewardReminderSetting
PUT    /email/updateReviewRequestSetting
PUT    /email/updateReviewReminderSetting
PUT    /email/updateRewardSetting
PUT    /email/updateRewardReminderSetting
```

### Email List & Stats

```
GET    /emails                                   → Danh sách emails đã gửi (paginated)
GET    /emails/{id}                              → Chi tiết email
GET    /email/stats                              → Thống kê tổng quan
GET    /email/stats/timeline                     → Stats theo thời gian
```

### Email Settings

```
GET    /email-settings                           → Lấy settings của shop
PUT    /email-settings                           → Cập nhật settings
PUT    /email-settings/exclude-emails            → Thêm/xoá excluded emails
PUT    /email-settings/exclude-products          → Thêm/xoá excluded products
```

### Templates

```
GET    /email-templates                          → Danh sách global templates
GET    /email-templates/{id}                     → Chi tiết template
POST   /email-templates/preview                  → Preview HTML của template
```

### Custom Sender

```
GET    /custom-email-sender                      → Lấy custom sender info
POST   /custom-email-sender                      → Tạo custom sender
PUT    /custom-email-sender/{id}                 → Cập nhật
DELETE /custom-email-sender/{id}                 → Xoá
POST   /custom-email-sender/{id}/verify          → Verify domain DNS
```

---

## 11. Các file quan trọng

### Controllers
| File | Mô tả |
|------|-------|
| `app/Http/Controllers/OrderWebHookController.php` | Entry point webhook từ Shopify |
| `app/Http/Controllers/API/EmailCampaignController.php` | CRUD campaigns |
| `app/Http/Controllers/API/EmailActivityController.php` | CRUD activities |
| `app/Http/Controllers/API/EmailController.php` | Email list & stats (new API) |
| `app/Http/Controllers/Client/EmailController.php` | Track open/click, unsubscribe |

### Jobs
| File | Mô tả |
|------|-------|
| `app/Jobs/ProcessWebhookOrderJob.php` | Core xử lý webhook → tạo email |
| `app/Jobs/PreHandleWebhookOrderUpdatedJob.php` | Pre-process archived webhook |
| `app/Jobs/AutomationEmailCampaign.php` | Gửi email thực sự |
| `app/Jobs/SendEmailTest.php` | Gửi test email |

### Services
| File | Mô tả |
|------|-------|
| `app/Services/Email/EmailService.php` | Build & send email |
| `app/Services/Job/SendEmailServiceImp.php` | Dispatch job |
| `app/Services/EmailCampaignDetail/EmailCampaignDetailServiceImp.php` | Business logic cho activities |
| `app/Services/EmailCampaign/NewEmailCampaignServiceImp.php` | New API service (2025) |

### Events & Listeners
| File | Mô tả |
|------|-------|
| `app/Events/AutomationEmailSend.php` | Event fired sau khi gửi email |
| `app/Listeners/AutomationEmailSent.php` | Schedule child emails |
| `app/Providers/EventServiceProvider.php` | Event-Listener mapping |

### Mail
| File | Mô tả |
|------|-------|
| `app/Mail/AutomationCampaign.php` | Mailable class |

### Models
| File | Mô tả |
|------|-------|
| `app/Models/Email.php` | Email tracking record |
| `app/Models/EmailCampaign.php` | Campaign container |
| `app/Models/EmailCampaignDetail.php` | Campaign activity |
| `app/Models/EmailType.php` | Lookup table |
| `app/Models/EmailTemplate.php` | Global templates |
| `app/Models/EmailSection.php` | Template blocks |
| `app/Models/EmailAttribute.php` | Event tracking |
| `app/Models/EmailSetting.php` | Per-shop settings |
| `app/Models/CustomEmailSender.php` | Custom sender domain |
| `app/Models/AbTestCampaign.php` | A/B test config |
| `app/Models/CouponSetting.php` | Coupon config |
| `app/Models/CustomerCoupon.php` | Per-customer coupons |

### Templates & Seeds
| File | Mô tả |
|------|-------|
| `app/Templates/automation.json` | Default template data (5 templates) |
| `app/Templates/activities.json` | MJML block data cho new editor |
| `database/seeds/EmailTypeSeeder.php` | Seed email types |
| `database/seeds/AutomationTemplateSeeder.php` | Seed global templates |

### Config
| File | Mô tả |
|------|-------|
| `config/mail.php` | AWS SES SMTP driver config |

---

---

## 12. Code thực tế — Implementation Notes

### 12.1 ProcessWebhookOrderJob — Các điểm quan trọng từ code

**Dependency injection (Constructor):**
```php
public function __construct(array $orderAttr, string $shopName, string $orderEvent)
{
    $this->orderAttr  = $orderAttr;   // raw Shopify webhook payload
    $this->shopName   = $shopName;    // myshop.myshopify.com
    $this->orderEvent = $orderEvent;  // 'Paid' | 'Fulfillment' | 'Archived' | 'Delivered'
}
```

**Feature gates thực tế:**
```php
$ability = $shop->subscription()->ability();

// Gate 1: Email feature có bật không?
if (!$ability->enabled('email_review_request')) return;

// Gate 2: DELIVERED event chỉ hoạt động nếu có feature riêng
if ($orderEvent === Order::DELIVERED && !$ability->enabled('review_request_upon_delivery')) return;
```

**Tránh gửi duplicate:**
```php
$alreadyQueued = Email::query()->where([
    'shop_id'                  => $shop->id,
    'order_id'                 => $savedOrder->id,
    'email_campaign_detail_id' => $reviewRequestEmail->id,
    'customer_id'              => $savedCustomer->id,
])->whereNotIn('status', [Email::CANCEL, Email::SKIPPED, Email::REACH_OUT])->exists();

if ($alreadyQueued) return;
```
> Lưu ý: Chỉ skip nếu email đang ở trạng thái hợp lệ. Nếu email cũ đã bị cancel/skipped/reach-out → cho phép gửi lại.

**isContinueSending logic:**
```php
private function isContinueSending(Order $order, $orderEvent): bool
{
    if ($orderEvent === Order::ARCHIVED && $order->isClosed())   return false;
    if ($orderEvent === Order::DELIVERED && $order->isDelivered()) return false;
    return true;
}
```
> Order đã được xử lý event đó rồi thì không xử lý nữa.

**Read replica cho Orders query:**
```php
$order = Order::on('mysql_readonly')->select()->where([...])->first();
```
> Hệ thống dùng **MySQL read replica** (`mysql_readonly`) cho query kiểm tra order, giảm tải master DB.

---

### 12.2 AutomationEmailCampaign Job — Code thực tế

**Khởi tạo tài nguyên:**
```php
private function initializeResources(Email $email): void
{
    $this->emailAttributeService = app(EmailAttributeService::class);
    $this->emailService          = app(EmailService::class);
    $this->shop                  = $email->shop;
    $this->activity              = $email->emailCampaignDetail;
    $this->currentEmail          = $email;
}
```

**shouldStopSending — Logic cancel reminder:**
```php
private function shouldStopSending(): bool
{
    $parent        = $this->currentEmail->parent ?? null;
    $attributeName = $parent 
        ? $this->emailService->mappingAttributeName($this->currentEmail) 
        : null;

    // mappingAttributeName trả về:
    // - 'rate'       nếu là Review Reminder (parent = Review Request)
    // - 'use-coupon' nếu là Reward Reminder (parent = Reward)

    if ($attributeName && $this->emailAttributeService->hasDone($attributeName, $parent)) {
        // Parent đã có attribute done (đã review hoặc đã dùng coupon)
        $this->cancelEmail(Email::CANCEL);
        return true;
    }
    return false;
}
```

**emailInterceptionHandler — 4 gates kiểm tra:**
```php
// Gate 1: Email không bị cancel/skipped/reach-out
if (in_array($email->status, [Email::SKIPPED, Email::CANCEL, Email::REACH_OUT])) return false;

// Gate 2: Shop chưa uninstall
if ($this->shop->uninstalled()) {
    $this->cancelEmail(Email::CANCEL);
    return false;
}

// Gate 3: Còn quota email tháng này
if (!$ability->canUse('email_volume')) {
    $this->cancelEmail(Email::REACH_OUT);
    return false;
}

// Gate 4: Reward email cần feature 'email_coupon'
if ($this->activity->isEmailReward() && !$ability->enabled('email_coupon')) {
    $this->cancelEmail(Email::CANCEL);
    return false;
}
```

**Difference giữa CANCEL vs REACH_OUT:**
- `CANCEL`: Email bị huỷ vĩnh viễn (shop uninstall, coupon feature disabled, đã review rồi)
- `REACH_OUT`: Hết quota email tháng — có thể gửi lại khi quota được reset

**canSendReminder — Điều kiện schedule children:**
```php
private function canSendReminder(Email $email): bool
{
    // Phải có children activities
    if ($this->activity->children()->count() <= 0) return false;
    
    // Email cha không bị cancel
    if (in_array($status, [Email::CANCEL, Email::REACH_OUT, Email::SKIPPED])) return false;
    
    // Còn quota
    if (!$ability->canUse('email_volume')) return false;
    
    // Coupon Reminder và Review Reminder KHÔNG tự tạo thêm children
    // (tránh infinite loop)
    if ($this->activity->isCouponReminder() || $this->activity->isReviewReminder()) return false;
    
    return true;
}
```

**Resend support:**
Job hỗ trợ resend qua field `email_data.is_resend`:
```php
private function resolveIsResend(Email $email): bool
{
    $emailData = $email->email_data ?? [];
    if (array_key_exists('is_resend', $emailData)) {
        return (bool) $emailData['is_resend'];
    }
    return false;
}
```
Khi `is_resend = true`, job **bỏ qua** `shouldStopSending()` — cho phép gửi lại dù đã review.

---

### 12.3 EmailServiceImp — send() thực tế

```php
public function send(Shop $shop, EmailCampaignDetail $activity, Customer $customer, 
                     Email $email, Collection $products = null, int $orderId = null,
                     CustomerCoupon $coupon = null, bool $isTest = false)
{
    // 1. Ghi nhận email_volume usage (tính vào quota)
    $shop->subscriptionUsage()->record('email_volume');

    // 2. Build HTML qua AutomationCampaignAdapter
    $adapter = new AutomationCampaignAdapter(
        $activity,
        $activity->pageBlock(),   // JSON decoded từ $activity->data
        $shop,
        $email->id,
        $customer,
        $order,
        $activity->couponSetting ?? null,
        $coupon
    );
    $htmlString = $adapter->generateEmailContent();

    // 3. Nếu không build được HTML → cancel email
    if (!$htmlString) {
        $email->update(["status" => Email::CANCEL]);
        return false;
    }

    // 4. Build subject (với variable substitution)
    $subject = generateEmailText(
        $customer,
        $activity->pageBlock()->data->value->subject ?? '',
        $shop
    );

    // 5. Gửi qua Laravel Mail
    Mail::to($customer->email)->send(
        new AutomationCampaign($shop, $activity, $email->id, $customer, 
                               $products, $coupon, $isTest, [], $htmlString)
    );

    // 6. Update status + EmailAttribute
    $email = $this->markAndRemovePendingStatus($email, Email::SENT, EmailAttribute::SENT);

    // 7. Lưu snapshot HTML + subject vào email_data (để preview sau này)
    $email->update([
        "email_data" => [
            "subject" => $subject,
            "content" => $adapter->generateEmailContent(true),  // true = preview mode
            "version" => "3.2.0"
        ],
    ]);

    return $email;
}
```

**markAndRemovePendingStatus:**
```php
public function markAndRemovePendingStatus(Email $email, string $status, string $attributeName)
{
    $email->update(['status' => $status]);
    
    // Xoá attribute 'pending' cũ
    $email->attributes()->where("attribute_name", EmailAttribute::PENDING)->delete();
    
    // Tạo/update attribute mới
    $email->attributes()->updateOrCreate(
        ['attribute_name' => $attributeName],
        ['is_done' => true]
    );
    return $email;
}
```

---

### 12.4 AutomationCampaign Mailable — Custom Sender Logic

```php
public function send(MailerContract $mailer)
{
    $customEmailSender = $this->shop->customEmailSender;

    if ($this->shop->canUseCustomSender() && $customEmailSender->account != 'tiennm') {
        // Dùng SendGrid SMTP
        $transport = new Swift_SmtpTransport('smtp.sendgrid.net', 587, 'tls');
        $transport->setUsername('apikey');
        $transport->setPassword(config("secom.sengrid.{$customEmailSender->account}"));
        
        // Swap mailer tạm thời → gửi → restore
        $backupMailer = Mail::getSwiftMailer();
        Mail::setSwiftMailer(new Swift_Mailer($transport));
        parent::send($mailer);
        Mail::setSwiftMailer($backupMailer);
        return;
    }

    // Default: AWS SES
    parent::send($mailer);
}
```

> **Lưu ý đặc biệt:** Account `tiennm` được hardcode để **bypass** custom sender — luôn dùng AWS SES. Đây là exception riêng cho một account cụ thể.

**setFromAndReplyTo logic:**
```php
if ($shop->canUseCustomSender()) {
    // Custom sender: from = custom email, reply-to = custom email
    $this->from($customerEmailSender->from_email, $customerEmailSender->from_name);
    $this->replyTo($email, $customerEmailSender->from_name);
} else {
    // Default: from = MAIL_FROM_ADDRESS (LAI app email)
    //          reply-to = shop's customer_email
    $shopEmail = $shopInfo->customer_email ?? config('mail.from.address');
    $this->from(config('mail.from.address'), $shopInfo->name);
    $this->replyTo($shopEmail, $shopInfo->name);
}
```

---

### 12.5 EmailCampaignDetail Model — Helper Methods

```php
// Kiểm tra loại activity
public function isEmailReward(): bool     { return $this->emailType->slug === 'reward-email'; }
public function isReviewRequest(): bool   { return $this->email_type_id === 1; }
public function isReviewReminder(): bool  { return $this->reference && $this->email_type_id === 2; }

// Reward Reminder = email_type_id = 2 NHƯNG reference là Reward (type_id = 3)
public function isCouponReminder(): bool
{
    $reference = $this->reference;
    return $reference && $reference->email_type_id === 3;
}

// Lấy reminders (chỉ active, email_type_id = 2)
public function reminders(): HasMany
{
    return $this->hasMany(EmailCampaignDetail::class, 'reference_email_campaign_details_id')
        ->where(['status' => true, 'email_type_id' => 2]);
}

// Lấy rewards (chỉ active, email_type_id = 3)
public function rewards(): HasMany
{
    return $this->hasMany(EmailCampaignDetail::class, 'reference_email_campaign_details_id')
        ->where(['status' => true, 'email_type_id' => 3]);
}

// pageBlock() — deserialize MJML blocks JSON từ field 'data'
public function pageBlock()
{
    return json_decode($this->data);
}
```

**Lưu ý về `reminders()` vs `rewards()`:**
- `reminders()` filter `email_type_id = 2` → chỉ lấy Reminder (kể cả Reward Reminder vì cùng type_id = 2)
- `rewards()` filter `email_type_id = 3` → chỉ lấy Reward  
- `canSendReminder()` trong Job gọi `$activity->children()` — lấy **tất cả** children (cả Reminder lẫn Reward), sau đó Listener sẽ gọi `$activity->reminders` (chỉ Reminder type) để schedule

---

### 12.6 Email Model — Constants và Helper Methods

```php
// Status constants
const PENDING   = 'pending';
const SENT      = 'sent';
const RATED     = 'rated';
const OPENED    = 'opened';
const CLICKED   = 'clicked';
const CANCEL    = 'cancel';
const REACH_OUT = 'reach-out';
const SKIPPED   = 'skipped';
// const FAILED = 'failed';  ← Đã comment out, không dùng nữa

// Category
const CATEGORY = [
    'AUTOMATIC' => 'automatic',
    'MANUAL'    => 'manual'
];

// Helper
public function isReviewRequestReminder(): bool {
    return $this->emailCampaignDetail->email_template_id === EmailTemplate::REVIEW_REMINDER;
}
public function isRewardReminder(): bool {
    return $this->emailCampaignDetail->email_template_id === EmailTemplate::REWARD_REMINDER;
}
public function isRewardEmail(): bool {
    return $this->emailCampaignDetail->email_template_id === EmailTemplate::REWARD;
}
public function isSent(): bool {
    // Kiểm tra qua attributes (not field status) — chính xác hơn
    return !!$this->attributes()->where("email_attributes.attribute_name", Email::SENT)->first();
}
```

---

### 12.7 createEmailToSend — Business Logic chi tiết

```php
public function createEmailToSend(int $shopId, int $customerId, EmailCampaignDetail $activity,
    string $emailAddress, int $orderId = null, int $emailTriggerId = null,
    Carbon $scheduledAt = null, string $category = 'automatic', int $testCampaignId = null)
{
    $shop    = Shop::find($shopId);
    $ability = $shop->subscription()->ability();
    $canUse  = $ability->canUse('email_volume');

    // Validation
    if (!$this->isAvailableForCreateEmail($emailAddress, $activity->email_template_id, $category, $activity, $ability)) {
        return false;
    }

    // Kiểm tra email/product bị exclude
    $shouldBlock = $emailSettingService->hasBlocked($shopId, $emailAddress, $orderId);

    // Xác định status ban đầu
    $emailStatus = $canUse 
        ? ($shouldBlock ? Email::SKIPPED : Email::PENDING)
        : Email::REACH_OUT;

    // Xác định scheduled_at
    $delayTime = $canUse 
        ? ($scheduledAt ?? $this->sendEmailService->getDelayTime($activity->timing, $activity->timing_unit))
        : null;  // REACH_OUT không cần schedule

    // Tạo Email record
    $email = Email::create([
        'shop_id'                  => $shopId,
        'order_id'                 => $orderId,
        'email_campaign_detail_id' => $activity->id,
        'customer_id'              => $customerId,
        'email_data'               => ['is_resend' => false, 'coupon_id' => null],
        'ab_test_campaign_id'      => $testCampaignId,
        'status'                   => $emailStatus,
        'scheduled_at'             => $delayTime,
        'email_trigger_id'         => $emailTriggerId,
        'category'                 => $category
    ]);

    // Tạo EmailAttribute ban đầu (track trạng thái pending/skipped/reach-out)
    $email->attributes()->create([
        'attribute_name' => $emailStatus,
        'is_done'        => false
    ]);

    // Nếu SKIPPED → return false (không dispatch job)
    if ($email->status == Email::SKIPPED) return false;
    
    return $email;
}
```

**isAvailableForCreateEmail — 4 validation rules:**
```php
// Rule 1: Phải có email address
if (!$emailAddress) return false;

// Rule 2: Automatic emails chỉ gửi khi activity đang active
if (!$activity->isActive() && $category == 'automatic') return false;

// Rule 3: Review request cần feature email_review_request
if ($emailTemplate === EmailTemplate::REVIEW_REQUEST && !$ability->enabled('email_review_request')) return false;

// Rule 4: Reward/Reward Reminder cần feature email_coupon
if (in_array($emailTemplate, [EmailTemplate::REWARD, EmailTemplate::REWARD_REMINDER]) 
    && !$ability->enabled('email_coupon')) return false;
```

---

### 12.8 AutomationEmailSent Listener — Reminder Scheduling

```php
public function handle(AutomationEmailSend $event)
{
    $activity      = $event->activity();
    $shop          = $event->shop();
    $reminderEmails = $activity->reminders;  // chỉ lấy active reminder activities (type = 2)
    
    // Chọn 1 reminder (có thể là A/B test)
    $activityChildren = $this->emailCampaignDetailService->filterAutomationEmailToSend(
        $shop, $reminderEmails
    );
    
    $emailAddress = $event->order()->email;
    if (!$activityChildren || !$emailAddress) return;

    // Tạo Email record cho reminder
    $email = $this->emailService->createEmailToSend(
        $shop->id,
        $event->customer()->id,
        $activityChildren,
        $emailAddress,
        $event->order()->id ?? null,
        $event->emailTriggerId()  // link về parent email
    );

    if ($email) {
        // Kế thừa coupon từ parent (Reward Reminder dùng lại coupon của Reward)
        $coupon = $event->coupon();
        if ($coupon) {
            $email->update([
                'email_data' => array_merge($email->email_data ?? [], [
                    'coupon_id' => $coupon->id,
                ]),
            ]);
        }
        
        // Dispatch job với delay
        $this->sendEmailService->automationSend($shop, $event->customer()->id, 
            $activityChildren->id, $email, $event->order()->id ?? null);
    }
}
```

> **Quan trọng:** Listener chỉ schedule **Review Reminder** (type = 2). **Reward email** được schedule riêng qua một listener khác hoặc sau khi khách submit review.

---

### 12.9 Ghi chú về sendTest và giới hạn

```php
// Limit test email: 15 emails/ngày/shop
public function checkLimitSentTestInDay($shop): bool
{
    $limit = 15;
    $keySetting = 'totalSentEmailTest';
    // Lưu format: "2026-05-26_3" (date_count)
    $totalSentTest = shopSetting($shop->id, $keySetting, null);
    // ...
}
```

**Send test với dummy customer:**
```php
$customer = new Customer([
    'id'         => 1,
    'first_name' => 'John',
    'last_name'  => 'Doe',
    'email'      => 'example@gmail.com'
]);
// Không tạo thật trong DB, chỉ dùng để render template
```

---

### 12.10 Customer.io Integration trong Email

Khi `markAsRate()` được gọi (khách submit review từ email):
```php
$count = $email->shop->emails()->whereHas('attributes', function ($builder) {
    $builder->where('attribute_name', 'rate');
})->count();

// Gửi event lên Customer.io tại milestone 1st và 10th review
if ($count == 1 || $count == 10) {
    $customerAttributeService->updateCustomer([
        'lai_email_review_count' => $count
    ]);
}
```

---

### 12.11 sendManually — Gửi email thủ công

```php
public function sendManually(int $shopId, Collection $orders, Carbon $scheduledAt)
{
    $shop     = Shop::find($shopId);
    $activity = $shop->requestReviewEmailCampaignDetails()->first();  // Review Request activity

    foreach ($orders as $order) {
        // Upsert customer nếu chưa có
        $customer = $this->customerService->findByEmail($order->email);
        if (!$customer) {
            $customer = Customer::create([
                'first_name'     => explode(" ", $order->customer)[0],
                'last_name'      => explode(" ", $order->customer)[1],
                'email'          => $order->email,
                'verified_email' => 1
            ]);
        }
        
        $email = $this->createEmailToSend(
            $shop->id, $customer->id, $activity, $order->email,
            $order->id, null, $scheduledAt,
            Email::CATEGORY['MANUAL']  // category = 'manual'
        );
        
        if (!$email) continue;
        
        $this->sendEmailService->automationSend($shop, $customer->id, 
            $activity->id, $email, $order->id);
    }
}
```

> Manual emails (`category = 'manual'`) bypass rule `activity->isActive()` — có thể gửi dù activity đang tắt.

---

---

## 13. SendEmailServiceImp — Dispatch và Delay Logic

### 13.1 getDelayTime — Tính thời gian delay

```php
public function getDelayTime(int $timing, string $timingUnit, Carbon $schedule = null)
{
    if ($schedule) return $schedule;         // manual override
    if (!$timing)  return now();             // timing = 0 → gửi ngay lập tức

    switch ($timingUnit) {
        case 'Hours': return now()->addHours($timing);
        case 'Days':  return now()->addDays($timing);
    }
}
```

**Timing cho từng activity mặc định:**

| Activity | timing | timing_unit | → scheduled_at |
|----------|--------|-------------|----------------|
| Review Request | 1 | Days | now + 1 ngày |
| Review Reminder | 3 | Days | (từ lúc gửi Request) + 3 ngày |
| Reward | 0 | Days | ngay lập tức sau khi review |
| Reward Reminder | 3 | Days | (từ lúc gửi Reward) + 3 ngày |

### 13.2 automationSend — Dispatch với các guards

```php
public function automationSend(Shop $shop, int $customerId, int $campaignDetailId, 
                                Email $email, int $orderId = null)
{
    $ability = $shop->subscription()->ability();
    $status  = $email->status;

    // Guard 1: Hết quota → log vào Elasticsearch + gửi event Customer.io
    if (!$ability->canUse('email_volume')) {
        ElasticLoggerUtil::logging($shopName, 'Email', 'Reach out quota');
        $this->sendCustomerIo($shop);  // gửi event 'lai_email_reach_limit' lên Customer.io
        return;
    }

    // Guard 2: Không có scheduled_at
    if (!$email->scheduled_at) return;

    // Guard 3: Email đã bị cancel/skipped/reach-out
    if (in_array($status, [Email::REACH_OUT, Email::SKIPPED, Email::CANCEL])) return;

    // Dispatch với delay
    AutomationEmailCampaign::dispatch($email->id)
        ->onQueue('emails')
        ->delay($email->scheduled_at);
}
```

**Khi hết quota email:**
1. Log vào Elasticsearch: `Email - 'Reach out quota'`
2. Gửi event lên Customer.io: `lai_email_reach_limit = true`
3. **Không** dispatch job nữa

---

## 14. EmailCampaignDetailServiceImp — Business Logic quan trọng

### 14.1 createDefaultCampaignDetails — Code thực tế

```php
public function createDefaultCampaignDetails(int $shopId)
{
    // Lấy automation campaign, tạo nếu chưa có
    $automationCampaign = $this->emailCampaignService->getAutomationCampaign($shopId)
        ?? $this->emailCampaignService->createDefaultCampaign($shopId);

    if ($automationCampaign) {
        $activities   = $automationCampaign->campaignDetails;
        $reviewRequest = $activities->where('email_type_id', 1);

        // Chỉ tạo nếu chưa có Review Request
        if ($reviewRequest->count() <= 0) {
            $reviewRequest = $this->createActivity($shopId, $automationCampaign->id, 1, 'Review request');
            if ($reviewRequest) {
                $this->createActivity($shopId, $automationCampaign->id, 2, 'Review reminder', $reviewRequest->id);
                $reward = $this->createActivity($shopId, $automationCampaign->id, 3, 'Reward', $reviewRequest->id);
                if ($reward) {
                    $this->createActivity($shopId, $automationCampaign->id, 4, 'Reward reminder', $reward->id);
                }
            }
        }
    }
}
```

**createActivity — Mapping template ID sang type string:**
```php
$type = [
    'Review request'  => 'review-request',
    'Review reminder' => 'review-reminder',
    'Reward'          => 'reward',
    'Reward reminder' => 'reward-reminder',
][$name];
```

**Data mỗi activity lấy từ 2 nguồn:**
1. `EmailTemplate->options` JSON: `email_type_id`, `subject`, `timing`, `timing_event`, `timing_unit`, `has_coupon`
2. `app/Templates/activities.json` → field `data` (MJML blocks JSON)

```php
$emailTemplate = EmailTemplate::find($templateId);
// options từ template:
$emailTypeId  = $emailTemplate->options['email_type_id'] ?? 5;
$subject      = $emailTemplate->options['subject'] ?? '';
$timing       = $emailTemplate->options['timing'] ?? 1;
$timingEvent  = $emailTemplate->options['timing_event'] ?? null;
$timingUnit   = $emailTemplate->options['timing_unit'] ?? 'Days';

// MJML blocks từ activities.json:
$activityData = $this->getDefaultActivityData($name);
// tìm item trong activities.json có "data.name" = "Review request" etc.
```

### 14.2 filterAutomationEmailToSend — A/B Testing Logic thực tế

```php
public function filterAutomationEmailToSend(Shop $shop, Collection $campaignDetails)
{
    $automationCampaign   = $this->emailCampaignService->getAutomationCampaign($shop->id);
    $totalCampaignDetails = $campaignDetails->count();

    if (!$automationCampaign || $totalCampaignDetails == 0) return null;

    $isTesting = $this->emailCampaignService->isAbTesting($automationCampaign->id);

    // A/B testing: chọn ngẫu nhiên
    // Normal: lấy phần tử đầu tiên
    return $isTesting
        ? $campaignDetails->all()[rand(0, $totalCampaignDetails - 1)]
        : $campaignDetails->first();
}
```

> **Lưu ý:** Khi A/B testing, mỗi order sẽ được assign ngẫu nhiên 1 activity variant. Không có session tracking — mỗi lần gọi là random mới.

### 14.3 sendAutomationRewardEmail — Trigger Reward sau khi review

Đây là method **quan trọng nhất** — được gọi sau khi khách submit review.

```php
public function sendAutomationRewardEmail(int $shopId, int $emailId, string $customerEmail)
{
    $shop = Shop::find($shopId);

    // 1. Kiểm tra shop có active Reward activity không
    if (!$this->isRewardEmailAvailable($shop)) return;

    $email    = Email::find($emailId);
    $customer = $this->customerService->findByEmail($customerEmail);

    // 2. Kiểm tra có nên gửi Reward không
    if (!$this->shouldSendRewardEmail($customerEmail, $email, $customer)) return;

    // 3. Tìm Reward activity phù hợp
    $rewardEmailCampaignDetail = $this->findRewardEmailToSend($email, $shop);
    if (!$rewardEmailCampaignDetail) return;

    // 4. Tạo Email record cho Reward
    $rewardEmail = $emailService->createEmailToSend(
        $shopId, $customer->id, $rewardEmailCampaignDetail,
        $email->order->email ?? "",
        $email->order_id,
        $email->id           // parent = email gốc (Review Request)
    );

    // 5. Dispatch job
    $sendEmailService->automationSend($shop, $customer->id, 
        $rewardEmailCampaignDetail->id, $rewardEmail, $rewardEmail->order_id);

    // 6. Đánh dấu coupon đã được issue (prevents duplicate reward)
    if ($ability->canUse('email_volume')) {
        $this->emailAttributeService->saveAttribute(EmailAttribute::COUPON, $email);
    }
}
```

**shouldSendRewardEmail — Các điều kiện:**
```php
// 1. Email gốc không bị cancel/reach-out
if (in_array($status, [Email::CANCEL, Email::REACH_OUT])) return false;

// 2. Email gốc chưa được cấp coupon (tránh duplicate)
if ($emailService->hasTakeDiscount($email)) return false;

// 3. Parent email (nếu có) cũng chưa được cấp coupon
if ($email->parent && $emailService->hasTakeDiscount($email->parent)) return false;

// 4. Customer phải tồn tại trong DB
if (!$customer) return false;
```

**findRewardEmailToSend — Tìm Reward activity:**
```php
public function findRewardEmailToSend(Email $email, Shop $shop): ?EmailCampaignDetail
{
    // Thử tìm từ activity của email hiện tại trước
    $reward = $this->filterAutomationEmailToSend($shop, $email->emailCampaignDetail->rewards);

    // Nếu không tìm thấy và có parent → tìm từ parent activity
    if (!$reward && $email->parent) {
        $reward = $this->filterAutomationEmailToSend(
            $shop, 
            $email->parent->emailCampaignDetail->rewards
        );
    }
    return $reward;
}
```

> Lý do cần tìm từ parent: Khi khách submit review từ **Review Reminder** (không phải Review Request), hệ thống vẫn cần gửi Reward. Reward activity link về Review Request, không link về Reminder.

### 14.4 updateCampaignDetails — Auto-register Shopify Webhooks

Khi shop thay đổi `timing_event`, hệ thống **tự động đăng ký webhook** với Shopify:

```php
$events = [
    'Fulfillment' => [
        'event' => 'orders/fulfilled',
        'route' => webhook_url(route('webhook.v2.orders.fulfilled'))
    ],
    'Archived' => [
        'event' => 'orders/updated',
        'route' => webhook_url(route('webhook.v2.orders.archived'))
    ],
    'Delivered' => [
        'event' => 'orders/updated',           // cùng event với Archived!
        'route' => webhook_url(route('webhook.v2.orders.archived'))  // cùng route!
    ]
];

$this->shopService->createWebhook($shop, $events[$timingEvent]['event'], $events[$timingEvent]['route']);
```

> **Lưu ý quan trọng:** `Delivered` và `Archived` đều dùng cùng Shopify webhook event `orders/updated` và cùng route. Sự phân biệt xảy ra ở `PreHandleWebhookOrderUpdatedJob` — kiểm tra `shipment_status` vs `closed_at`.

**Business rules khi update:**
```php
// Request Email: không có reference parent
if ($emailType->slug == 'request-email') {
    $attributes['reference_email_campaign_details_id'] = null;
}

// Non-Request: không có timing_event (chỉ Request mới có trigger event)
else {
    $attributes['timing_event'] = null;
}

// Reward Email: timing luôn = 0 (gửi ngay sau review)
if ($emailType->slug == 'reward-email') {
    $attributes['timing'] = 0;
    // Cũng update CouponSetting
}
```

### 14.5 deleteActivity — Cascade delete logic

```php
public function deleteActivity(int $campaignId, int $activityId)
{
    // 1. Xoá CouponSetting của activity
    $activity->couponSetting()->delete();

    // 2. Detach children (không xoá, chỉ set reference = null)
    $activity->children()->update([
        'reference_email_campaign_details_id' => null
    ]);

    // 3. Detach tất cả emails đã gửi (set campaign_detail_id = null)
    $activity->emails()->update([
        'email_campaign_detail_id' => null
    ]);

    // 4. Xoá discount sections và CouponSetting của từng section
    $activity->sections()->each(function ($section) {
        if ($section->type == 'discount') {
            $section->discount()->delete();
        }
    });

    // 5. Xoá activity
    return $activity->delete();
}
```

---

## 15. PreHandleWebhookOrderUpdatedJob — Code thực tế

```php
public function handle(EmailCampaignService $emailCampaignService, ShopService $shopService)
{
    $shopId        = $shopService->getShopByShopName($this->shopName)->id;
    $emailCampaign = $emailCampaignService->getAutomationCampaign($shopId);

    if (!$emailCampaign) return;

    // Lấy timing_event từ activity ĐẦU TIÊN của campaign
    $timingEvent = $emailCampaign->campaignDetails->first()->timing_event;

    if ($timingEvent === Order::ARCHIVED) {
        // Chỉ xử lý nếu order đã closed
        if (!$this->request['closed_at']) return;

        ProcessWebhookOrderJob::dispatch($this->request, $this->shopName, Order::ARCHIVED)
            ->onQueue('webhook');
    } else {
        // Xử lý Delivered event
        $this->handleWebhookDeliveryOrder($this->request);
    }
}

private function handleWebhookDeliveryOrder($request)
{
    $fulfillments   = collect(collect($request)->get('fulfillments'));
    $shipmentStatus = $fulfillments->first()['shipment_status'] ?? null;

    if (!$shipmentStatus || $shipmentStatus != 'delivered') return;

    ProcessWebhookOrderJob::dispatch($request, $this->shopName, Order::DELIVERED)
        ->onQueue('webhook');
}
```

> **Điểm quan trọng:** Job này lấy `timing_event` từ `$emailCampaign->campaignDetails->first()->timing_event` — tức là lấy timing_event của **activity đầu tiên** trong campaign (Review Request). Điều này có nghĩa **toàn bộ campaign chỉ có 1 timing_event** — không thể có Fulfillment + Archived cùng lúc.

---

## 16. Luồng Reward Email — Chi tiết đầy đủ

```
Khách mở email Request → click "Write a review" 
        │
        ▼ redirect_url với params
  [GET /reviews/submit?email_id={id}&store={shop}]
  ReviewController
        │
        ▼ submit review
  ReviewService->create()
        │
        ▼ sau khi review được lưu
  EmailCampaignDetailService->sendAutomationRewardEmail(shopId, emailId, customerEmail)
        │
        ├─ isRewardEmailAvailable()    → shop có active Reward activity không?
        ├─ shouldSendRewardEmail()     → chưa issue coupon, customer tồn tại
        ├─ findRewardEmailToSend()     → tìm Reward activity (từ current hoặc parent)
        ├─ createEmailToSend()         → tạo Email record (parent = review request email)
        │
        ▼ dispatch với timing = 0 (ngay lập tức)
  AutomationEmailCampaign Job (queue: emails)
        │
        ├─ emailInterceptionHandler()  → kiểm tra quota, feature email_coupon
        ├─ shouldStopSending()         → skip (Reward không có parent attribute check)
        ├─ createCoupon()              → tạo CustomerCoupon qua Shopify API
        ├─ emailService->send()        → build HTML, gửi email có coupon code
        │
        ▼ fire event
  AutomationEmailSend
        │
        ▼ listener schedule
  Review Reminder? KHÔNG (canSendReminder = false vì activity.children rỗng)
  Reward Reminder? CÓ nếu có Reward Reminder activity
```

**Coupon trong Reward email được kế thừa sang Reward Reminder:**
```php
// Trong AutomationEmailCampaign:
if ($coupon) {
    $this->attachCouponToEmails($sentEmail, $coupon);
}

private function attachCouponToEmails(Email $email, CustomerCoupon $coupon): void
{
    $email->coupons()->save($coupon);                    // Reward email
    if ($this->currentEmail->parent) {
        $this->currentEmail->parent->coupons()->save($coupon);  // Review Request email (parent)
    }
}

// Trong AutomationEmailSent Listener (schedule Reward Reminder):
if ($coupon) {
    $email->update([
        'email_data' => array_merge($email->email_data ?? [], [
            'coupon_id' => $coupon->id,   // lưu vào email_data để Reward Reminder dùng lại
        ]),
    ]);
}
```

---

## 17. Email Attributes — Trạng thái đầy đủ

| attribute_name | is_done | Ý nghĩa | Ghi chú |
|----------------|---------|---------|---------|
| `pending` | false | Email đang chờ gửi | Xoá khi gửi thành công |
| `sent` | true | Email đã gửi | Thêm sau khi Mail::send() thành công |
| `opened` | true | Email đã được mở | Trigger bởi tracking pixel |
| `clicked` | true | Link trong email đã được click | Trigger bởi redirect URL |
| `rate` | true | Khách đã submit review | Trigger cancel Reminder |
| `coupon` | true | Coupon đã được issue | Ngăn gửi Reward thứ 2 |
| `cancel` | false | Email bị huỷ | Shop uninstall, review rồi, etc. |
| `reach-out` | - | Hết quota | Có thể retry sau |
| `skipped` | - | Email/product bị exclude | |

**Constants trong EmailAttribute model:**
```php
const PENDING   = 'pending';
const SENT      = 'sent';
const RATE      = 'rate';
const OPENED    = 'opened';
const REACH_OUT = 'reach-out';
const CANCEL    = 'cancel';
const CLICKED   = 'clicked';
const COUPON    = 'coupon';
const SKIPPED   = 'skipped';
```

---

## 18. Tóm tắt toàn bộ luồng từ A đến Z

```
1. SHOPIFY ORDER EVENT
   └─ Shopify gọi POST /webhook/v2/orders/{fulfilled|paid|updated}
   └─ VerifyShopifyWebhook middleware xác thực HMAC

2. CONTROLLER → QUEUE (webhook)
   └─ fulfilled/paid → ProcessWebhookOrderJob(event: Fulfillment|Paid)
   └─ updated        → PreHandleWebhookOrderUpdatedJob
                          → nếu timingEvent=Archived + closed_at → ProcessWebhookOrderJob(Archived)
                          → nếu shipmentStatus=delivered → ProcessWebhookOrderJob(Delivered)

3. ProcessWebhookOrderJob (queue: webhook)
   └─ Check: feature enabled, DELIVERED feature, customer email tồn tại
   └─ Check: duplicate (không gửi nếu đã queue)
   └─ saveOrder() → bảng `orders`
   └─ saveCustomer() → bảng `customers`
   └─ Klaviyo/Omnisend integration nếu enabled
   └─ Lấy active Review Request activities
   └─ filterAutomationEmailToSend() → 1 activity (random nếu A/B test)
   └─ createEmailToSend() → INSERT `emails` (status=pending, scheduled_at=now+1day)
   └─ automationSend() → dispatch AutomationEmailCampaign với delay(scheduled_at)

4. AutomationEmailCampaign (queue: emails, delay ~1 ngày)
   └─ emailInterceptionHandler():
      - email không cancel/skip
      - shop còn installed
      - còn email_volume quota
      - reward cần feature email_coupon
   └─ shouldStopSending(): nếu parent đã rated/coupon-used → cancel
   └─ createCoupon(): nếu Reward email → tạo Shopify discount code
   └─ emailService->send():
      - AutomationCampaignAdapter.generateEmailContent() → HTML
      - generateEmailText() → subject
      - Mail::to(customer.email).send(AutomationCampaign)
      - Custom sender: SendGrid SMTP hoặc AWS SES
   └─ markAndRemovePendingStatus(SENT) + lưu snapshot vào email_data
   └─ fire AutomationEmailSend event

5. AutomationEmailSent Listener
   └─ Lấy $activity->reminders (active, type=2)
   └─ filterAutomationEmailToSend() → 1 reminder activity
   └─ createEmailToSend() → INSERT `emails` (parent=current, scheduled_at=now+3days)
   └─ automationSend() → dispatch AutomationEmailCampaign với delay(scheduled_at)

6. Khách mở email (tracking pixel)
   └─ GET /email/track-open → status=opened + attribute opened

7. Khách click link "Write a review"
   └─ GET /email/track-click → status=clicked + attribute clicked
   └─ Redirect đến review form

8. Khách submit review
   └─ EmailCampaignDetailService.sendAutomationRewardEmail()
   └─ markAsRate(emailId) → attribute RATE=true (cancels pending reminders)
   └─ Tạo Reward email → dispatch với delay(0) → ngay lập tức
   └─ Reward email tạo Shopify coupon → gửi email có coupon code
   └─ AutomationEmailSent → schedule Reward Reminder với delay(3 days)

9. Reward Reminder (3 ngày sau khi nhận Reward)
   └─ shouldStopSending(): nếu coupon đã được dùng → cancel
   └─ Nếu chưa dùng → gửi reminder kèm cùng coupon code
```

---

---

## 19. AutomationCampaignAdapter — Render HTML Email

File: `app/Adapters/Email/AutomationCampaignAdapter.php`

Đây là class chịu trách nhiệm **build HTML email** từ MJML blocks JSON. Sử dụng thư viện `mjml` binary.

### 19.1 generateEmailContent — Quy trình render

```php
public function generateEmailContent(bool $isPreview = false): string
{
    // 1. Chuẩn bị blocks (inject sản phẩm + coupon vào đúng block)
    if ($this->activity) {
        $this->prepareEmailBlocks();
    }

    // 2. Collect variables để substitute vào template
    $variables = $this->collectEmailVariables();
    // {shopName}, {firstName}, {lastName}, {fullName}, {email}, {couponValue}, {couponCode}

    // 3. Build MJML string qua MjmlBuilder
    $mjml = $builder->start($variables)
        ->startHead()
        ->withPreview($value->preview ?? "")
        ->withStyle($style)             // nếu isPreview: "body { pointer-events: none }"
        ->withBreakpoint($value->breakpoint)
        ->withFonts($value->fonts)
        ->withAttributes($value)
        ->endHead()
        ->withBody($this->pageBlock->attributes, $this->pageBlock->children)
        ->withEmailTrackingImage($this->emailId, $shopName, $isPreview)  // tracking pixel
        ->end();

    // 4. Compile MJML → HTML qua binary
    $renderer = new BinaryRenderer(base_path() . '/node_modules/.bin/mjml');
    return $renderer->render($mjml);
}
```

> **MJML binary:** Hệ thống gọi `node_modules/.bin/mjml` để compile MJML → responsive HTML. Binary này phải được cài đặt qua `npm install`.

### 19.2 prepareEmailBlocks — Inject data vào blocks

```php
private function prepareEmailBlocks()
{
    // Review Request & Review Reminder: inject sản phẩm từ order
    if ($this->activity->isReviewRequest() || $this->activity->isReviewReminder()) {
        $products = $productService->remainingPreviewProductsInEmail($shop->id, $order->id);
        $productIds = $products->pluck("id")->toArray();

        // Duplicate block product_review N lần (theo số sản phẩm)
        $this->duplicateProductReviewBlock(count($productIds));
        
        // Populate từng block với data sản phẩm
        $this->populateProductReviewBlockWithData($productIds);
    }

    // Reward & Reward Reminder: inject coupon
    if ($this->activity->isEmailReward() || $this->activity->isCouponReminder()) {
        $this->populateCouponBlockWithData();  // inject $customerCoupon vào coupon block
    }
}
```

### 19.3 populateProductReviewBlockWithData — Inject sản phẩm + Review link

```php
// Mỗi sản phẩm được inject vào block product_review với:
$product->title = $record->title;
$product->rate  = ceil($record->rate);  // số sao hiển thị gợi ý
$product->url   = generateEmailLink(
    $record->url,
    $this->emailId,
    $shopName,
    [
        'scm_review_mail' => 1,
        'scm_mail'        => $customer->email,
        'scm_rating'      => 5,          // default rating suggestion = 5 sao
        'scm_name'        => "{$customer->first_name} {$customer->last_name}"
    ]
);  // URL tracking click + redirect đến review form
$product->image = $record->image;
```

> **URL tracking:** Tất cả product links đều được wrap qua `generateEmailLink()` để track click. Params `scm_*` được dùng để pre-fill review form (email, rating, name).

### 19.4 collectEmailVariables — Variables trong template

```php
return [
    "shopName"    => shopNameFromDomain($this->shop->shop),
    "firstName"   => $customer->first_name ?? "",
    "lastName"    => $customer->last_name ?? "",
    "fullName"    => "{$customer->first_name} {$customer->last_name}",
    "email"       => $customer->email ?? "",
    "couponValue" => $couponSetting ? 
        "{$couponSetting->discount_value}{$unit}" : "",  // vd: "10%" hoặc "$5"
    "couponCode"  => $couponSetting->coupon->coupon ?? ""
];
```

**Currency symbol:** Lấy từ `$shop->info->currency` → `getCurrencySymbol($currency)`

### 19.5 Test send — Dummy data

Khi `isTest = true`:
```php
// Dùng 3 sản phẩm đầu tiên của shop (không theo order thực)
$productIds = Product::where("shop_id", $shop->id)->limit(3)->pluck("id")->toArray();

// Coupon dummy nếu không có coupon thực
$this->customerCoupon = new CustomerCoupon(["coupon" => "LAI-DISCOUNT"]);
```

---

## 20. EmailSettingService — Exclude & Block Logic

File: `app/Services/EmailSetting/EmailSettingServiceImp.php`

### 20.1 hasBlocked — Kiểm tra email/product bị exclude

```php
public function hasBlocked(int $shopId, string $email, int $orderId = null): bool
{
    $ability = $shop->subscription()->ability();
    $setting = $this->getSetting($shopId);

    // Kiểm tra email bị exclude (cần feature 'exclude_emails')
    if ($ability->enabled('exclude_emails') && in_array($email, $setting->exclude_emails)) {
        return true;
    }

    // Kiểm tra tất cả products trong order bị exclude (cần feature 'exclude_products')
    if ($ability->enabled('exclude_products')) {
        $products   = $productService->getProductsByOrder($shop, $order);
        $productIds = $products->pluck('id')->toArray();
        $intersect  = array_intersect($productIds, $setting->exclude_product_ids);

        // Chỉ block nếu TẤT CẢ sản phẩm trong order đều bị exclude
        if (count($intersect) === count($productIds)) {
            return true;
        }
    }

    return false;
}
```

> **Lưu ý quan trọng về exclude product:** Hệ thống chỉ skip email nếu **TẤT CẢ** sản phẩm trong order nằm trong danh sách exclude. Nếu order có 3 sản phẩm mà chỉ 2 bị exclude → vẫn gửi email.

### 20.2 skipEmails — Retroactive cancel pending emails

Khi shop thêm email/product vào exclude list, hệ thống **ngay lập tức** cancel tất cả emails đang pending:

```php
public function skipEmails(Collection $pendingEmails, array $emailAddresses, array $productIds)
{
    $pendingEmails->each(function (Email $pendingEmail) use ($emailAddresses, $productIds) {
        $order = $pendingEmail->order;

        // Nếu email bị exclude → skip
        if (in_array($order->email, $emailAddresses)) {
            $this->emailService->markAndRemovePendingStatus($pendingEmail, Email::SKIPPED, EmailAttribute::SKIPPED);
            return;
        }

        // Nếu TẤT CẢ products của order bị exclude → skip
        if (count($productIds) > 0 && $order) {
            $products      = $productService->getProductsByOrder($shop, $order);
            $productIdsInOrder = $products->pluck('id')->toArray();
            $intersect     = array_intersect($productIdsInOrder, $productIds);

            if (count($intersect) == count($productIdsInOrder)) {
                $this->emailService->markAndRemovePendingStatus($pendingEmail, Email::SKIPPED, EmailAttribute::SKIPPED);
            }
        }
    });
}
```

**Khi nào skipEmails được gọi:**
1. `updateOrCreateSetting()` — khi shop cập nhật toàn bộ settings
2. `excludeProducts(action: 'block')` — khi block thêm products
3. `excludeEmails()` — khi thêm email vào exclude list

### 20.3 excludeProducts — Block/Unblock sản phẩm

```php
public function excludeProducts(int $shopId, array $attributes)
{
    $productIds = $attributes['product_ids'];
    $action     = $attributes['action'];  // 'block' | 'unblock'

    if ($action == 'block') {
        // Merge vào list hiện tại
        $newExcludeProductIds = array_merge($excludeProductIds, $productIds);
        // Ngay lập tức skip pending emails liên quan
        $this->skipEmails($pendingEmails, [], $newExcludeProductIds);
    }

    if ($action == 'unblock') {
        // Loại bỏ khỏi list
        $newExcludeProductIds = array_diff($excludeProductIds, $productIds);
        // Không tự động resume emails đã skipped
    }
}
```

> **Unblock:** Khi unblock, emails đã bị skip **không** được tự động resume. Chúng vẫn ở trạng thái `SKIPPED`.

---

## 21. Config Mail — AWS SES Setup

File: `config/mail.php`

```php
'driver'       => env('MAIL_DRIVER', 'smtp'),
'host'         => env('AWS_SES_SMTP_ENDPOINT', ''),
'port'         => env('MAIL_PORT', 587),
'from'         => [
    'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
    'name'    => env('MAIL_FROM_NAME', 'LAI Review'),
],
'username'     => env('AWS_SES_USERNAME'),
'password'     => env('AWS_SES_PASSWORD'),
'encryption'   => 'tls',
```

**Biến môi trường cần thiết:**
```env
MAIL_DRIVER=smtp
AWS_SES_SMTP_ENDPOINT=email-smtp.us-east-1.amazonaws.com
AWS_SES_USERNAME=AKIAIOSFODNN7EXAMPLE
AWS_SES_PASSWORD=wJalrXUtnFEMI...
MAIL_FROM_ADDRESS=noreply@laireviews.com
MAIL_FROM_NAME="LAI Reviews"

# Custom sender (SendGrid)
CUSTOM_EMAIL_SENDER_PORT=587
# Passwords lưu trong config/secom.php:
# 'sengrid' => ['account_name' => 'SG.xxxxx']
```

---

## 22. Các migrations theo thứ tự thời gian

| Năm | Migration | Thay đổi chính |
|-----|-----------|----------------|
| 2019 | create_email_settings | Bảng email_settings với đầy đủ fields (timing, template, subject...) |
| 2019 | create_email_campaigns | Bảng email_campaigns cơ bản |
| 2019 | create_email_campaign_details | Bảng email_campaign_details cơ bản |
| 2019 | create_emails | Bảng emails cơ bản |
| 2019 | create_email_types | Bảng email_types |
| 2021 | add_email_attributes | Bảng email_attributes (polymorphic tracking) |
| 2022 | add_email_template | Bảng email_templates + email_categories |
| 2022 | add_ab_test | Bảng ab_test_campaigns + ab_test_campaign_id vào emails |
| 2022 | add_custom_email_sender | Bảng custom_email_senders |
| 2022 | add_email_template_id_to_details | Thêm email_template_id, thumbnail vào email_campaign_details |
| 2023 | rebuild_email_settings | Drop toàn bộ columns cũ của email_settings, rebuild đơn giản hơn |
| 2023 | add_delivered_to_timing_event | Thêm 'Delivered' vào ENUM timing_event |
| 2024 | add_data_to_campaign_details | Thêm `data` JSON (MJML blocks) vào email_campaign_details |
| 2024 | add_coupon_setting_id | Thêm coupon_setting_id vào email_campaign_details |
| 2025 | add_shop_id_type_to_details | Thêm shop_id và type vào email_campaign_details (new API) |

---

## 23. Feature Flags liên quan Email

| Feature slug | Mô tả | Khi disabled |
|-------------|-------|-------------|
| `email_review_request` | Tính năng gửi Review Request | Toàn bộ email pipeline không hoạt động |
| `review_request_upon_delivery` | Trigger email khi delivered | DELIVERED event bị ignore |
| `email_coupon` | Gửi Reward email có coupon | Reward email bị cancel |
| `exclude_emails` | Exclude specific emails | hasBlocked() skip email check |
| `exclude_products` | Exclude specific products | hasBlocked() skip product check |
| `klaviyo` | Klaviyo integration | Không gửi event sang Klaviyo |
| `omnisend` | Omnisend integration | Không gửi event sang Omnisend |
| `email_volume` | Quota email tháng | Email vào REACH_OUT, không dispatch job |

---

---

## 24. ReviewSubmittedSideEffectJob — Trigger Reward sau khi Review

File: `app/Jobs/ReviewSubmittedSideEffectJob.php`

Đây là job được dispatch sau khi khách submit review. Chứa toàn bộ side effects khi có review mới.

### 24.1 Constructor params

```php
public function __construct(
    int $shopId,
    $review,
    $trackedEmailId = null,    // email ID từ tracking URL (nếu review từ email)
    $trackedProvider = null,   // 'klaviyo' | 'omnisend' (nếu review từ integration link)
    $coupon = null             // coupon nếu có
)
```

### 24.2 handle() — Toàn bộ side effects

```php
public function handle()
{
    $shop = Shop::find($shopId);
    if (!$shop || $shop->uninstalled()) return;

    $ability         = $shop->subscription()->ability();
    $canUseEmailCoupon = $ability->enabled('email_coupon') && $ability->canUse('email_volume');
    $sendImmediately = !$reviewSetting->enable_email_coupon_waiting;
    // enable_email_coupon_waiting = false (default) → gửi ngay
    // enable_email_coupon_waiting = true → đợi admin approve review trước

    // 1. Gán email_id vào review nếu có tracked email
    if ($trackedEmailId && $canUseEmailCoupon && is_numeric($trackedEmailId)) {
        $review->email_id = $trackedEmailId;
    }

    // 2. Trigger Customer.io event 'submit_review'
    $customerAttributeService->triggerSubmitReview($shop);

    // 3. Update email status → 'rated' + thêm attribute 'rate'
    if ($trackedEmailId && is_numeric($trackedEmailId)) {
        $emailService->updateEmailStatus($emailId, 'rated');
        $emailService->markAsRate((int)$emailId);  // + Customer.io milestone 1st/10th
    }

    // 4. Gửi Reward email (nếu đủ điều kiện)
    if ($canUseEmailCoupon && $trackedEmailId && $review->email && $sendImmediately) {
        $emailCampaignDetailService->sendAutomationRewardEmail($shopId, $trackedEmailId, $review->email);
    }

    // 5. Gửi coupon email cho merchant (thông báo có review mới kèm coupon)
    if ($coupon && $review->email) {
        $notificationService->notifyCouponForSubmitReview($review, $coupon, $shop);
    }

    // 6. Gửi event REVIEW_SUBMITTED lên Klaviyo
    if ($canUseKlaviyo && $review->email) {
        $this->sendEventToKlaviyo();
        // Lưu integration_attributes: attribute_name='submitted_event', value=review->id
    }

    // 7. Gửi event REVIEW_SUBMITTED lên Omnisend (cần phone)
    if ($canUseOmnisend && $review->email) {
        $this->sendEventToOmnisend();
        // Event system name: 'LAI_reviews_are_submitted'
    }

    // 8. Track nếu review từ Klaviyo link
    if ($trackedProvider == 'klaviyo') {
        // Lưu integration_attributes: attribute_name='review_submitted'
    }

    // 9. Track nếu review từ Omnisend link
    if ($trackedProvider == 'omnisend') {
        // Lưu integration_attributes: attribute_name='review_submitted'
    }
}
```

### 24.3 enable_email_coupon_waiting — Cơ chế delay Reward

Setting `enable_email_coupon_waiting` trong `review_settings`:
- `false` (default): Gửi Reward ngay lập tức khi khách submit review
- `true`: **Không** gửi Reward trong `ReviewSubmittedSideEffectJob`. Thay vào đó, Reward chỉ được gửi khi admin **approve** review

Khi admin approve:
```
ReviewService->approve()
  → NotificationServiceImp->notifyMerchantHasReview()
      → nếu review có reviewEmail && availableEmailCouponWaiting
          → emailCampaignDetailService->sendAutomationRewardEmail()
```

### 24.4 Integration tracking khi review từ email

Khi khách click review link từ email:
- URL chứa `?email_id={id}` (hoặc `scm_review_mail=1`)
- `ReviewController` extract `email_id` và pass vào `ReviewSubmittedSideEffectJob`
- Job update `emails.status = 'rated'` và thêm attribute `rate`
- Review record có `email_id` để track nguồn

### 24.5 Shopify Flow Integration

```php
TriggerFlowJob::dispatch($shop, $triggerId, $review);
// Gửi trigger đến Shopify Flow workflow khi có review mới
// trigger_id lấy từ config('shopify.trigger_id')
```

---

## 25. Event-Listener Map đầy đủ (Email liên quan)

```
AutomationEmailSend     →  AutomationEmailSent    (schedule reminders sau khi gửi)
CouponEmailSend         →  CouponEmailSent         (coupon-specific email flow)
ReviewSubmittedEvent    →  ReviewSubmittedListener (integrations + webhook)
```

**Luồng event chi tiết:**

```
1. AutomationEmailCampaign Job gửi email xong
   └─ event(AutomationEmailSend)
   └─ AutomationEmailSent::handle()
      └─ schedule Review Reminder (delay 3 ngày)

2. Khách submit review
   └─ ReviewSubmittedSideEffectJob::handle()
      ├─ markAsRate(emailId) → attribute 'rate' = true
      │     → shouldStopSending() sẽ cancel Review Reminder khi chạy
      ├─ sendAutomationRewardEmail() → dispatch Reward email
      ├─ sendEventToKlaviyo() → REVIEW_SUBMITTED event
      └─ sendEventToOmnisend() → LAI_reviews_are_submitted trigger

3. Reward email được gửi
   └─ event(AutomationEmailSend) với Reward activity
   └─ AutomationEmailSent::handle()
      └─ $activity->reminders → Reward Reminder (delay 3 ngày)

4. Reward Reminder chạy
   └─ shouldStopSending():
      └─ parent = Reward email
      └─ mappingAttributeName() → 'use-coupon'
      └─ hasDone('use-coupon', parent)? → nếu có → cancel
```

---

## 26. Bảng tóm tắt các Queue và Jobs

| Job | Queue | Khi nào dispatch | Delay |
|-----|-------|-----------------|-------|
| `PreHandleWebhookOrderUpdatedJob` | webhook | webhook orders/updated | 0 |
| `ProcessWebhookOrderJob` | webhook | sau Pre-handle hoặc từ Controller | 0 |
| `AutomationEmailCampaign` | emails | sau createEmailToSend | scheduled_at (1-3 ngày) |
| `ReviewSubmittedSideEffectJob` | default | sau khi review được submit | 0 |
| `SendEmailTest` | emails | shop click "Send test" | 0 |
| `TriggerFlowJob` | default | sau khi review submit | 0 |

---

## 27. Ghi chú đặc biệt — Edge cases

### Trường hợp 1: A/B test randomness
Hàm `filterAutomationEmailToSend()` dùng `rand()` — không dùng seeded random. Điều này nghĩa là cùng 1 customer có thể nhận activity khác nhau nếu order bị re-processed.

### Trường hợp 2: Hardcoded account 'tiennm'
Trong `AutomationCampaign::send()`:
```php
if ($shop->canUseCustomSender() && $customEmailSender->account != 'tiennm') {
    $this->useCustomMailer(...)
}
```
Account `tiennm` bị force về AWS SES dù shop có custom sender. Đây là exception debug/internal.

### Trường hợp 3: MySQL Read Replica
```php
$order = Order::on('mysql_readonly')->select()->where([...])->first();
```
`ProcessWebhookOrderJob` query orders trên replica để giảm tải. Có thể gây **replication lag** — order vừa insert ở master chưa sync sang replica → `$order = null` → cho phép tạo email mới dù order đã có.

### Trường hợp 4: Reward timing = 0
Reward email không có delay (`timing = 0`). Nhưng job vẫn được dispatch qua queue — không phải synchronous. Trong production với queue worker busy, có thể delay vài giây đến vài phút.

### Trường hợp 5: Email snapshot
Sau khi gửi, `email_data` được update với:
```json
{
    "subject": "Please let us know...",
    "content": "<html>...(full HTML)...</html>",
    "version": "3.2.0",
    "is_resend": false,
    "coupon_id": null
}
```
Field `content` là snapshot HTML đầy đủ — dùng để preview lại email đã gửi mà không cần re-render.

### Trường hợp 6: Email SKIPPED vs REACH_OUT
- **SKIPPED**: Email bị block bởi exclude list → không dispatch job, không re-queue
- **REACH_OUT**: Hết quota → không dispatch job, nhưng khi quota reset **không** tự resume. Admin phải resend thủ công.

### Trường hợp 7: Double listener cho Reward
`sendAutomationRewardEmail` được gọi từ 2 nơi:
1. `ReviewSubmittedSideEffectJob` (khi `sendImmediately = true`)
2. `NotificationServiceImp::notifyMerchantHasReview()` (khi `enable_email_coupon_waiting = true` và admin approve)

Hàm `shouldSendRewardEmail()` protect duplicate: kiểm tra `hasTakeDiscount(email)` — nếu coupon đã được issue rồi thì return false.

---

---

## 28. CustomerCouponService — Tạo Coupon Shopify

File: `app/Services/CustomerCoupon/CustomerCouponServiceImp.php`

### 28.1 createCustomerCoupon — Full flow

```php
public function createCustomerCoupon(Shop $shop, $type)
{
    // 1. Lấy CouponSetting theo shop + type ('email' | 'review-setting')
    $couponSetting = $couponSettingService->getCouponSetting($shopId, $type);

    $isAutoGenerate = $couponSetting->code_type; // 'auto' | null(static)

    if (!$isAutoGenerate) {
        // Static code: dùng code cố định từ CouponSetting
        return CustomerCoupon::create([
            'shop_id'           => $shopId,
            'coupon_setting_id' => $couponSetting->id,
            'coupon'            => $couponSetting->code,
            'type'              => 'static'
        ]);
    }

    // Auto-generate: tạo unique code
    $code = $this->generateUniqueCouponCode($shop);  // format: 'LAI-xxxxxxx'

    // Tính expires_at
    $endsAt = $couponSetting->expires_after
        ? Carbon::now()->addDays($couponSetting->expires_after)->toIso8601String()
        : null;

    // 2. Tạo Price Rule trên Shopify
    $priceRule = $this->createPriceRule($shop, [
        'title'              => $code,
        'target_type'        => 'line_item',
        'target_selection'   => 'all',
        'allocation_method'  => 'across',
        'customer_selection' => 'all',
        'once_per_customer'  => true,           // chỉ dùng 1 lần/khách
        'value_type'         => $couponSetting->discount_type,  // 'percentage' | 'fixed_amount'
        'value'              => "-{$couponSetting->discount_value}",  // âm vì là discount
        'starts_at'          => Carbon::now()->toIso8601String(),
        'ends_at'            => $endsAt
    ]);

    // 3. Tạo Discount Code gắn vào Price Rule
    $discountCode = $this->createDiscountCode($shop, $priceRule['id'], $code);

    // 4. Lưu vào customer_coupons
    return CustomerCoupon::create([
        'shop_id'                  => $shopId,
        'shopify_discount_code_id' => $discountCode['id'],
        'shopify_price_rule_id'    => $priceRule['id'],
        'coupon_setting_id'        => $couponSetting->id,
        'coupon'                   => $code,
        'type'                     => 'auto',
        'expired_at'               => $expiresAfter ? $endsAt : Carbon::now()
    ]);
}
```

### 28.2 generateUniqueCouponCode — Format và uniqueness check

```php
public function generateUniqueCouponCode(Shop $shop): string
{
    $tries = 3;  // thử tối đa 3 lần
    do {
        // Format: 'LAI-' + base36(timestamp)
        // vd: 'LAI-lhzpg4c' (timestamp 1716700000 → base36)
        $tempCouponCode = 'LAI-' . base_convert(time(), 10, 36);

        // Kiểm tra với Shopify API xem code đã tồn tại chưa
        $res = $clientApi->get("discount_codes/lookup.json", null, ['code' => $tempCouponCode]);

        if (data_get($res, 'errors') == 'Not Found') {
            return $tempCouponCode;  // Code unique → sử dụng
        }
        $tries--;
    } while ($tries > 0);

    return '';  // Không tìm được unique code sau 3 lần
}
```

> **Vấn đề tiềm ẩn:** Nếu 3 lần thử đều fail (code đã tồn tại), trả về empty string. Hàm caller check `empty($code)` và throw exception. Email sẽ không được gửi.

### 28.3 Shopify API Integration

```php
// createPriceRule — Tạo Price Rule
$client = get_client($shop);               // Shopify API client
$priceRuleApi = new PriceRule($client);    // secomapp/laravel-shopify package
$priceRule = $priceRuleApi->create($attributes);

// createDiscountCode — Gắn code vào Price Rule
$discountCodeApi = new DiscountCode($client);
$discountCode = $discountCodeApi->create($priceRuleId, $code);
```

**Package:** `secomapp/laravel-shopify`

### 28.4 Error tracking

Tất cả lỗi đều được log 2 nơi:
```php
logger()->error("Shop $shop->id - Failed to create coupon: ...");
logExceptionToMixpanel($e, $shop->id, "Failed to create coupon");  // Mixpanel event
```

---

## 29. Routes thực tế (routes/api.php + routes/web.php)

### 29.1 Public API routes (api.php — không cần auth)

```
GET  /api/mail/action              → EmailController@action
GET  /api/mail/trackOpenEmail      → EmailController@trackOpenEmail
GET  /api/mail/trackClickEmail     → EmailController@trackClickEmail
GET  /api/email                    → EmailController@index (review form)
```

### 29.2 Admin API routes (web.php — cần auth-shop)

**New Email API (2025):**
```
GET   /email/getCampaignStatuses
GET   /email/getReviewRequestSetting       → getReviewRequestSetting
POST  /email/updateReviewRequestSetting    → updateReviewRequestSetting
GET   /email/getReviewReminderSetting
POST  /email/updateReviewReminderSetting
GET   /email/getRewardSetting
POST  /email/updateRewardSetting
GET   /email/getRewardReminderSetting
POST  /email/updateRewardReminderSetting
GET   /email/getEmailTemplateData
POST  /email/updateEmailTemplateData
POST  /email/sendTest
```

**Campaign API (v2):**
```
GET   /v2/email-campaigns                       → list campaigns
PUT   /v2/email-campaigns/{id}                  → update campaign
GET   /v2/email-campaigns/{id}/activities       → list activities
POST  /v2/email-campaigns/{id}/activities       → create activity
PUT   /v2/email-campaigns/{id}/activities/{aid} → update activity
GET   /v2/email-campaigns/{id}/activities/{aid}/preview → preview HTML
PATCH /v2/email-campaigns/{id}/activities/{aid} → toggle status
GET   /v2/email-templates/                      → list templates
```

**Email list & management (v2):**
```
GET  /v2/emails                    → list emails (paginated)
GET  /v2/emails/{id}               → detail
PUT  /v2/emails/cancel             → cancel pending [middleware: cancel.pending.email]
PUT  /v2/emails/re-send            → resend [middleware: available.volume:email, re-send.email]
POST /v2/emails/send-manually      → manual send
```

**Email settings:**
```
GET  /email-settings               → get settings
POST /email-settings/store         → create/update
PUT  /email-settings/exclude-emails    → [feature: exclude_emails]
PUT  /email-settings/unblock-emails
PUT  /email-settings/exclude-products  → [feature: exclude_products]
```

**Custom sender:**
```
GET  /custom-email/               → get custom sender info
POST /custom-email/verify         → verify domain DNS
POST /custom-email/validate       → validate domain
POST /custom-email/send-test      → [throttle: 30 per 1440 min = 30/day]
```

**Reports:**
```
GET /reports/email/dashboard      → email stats overview
GET /reports/email/metrics        → detailed metrics
```

### 29.3 Middleware quan trọng trên email routes

| Middleware | Tác dụng |
|-----------|---------|
| `cancel.pending.email` | Cancel tất cả pending emails của order trước khi cancel request |
| `available.volume:email` | Kiểm tra còn email_volume quota |
| `re-send.email` | Validation cho resend (email đã gửi xong mới được resend) |
| `feature.available:exclude_emails` | Kiểm tra feature flag |
| `feature.available:exclude_products` | Kiểm tra feature flag |
| `throttle:30,1440` | Rate limit custom sender test: 30 requests/ngày |

---

## 30. TrackClickEmail — URL Structure chi tiết

URL trong email khi click bất kỳ link:
```
/api/mail/trackClickEmail?email_attribute={emailId}&shop_name={shopName}&redirect_url={originalUrl}
```

Sau khi track → redirect đến:
```
{originalUrl}?email_attribute={emailId}&shop_name={shopName}&redirect_url={originalUrl}
```

> **Lưu ý:** trackClickEmail gộp **tất cả** params vào redirect URL. Điều này có nghĩa params tracking (`email_attribute`, `shop_name`) sẽ xuất hiện trong URL đích. Review form có thể dùng `email_attribute` để map review về đúng email.

**Special case — email_attribute = 1:**
```php
if ($emailAttribute != 1) {
    // Track normally
}
// email_attribute = 1 → skip tracking (test/fallback)
```

---

## 31. Bảng `couponables` — Polymorphic coupon pivot

Bảng pivot không xuất hiện trong migrations riêng mà được tạo qua `morphToMany`:

```sql
CREATE TABLE couponables (
    id                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    couponable_id     BIGINT UNSIGNED NOT NULL,
    couponable_type   VARCHAR(255) NOT NULL,
    -- 'App\Models\Email'   → email đã gửi coupon
    -- 'App\Models\Review'  → review được tặng coupon
    customer_coupon_id BIGINT UNSIGNED NOT NULL,
    created_at        TIMESTAMP,
    updated_at        TIMESTAMP
);
```

**Tại sao cần couponables:**
- Một coupon có thể gắn với nhiều `Email` records (Reward email + parent Review Request email)
- Một coupon có thể gắn với `Review` nếu submit từ coupon form
- Cho phép query "email nào đã có coupon" và "review nào có coupon"

---

---

## 32. NewEmailCampaignService — New API (2025 Refactor)

File: `app/Services/EmailCampaign/NewEmailCampaignServiceImp.php`

API mới query thẳng theo `shop_id + type` thay vì đi qua `campaign_id`:

```php
// Get
public function getEmailCampaignDetail($shopId, $type)
{
    return EmailCampaignDetail::query()->where([
        'shop_id' => $shopId,
        'type'    => $type,  // 'review-request' | 'review-reminder' | 'reward' | 'reward-reminder'
    ])->first();
}

// Update
public function updateEmailCampaignDetail($shopId, $type, $attributes)
{
    return EmailCampaignDetail::query()->where([
        'shop_id' => $shopId,
        'type'    => $type,
    ])->update($attributes);
}

// Update campaign (parent)
public function updateEmailCampaign($shopId, $attributes)
{
    return EmailCampaign::query()->where('shop_id', $shopId)->update($attributes);
}
```

---

## 33. EmailCampaignController — Business Rules quan trọng

File: `app/Http/Controllers/API/EmailCampaignController.php`

### 33.1 getCampaignStatuses — Response format

```json
{
    "review-request":  true,
    "review-reminder": false,
    "reward":          true,
    "reward-reminder": false
}
```

### 33.2 updateReviewRequestSetting — Cascade disable

Khi tắt Review Request (`status = false`), **tự động tắt** tất cả children:
```php
if (!$status) {
    $emailCampaignService->updateEmailCampaignDetail($shop->id, 'review-reminder', ['status' => false]);
    $emailCampaignService->updateEmailCampaignDetail($shop->id, 'reward', ['status' => false]);
    $emailCampaignService->updateEmailCampaignDetail($shop->id, 'reward-reminder', ['status' => false]);
}
// Đồng thời cập nhật EmailCampaign.status cho tương thích legacy
$emailCampaignService->updateEmailCampaign($shop->id, ['status' => $status]);
```

> **Quan trọng:** Khi tắt Review Request → cả Reminder, Reward, Reward Reminder đều bị tắt. Nhưng khi bật lại Review Request, các children KHÔNG tự động bật.

### 33.3 updateRewardSetting — Cascade disable + CouponSetting

```php
// Tắt Reward → tắt Reward Reminder
if (!$status) {
    $emailCampaignService->updateEmailCampaignDetail($shop->id, 'reward-reminder', ['status' => false]);
}

// Update CouponSetting (default: 10%, no expiry)
$couponSettingService->updateOrCreateCouponSetting($shop->id, 'email', [
    'discount_value' => $request->input('discountValue', 10),
    'discount_type'  => $request->input('discountType', 'percentage'),
    'expires_after'  => $request->input('expiresAfter', 0),
]);

// Update review_settings.enable_email_coupon_waiting
$reviewSettingService->updateOrCreateReviewSetting($shop->id, [
    'enable_email_coupon_waiting' => $request->input('sendDiscountAfterApproval', false),
]);
```

**Response data của getRewardSetting:**
```json
{
    "status": true,
    "discountValue": 15,
    "discountType": "percentage",
    "expiresAfter": 0,
    "sendDiscountAfterApproval": false
}
```

### 33.4 Dependency validation khi enable

| Bật | Yêu cầu |
|-----|---------|
| Review Reminder | Review Request phải đang bật |
| Reward | Review Request phải đang bật |
| Reward Reminder | Reward phải đang bật |

```php
// Ví dụ: Reward Reminder
if ($status) {
    $rewardSetting = $emailCampaignService->getEmailCampaignDetail($shop->id, 'reward');
    if (!data_get($rewardSetting, 'status', false)) {
        return failedResult('REQUIRE_REWARD_ENABLED');
    }
}
```

### 33.5 Feature mapping cho updateEmailTemplateData

| type | Feature cần |
|------|------------|
| review-request | email_review_request |
| review-reminder | email_review_reminder |
| reward | email_coupon |
| reward-reminder | email_coupon |

### 33.6 sendTest — Guards đặc biệt

```php
// Guard 1: Giới hạn 15 test emails/ngày
$checkRun = $emailService->checkLimitSentTestInDay($shop);
if (!$checkRun) return failedResult("Limit sent test");

// Guard 2: Hardcode blacklist 2 shop cụ thể
if ($shop->shop == 'cmb43e-09.myshopify.com' || $shop->shop == 'jay6yz-zd.myshopify.com') {
    return failedResult("You're not allowed to send test email");
}

// Guard 3: Phải còn quota
if (!$shop->subscription()->ability()->canUse('email_volume')) {
    return failedResult("Reach out quota");
}
```

> **Lưu ý:** 2 shop bị hardcode block gửi test — có thể là shops spam hoặc test accounts nội bộ.

---

## 34. getEmailTemplateData vs getEmailTemplateDataTemp

```php
// getEmailTemplateData: trả raw JSON string
public function getEmailTemplateData(Request $request)
{
    return data_get($campaign, 'data');  // JSON string
}

// getEmailTemplateDataTemp: trả decoded với key 'root'
public function getEmailTemplateDataTemp(Request $request)
{
    return ['root' => json_decode(data_get($campaign, 'data'))];
}
```

`getEmailTemplateDataTemp` là endpoint **tạm thời** (suffix Temp) — được dùng để thử nghiệm format mới. Frontend dùng format này để load editor với `root` key.

---

## 35. Dependency Injection Map — Services binding

File: `app/Providers/AppServiceProvider.php` (hoặc ServiceProvider riêng)

```php
// Interface → Implementation
EmailService             → EmailServiceImp
EmailCampaignDetailService → EmailCampaignDetailServiceImp
EmailCampaignService     → EmailCampaignServiceImp (hoặc NewEmailCampaignServiceImp)
NewEmailCampaignService  → NewEmailCampaignServiceImp
SendEmailService         → SendEmailServiceImp
EmailSettingService      → EmailSettingServiceImp
CustomerCouponService    → CustomerCouponServiceImp
CouponSettingService     → CouponSettingServiceImp
```

**Các service dùng `app()` resolve thay vì constructor injection:**
- `app(CouponSettingService::class)` trong `EmailCampaignDetailServiceImp::createActivity()`
- `app(CustomerCouponService::class)` trong `AutomationEmailCampaign::createCoupon()`
- `app(EmailService::class)` trong `AutomationEmailCampaign::handle()`
- `app(EmailAttributeService::class)` trong nhiều nơi

---

## 36. Tóm tắt — Quy tắc business quan trọng nhất

1. **Activity hierarchy phải từ trên xuống:** Review Request → Reminder/Reward → Reward Reminder. Không thể enable child nếu parent disabled.

2. **Timing event chỉ ở Review Request:** Chỉ Review Request có `timing_event` (Fulfillment/Paid/Archived/Delivered). Các activity con không có timing_event.

3. **Reward timing luôn = 0:** Reward gửi ngay khi khách review. Không thể cấu hình delay.

4. **Coupon issue idempotent:** `hasTakeDiscount(email)` guard ngăn gửi 2 Reward cho cùng 1 review.

5. **shouldStopSending là lazyevaluation:** Reminder chỉ biết bị cancel khi đến lúc chạy — không cancel real-time khi review. Điều này có nghĩa nếu timing ngắn, Reminder có thể được gửi trước khi `rate` attribute được set.

6. **Resend bypass shouldStopSending:** `is_resend = true` cho phép gửi lại dù đã review. Dùng cho admin resend thủ công.

7. **A/B test là pure random:** Không có session tracking, không guarantee balanced split. Mỗi order là independent roll.

8. **Webhook registration tự động:** Mỗi khi thay đổi timing_event, Shopify webhook được tạo/update tự động.

9. **MySQL read replica cho order lookup:** `Order::on('mysql_readonly')` — có thể bị lag nếu order vừa được tạo.

10. **Email snapshot lưu HTML:** `email_data.content` là full HTML snapshot — cho phép preview email đã gửi ngay cả khi template sau này thay đổi.

---

*Tài liệu được tạo tự động bằng Claude Cowork — 2026-05-26*  
*Version 7.0 — Hoàn chỉnh với NewEmailCampaignService, EmailCampaignController đầy đủ, business rules tổng hợp*
