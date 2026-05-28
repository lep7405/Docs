# Tất cả các trường hợp gửi email — Bài toán & Giải pháp kỹ thuật

> Tài liệu này liệt kê đầy đủ các case gửi email trong hệ thống, mô tả bài toán cần giải quyết và cách giải quyết bằng kỹ thuật.

---

## Tổng quan các case

| # | Case | Trigger | Queue |
|---|------|---------|-------|
| 1 | Review Request tự động | Order webhook | emails |
| 2 | Review Reminder tự động | X ngày sau Request chưa review | emails |
| 3 | Reward tự động (gửi ngay) | Review submit | emails |
| 4 | Reward chờ duyệt (coupon_waiting) | Admin approve review | emails |
| 5 | Reward Reminder tự động | X ngày sau Reward chưa dùng coupon | emails |
| 6 | Rejection review email | Admin reject review | sync (Mail::send trực tiếp) |
| 7 | Coupon cho photo review | Review submit có ảnh | sync |
| 8 | Manual send (artisan) | Ops chạy tay | emails |
| 9 | Manual remind (artisan) | Ops chạy tay | emails |
| 10 | Resend batch | Ops/Admin chọn lại emails | emails |
| 11 | Send test | Admin nhấn "Send test" trên UI | default |

---

## Case 1 — Review Request tự động

### Bài toán

Sau khi khách đặt hàng và đơn hàng được fulfil/archive/deliver, hệ thống cần tự động gửi email mời khách để lại review. Thách thức:
- Không gửi trùng nếu webhook gửi đến nhiều lần
- Cần delay X ngày sau sự kiện, không gửi ngay
- Shop có thể chọn trigger khác nhau: lúc fulfil, archive, hay delivery
- Nếu shop chưa bật tính năng hoặc hết quota thì không gửi
- Nếu địa chỉ email bị block thì bỏ qua

### Giải pháp kỹ thuật

```
Shopify gửi webhook → routes/webhook.php
  → ProcessWebhookOrderJob (queue: webhook)

Trong job:
  1. [Guard] feature 'email_review_request' có enabled không?
     → Không → return

  2. [Guard] Shop đã uninstall chưa?
     → Rồi → return

  3. [Dedup] Query emails table:
     SELECT * FROM emails
     WHERE order_id = ?
       AND email_campaign_detail_id IN (active review-request activities)
       AND status NOT IN ('cancel','skipped','reach-out')
     → Tìm thấy → isContinueSending() = false → return

  4. [Blocklist] emailSettingService->hasBlocked(shopId, email, orderId)
     → Bị block → status = SKIPPED → return

  5. [Quota] canUse('email_volume')
     → Hết → status = REACH-OUT, scheduled_at = null

  6. Tính scheduled_at = now() + activity.timing (days/hours)

  7. INSERT emails(status=PENDING, scheduled_at=X, category=automatic)
     INSERT email_attributes(attribute_name='pending')

  8. dispatch(AutomationEmailCampaign, emailId)
       .onQueue('emails')
       .delay(scheduled_at)       ← Job nằm trong Redis, chờ đến giờ

Khi job chạy (sau X ngày):
  → emailInterceptionHandler()   ← kiểm tra lại tất cả gate
  → shouldStopSending()          ← nếu đã rate → CANCEL
  → EmailService::send()         ← render MJML → Mail::send() → AWS SES
```

**Key files:**
- `app/Jobs/ProcessWebhookOrderJob.php`
- `app/Jobs/AutomationEmailCampaign.php`
- `app/Services/Email/EmailServiceImp.php::createEmailToSend()`
- `routes/webhook.php`

---

## Case 2 — Review Reminder tự động

### Bài toán

Sau khi gửi Review Request, nếu khách không để lại review sau X ngày thì gửi email nhắc lại. Thách thức:
- Phải biết "khách đã review chưa" tại thời điểm job chạy (không phải lúc tạo job)
- Không được tạo reminder của reminder (vòng lặp vô hạn)
- Reminder phải liên kết với email cha để biết cần cancel khi nào

### Giải pháp kỹ thuật

```
AutomationEmailCampaign (Review Request) gửi xong
  → canSendReminder() = true?
       ├── activity.children().count() > 0?  (có reminder activity?)
       ├── email.status không phải CANCEL/REACH-OUT/SKIPPED?
       ├── canUse('email_volume')?
       └── activity KHÔNG phải isCouponReminder() và KHÔNG phải isReviewReminder()?
  → true → fire event AutomationEmailSend

AutomationEmailSend listener
  → createEmailToSend(
       activity = reminder_activity,
       emailTriggerId = request_email.id   ← liên kết về cha
     )
  → dispatch(AutomationEmailCampaign).delay(reminder_activity.timing)

Khi Reminder job chạy:
  → shouldStopSending():
       parent = email.parent (Review Request email)
       attributeName = mappingAttributeName() → 'rate'
       hasDone('rate', parent)?
         → YES → CANCEL (khách đã review rồi)
         → NO  → tiếp tục gửi
```

**Key point:** `email_trigger_id` là cầu nối giữa Reminder và Request. Khi request email có attribute `rate` → reminder tự cancel.

**Key files:**
- `app/Jobs/AutomationEmailCampaign.php::canSendReminder()`
- `app/Jobs/AutomationEmailCampaign.php::shouldStopSending()`
- `app/Listeners/AutomationEmailSent.php`
- `app/Services/Email/EmailServiceImp.php::mappingAttributeName()`

---

## Case 3 — Reward tự động (gửi ngay sau review)

### Bài toán

Ngay khi khách submit review, gửi email cảm ơn kèm coupon giảm giá. Thách thức:
- Phải tạo Shopify coupon (price rule + discount code) trước khi gửi
- Mỗi coupon là unique per customer, once-per-customer
- Không gửi nếu khách đã từng nhận coupon từ email này
- Coupon code phải unique trong Shopify store (tối đa 3 lần retry)

### Giải pháp kỹ thuật

```
Review submit → ReviewSubmittedSideEffectJob

  1. sendImmediately = !reviewSetting->enable_email_coupon_waiting
     → true  → gửi ngay (Case 3)
     → false → chờ admin approve (Case 4)

  2. [Guard] shouldSendRewardEmail():
       ├── email.status không phải CANCEL/REACH-OUT?
       ├── hasTakeDiscount(email) = false?        ← chưa có coupon
       └── customer tồn tại?

  3. findRewardEmailToSend():
       → activity.rewards (children có email_type_id=3, status=true)
       → nếu A/B testing: rand() pick
       → nếu không: first()

  4. createEmailToSend(reward_activity, emailTriggerId=request_email.id)
     → INSERT emails

  5. dispatch(AutomationEmailCampaign).delay(scheduled_at)

Trong AutomationEmailCampaign:
  → createCoupon():
       customerCouponService->createCustomerCoupon(shop, 'email')
         ├── Static code: dùng CouponSetting.code
         └── Auto: format LAI-{base36(timestamp)}
               → Shopify: createPriceRule (once_per_customer=true)
               → Shopify: createDiscountCode
               → 3 lần retry nếu code bị trùng
               → INSERT customer_coupons
  → send() với coupon đính kèm
  → attachCouponToEmails(): email.coupons().save(coupon)
```

**Key files:**
- `app/Jobs/ReviewSubmittedSideEffectJob.php`
- `app/Services/EmailCampaignDetail/EmailCampaignDetailServiceImp.php::sendAutomationRewardEmail()`
- `app/Services/CustomerCoupon/CustomerCouponServiceImp.php::createCustomerCoupon()`
- `app/Jobs/AutomationEmailCampaign.php::createCoupon()`

---

## Case 4 — Reward chờ duyệt (enable_email_coupon_waiting = true)

### Bài toán

Merchant muốn kiểm tra review trước khi phát coupon — chỉ gửi Reward sau khi admin approve review. Thách thức:
- Review lúc submit có status = pending
- Phải trigger gửi Reward vào đúng thời điểm admin approve, không phải lúc submit
- Nếu admin reject thay vì approve → gửi email rejection thay thế

### Giải pháp kỹ thuật

```
[Khi review submit]
ReviewSubmittedSideEffectJob:
  → sendImmediately = false → KHÔNG gửi Reward

[Khi admin approve review]
ReviewServiceImp::approveReviews()
  → notificationService->notifyCouponAfterApprovedReview(shop, review)
      ├── [Guard] feature 'discount_write_review' enabled?
      ├── [Guard] review chưa có coupon (hasSentCoupon)?
      ├── [Guard] review.status == pending?
      └── availableEmailCouponWaiting && review.reviewEmail?
            → emailCampaignDetailService->sendAutomationRewardEmail(
                shopId,
                requestReviewEmail.id,
                review.email
              )
            → Luồng tiếp theo giống Case 3

[Khi admin reject review]
  → Xem Case 6
```

**Key files:**
- `app/Services/Review/ReviewServiceImp.php::approveReviews()`
- `app/Services/Notification/NotificationServiceImp.php::notifyCouponAfterApprovedReview()`
- `config/secom.php` → `release_coupon_waiting` (ngày feature được bật)

---

## Case 5 — Reward Reminder tự động

### Bài toán

Sau khi gửi Reward, nếu khách chưa dùng coupon sau X ngày thì nhắc lại. Thách thức:
- Phải biết khách "đã dùng coupon chưa" — không phải "đã mở email chưa"
- Coupon được tạo ở Reward, Reminder phải tái sử dụng cùng coupon đó
- Không gửi reminder của reminder (vòng lặp vô hạn)

### Giải pháp kỹ thuật

```
AutomationEmailCampaign (Reward) gửi xong
  → canSendReminder() → true
  → fire AutomationEmailSend event
  → Tạo Reward Reminder email (email_trigger_id = reward_email.id)
  → dispatch với delay

Khi Reminder job chạy:
  → shouldStopSending():
       parent = reward_email
       attributeName = mappingAttributeName() → 'use-coupon'
       hasDone('use-coupon', parent)?
         → YES → CANCEL (đã dùng coupon rồi)
         → NO  → gửi tiếp

  → resolveCoupon(): lấy coupon từ email_data.coupon_id
    (coupon đã được tạo từ Reward, không tạo mới)

  → send() với coupon cũ đính kèm
```

**Khi khách dùng coupon:**
```
Shopify webhook → mark coupon used
  → email_attributes: attribute_name='use-coupon', is_done=true
  → lần sau shouldStopSending() sẽ = true → Reminder bị CANCEL
```

**Key files:**
- `app/Jobs/AutomationEmailCampaign.php::shouldStopSending()`
- `app/Services/Email/EmailServiceImp.php::mappingAttributeName()`
- `app/Jobs/AutomationEmailCampaign.php::resolveCoupon()`

---

## Case 6 — Rejection review email

### Bài toán

Khi admin reject một review (thường là review để nhận coupon photo/email reward), cần notify khách biết review bị từ chối và cho phép submit lại. Thách thức:
- Email này không qua queue — gửi đồng bộ ngay lập tức
- Không lưu vào bảng `emails` (không có tracking)
- Template riêng: `app/Templates/reject-review.json`
- Link trong email phải chứa tracking params để khách submit review lại

### Giải pháp kỹ thuật

```
ReviewServiceImp::rejectReviews(shop, reviewIds, reason)
  → Điều kiện gửi rejection email:
       ├── feature 'coupon_waiting' enabled?
       ├── review có ảnh (files > 0) → availableReviewCouponWaiting?
       └── review có reviewEmail (từ email campaign) → availableEmailCouponWaiting?

  → notificationService->notifyRejectReviewForCustomer(review, shop, reason)
       ├── [Guard] review.created_at >= release_coupon_waiting date?
       ├── [Guard] review.status == pending?
       └── [Guard] review.source == 'submit'?

  → Load template: app/Templates/reject-review.json
  → Inject vào blocks:
       - header/shopName: tên shop
       - body: lý do reject (reason)
       - submit_button.href: URL product + tracking params
         (scm_review_mail=1, scm_mail, scm_rating=5, scm_name)
       - copyright: © year shopName

  → AutomationCampaignAdapter(activity=null, pageBlock)
     → generateEmailContent() → HTML
  → Mail::to(review.email)->send(AutomationCampaign) ← SYNC, không qua queue
```

**Khác biệt với các case khác:**
- `activity = null` → `prepareEmailBlocks()` bị skip
- Không có `email` record → không tracking
- Gửi đồng bộ trong request → có thể slow nếu SES chậm

**Key files:**
- `app/Services/Notification/NotificationServiceImp.php::notifyRejectReviewForCustomer()`
- `app/Services/Review/ReviewServiceImp.php::rejectReviews()`
- `app/Templates/reject-review.json`

---

## Case 7 — Coupon cho photo review (không qua email campaign)

### Bài toán

Merchant bật tính năng tặng coupon cho review có ảnh, nhưng KHÔNG bật email campaign reward. Khách submit review có ảnh → nhận coupon qua email ngay lập tức, không qua pipeline thông thường.

### Giải pháp kỹ thuật

```
notifyCouponAfterApprovedReview():
  → availableReviewCouponWaiting && review.files.count() > 0?
      → customerCouponService->createCustomerCoupon(shop, 'review')
         ← source = 'review' (khác với 'email' ở Case 3)
      → notifyCouponForSubmitReview(review, coupon, shop)
           → Lấy reward activity đầu tiên của shop
           → Render với AutomationCampaignAdapter (activity = reward)
           → review.coupons().save(coupon)  ← gắn coupon vào review
           → Mail::to(review.email)->send() ← SYNC
```

**Khác với Reward email (Case 3):**
- Không tạo Email record
- Coupon source = `review`, không phải `email`
- Gửi đồng bộ, không delay

**Key files:**
- `app/Services/Notification/NotificationServiceImp.php::notifyCouponForSubmitReview()`
- `app/Services/CustomerCoupon/CustomerCouponServiceImp.php::createCustomerCoupon(source='review')`

---

## Case 8 — Manual send (Artisan)

### Bài toán

Ops cần gửi Review Request cho một loạt đơn hàng cũ chưa được gửi (ví dụ: shop mới cài app, chưa có email nào). Thách thức:
- Có thể cần gửi hàng trăm/nghìn đơn
- Cần chọn range order number, exclude một số đơn cụ thể
- Cần tránh gửi trùng với email đã tồn tại
- Có mode `--force`: tự động lọc những đơn chưa có email nào

### Giải pháp kỹ thuật

```bash
php artisan email:request-review \
  --shopId=123 \
  --fromOrder=1000 \
  --toOrder=2000 \
  --excludeOrder=1050,1075 \
  --delayTime="2026-06-01 08:00:00"

# Hoặc force mode (tự tìm đơn chưa có email):
php artisan email:request-review --shopId=123 --force
```

```
SendReviewRequest::handle()
  → Nếu không có --force:
       → Query orders WHERE number BETWEEN fromOrder AND toOrder
       → Loại bỏ excludeOrder
  → Nếu có --force:
       → Lấy tất cả orders của shop
       → Filter: chỉ lấy đơn KHÔNG có email nào (cả automatic lẫn manual)

  → emailService->sendManually(shopId, orders, scheduledAt)
       → Lấy activity review-request đang active
       → Với mỗi order:
            → findOrCreate customer
            → createEmailToSend(category=MANUAL, scheduledAt=delayTime)
            → automationSend() → dispatch AutomationEmailCampaign.delay(scheduledAt)
```

**Khác với auto send:** `category = 'manual'` — dùng để phân biệt trong báo cáo và tránh `--force` lần sau gửi trùng.

**Key files:**
- `app/Console/Commands/SendReviewRequest.php`
- `app/Services/Email/EmailServiceImp.php::sendManually()`

---

## Case 9 — Manual remind (Artisan)

### Bài toán

Ops cần gửi Reminder ngay lập tức cho một số email ID cụ thể (không cần đợi timing). Dùng khi reminder tự động bị fail hoặc bị CANCEL nhầm.

### Giải pháp kỹ thuật

```bash
php artisan email:remind \
  --shop=myshop \
  --emailIds=1001,1002,1003
```

```
SendReminderEmailManual::handle()
  → Load emails theo IDs
  → Với mỗi email:
       → Kiểm tra campaign có phải review-request không
         (chỉ remind từ request, không remind từ reward)
       → Lấy reminder activities: campaign.reminders
       → filterAutomationEmailToSend() → random (A/B) hoặc first
       → automationSend(shop, customerId, activityId, ...) → dispatch ngay (now())
```

**Lưu ý:** Command này có bug nhỏ — `automationSend()` được gọi với signature sai (thừa params). Cần fix trước khi dùng production.

**Key files:**
- `app/Console/Commands/SendReminderEmailManual.php`

---

## Case 10 — Resend batch

### Bài toán

Admin muốn gửi lại cho một batch emails đã bị REACH-OUT (hết quota tháng trước) hoặc SKIPPED. Khách nhận lại email nhưng vẫn dùng coupon cũ (không tạo coupon mới).

### Giải pháp kỹ thuật

```
EmailServiceImp::resend(shop, emails, scheduledAt)
  → Với mỗi email:
       → getCouponFromEmail(email):
            ├── isRewardEmail? → email.coupons().first()
            └── có parent?    → parent.coupons().first()
       → createEmailToSend(... , emailTriggerId=email.email_trigger_id)
       → newEmail.update(['email_data' => ['is_resend' => true, 'coupon_id' => coupon.id]])
       → automationSend() → dispatch

Trong AutomationEmailCampaign:
  → resolveIsResend() = true (đọc email_data.is_resend)
  → resolveIsResend() = true → SKIP shouldStopSending() check
    (vì dù đã rate, đã dùng coupon, ops vẫn muốn gửi lại)
  → resolveCoupon() → đọc email_data.coupon_id → load CustomerCoupon cũ
    (không tạo coupon mới, tái sử dụng)
  → send() với coupon cũ
```

**Key logic:** `is_resend = true` bypass `shouldStopSending()` — cho phép gửi dù email cha đã có attribute `rate` hay `use-coupon`.

**Key files:**
- `app/Services/Email/EmailServiceImp.php::resend()`
- `app/Jobs/AutomationEmailCampaign.php::resolveIsResend()`
- `app/Jobs/AutomationEmailCampaign.php::resolveCoupon()`

---

## Case 11 — Send test email

### Bài toán

Admin muốn preview email trước khi bật campaign, gửi thử đến địa chỉ email tùy ý. Thách thức:
- Không có order thật, không có customer thật
- Dùng dữ liệu giả (John Doe, 3 sản phẩm đầu của shop)
- Không được gây side effect (không tạo coupon thật, không tính vào quota)
- Giới hạn 15 lần/ngày/shop

### Giải pháp kỹ thuật

```
POST /api/email-campaigns/{id}/activities/{id}/send-test
  → EmailActivityController::sendTest()

  [Guard] checkLimitSentTestInDay(shop):
    → shopSetting 'totalSentEmailTest' = "2026-05-28_5"
    → Nếu cùng ngày và count <= 15 → cho qua, tăng counter
    → Nếu > 15 → failedResult("Limit sent test")

  [Guard] shop.subscription().canUse('email_volume')

  → emailService->sendTest(shop, campaignId, activityId, email)
       → record usage 'email_volume'  ← GHI VÀO QUOTA (khác với resend)
       → tạo Customer giả: { first_name: 'John', last_name: 'Doe', email: 'example@gmail.com' }
       → AutomationCampaignAdapter(..., isTest=true)
            → prepareTestSend():
                 Review Request: lấy 3 products đầu của shop (không cần order)
                 Reward: tạo CustomerCoupon giả { coupon: "LAI-DISCOUNT" }
       → generateEmailContent() → HTML
       → dispatch(SendEmailTest, html) .onQueue('default')  ← queue riêng
```

**Blacklist cứng:** Shops `cmb43e-09.myshopify.com` và `jay6yz-zd.myshopify.com` không được gửi test.

**Key files:**
- `app/Http/Controllers/API/EmailActivityController.php::sendTest()`
- `app/Services/Email/EmailServiceImp.php::sendTest()`
- `app/Jobs/SendEmailTest.php`

---

## So sánh tổng hợp tất cả cases

| Case | Có Email record? | Qua queue? | Tính quota? | Coupon? | is_resend bypass? |
|------|-----------------|------------|-------------|---------|-------------------|
| 1. Review Request | ✅ | ✅ emails | ✅ | ❌ | ❌ |
| 2. Review Reminder | ✅ | ✅ emails | ✅ | ❌ | ❌ |
| 3. Reward (ngay) | ✅ | ✅ emails | ✅ | ✅ tạo mới | ❌ |
| 4. Reward (duyệt) | ✅ | ✅ emails | ✅ | ✅ tạo mới | ❌ |
| 5. Reward Reminder | ✅ | ✅ emails | ✅ | ✅ tái sử dụng | ❌ |
| 6. Rejection email | ❌ | ❌ sync | ❌ | ❌ | N/A |
| 7. Coupon photo | ❌ | ❌ sync | ❌ | ✅ tạo mới | N/A |
| 8. Manual send | ✅ category=manual | ✅ emails | ✅ | ❌ | ❌ |
| 9. Manual remind | ✅ | ✅ emails | ✅ | ❌ | ❌ |
| 10. Resend | ✅ is_resend=true | ✅ emails | ✅ | ✅ tái sử dụng | ✅ |
| 11. Send test | ❌ | ✅ default | ✅ | ✅ fake | N/A |

---

## Các cơ chế bảo vệ chung

### 1. Idempotency — Tránh gửi trùng

```
emails table:
  WHERE order_id = ? AND email_campaign_detail_id IN (?) AND status NOT IN ('cancel','skipped','reach-out')
  → Đã tồn tại → bỏ qua
```

### 2. Gate check khi job thực sự chạy (double check)

`emailInterceptionHandler()` kiểm tra lại tại thời điểm job execute (không chỉ lúc create):
```
1. email.status ∈ {SKIPPED, CANCEL, REACH-OUT} → return
2. shop.uninstalled() → CANCEL
3. email_volume quota hết → REACH-OUT
4. Reward nhưng feature email_coupon tắt → CANCEL
```

### 3. Stop-sending logic

`shouldStopSending()` dựa trên `email_attributes` của email cha:
```
Review Reminder → check 'rate' attribute trên Review Request email
Reward Reminder → check 'use-coupon' attribute trên Reward email
```

### 4. Quota tracking

```
shop.subscriptionUsage().record('email_volume')
  → tăng counter usage trong subscription
  → canUse('email_volume') check monthly limit
  → Khi hết: REACH-OUT (không gửi, nhưng có thể retry tháng sau)
```

### 5. Coupon uniqueness

```
generateUniqueCouponCode():
  → Tối đa 3 lần thử
  → Format: LAI-{base36(timestamp)}
  → Verify với Shopify API trước khi dùng
  → Shopify price rule: once_per_customer = true
```

---

## Decision tree tổng quát khi có sự kiện mới

```
Sự kiện xảy ra (webhook/review submit)
          │
          ▼
  Feature gate OK?
    NO  → bỏ qua
    YES │
        ▼
  Shop uninstalled?
    YES → bỏ qua
    NO  │
        ▼
  Email đã tồn tại? (dedup check)
    YES → bỏ qua
    NO  │
        ▼
  Email bị block?
    YES → status = SKIPPED → bỏ qua
    NO  │
        ▼
  Còn quota email_volume?
    NO  → status = REACH-OUT → lưu DB, không dispatch
    YES │
        ▼
  Tính scheduled_at = now() + timing
  INSERT emails(status=PENDING)
  dispatch(AutomationEmailCampaign).delay(scheduled_at)
          │
          │ [sau X ngày/giờ]
          ▼
  Double-check tất cả gates lại
          │
          ▼
  shouldStopSending()?  ← kiểm tra parent email attributes
    YES → CANCEL
    NO  │
        ▼
  Tạo coupon nếu cần (Reward only)
          │
          ▼
  Render MJML → HTML → Mail::send()
          │
          ▼
  Lưu HTML snapshot vào email_data
  Dispatch Reminder nếu cần
```

---

*Tài liệu được tạo bởi Claude Cowork — 2026-05-27*
