---
name: web-wanderer-buyer-reviews
description: Use Apify's web_wanderer TikTok reviews scraper to collect Mexico TikTok Shop product-card buyer reviews and turn them into good/neutral/bad review summaries and product-selection insights. Trigger when the user provides a TikTok Shop product/PDP URL or product ID and asks for buyer review analysis, purchase-experience evidence, or product demand gaps. Do not use for TikTok video comments, live-room chat, creator discovery, or video URLs.
---

# Web Wanderer Buyer Reviews

## Overview

Collect public buyer-review records attached to a TikTok Shop product in a specified market (default `MX`), preserve the distinction between product-card reviews and content comments, and summarize what buyers say was or was not satisfied. Use the Apify API with the existing local token; never print or save the token, signed dataset URL, reviewer identifiers, emails, or other personal information.

## First-call usage notice

On the first invocation in a conversation, show this short notice before running:

> 使用说明：请提供 TikTok Shop 商品页 URL（例如 `/mx/pdp/<商品ID>`）或商品 ID。此 Skill 抓取的是商品卡/商品详情页买家评价，不是短视频评论或直播聊天；首次先做小样本与字段门禁，空结果或市场错配会停止，不把“抓不到”解释成“没有差评”。

Then proceed if the user has already supplied a valid product URL/ID. Do not ask again for information already present.

## Input validation

- Accept `https://shop.tiktok.com/<market>/pdp/<id>`, `https://www.tiktok.com/view/product/<id>`, or a numeric product ID.
- Extract the numeric product ID; preserve the market from the URL when present, otherwise use `MX`.
- Before calling the Actor, normalize every accepted input to `https://www.tiktok.com/shop/<market>/pdp/<id>`. Do not pass a bare numeric ID to web_wanderer: Actor build `0.2.5` can fail its MX regional routing for numeric IDs even while the PDP visibly has reviews.
- Reject video URLs (`/video/`, `/@user/video/`, `/t/`) and explain that this Skill cannot fetch video comments.
- Reject live-room URLs and ordinary shop/category URLs that do not contain a product ID.
- Never infer a product ID from an unrelated video or creator page.

## Retrieval workflow

1. Read `APIFY_TOKEN` from the local project `.env` (or configured secret store). Never include it in logs, prompts, files, or final output.
2. Run `web_wanderer~tiktok-reviews-scraper` with:
   ```json
   {
     "region": "MX",
     "product_ids": ["https://www.tiktok.com/shop/mx/pdp/<id>"],
     "reviews_limit": 0,
     "reviews_filter": "all",
     "reviews_sort": "most_recent",
     "include_personal_information": false
   }
   ```
3. Set a per-run spend ceiling of `$0.50` unless the user explicitly sets a lower ceiling. One product is the default probe; do not scale to multiple products automatically.
4. Read the dataset through the run's authorized output. Keep only sanitized aggregates and review text needed for the requested analysis; do not persist raw reviewer IDs, names, signed URLs, or images.
5. Record run ID, status, captured time, returned count, market, and actual charge in a local research artifact only when the user asks for a saved result.

## Data gates

- Require: successful run, non-empty records, requested product ID match, and `review_country` matching the target market when present.
- Check field coverage for rating, text, SKU, verification, incentive flag, date, and country. Report missing fields instead of filling them in.
- If the result is empty, wrong-market, or mismatched, stop and report the evidence. Do not claim the product has no reviews.
- If an empty run used a bare numeric ID, retry once with the canonical regional product URL before applying the empty-result stop gate. Record whether the log says `Could not process product` or `No reviews found`; these are scraper failures/signals, not proof that the PDP has no reviews.
- Treat `total_reviews`/`rating_result` as the platform aggregate and returned rows as the captured review sample. If they differ, state both.
- Do not silently rerun an empty or failed Actor; rerun only when the user explicitly requests another attempt or supplies a corrected product URL.

## Analysis rules

- Star buckets: 4–5 = good, 3 = neutral/mixed, 1–2 = bad. Use text to explain the reason, but never override an explicit star rating without calling out the conflict.
- Mark blank-text reviews as rating-only. Mark contradictory or sarcastic text (for example a 1-star row ending with “I give it a 5”) as a data-quality issue and classify by the explicit star plus complaint substance.
- Separate three counts: all returned reviews, non-empty text reviews, and verified/incentivized reviews.
- Translate Spanish text for the user, but retain short original excerpts only when useful and safe. Paraphrase rather than expose personal information.
- Derive selection insights from fulfilled and unfulfilled needs: function, quality, size/capacity, leakage/durability, appearance, delivery, packaging, SKU differences, and expectation mismatch.
- Never call product-card buyer reviews “video comments,” “live comments,” or proof of video/live attribution. The Actor does not expose order-origin fields.

## Required output

Use this compact structure:

1. **测试状态**：商品、市场、Actor、run status、实际费用、返回条数。
2. **数据口径**：商品卡买家评价；是否能确认 MX；平台汇总 vs 返回行数。
3. **好评 / 中评 / 差评**：数量、占比、主题、代表性文本的中文概括。
4. **验证与质量信号**：verified purchase、incentivized、空文本、SKU、字段缺失、矛盾文本。
5. **选品判断**：已满足需求、未满足需求、风险、建议验证动作。
6. **边界**：明确没有覆盖短视频评论、直播聊天、达人/视频归因。
