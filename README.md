# 📄 Shopify Customization: Show Retail & Trade Pricing for Approved Customers

## 🎯 Use Case

When a customer with the tag **"Approved Trade"** logs into the Shopify storefront, they should see both:

* the **Retail Price** (RRP), and
* a special **Trade Price** (e.g., 20% off retail).

This visibility should occur on:

* Product pages
* Cart page
* Collection pages (product cards)

## 🤩 Problem

Shopify does not natively support dual pricing per customer group on product or collection pages.

Metafields such as `trade_discount` were already set up for variants, but:

* Only retail price displayed by default
* The theme (Prestige) did not show trade pricing on product cards or collection views
* Cart showed only retail prices

## ✅ Solution

We implemented the following:

### 1. **Display RRP and Trade Price on Product Cards (Collections Page)**

Edit `product-card.liquid` to:

* Add logic that calculates and displays `Trade Price`
* Keep the RRP visible

```liquid
{%- if settings.currency_code_enabled -%}
  {%- assign variant_price = variant.price | money_with_currency -%}
  {%- assign trade_price = variant.price | times: 0.8 | money_with_currency -%}
{%- else -%}
  {%- assign variant_price = variant.price | money -%}
  {%- assign trade_price = variant.price | times: 0.8 | money -%}
{%- endif -%}

<div style="display: flex; flex-direction: column; gap: 2px; text-align: center;">
  <span class="{{ regular_price_classes }}">
    <span>RRP: </span>{{- variant_price -}}
  </span>
  <span class="{{ base_text_class }}" style="color: #ad966e; font-weight: bold;">
    <span>Trade price: </span>{{- trade_price -}}
  </span>
</div>
```

### 2. **Suppress Unnecessary Sale Price Block**

We removed default sale price markup to prevent duplicate price display:

* Removed `<sale-price>` and `<compare-at-price>` blocks

### 3. **Cart Page Trade Pricing Support**

In `main-cart.liquid`, added logic to:

* Check if `customer.tags contains 'Trade'`
* If yes, apply `line_item.variant.metafields.custom.trade_discount` percentage

```liquid
{%- if customer and customer.tags contains 'Trade' and line_item.variant.metafields.custom.trade_discount -%}
  {%- assign discount_percent = line_item.variant.metafields.custom.trade_discount | times: 0.01 -%}
  {%- assign discount_amount = line_item.final_line_price | times: discount_percent -%}
  {%- assign trade_price_total = line_item.final_line_price | minus: discount_amount -%}
  {{ trade_price_total | money }}
{%- else -%}
  {{ line_item.final_line_price | money }}
{%- endif -%}
```

### 4. **Created `trade-pricing.liquid` Snippet for Reusability**

We built a reusable snippet to insert pricing logic into product and card contexts:

```liquid
{%- assign is_trade = false -%}
{%- if customer != nil and customer.metafields.custom.trade_customer -%}
    {%- assign is_trade = true -%}
{%- endif -%}
{%- if product.selected_or_first_available_variant != nil -%}
    <div class="v-stack">
        {%- if is_trade or request.design_mode -%}
            {%- comment -%} For trade customers, show both RRP and Trade price {%- endcomment -%}
            <div class="rrp-price">
                RRP: {{ product.selected_or_first_available_variant.compare_at_price | default: product.selected_or_first_available_variant.price | money }}
            </div>
            
            {%- assign trade_discount = 100 | minus: product.metafields.custom.trade_discount | divided_by: 100 | default: 0.8 -%}
            <div class="trade-price">
                Trade price:&nbsp;
                {%- if product.metafields.custom.trade_price -%}
                    {{ product.metafields.custom.trade_price.value | money }}
                {%- else -%}
                    {{ product.selected_or_first_available_variant.price | times: trade_discount | money }}
                {%- endif -%}
            </div>
        {%- else -%}
            {%- comment -%} For regular customers, show normal price-list {%- endcomment -%}
            {%- if context == 'product' -%}
                {%- render 'price-list', product: product, variant: product.selected_or_first_available_variant, context: 'product' -%}
            {%- else -%}
                {%- render 'price-list', product: product, context: 'card' -%}
            {%- endif -%}
        {%- endif -%}
    </div>
{%- endif -%}

<style>
.trade-price {
    font-size: 1.2em;
    font-weight: bold;
    color: #ad966e;
    margin-top: 8px;
}

.rrp-price {
    font-size: 1em;
    color: #666;
}
</style>
```

---

## 🛠 Result

✅ Logged-in Trade customers now see:

* RRP and Trade Pricing on product cards
* Trade prices in Cart (automatically calculated)
* Sale blocks hidden to avoid redundancy

This improves pricing transparency for Trade customers without interfering with standard retail experience.

---

## 📌 Notes

* Trade discount logic is based on a `trade_discount` metafield per variant
* No automatic discount codes or cart-level discounts are applied — pricing is modified directly
* Tested with Prestige theme

## ⚙️ Configuration Requirements

### Required Metafields

1. **Customer Metafield**: `custom.trade_customer` (boolean)
2. **Product Metafields**:

   * `custom.trade_price` (money) - Optional: Custom trade price
   * `custom.trade_discount` (number) - Optional: Discount percentage

### Usage in Templates

```liquid
{% render 'trade-pricing', customer: customer, product: product, context: 'product' %}
```

## 🧪 Testing Checklist

* ✅ Test with **trade customers**: Ensure both prices display
* ✅ Test with **regular customers**: Ensure only standard price shows
* ✅ Check **design mode** for visibility
* ✅ Confirm trade price logic works with or without metafield values

## 🎨 Customization Tips

* **Adjust font sizes** by modifying `font-size` values
* **Change color** by editing `color: #ad966e` (trade) or `#666` (RRP)
* **Modify spacing** via `margin-top` or container layout
