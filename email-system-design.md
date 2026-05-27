# Thiết kế hệ thống Email Automation

> Tài liệu này mô tả kiến trúc, các thành phần, và luồng xây dựng một hệ thống email automation gắn với vòng đời đơn hàng và review — dựa trên codebase LAI Product Reviews.

---

## 1. Tổng quan & Mục tiêu

Hệ thống email automation cho phép merchant tự động gửi email đến khách hàng theo các mốc sự kiện:

| Loại email | Trigger | Mục tiêu |
|---|---|---|
| **Review Request** | Đơn hàng fulfilled / archived / delivered | Mời khách để lại review |
| **Review Reminder** | X ngày sau Review Request chưa có review | Nhắc lại |
| **Reward** | Khách submit review | Tặng coupon khuyến mãi |
| **Reward Reminder** | X ngày sau Reward chưa dùng coupon | Nhắc dùng coupon |

Ngoài 4 loại automation còn có email thủ công (manual send) và email test.

---

## 2. Kiến trúc tổng thể

```
┌─────────────────────────────────────────────────┐
│                   TRIGGERS                       │
│  Shopify Webhook  │  Review Submitted  │  Manual │
└────────┬──────────┴────────┬───────────┴────┬────┘
         │                   │                │
         ▼                   ▼                ▼
┌─────────────────────────────────────────────────┐
│              QUEUE JOBS (emails queue)           │
│  ProcessWebhookOrderJob  │  ReviewSideEffectJob  │
└────────────────┬────────────────────────────────┘
                 │ dispatch AutomationEmailCampaign
                 ▼
┌─────────────────────────────────────────────────┐
│           AutomationEmailCampaign (Job)          │
│  Gate checks → Template render → Mail::send()   │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│              AWS SES / SendGrid SMTP             │
└─────────────────────────────────────────────────┘
```

---

## 3. Các bảng dữ liệu cần thiết

### 3.1 Bảng cấu hình campaign

```
email_categories          email_types
─────────────────         ─────────────────────
id                        id
name                      name
slug (automation)         slug (request-email, reminder-email,
                               reward-email, rejection-email)


email_campaigns           email_campaign_details (= activities)
─────────────────         ──────────────────────────────────────
id                        id
shop_id                   shop_id            ← thêm 2025 để query nhanh
name                      email_campaign_id
status                    email_type_id
                          type               ← slug ngắn (review-request, ...)
                          name
                          email_data (json)  ← subject, previewText, ...
                          timing             ← số đơn vị (1, 3, 7)
                          timing_unit        ← Hours / Days
                          timing_event       ← Fulfillment/Paid/Archived/Delivered
                          has_coupon
                          reference_email_campaign_details_id  ← self FK
                          status             ← on/off
                          email_template_id
                          thumbnail
                          data (json)        ← cấu trúc block MJML
                          coupon_setting_id
```

### 3.2 Bảng template

```
email_templates
───────────────────────────────────────────
id
name                    (Review request / Reward / ...)
category_id             → email_categories
email_type_id           → email_types
required_sections       (json array)
exclude_sections        (json array)
default_sections        (json array)   ← cấu trúc section mặc định
options                 (json)         ← subject, timing, section_style, font
thumbnail
email_reference_type_id ← template nào được phép làm cha
```

### 3.3 Bảng email sections (blocks trong template)

```
email_sections
───────────────────────────────────────────
id
email_campaign_detail_id
type                    (header / text / productReview / discount / footer / button)
contents                (json)
style                   (json)
index                   (thứ tự hiển thị)
coupon_setting_id       ← chỉ section discount
```

### 3.4 Bảng emails (lịch sử gửi)

```
emails
───────────────────────────────────────────
id
shop_id
order_id                → orders
customer_id             → customers
email_campaign_detail_id → email_campaign_details
email_trigger_id        → emails (self FK: email cha trigger ra email này)
status                  ENUM: pending / sent / rated / opened / clicked / cancel / reach-out / skipped
scheduled_at            ← thời điểm sẽ gửi (delay)
email_data              (json) ← subject, content HTML snapshot, is_resend, coupon_id
category                automatic / manual
ab_test_campaign_id
```

### 3.5 Bảng tracking sự kiện trên email

```
email_attributes
───────────────────────────────────────────
id
attributeable_id        ← polymorphic: Email hoặc EmailCampaignDetail
attributeable_type
attribute_name          (pending / sent / opened / clicked / rate / coupon / use-coupon)
is_done                 boolean
```

### 3.6 Bảng coupon

```
coupon_settings
───────────────────────────────────────────
id
shop_id
source                  (email / review)
discount_value
discount_type           (percentage / fixed)
expires_after           (số ngày, 0 = không hết hạn)
code                    ← static code nếu có


customer_coupons
───────────────────────────────────────────
id
shop_id
customer_id
email_campaign_detail_id
discount_code
shopify_price_rule_id
shopify_discount_code_id
expires_at


couponables (pivot)
───────────────────────────────────────────
coupon_id
couponable_id           ← Email.id
couponable_type         ← "App\Models\Email"
```

### 3.7 Bảng blocklist email

```
email_settings
───────────────────────────────────────────
id
shop_id
email                   ← email address bị block
order_id                ← (optional) chỉ block cho order cụ thể
```

---

## 4. Sơ đồ quan hệ (ERD rút gọn)

```
Shop
 ├── EmailCampaign (1 shop : 1 automation campaign)
 │     └── EmailCampaignDetail x4 (activities)
 │           ├── EmailSection[] (blocks MJML)
 │           ├── CouponSetting (chỉ Reward)
 │           └── Email[] (lịch sử gửi)
 │                 ├── EmailAttribute[] (tracking)
 │                 └── CustomerCoupon[] (pivot)
 ├── CouponSetting (source=email, global)
 └── EmailSetting[] (blocklist)

EmailCampaignDetail ──self-ref──► EmailCampaignDetail
  (Reminder → Request)           (parent / reference)
  (Reward → Request)
  (Reward Reminder → Reward)

Email ──self-ref──► Email
  (Reminder.email_trigger_id → Request.email.id)
  (Reward.email_trigger_id → Request.email.id)
```

---

## 5. Các thành phần cốt lõi

### 5.1 Layer Controllers

```
EmailCampaignController     → CRUD settings (status, timing, timingEvent)
EmailActivityController     → CRUD activities, send test, preview
NewEmailSettingController   → Settings mới (dùng NewEmailCampaignService)
```

### 5.2 Layer Services

```
EmailCampaignDetailService
  ├── createDefaultCampaignDetails()  ← tạo 4 activities mặc định
  ├── createActivity()                ← tạo 1 activity (A/B test)
  ├── updateCampaignDetails()         ← cập nhật timing, status, ...
  ├── deleteActivity()                ← cascade delete
  ├── sendAutomationRewardEmail()     ← gửi reward sau khi có review
  └── filterAutomationEmailToSend()   ← random (A/B) hoặc first

EmailService
  ├── createEmailToSend()             ← tạo Email record + tính scheduled_at
  ├── send()                          ← render MJML → Mail::send()
  ├── resend()                        ← gửi lại batch emails
  ├── markAsRate()                    ← đánh dấu đã review
  ├── hasTakeDiscount()               ← kiểm tra đã dùng coupon chưa
  └── checkLimitSentTestInDay()       ← giới hạn 15 test/ngày

NewEmailCampaignService
  ├── getEmailCampaignDetail()        ← query theo shop_id + type
  ├── updateEmailCampaign()
  └── updateEmailCampaignDetail()

SendEmailService (Job dispatcher)
  ├── getDelayTime()                  ← tính thời gian delay
  └── automationSend()                ← dispatch AutomationEmailCampaign job
```

### 5.3 Layer Jobs

```
ProcessWebhookOrderJob          ← Nhận webhook Shopify order
  → Kiểm tra feature gate
  → Tạo Email (createEmailToSend)
  → Dispatch AutomationEmailCampaign với delay

AutomationEmailCampaign         ← Job thực sự gửi email
  → emailInterceptionHandler()  ← 4 gate checks
  → shouldStopSending()         ← check parent email đã rate/coupon chưa
  → EmailService::send()
  → Tạo Reminder/Reward nếu cần

ReviewSubmittedSideEffectJob    ← Sau khi khách submit review
  → markAsRate()                ← cancel reminders của request email
  → sendAutomationRewardEmail() ← gửi Reward (ngay hoặc chờ duyệt)
  → Klaviyo / Omnisend event
```

### 5.4 Template Engine

```
AutomationCampaignAdapter
  ├── prepareEmailBlocks()      ← inject data vào blocks (products, coupon, URLs)
  ├── collectEmailVariables()   ← {shopName, firstName, couponCode, ...}
  └── generateEmailContent()    ← build MJML string → render qua binary mjml

MjmlBuilder                     ← Build MJML string từ JSON blocks

AutomationCampaign (Mailable)   ← Cấu hình From/Reply-To
  └── Custom sender (SendGrid)  ← nếu shop có custom email sender
```

---

## 6. Vòng đời trạng thái Email

```
                    ┌──────────────────────┐
                    │     createEmailToSend │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼──────────────┐
              │                │               │
         email_volume      shouldBlock     email_volume
          quota OK?          = true?        exceeded?
              │                │               │
              ▼                ▼               ▼
          PENDING           SKIPPED         REACH-OUT
              │            (không gửi)    (không dispatch)
              │
              ▼ (AutomationEmailCampaign job chạy)
           SENT ──► OPENED ──► CLICKED
              │
              ├── Khách submit review ──► RATED
              │
              └── shouldStopSending() = true ──► CANCEL
                  (parent email đã RATED hoặc dùng coupon)
```

**Trạng thái tracking qua `email_attributes`:**

| attribute_name | Ý nghĩa |
|---|---|
| `pending` | Đang chờ gửi |
| `sent` | Đã gửi thành công |
| `opened` | Khách mở email (pixel tracking) |
| `clicked` | Khách click link trong email |
| `rate` | Khách đã submit review từ email này |
| `coupon` | Email này đã có coupon đính kèm |
| `use-coupon` | Khách đã dùng coupon |

---

## 7. Luồng xử lý chi tiết

### 7.1 Review Request — từ order webhook

```
Shopify order/fulfilled webhook
  → ProcessWebhookOrderJob (queue: webhook)
      │
      ├─ [check] feature 'email_review_request' enabled?
      ├─ [check] shop chưa uninstall?
      ├─ [check] email chưa có trong DB (tránh duplicate)?
      ├─ [check] isContinueSending() (tránh re-process)?
      │
      ├─ Tìm activity type 'review-request' đang active
      ├─ createEmailToSend() → Email record (status=PENDING, scheduled_at = now + timing)
      │
      └─ automationSend() → dispatch AutomationEmailCampaign.delay(scheduled_at)

AutomationEmailCampaign (chạy sau X ngày)
      │
      ├─ emailInterceptionHandler():
      │     [1] activity.status = false? → return
      │     [2] shop uninstalled? → return
      │     [3] email_volume quota exceeded? → status=REACH-OUT, return
      │     [4] feature email_review_request disabled? → return
      │
      ├─ shouldStopSending()? → check attribute 'rate' / 'use-coupon' → CANCEL
      │
      ├─ EmailService::send()
      │     → AutomationCampaignAdapter → MJML → HTML
      │     → Mail::to(customer)->send(AutomationCampaign mailable)
      │     → lưu HTML snapshot vào email_data
      │
      └─ Tạo Review Reminder (nếu reminder activity active)
            → createEmailToSend(reminder_activity, trigger=this_email.id)
            → dispatch với delay mới
```

### 7.2 Reward — sau khi khách submit review

```
Review submitted
  → ReviewSubmittedSideEffectJob
      │
      ├─ markAsRate(email_id)
      │     → thêm attribute 'rate' vào email → cancel reminders
      │
      ├─ sendImmediately = !enable_email_coupon_waiting
      │     true  → gửi Reward ngay
      │     false → chờ admin approve review rồi gửi
      │
      └─ sendAutomationRewardEmail()
            → shouldSendRewardEmail(): kiểm tra hasTakeDiscount()
            → findRewardEmailToSend(): lấy reward activity
            → createCustomerCoupon()
                  → tạo Shopify PriceRule + DiscountCode
                  → lưu vào customer_coupons
            → createEmailToSend(reward_activity)
            → automationSend() → dispatch AutomationEmailCampaign
```

### 7.3 Reminder — chạy sau email cha không được phản hồi

```
AutomationEmailCampaign (Review Request / Reward)
  │ (sau khi send xong)
  └─ [Tạo Reminder nếu activity active]
        → createEmailToSend(reminder_activity, trigger=parent_email.id)
        → dispatch với delay mới

AutomationEmailCampaign (Reminder)
  │
  ├─ shouldStopSending()
  │     → mappingAttributeName() → tìm 'rate' (review reminder) hoặc 'use-coupon' (reward reminder)
  │     → nếu có → CANCEL (không gửi)
  │
  └─ canSendReminder() → guard vô hạn loop
        isCouponReminder() → false (không tạo reminder của reminder)
        isReviewReminder() → false
```

---

## 8. Hệ thống Template (MJML)

### Kiến trúc render

```
EmailCampaignDetail.data (JSON blocks)
         │
         ▼
AutomationCampaignAdapter
  ├── prepareEmailBlocks()
  │     ├── Duplicate productReview block x số sản phẩm trong order
  │     ├── Inject product title, image, review URL (với tracking params)
  │     ├── Inject coupon code vào discount block
  │     └── Replace variables: {shopName}, {firstName}, {couponValue}, ...
  │
  └── generateEmailContent()
        → MjmlBuilder → build MJML string
        → exec(node_modules/.bin/mjml) → compile ra HTML
        → trả về HTML string
```

### Các loại section

| type | Mô tả |
|---|---|
| `header` | Logo / tên shop |
| `text` | Đoạn văn bản |
| `productReview` | Card sản phẩm + nút Review (duplicate per product) |
| `discount` | Coupon code block |
| `button` | CTA button |
| `footer` | Địa chỉ, phone, copyright |

### Tracking URL trong email

Review URL được gắn params:
```
https://shop.com/products/xxx?
  scm_review_mail=1
  &scm_mail={email_id}
  &scm_rating=5          ← pre-fill 5 sao
  &scm_name={customer_name}
```

---

## 9. Hệ thống Coupon

```
Khi tạo Reward activity
  → createDefaultCouponSetting(shopId, 'email')
  → coupon_settings record (discount 15%, expires 0)

Khi gửi Reward email
  → createCustomerCoupon()
        ├── Static code: dùng CouponSetting.code
        └── Auto-generated: format LAI-{base36(timestamp)}
              → verify uniqueness với Shopify API (max 3 tries)
        → Shopify: createPriceRule() (once_per_customer: true)
        → Shopify: createDiscountCode()
        → customer_coupons record

Khi khách dùng coupon
  → Shopify webhook → markCouponUsed()
  → email_attributes: attribute_name='use-coupon'
  → shouldStopSending() → cancel Reward Reminder
```

---

## 10. Gate checks & Business Rules

### Feature gates (subscription-based)

| Feature key | Ảnh hưởng |
|---|---|
| `email_review_request` | Bật/tắt Review Request |
| `email_review_reminder` | Bật/tắt Review Reminder |
| `email_coupon` | Bật/tắt Reward + Reward Reminder |
| `email_volume` | Quota số email/tháng |
| `review_request_upon_delivery` | Cho phép timing_event=Delivered |

### Dependency rules

```
Review Request OFF → tự động OFF: Reminder, Reward, Reward Reminder
Reward OFF         → tự động OFF: Reward Reminder
Reminder ON        → yêu cầu Review Request phải ON
Reward ON          → yêu cầu Review Request phải ON
Reward Reminder ON → yêu cầu Reward phải ON
```

### Email blocklist

`email_settings` cho phép merchant block email cụ thể (toàn bộ hoặc theo order).
`createEmailToSend()` check `hasBlocked()` → status=SKIPPED nếu bị block.

### Duplicate prevention

`ProcessWebhookOrderJob` kiểm tra:
```sql
SELECT id FROM emails
WHERE order_id = ?
AND email_campaign_detail_id IN (active request activities)
AND status NOT IN ('cancel', 'skipped', 'reach-out')
```
Nếu đã tồn tại → bỏ qua, không tạo mới.

---

## 11. Queue Architecture

```
Queue: webhook    → ProcessWebhookOrderJob (nhận webhook Shopify)
Queue: emails     → AutomationEmailCampaign (gửi email thực sự)
Queue: default    → ReviewSubmittedSideEffectJob, SendEmailTest
Queue: install    → AppInstalledJob (setup ban đầu)
```

**Tại sao tách queue `emails`?**  
Để có thể scale worker riêng cho việc gửi email, không bị block bởi các job khác. Cũng dễ monitor qua Horizon.

---

## 12. Custom Email Sender

Merchant có thể cấu hình custom sender (SMTP riêng qua SendGrid):

```
AutomationCampaign (Mailable)::build()
  │
  ├─ [Có custom sender]:
  │     → Tạo Swift_SmtpTransport với SendGrid credentials
  │     → Swap mailer tạm thời
  │     → From: shop.from_email
  │     → Reply-to: shop.customer_email
  │
  └─ [Không có / exception tiennm]:
        → Dùng AWS SES mặc định
        → From: MAIL_FROM_ADDRESS
        → Reply-to: shop.customer_email
```

---

## 13. A/B Testing

```
AbTestCampaign
  ├── active_activities (json array of activity IDs)
  ├── stopping_rule (duration / quantity / performance)
  ├── start_date, end_date
  └── winner_activity_percentage_to_stop

filterAutomationEmailToSend():
  ├── isAbTesting()? → true  → random pick (rand())
  └──              → false → first() (activity đầu tiên active)
```

---

## 14. Checklist xây dựng từ đầu

Để xây dựng hệ thống tương tự, cần thực hiện theo thứ tự:

### Phase 1 — Database & Models

- [ ] Tạo migrations: `email_categories`, `email_types`
- [ ] Tạo migrations: `email_campaigns`, `email_campaign_details`
- [ ] Tạo migrations: `email_templates`
- [ ] Tạo migrations: `email_sections`
- [ ] Tạo migrations: `emails`, `email_attributes`
- [ ] Tạo migrations: `coupon_settings`, `customer_coupons`, `couponables`
- [ ] Tạo migrations: `email_settings` (blocklist)
- [ ] Tạo Eloquent models với relationships đầy đủ

### Phase 2 — Seed data

- [ ] `AutomationTemplateSeeder` — seed 4 email templates (Request, Reminder, Reward, Reward Reminder) từ `automation.json`
- [ ] Seed `email_types`: request-email, reminder-email, reward-email, rejection-email, promotion-email
- [ ] Tạo file `app/Templates/activities.json` chứa block MJML mặc định cho từng loại

### Phase 3 — Template Engine

- [ ] Cài `mjml` binary qua npm
- [ ] Build `MjmlBuilder` — convert JSON blocks sang MJML string
- [ ] Build `AutomationCampaignAdapter` — inject data vào blocks, compile HTML
- [ ] Build `AutomationCampaign` Mailable (hỗ trợ custom SMTP sender)

### Phase 4 — Core Services

- [ ] `EmailCampaignService` — tạo/lấy automation campaign
- [ ] `EmailCampaignDetailService` — CRUD activities, `createDefaultCampaignDetails()`
- [ ] `EmailService` — `createEmailToSend()`, `send()`, tracking methods
- [ ] `SendEmailService` — `getDelayTime()`, `automationSend()` (dispatch job với delay)
- [ ] `CouponSettingService` — tạo default coupon, update
- [ ] `CustomerCouponService` — tạo Shopify price rule + discount code

### Phase 5 — Jobs

- [ ] `ProcessWebhookOrderJob` — nhận order webhook, tạo Email, dispatch send job
- [ ] `AutomationEmailCampaign` — gate checks, render, send, tạo reminders
- [ ] `ReviewSubmittedSideEffectJob` — mark as rate, gửi reward

### Phase 6 — Setup hook vào Install

- [ ] `AppInstalledJob` gọi `createDefaultCampaignDetails(shopId)`
- [ ] `AppLoggedInListener` gọi `createDefaultCampaignDetails(shopId)` (safety net)
- [ ] Đăng ký Shopify webhook `orders/fulfilled` khi shop enable Review Request

### Phase 7 — API & Admin UI

- [ ] `EmailCampaignController` — settings (status, timing, timing_event)
- [ ] `EmailActivityController` — CRUD activities, send test, preview
- [ ] Routes đầy đủ trong `routes/web.php` hoặc `routes/api.php`
- [ ] Frontend: form cấu hình campaign, email editor (MJML blocks)

### Phase 8 — Monitoring & Operations

- [ ] Laravel Horizon cho queue monitoring
- [ ] Log lỗi gửi email về Mattermost / Slack
- [ ] Dashboard thống kê: sent, opened, clicked, rated per campaign
- [ ] Artisan commands: `send:review-request`, `send:reminder-email-manual`, `sync:default-template`

---

## 15. Các điểm cần lưu ý khi implement

**Idempotency:** `createDefaultCampaignDetails()` phải check trước khi tạo — tránh duplicate khi gọi nhiều lần.

**Delay job:** Email được dispatch với `->delay($scheduled_at)` chứ không phải cron. Queue worker phải luôn chạy. Nếu worker restart thì job vẫn nằm trong Redis, không mất.

**MJML binary:** Phụ thuộc vào `node_modules/.bin/mjml`. Nếu deploy mới cần `npm install` trước khi gửi email được.

**Shopify API rate limit:** Tạo coupon gọi 2 API calls (PriceRule + DiscountCode). Với nhiều email đồng thời sẽ bị rate limit → cần retry logic hoặc queue riêng.

**Status REACH-OUT vs CANCEL:** REACH-OUT là tạm thời (quota hết tháng này → tháng sau có thể retry). CANCEL là vĩnh viễn (đã review rồi, không gửi lại).

**Self-referencing Email:** `email_trigger_id` liên kết email con (Reminder/Reward) về email cha (Request). Khi cha bị cancel → các con cũng nên cancel (xử lý trong `shouldStopSending`).

**Custom sender bypass:** Account `tiennm` hardcode bypass custom sender — đây là account test nội bộ, luôn dùng AWS SES.

---

*Tài liệu được tạo bởi Claude Cowork — 2026-05-27*
