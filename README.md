# Web Wanderer Buyer Reviews

使用 Apify 的 `web_wanderer~tiktok-reviews-scraper` 抓取 TikTok Shop 商品详情页买家评价，并输出好评、中评、差评和选品洞察。

## 适用场景

- 判断真实购买者哪些需求已经满足、哪些尚未满足
- 发现质量、尺寸、容量、漏液、包装、配送和 SKU 差异
- 分析验证购买、激励评价、空文本评价比例
- 为选品周报提供需求和风险证据

## 数据边界

本 Skill 只分析商品卡 / 商品详情页买家评价，不分析：

- 短视频评论区
- 直播间聊天
- 达人画像或视频归因
- 评价订单来自短视频、直播还是商品卡

TikTok 可能把不同入口产生的订单评价汇总到商品页，但 Actor 不返回下单来源字段，因此不能判断渠道归因。

## 输入格式

推荐输入 TikTok Shop 商品页 URL：

```text
https://shop.tiktok.com/mx/pdp/<商品ID>
```

也接受 `https://www.tiktok.com/view/product/<商品ID>` 或纯数字商品 ID。Skill 会先提取 ID，再统一转换为：

```text
https://www.tiktok.com/shop/mx/pdp/<商品ID>
```

不要直接把裸数字 ID 传给 Actor。`web_wanderer` build `0.2.5` 可能无法正确解析 MX 区域，即使商品页明确显示评价，也会返回 `Could not process product` 或空结果。

## 首次调用提示

首次使用时会先提示：

> 请提供 TikTok Shop 商品页 URL（例如 `/mx/pdp/<商品ID>`）或商品 ID。此 Skill 抓取的是商品卡买家评价，不是短视频评论或直播聊天；首次先做小样本与字段门禁，空结果不会被解释成“商品没有评价”。

## 推荐提示词

```text
请使用 web-wanderer-buyer-reviews 分析这个 TikTok Shop 商品页的全部公开买家评价。

商品链接：<粘贴商品页 URL>
市场：MX

请输出：
1. 测试状态、实际返回条数和实际费用
2. 商品平台评分与星级分布
3. 好评、中评、差评数量、占比和主题
4. 已满足的购买需求
5. 尚未满足的需求与负面问题
6. SKU 差异
7. 验证购买、激励评价、空文本评价比例
8. 选品建议、主要风险和下一步验证动作

明确区分商品卡买家评价、短视频评论和直播聊天。
```

## 空结果排查

1. 检查输入是否为商品页，而不是视频、直播、店铺或分类页。
2. 从输入提取数字商品 ID。
3. 转换为 `https://www.tiktok.com/shop/<market>/pdp/<id>` 后重试一次。
4. 区分日志：
   - `Could not process product`：Actor 处理或路由失败。
   - `No reviews found`：Actor 本次未找到评价。
5. 两种日志都不能证明商品没有评价；同时报告平台汇总数与实际返回行数。
6. 空结果、市场错配或商品 ID 不匹配时停止扩大抓取。

## 输出口径

- 4–5 星：好评
- 3 星：中评 / 混合评价
- 1–2 星：差评
- 平台 `total_reviews` / `rating_result` 与 Actor 返回行数分开报告
- 高激励评价占比需要降低总体评分可信度
- 不保存 API Token、签名 URL、评价者身份、邮箱或其他个人信息

默认市场：`MX`。默认单次费用上限：`$0.50`。默认先测试一个商品。
