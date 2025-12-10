# E-commerce Partner Integration API & Webhook Specification

**Version:** 1.0  
**Last Updated:** December 2024  
**Format:** JSON (all requests and responses)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Common Field Definitions](#2-common-field-definitions)
3. [Authentication](#3-authentication)
4. [Product APIs](#4-product-apis)
5. [Order APIs](#5-order-apis)
6. [Webhooks (Partner → Fabrilife)](#6-webhooks-partner--fabrilife)
7. [Webhooks (Fabrilife → Partner)](#7-webhooks-fabrilife--partner)
8. [Error Handling](#8-error-handling)
9. [SKU Mapping](#9-sku-mapping)
10. [Implementation Checklist](#10-implementation-checklist)
11. [Testing](#11-testing)

---

## 1. Overview

This document outlines the API endpoints and webhook specifications for integrating with Fabrilife's inventory and order management system.

### Base URL
```
Production: https://api.partner.com/v1
Sandbox: https://sandbox-api.partner.com/v1
```

### Common Headers
```
Content-Type: application/json
Authorization: Bearer {access_token}
```

### Timezone
All timestamps must be in ISO 8601 format with timezone offset:
```
2024-12-09T10:30:00+06:00
```

---

## 2. Common Field Definitions

This section defines common fields used throughout the API. Understanding these fields is essential for proper integration.

### 2.1 Identifier Fields

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `item_id` | int | Unique product identifier in the partner's e-commerce platform. A single product can have multiple SKU variants (sizes, colors). | `10045` |
| `sku_id` | int | Unique identifier for a specific SKU variant within the partner's e-commerce platform. | `50234` |
| `seller_sku` | string | **Fabrilife's SKU identifier.** This is the primary SKU used by Fabrilife to identify a specific product variant (combination of product + color + size). Use this for all inventory sync and order operations with Fabrilife. | `FL-NAVY-TS-M` |
| `shop_sku` | string | **Partner's e-commerce platform SKU.** This is the SKU code used on the partner's platform to identify the same product variant. Used for display and partner's internal operations. | `PARTNER-FL-NAVY-TS-M` |
| `barcode_ean` | string | EAN/UPC barcode number for the product variant. Used for scanning and inventory management. | `8901234567891` |
| `order_id` | int | Unique order identifier in the partner's e-commerce platform database. | `987654` |
| `order_number` | string | Human-readable order reference number displayed to customers. Format typically includes date. | `2024120901234` |
| `order_item_id` | int | Unique identifier for a specific line item within an order in the partner's platform. | `1234567` |

### 2.2 SKU Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│              PARTNER PLATFORM PRODUCT (item_id)                 │
│                    "Premium Cotton T-Shirt"                     │
│                      item_id: 10045                             │
└─────────────────────────────────────────────────────────────────┘
                                │
                                │ has multiple variants
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              PARTNER PLATFORM SKU VARIANTS (sku_id)             │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  sku_id: 50234  │  sku_id: 50235  │  sku_id: 50236              │
│  Size: S        │  Size: M        │  Size: L                    │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
                                │ identified on partner platform by
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              SHOP SKU (Partner's SKU code - string)             │
├─────────────────┬─────────────────┬─────────────────────────────┤
│PARTNER-FL-      │PARTNER-FL-      │PARTNER-FL-                  │
│NAVY-TS-S        │NAVY-TS-M        │NAVY-TS-L                    │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
                                │ mapped to Fabrilife's
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              SELLER SKU (Fabrilife's SKU - string)              │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ FL-NAVY-TS-S    │ FL-NAVY-TS-M    │ FL-NAVY-TS-L                │
└─────────────────┴─────────────────┴─────────────────────────────┘

Key Points:
• item_id, sku_id, order_id, order_item_id = Partner platform's internal IDs (integers)
• shop_sku = Partner platform's SKU code (string)
• seller_sku = Fabrilife's SKU code (string) - USE THIS FOR INVENTORY SYNC
```

### 2.3 Product & Inventory Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Product display name including variant info. |
| `status` | string | Product/SKU status: `live`, `inactive`, `deleted`. |
| `quantity` | int | Total stock quantity for the SKU. |
| `available` | int | Stock available for sale (quantity minus reserved). |
| `reserved` | int | Stock reserved in pending/unpaid orders. |
| `price` | float | Regular selling price (MRP). |
| `special_price` | float | Discounted/sale price. `null` if no active sale. |
| `special_from_date` | date | Sale start date (YYYY-MM-DD). Required if `special_price` is set. |
| `special_to_date` | date | Sale end date (YYYY-MM-DD). Required if `special_price` is set. |
| `size` | string | Size variant (e.g., `S`, `M`, `L`, `XL`, `XXL`). |
| `color` | string | Color variant (e.g., `Navy Blue`, `Black`). |

### 2.4 Order Fields

| Field | Type | Description |
|-------|------|-------------|
| `statuses` | array | Array of item statuses within the order. An order may have items in different statuses. |
| `payment_method` | string | Payment method: `cod` (Cash on Delivery), `online`, `bkash`, `nagad`, `card`. |
| `items_count` | int | Total number of items in the order. |
| `price` | float | Order subtotal (sum of item prices before shipping and discounts). |
| `shipping_fee` | float | Delivery/shipping charge. |
| `shipping_fee_discount` | float | Discount applied to shipping fee. |
| `voucher` | float | Voucher/coupon discount amount applied to the order. |
| `voucher_code` | string | The voucher/coupon code used, if any. |
| `item_price` | float | Original price per unit for an order item. |
| `paid_price` | float | Actual price paid per unit after discounts. |
| `voucher_amount` | float | Voucher discount allocated to this specific item. |
| `tracking_code` | string | Shipment tracking number from logistics provider. |
| `shipment_provider` | string | Logistics/courier company name. |

### 2.5 Address Fields

| Field | Type | Description |
|-------|------|-------------|
| `first_name` | string | Recipient's first name. |
| `last_name` | string | Recipient's last name. |
| `phone` | string | Primary phone number (preferably E.164 format). |
| `phone2` | string | Secondary/alternate phone number. |
| `address1` | string | Primary address line (house/building number, street). |
| `address2` | string | Secondary address line (apartment, suite, floor). |
| `address3` | string | Area/locality name. |
| `address4` | string | Additional address information. |
| `address5` | string | Additional address information. |
| `city` | string | City name. |
| `post_code` | string | Postal/ZIP code. |
| `country` | string | Country name. |

### 2.6 Timestamp Fields

| Field | Type | Description |
|-------|------|-------------|
| `created_at` | datetime | When the record was created. ISO 8601 format with timezone. |
| `updated_at` | datetime | When the record was last modified. ISO 8601 format with timezone. |
| `timestamp` | datetime | Event timestamp in webhooks. ISO 8601 format with timezone. |

### 2.7 Order Status Definitions

| Status | Description | Stock State |
|--------|-------------|-------------|
| `unpaid` | Order placed but payment not yet received. | Reserved |
| `pending` | Payment received, awaiting seller confirmation/processing. | Deducted |
| `confirmed` | Order confirmed by seller, preparing for shipment. | Deducted |
| `ready_to_ship` | Order packed and ready for courier pickup. | Deducted |
| `shipped` | Order handed over to logistics partner, in transit. | Deducted |
| `delivered` | Order successfully delivered to customer. | Sold |
| `canceled` | Order canceled (by customer or seller). | Restored |
| `failed` | Delivery attempt failed. | Restored |
| `returned` | Order returned by customer after delivery. | Restored |

### 2.8 Product Status Definitions

| Status | Description | Visibility |
|--------|-------------|------------|
| `live` | Product is active and available for sale. | Visible to customers |
| `inactive` | Product is temporarily disabled. | Hidden from customers |
| `deleted` | Product is permanently removed. | Not accessible |

### 2.9 Webhook Event Types

| Event | Trigger | Direction |
|-------|---------|-----------|
| `order.created` | New order placed | Partner → Fabrilife |
| `order.status_updated` | Order/item status changed | Partner → Fabrilife |
| `order.item_removed` | Item removed from order | Partner → Fabrilife |
| `order.item_quantity_changed` | Item quantity modified | Partner → Fabrilife |
| `orders.bulk_updated` | Multiple orders updated | Partner → Fabrilife |
| `inventory.sync_request` | Request current stock levels | Partner → Fabrilife |
| `stock.updated` | Inventory changed | Fabrilife → Partner |
| `price.updated` | Pricing changed | Fabrilife → Partner |
| `product.status_updated` | Product availability changed | Fabrilife → Partner |
| `product.created` | New product added | Fabrilife → Partner |

---

## 3. Authentication

### 3.1 Create Access Token (Initial Authorization)

Called after OAuth authorization to exchange authorization code for tokens.

**Business Use Case:**
This is the first step in establishing a secure connection between Fabrilife and the partner platform. Used when:
- Initial integration setup between Fabrilife and partner
- Re-authorization after refresh token expires
- Setting up new seller accounts on partner platform

**Endpoint:** `POST /auth/token/create`

**Headers:**
```
Content-Type: application/json
```

**Request:**
```json
{
  "code": "authorization_code_from_oauth_redirect"
}
```

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "expires_in": 604800,
  "refresh_expires_in": 2592000,
  "account": "partner@example.com",
  "account_id": "PARTNER-12345",
  "country": "BD"
}
```

**Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| code | int | 0 for success, non-zero for error |
| access_token | string | Token for API authentication |
| refresh_token | string | Token used to refresh access_token |
| expires_in | int | Access token validity in seconds (7 days) |
| refresh_expires_in | int | Refresh token validity in seconds (30 days) |
| account | string | Account email/identifier |
| account_id | string | Unique account ID |

---

### 3.2 Refresh Access Token

Called when access token expires. Requires the current access token.

**Business Use Case:**
Access tokens have limited validity (typically 7 days) for security. This API allows:
- Automated token renewal without user intervention
- Maintaining uninterrupted API access for scheduled sync jobs
- Avoiding full re-authorization flow for expired tokens

**Endpoint:** `POST /auth/token/refresh`

**Headers:**
```
Content-Type: application/json
Authorization: Bearer {current_access_token}
```

**Request:**
```json
{
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..."
}
```

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9_NEW...",
  "refresh_token": "dGhpcyBpcyBhIG5ldyByZWZyZXNoIHRva2Vu...",
  "expires_in": 604800,
  "refresh_expires_in": 2592000,
  "account": "partner@example.com",
  "account_id": "PARTNER-12345",
  "country": "BD"
}
```

**Response (Error - Invalid Token):**
```json
{
  "code": 1001,
  "message": "Invalid or expired access token",
  "request_id": "req_abc123"
}
```

---

## 4. Product APIs

### 4.1 Get Products (Paginated)

Retrieve product catalog with SKU-level details.

**Business Use Case:**
This API enables Fabrilife to sync the complete product catalog to the partner platform. Used for:
- Initial product catalog sync during integration setup
- Periodic full catalog sync to catch any missed updates
- Discovering new products added to the catalog
- Building product mapping between Fabrilife SKUs and partner SKUs

**Endpoint:** `GET /products`

**Headers:**
```
Authorization: Bearer {access_token}
```

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| filter | string | No | `all` | Filter products: `all`, `live`, `inactive`, `deleted` |
| limit | int | No | 50 | Products per page (max: 100) |
| offset | int | No | 0 | Pagination offset |
| updated_after | datetime | No | - | Filter products updated after this time (ISO 8601) |

**Example Request:**
```
GET /products?filter=live&limit=50&offset=0
```

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "total_products": 1250,
    "products": [
      {
        "item_id": 10045,
        "primary_category": 10001,
        "status": "live",
        "created_at": "2024-01-15T10:30:00+06:00",
        "updated_at": "2024-12-01T14:22:00+06:00",
        "attributes": {
          "name": "Premium Cotton T-Shirt - Navy Blue",
          "short_description": "Comfortable 100% cotton t-shirt",
          "description": "<p>Premium quality 100% cotton t-shirt...</p>",
          "brand": "Fabrilife",
          "model": "TS-PREMIUM-2024",
          "warranty_type": "No Warranty",
          "color_family": "Blue"
        },
        "skus": [
          {
            "sku_id": 50234,
            "seller_sku": "FL-NAVY-TS-S",
            "shop_sku": "PARTNER-FL-NAVY-TS-S",
            "barcode_ean": "8901234567890",
            "size": "S",
            "color": "Navy Blue",
            "quantity": 150,
            "available": 145,
            "reserved": 5,
            "price": 850.00,
            "special_price": 699.00,
            "special_from_date": "2024-12-01",
            "special_to_date": "2024-12-31",
            "status": "active",
            "package_weight": "0.25",
            "package_length": "30",
            "package_width": "25",
            "package_height": "3",
            "images": [
              "https://cdn.partner.com/products/navy-ts-s-1.jpg",
              "https://cdn.partner.com/products/navy-ts-s-2.jpg"
            ]
          },
          {
            "sku_id": 50235,
            "seller_sku": "FL-NAVY-TS-M",
            "shop_sku": "PARTNER-FL-NAVY-TS-M",
            "barcode_ean": "8901234567891",
            "size": "M",
            "color": "Navy Blue",
            "quantity": 200,
            "available": 190,
            "reserved": 10,
            "price": 850.00,
            "special_price": 699.00,
            "special_from_date": "2024-12-01",
            "special_to_date": "2024-12-31",
            "status": "active",
            "package_weight": "0.27",
            "package_length": "30",
            "package_width": "25",
            "package_height": "3",
            "images": [
              "https://cdn.partner.com/products/navy-ts-m-1.jpg"
            ]
          }
        ],
        "images": [
          {
            "url": "https://cdn.partner.com/products/navy-ts-main.jpg",
            "position": 1
          },
          {
            "url": "https://cdn.partner.com/products/navy-ts-back.jpg",
            "position": 2
          }
        ]
      }
    ]
  }
}
```

**SKU Fields Explanation:**

| Field | Type | Description |
|-------|------|-------------|
| seller_sku | string | Fabrilife's SKU identifier (use this for inventory sync) |
| shop_sku | string | Partner's SKU identifier on their platform |
| quantity | int | Total stock quantity |
| available | int | Stock available for sale |
| reserved | int | Stock reserved in pending orders |
| price | float | Regular price |
| special_price | float | Sale/discounted price (null if no sale) |
| special_from_date | date | Sale start date (YYYY-MM-DD) |
| special_to_date | date | Sale end date (YYYY-MM-DD) |

---

### 4.2 Get Product by ID

Retrieve a single product with all its SKU variants.

**Business Use Case:**
Fetch complete details for a specific product when:
- Verifying product information before creating listings
- Debugging inventory discrepancies for a specific product
- Getting all size/color variants for a product at once
- Refreshing product data after receiving update notifications

**Endpoint:** `GET /products/{item_id}`

**Headers:**
```
Authorization: Bearer {access_token}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| item_id | string | Yes | Unique product identifier |

**Example Request:**
```
GET /products/10045
```

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "item_id": 10045,
    "primary_category": 10001,
    "status": "live",
    "created_at": "2024-01-15T10:30:00+06:00",
    "updated_at": "2024-12-01T14:22:00+06:00",
    "attributes": {
      "name": "Premium Cotton T-Shirt - Navy Blue",
      "short_description": "Comfortable 100% cotton t-shirt",
      "description": "<p>Premium quality 100% cotton t-shirt with modern fit. Features reinforced stitching and pre-shrunk fabric for lasting comfort and durability.</p>",
      "brand": "Fabrilife",
      "model": "TS-PREMIUM-2024",
      "warranty_type": "No Warranty",
      "color_family": "Blue",
      "material": "100% Cotton",
      "weight": "180 GSM",
      "care_instructions": "Machine wash cold, tumble dry low"
    },
    "skus": [
      {
        "sku_id": 50234,
        "seller_sku": "FL-NAVY-TS-S",
        "shop_sku": "PARTNER-FL-NAVY-TS-S",
        "barcode_ean": "8901234567890",
        "size": "S",
        "color": "Navy Blue",
        "quantity": 150,
        "available": 145,
        "reserved": 5,
        "price": 850.00,
        "special_price": 699.00,
        "special_from_date": "2024-12-01",
        "special_to_date": "2024-12-31",
        "status": "active",
        "package_weight": "0.25",
        "package_length": "30",
        "package_width": "25",
        "package_height": "3",
        "images": [
          "https://cdn.partner.com/products/navy-ts-s-1.jpg",
          "https://cdn.partner.com/products/navy-ts-s-2.jpg"
        ]
      },
      {
        "sku_id": 50235,
        "seller_sku": "FL-NAVY-TS-M",
        "shop_sku": "PARTNER-FL-NAVY-TS-M",
        "barcode_ean": "8901234567891",
        "size": "M",
        "color": "Navy Blue",
        "quantity": 200,
        "available": 190,
        "reserved": 10,
        "price": 850.00,
        "special_price": 699.00,
        "special_from_date": "2024-12-01",
        "special_to_date": "2024-12-31",
        "status": "active",
        "package_weight": "0.27",
        "package_length": "30",
        "package_width": "25",
        "package_height": "3",
        "images": [
          "https://cdn.partner.com/products/navy-ts-m-1.jpg"
        ]
      },
      {
        "sku_id": 50236,
        "seller_sku": "FL-NAVY-TS-L",
        "shop_sku": "PARTNER-FL-NAVY-TS-L",
        "barcode_ean": "8901234567892",
        "size": "L",
        "color": "Navy Blue",
        "quantity": 180,
        "available": 175,
        "reserved": 5,
        "price": 850.00,
        "special_price": 699.00,
        "special_from_date": "2024-12-01",
        "special_to_date": "2024-12-31",
        "status": "active",
        "package_weight": "0.29",
        "package_length": "32",
        "package_width": "26",
        "package_height": "3",
        "images": [
          "https://cdn.partner.com/products/navy-ts-l-1.jpg"
        ]
      },
      {
        "sku_id": 50237,
        "seller_sku": "FL-NAVY-TS-XL",
        "shop_sku": "PARTNER-FL-NAVY-TS-XL",
        "barcode_ean": "8901234567893",
        "size": "XL",
        "color": "Navy Blue",
        "quantity": 120,
        "available": 118,
        "reserved": 2,
        "price": 850.00,
        "special_price": 699.00,
        "special_from_date": "2024-12-01",
        "special_to_date": "2024-12-31",
        "status": "active",
        "package_weight": "0.31",
        "package_length": "34",
        "package_width": "28",
        "package_height": "3",
        "images": [
          "https://cdn.partner.com/products/navy-ts-xl-1.jpg"
        ]
      }
    ],
    "images": [
      {
        "url": "https://cdn.partner.com/products/navy-ts-main.jpg",
        "position": 1
      },
      {
        "url": "https://cdn.partner.com/products/navy-ts-back.jpg",
        "position": 2
      },
      {
        "url": "https://cdn.partner.com/products/navy-ts-detail.jpg",
        "position": 3
      }
    ],
    "total_stock": 650,
    "total_available": 628,
    "total_reserved": 22
  }
}
```

**Response (Product Not Found):**
```json
{
  "code": 3002,
  "message": "Product not found",
  "request_id": "req_abc123xyz",
  "timestamp": "2024-12-09T10:30:00+06:00",
  "errors": [
    {
      "field": "item_id",
      "message": "No product found with the specified ID",
      "value": 99999
    }
  ]
}
```

---

### 4.3 Get Product by SKU

Retrieve product information using a seller SKU.

**Business Use Case:**
Quick lookup when you only have the SKU identifier. Used for:
- Order processing when you need product details from SKU
- Inventory verification for specific variants
- Mapping validation to ensure SKU exists before creating orders
- Customer service inquiries about specific product variants

**Endpoint:** `GET /products/sku/{seller_sku}`

**Headers:**
```
Authorization: Bearer {access_token}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| seller_sku | string | Yes | Fabrilife seller SKU |

**Example Request:**
```
GET /products/sku/FL-NAVY-TS-M
```

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "item_id": 10045,
    "product_name": "Premium Cotton T-Shirt - Navy Blue",
    "status": "live",
    "sku": {
      "sku_id": 50235,
      "seller_sku": "FL-NAVY-TS-M",
      "shop_sku": "PARTNER-FL-NAVY-TS-M",
      "barcode_ean": "8901234567891",
      "size": "M",
      "color": "Navy Blue",
      "quantity": 200,
      "available": 190,
      "reserved": 10,
      "price": 850.00,
      "special_price": 699.00,
      "special_from_date": "2024-12-01",
      "special_to_date": "2024-12-31",
      "status": "active"
    },
    "other_variants": [
      {
        "sku_id": 50234,
        "seller_sku": "FL-NAVY-TS-S",
        "size": "S",
        "available": 145
      },
      {
        "sku_id": 50236,
        "seller_sku": "FL-NAVY-TS-L",
        "size": "L",
        "available": 175
      },
      {
        "sku_id": 50237,
        "seller_sku": "FL-NAVY-TS-XL",
        "size": "XL",
        "available": 118
      }
    ]
  }
}
```

**Response (SKU Not Found):**
```json
{
  "code": 3002,
  "message": "SKU not found",
  "request_id": "req_def456uvw",
  "timestamp": "2024-12-09T10:30:00+06:00",
  "errors": [
    {
      "field": "seller_sku",
      "message": "No product found with the specified SKU",
      "value": "INVALID-SKU"
    }
  ]
}
```

---

### 4.4 Update Product Price & Quantity

Update stock levels and/or pricing for multiple SKUs.

**Business Use Case:**
Critical for maintaining accurate inventory across platforms. Used for:
- Real-time stock updates when Fabrilife inventory changes
- Price synchronization during sales campaigns or price changes
- Bulk inventory updates after stock receiving or adjustments
- Setting products to zero stock when out of inventory
- Applying promotional pricing with sale dates

**Endpoint:** `POST /products/price-quantity/update`

**Headers:**
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request:**
```json
{
  "products": [
    {
      "seller_sku": "FL-NAVY-TS-S",
      "quantity": 145
    },
    {
      "seller_sku": "FL-NAVY-TS-M",
      "quantity": 0,
      "price": 899.00,
      "special_price": 749.00,
      "special_from_date": "2024-12-10",
      "special_to_date": "2024-12-31"
    },
    {
      "seller_sku": "FL-BLACK-TS-L",
      "price": 850.00,
      "special_price": null
    }
  ]
}
```

**Request Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| seller_sku | string | Yes | Fabrilife SKU identifier |
| quantity | int | No | New stock quantity |
| price | float | No | New regular price |
| special_price | float | No | New sale price (null to remove sale) |
| special_from_date | date | No* | Sale start date (*required if special_price set) |
| special_to_date | date | No* | Sale end date (*required if special_price set) |

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "updated_count": 3,
    "results": [
      {
        "seller_sku": "FL-NAVY-TS-S",
        "status": "success",
        "message": "Updated successfully"
      },
      {
        "seller_sku": "FL-NAVY-TS-M",
        "status": "success",
        "message": "Updated successfully"
      },
      {
        "seller_sku": "FL-BLACK-TS-L",
        "status": "success",
        "message": "Updated successfully"
      }
    ]
  }
}
```

**Response (Partial Success):**
```json
{
  "code": 0,
  "message": "Partial success",
  "data": {
    "updated_count": 2,
    "failed_count": 1,
    "results": [
      {
        "seller_sku": "FL-NAVY-TS-S",
        "status": "success",
        "message": "Updated successfully"
      },
      {
        "seller_sku": "INVALID-SKU",
        "status": "failed",
        "message": "SKU not found"
      }
    ]
  }
}
```

---

## 5. Order APIs

### 5.1 Get Orders (Paginated)

Retrieve orders with filtering options, including order items.

**Business Use Case:**
This API is the primary method for fetching orders from the partner platform. Fabrilife uses this to:
- Sync new orders periodically to update local inventory
- Monitor order status changes for stock management
- Generate reports and analytics on partner sales
- Reconcile orders between systems during daily/weekly audits

**Endpoint:** `GET /orders`

**Headers:**
```
Authorization: Bearer {access_token}
```

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| created_after | datetime | No* | - | Orders created after this time (ISO 8601) |
| created_before | datetime | No | - | Orders created before this time |
| update_after | datetime | No* | - | Orders updated after this time |
| update_before | datetime | No | - | Orders updated before this time |
| status | string | No | - | Filter by status (see Order Statuses) |
| sort_by | string | No | `created_at` | Sort field: `created_at` or `updated_at` |
| sort_direction | string | No | `DESC` | Sort direction: `ASC` or `DESC` |
| limit | int | No | 50 | Results per page (max: 100) |
| offset | int | No | 0 | Pagination offset |

*At least one time filter (`created_after` or `update_after`) is required.

**Order Statuses:**

| Status | Description | Stock Impact |
|--------|-------------|--------------|
| `unpaid` | Order placed, awaiting payment | Stock reserved |
| `pending` | Payment received, awaiting processing | Stock deducted |
| `confirmed` | Order confirmed by seller | No change |
| `ready_to_ship` | Order packed, ready for pickup | No change |
| `shipped` | Handed to logistics partner | No change |
| `delivered` | Successfully delivered | No change |
| `canceled` | Order canceled | Stock restored |
| `failed` | Delivery failed | Stock restored |
| `returned` | Order returned by customer | Stock restored |

**Example Request:**
```
GET /orders?created_after=2024-12-01T00:00:00%2B06:00&status=pending&limit=50&offset=0&sort_by=created_at&sort_direction=ASC
```

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "total_orders": 45,
    "count": 2,
    "orders": [
      {
        "order_id": 987654,
        "order_number": "2024120901234",
        "created_at": "2024-12-09T10:30:00+06:00",
        "updated_at": "2024-12-09T11:45:00+06:00",
        "statuses": ["pending"],
        "payment_method": "cod",
        "remarks": "Please call before delivery",
        "delivery_info": "Leave at door if not home",
        "customer_first_name": "John",
        "customer_last_name": "Doe",
        "items_count": 2,
        "price": 1398.00,
        "shipping_fee": 60.00,
        "shipping_fee_discount": 0.00,
        "voucher": 100.00,
        "voucher_code": "WINTER100",
        "gift_option": false,
        "gift_message": null,
        "address_billing": {
          "first_name": "John",
          "last_name": "Doe",
          "phone": "+8801712345678",
          "phone2": "",
          "address1": "House 12, Road 5",
          "address2": "Block A",
          "address3": "Banani",
          "address4": "",
          "address5": "",
          "city": "Dhaka",
          "post_code": "1213",
          "country": "Bangladesh"
        },
        "address_shipping": {
          "first_name": "John",
          "last_name": "Doe",
          "phone": "+8801712345678",
          "phone2": "",
          "address1": "House 12, Road 5",
          "address2": "Block A",
          "address3": "Banani",
          "address4": "",
          "address5": "",
          "city": "Dhaka",
          "post_code": "1213",
          "country": "Bangladesh"
        },
        "items": [
          {
            "order_item_id": 1234567,
            "sku": "FL-NAVY-TS-M",
            "shop_sku": "PARTNER-FL-NAVY-TS-M",
            "name": "Premium Cotton T-Shirt - Navy Blue",
            "variation": "Size: M, Color: Navy Blue",
            "quantity": 1,
            "item_price": 850.00,
            "paid_price": 699.00,
            "tax_amount": 0.00,
            "voucher_amount": 50.00,
            "status": "pending",
            "tracking_code": null,
            "shipment_provider": null
          },
          {
            "order_item_id": 1234568,
            "sku": "FL-BLACK-TS-L",
            "shop_sku": "PARTNER-FL-BLACK-TS-L",
            "name": "Premium Cotton T-Shirt - Black",
            "variation": "Size: L, Color: Black",
            "quantity": 1,
            "item_price": 850.00,
            "paid_price": 699.00,
            "tax_amount": 0.00,
            "voucher_amount": 50.00,
            "status": "pending",
            "tracking_code": null,
            "shipment_provider": null
          }
        ]
      },
      {
        "order_id": 987655,
        "order_number": "2024120901235",
        "created_at": "2024-12-09T11:00:00+06:00",
        "updated_at": "2024-12-09T11:00:00+06:00",
        "statuses": ["pending"],
        "payment_method": "online",
        "remarks": "",
        "delivery_info": "",
        "customer_first_name": "Jane",
        "customer_last_name": "Smith",
        "items_count": 1,
        "price": 699.00,
        "shipping_fee": 60.00,
        "shipping_fee_discount": 60.00,
        "voucher": 0.00,
        "voucher_code": null,
        "gift_option": false,
        "gift_message": null,
        "address_billing": {
          "first_name": "Jane",
          "last_name": "Smith",
          "phone": "+8801898765432",
          "phone2": "+8801712345678",
          "address1": "Apt 5B, Tower 3",
          "address2": "Gulshan Heights",
          "address3": "Gulshan-2",
          "address4": "",
          "address5": "",
          "city": "Dhaka",
          "post_code": "1212",
          "country": "Bangladesh"
        },
        "address_shipping": {
          "first_name": "Jane",
          "last_name": "Smith",
          "phone": "+8801898765432",
          "phone2": "+8801712345678",
          "address1": "Apt 5B, Tower 3",
          "address2": "Gulshan Heights",
          "address3": "Gulshan-2",
          "address4": "",
          "address5": "",
          "city": "Dhaka",
          "post_code": "1212",
          "country": "Bangladesh"
        },
        "items": [
          {
            "order_item_id": 1234569,
            "sku": "FL-WHITE-TS-S",
            "shop_sku": "PARTNER-FL-WHITE-TS-S",
            "name": "Premium Cotton T-Shirt - White",
            "variation": "Size: S, Color: White",
            "quantity": 2,
            "item_price": 850.00,
            "paid_price": 699.00,
            "tax_amount": 0.00,
            "voucher_amount": 0.00,
            "status": "pending",
            "tracking_code": null,
            "shipment_provider": null
          }
        ]
      }
    ]
  }
}
```

**Order Response Fields:**

| Field | Type | Description |
|-------|------|-------------|
| order_id | string | Unique order identifier |
| order_number | string | Display order number |
| statuses | array | Array of item statuses in this order |
| price | float | Subtotal (sum of item prices) |
| shipping_fee | float | Delivery charge |
| shipping_fee_discount | float | Discount on shipping |
| voucher | float | Voucher/coupon discount amount |

---

### 5.2 Get Order Items

Retrieve detailed item information for a specific order.

**Business Use Case:**
Get comprehensive item-level details for a single order. Used for:
- Order fulfillment - knowing exactly what to pick and pack
- Tracking individual item statuses in multi-item orders
- Customer service - answering queries about specific items
- Returns processing - identifying which items are being returned
- Invoice generation with complete item details

**Endpoint:** `GET /orders/{order_id}/items`

**Headers:**
```
Authorization: Bearer {access_token}
```

**Path Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| order_id | string | Yes | The order ID |

**Example Request:**
```
GET /orders/987654/items
```

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "order_id": 987654,
    "items": [
      {
        "order_item_id": 1234567,
        "sku": "FL-NAVY-TS-M",
        "shop_sku": "PARTNER-FL-NAVY-TS-M",
        "name": "Premium Cotton T-Shirt - Navy Blue",
        "variation": "Size: M, Color: Navy Blue",
        "quantity": 1,
        "item_price": 850.00,
        "paid_price": 699.00,
        "tax_amount": 0.00,
        "shipping_fee": 30.00,
        "shipping_fee_discount": 0.00,
        "voucher_amount": 50.00,
        "wallet_credits": 0.00,
        "status": "pending",
        "is_digital": false,
        "digital_delivery_info": null,
        "reason": null,
        "reason_detail": null,
        "tracking_code": null,
        "tracking_code_pre": null,
        "shipment_provider": null,
        "invoice_number": null,
        "package_id": null,
        "product_main_image": "https://cdn.partner.com/products/navy-ts-m-1.jpg",
        "product_detail_url": "https://partner.com/product/navy-ts",
        "purchase_order_id": null,
        "purchase_order_number": null,
        "promised_shipping_time": "2024-12-11T18:00:00+06:00",
        "extra_attributes": null,
        "created_at": "2024-12-09T10:30:00+06:00",
        "updated_at": "2024-12-09T10:30:00+06:00"
      },
      {
        "order_item_id": 1234568,
        "sku": "FL-BLACK-TS-L",
        "shop_sku": "PARTNER-FL-BLACK-TS-L",
        "name": "Premium Cotton T-Shirt - Black",
        "variation": "Size: L, Color: Black",
        "quantity": 1,
        "item_price": 850.00,
        "paid_price": 699.00,
        "tax_amount": 0.00,
        "shipping_fee": 30.00,
        "shipping_fee_discount": 0.00,
        "voucher_amount": 50.00,
        "wallet_credits": 0.00,
        "status": "pending",
        "is_digital": false,
        "digital_delivery_info": null,
        "reason": null,
        "reason_detail": null,
        "tracking_code": null,
        "tracking_code_pre": null,
        "shipment_provider": null,
        "invoice_number": null,
        "package_id": null,
        "product_main_image": "https://cdn.partner.com/products/black-ts-l-1.jpg",
        "product_detail_url": "https://partner.com/product/black-ts",
        "purchase_order_id": null,
        "purchase_order_number": null,
        "promised_shipping_time": "2024-12-11T18:00:00+06:00",
        "extra_attributes": null,
        "created_at": "2024-12-09T10:30:00+06:00",
        "updated_at": "2024-12-09T10:30:00+06:00"
      }
    ]
  }
}
```

**Order Item Fields:**

| Field | Type | Description |
|-------|------|-------------|
| order_item_id | string | Unique item identifier |
| sku | string | Fabrilife's seller SKU |
| shop_sku | string | Partner's SKU on their platform |
| quantity | int | Number of units ordered |
| item_price | float | Original price per unit |
| paid_price | float | Actual price paid per unit |
| voucher_amount | float | Voucher discount applied to this item |
| status | string | Item-level status |
| tracking_code | string | Shipment tracking number |
| shipment_provider | string | Logistics provider name |

---

### 5.3 Update Order Status

Update the status of an order or specific items.

**Business Use Case:**
Essential for order lifecycle management and inventory accuracy. Used for:
- Confirming orders after inventory verification
- Marking orders ready for shipment after packing
- Canceling orders (triggers stock restoration)
- Handling delivery failures and returns
- Partial cancellations when some items are unavailable
- Triggering appropriate stock adjustments based on status changes

**Endpoint:** `POST /orders/status/update`

**Headers:**
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request:**
```json
{
  "order_id": 987654,
  "status": "ready_to_ship",
  "order_item_ids": [1234567, 1234568],
  "reason": null,
  "reason_detail": null
}
```

**Request Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| order_id | int | Yes | The order ID |
| status | string | Yes | New status (see Order Statuses) |
| order_item_ids | array[int] | No | Specific items to update (all if omitted) |
| reason | string | No* | Reason code (*required for cancel/return) |
| reason_detail | string | No | Additional reason details |

**Cancellation Reason Codes:**

| Code | Description |
|------|-------------|
| `customer_request` | Customer requested cancellation |
| `out_of_stock` | Item not available |
| `pricing_error` | Incorrect pricing |
| `duplicate_order` | Duplicate order placed |
| `fraud_suspected` | Suspected fraudulent order |
| `delivery_failed` | Could not deliver |
| `customer_refused` | Customer refused delivery |
| `wrong_item` | Wrong item shipped |
| `damaged` | Item damaged |
| `other` | Other reason (provide detail) |

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "order_id": 987654,
    "previous_status": "pending",
    "new_status": "ready_to_ship",
    "updated_items": [
      {
        "order_item_id": 1234567,
        "status": "ready_to_ship"
      },
      {
        "order_item_id": 1234568,
        "status": "ready_to_ship"
      }
    ],
    "updated_at": "2024-12-09T14:30:00+06:00"
  }
}
```

**Response (Error - Invalid Transition):**
```json
{
  "code": 4001,
  "message": "Invalid status transition",
  "data": {
    "order_id": 987654,
    "current_status": "delivered",
    "requested_status": "pending",
    "allowed_transitions": ["returned"]
  }
}
```

**Valid Status Transitions:**

| From Status | Allowed To |
|-------------|------------|
| `unpaid` | `pending`, `canceled` |
| `pending` | `confirmed`, `ready_to_ship`, `canceled` |
| `confirmed` | `ready_to_ship`, `canceled` |
| `ready_to_ship` | `shipped`, `canceled` |
| `shipped` | `delivered`, `failed`, `returned` |
| `delivered` | `returned` |
| `failed` | `shipped`, `returned`, `canceled` |

---

### 5.4 Set Tracking Information

Add or update shipment tracking details.

**Business Use Case:**
Enables end-to-end shipment visibility for customers and operations. Used for:
- Adding tracking numbers after courier pickup
- Enabling customer order tracking on partner platform
- Logistics performance monitoring and reporting
- Proof of shipment for dispute resolution
- Automated delivery status updates

**Endpoint:** `POST /orders/tracking/update`

**Headers:**
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request:**
```json
{
  "order_id": 987654,
  "tracking_code": "TRK123456789",
  "shipment_provider": "Pathao",
  "order_item_ids": [1234567, 1234568],
  "serial_number": null
}
```

**Request Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| order_id | int | Yes | The order ID |
| tracking_code | string | Yes | Shipment tracking number |
| shipment_provider | string | Yes | Logistics provider name |
| order_item_ids | array[int] | No | Specific items (all if omitted) |
| serial_number | array | No | Serial numbers for serialized items |

**Supported Shipment Providers:**

| Provider Code | Provider Name |
|---------------|---------------|
| `pathao` | Pathao Courier |
| `paperfly` | Paperfly |
| `redx` | RedX |
| `steadfast` | Steadfast |
| `sundarban` | Sundarban Courier |
| `ecourier` | eCourier |
| `other` | Other (specify in name) |

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "order_id": 987654,
    "tracking_code": "TRK123456789",
    "shipment_provider": "Pathao",
    "tracking_url": "https://tracking.pathao.com/TRK123456789",
    "updated_items": [1234567, 1234568],
    "updated_at": "2024-12-09T15:00:00+06:00"
  }
}
```

---

### 5.5 Get Multiple Order Items (Bulk)

Retrieve items for multiple orders in a single request.

**Business Use Case:**
Efficient bulk data retrieval for batch processing. Used for:
- Batch order fulfillment - getting items for multiple orders at once
- Generating pick lists for warehouse operations
- Bulk invoice generation
- Daily/weekly order reconciliation
- Reducing API calls during high-volume sync operations

**Endpoint:** `POST /orders/items/get`

**Headers:**
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request:**
```json
{
  "order_ids": [987654, 987655, 987656]
}
```

**Response (Success):**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "orders": [
      {
        "order_id": 987654,
        "items": [
          {
            "order_item_id": 1234567,
            "sku": "FL-NAVY-TS-M",
            "shop_sku": "PARTNER-FL-NAVY-TS-M",
            "quantity": 1,
            "status": "pending"
          }
        ]
      },
      {
        "order_id": 987655,
        "items": [
          {
            "order_item_id": 1234569,
            "sku": "FL-WHITE-TS-S",
            "shop_sku": "PARTNER-FL-WHITE-TS-S",
            "quantity": 2,
            "status": "pending"
          }
        ]
      }
    ]
  }
}
```

---

## 6. Webhooks (Partner → Fabrilife)

The partner system must send webhook notifications to Fabrilife when order events occur. This enables real-time inventory synchronization.

**Why Webhooks Are Critical:**
Webhooks provide real-time event notifications, eliminating the need for constant polling. This ensures:
- Immediate inventory updates when orders are placed or canceled
- Accurate stock levels across all sales channels
- Prevention of overselling due to delayed sync
- Reduced API load compared to polling-based approaches

### 6.1 Webhook Configuration

**Business Use Case:**
Secure webhook setup ensures reliable event delivery. Proper configuration enables:
- Authenticated communication preventing unauthorized webhook submissions
- Guaranteed delivery through retry mechanisms
- Event ordering and deduplication using event IDs

**Fabrilife Webhook Endpoint:**
```
POST https://api.fabrilife.com/webhooks/partner/{partner_id}
```

**Required Headers:**
```
Content-Type: application/json
X-Webhook-Signature: sha256={hmac_sha256_signature}
X-Webhook-Timestamp: {unix_timestamp}
X-Partner-ID: {partner_id}
```

**Signature Generation (PHP Example):**
```php
$payload = json_encode($data);
$timestamp = time();
$signature_string = $timestamp . '.' . $payload;
$signature = hash_hmac('sha256', $signature_string, $webhook_secret);

// Header value
$header = 'sha256=' . $signature;
```

**Webhook Retry Policy:**
- Initial attempt immediately
- Retry 1: After 1 minute
- Retry 2: After 5 minutes
- Retry 3: After 30 minutes
- Retry 4: After 2 hours
- Retry 5: After 24 hours
- After 5 retries, webhook is marked as failed

**Expected Response:**
- HTTP 200 with body: `{"code": 0, "message": "Success", "received_event_id": "{event_id}"}`
- Response within 30 seconds
- Any non-200 response triggers retry

---

### 6.2 Order Created Webhook

**Event:** `order.created`

Triggered immediately when a new order is placed.

**Business Use Case:**
The most critical webhook for inventory management. When a customer places an order:
- Fabrilife immediately deducts stock for ordered SKUs
- Prevents the same inventory from being sold on Fabrilife website or other channels
- Enables real-time inventory accuracy across all platforms
- Creates local order record for fulfillment tracking

**Payload:**
```json
{
  "event": "order.created",
  "event_id": "evt_abc123xyz",
  "timestamp": "2024-12-09T10:30:00+06:00",
  "data": {
    "order_id": 987654,
    "order_number": "2024120901234",
    "created_at": "2024-12-09T10:30:00+06:00",
    "updated_at": "2024-12-09T10:30:00+06:00",
    "statuses": ["pending"],
    "payment_method": "cod",
    "items_count": 2,
    "price": 1398.00,
    "shipping_fee": 60.00,
    "voucher": 100.00,
    "voucher_code": "WINTER100",
    "items": [
      {
        "order_item_id": 1234567,
        "sku": "FL-NAVY-TS-M",
        "shop_sku": "PARTNER-FL-NAVY-TS-M",
        "name": "Premium Cotton T-Shirt - Navy Blue - M",
        "quantity": 1,
        "item_price": 850.00,
        "paid_price": 699.00,
        "status": "pending"
      },
      {
        "order_item_id": 1234568,
        "sku": "FL-BLACK-TS-L",
        "shop_sku": "PARTNER-FL-BLACK-TS-L",
        "name": "Premium Cotton T-Shirt - Black - L",
        "quantity": 1,
        "item_price": 850.00,
        "paid_price": 699.00,
        "status": "pending"
      }
    ],
    "customer": {
      "first_name": "John",
      "last_name": "Doe",
      "phone": "+8801712345678",
      "email": "john.doe@example.com"
    },
    "address_billing": {
      "first_name": "John",
      "last_name": "Doe",
      "phone": "+8801712345678",
      "address1": "House 12, Road 5",
      "address2": "Block A",
      "address3": "Banani",
      "address4": "",
      "address5": "",
      "city": "Dhaka",
      "post_code": "1213",
      "country": "Bangladesh"
    },
    "address_shipping": {
      "first_name": "John",
      "last_name": "Doe",
      "phone": "+8801712345678",
      "address1": "House 12, Road 5",
      "address2": "Block A",
      "address3": "Banani",
      "address4": "",
      "address5": "",
      "city": "Dhaka",
      "post_code": "1213",
      "country": "Bangladesh"
    }
  }
}
```

**Fabrilife Action:**
- Deduct stock for each SKU based on quantity
- Create order record in local database

**Expected Response:**
```json
{
  "code": 0,
  "message": "Webhook received successfully",
  "received_event_id": "evt_abc123xyz",
  "processed_skus": [
    {"sku": "FL-NAVY-TS-M", "quantity_deducted": 1},
    {"sku": "FL-BLACK-TS-L", "quantity_deducted": 1}
  ]
}
```

---

### 6.3 Order Status Updated Webhook

**Event:** `order.status_updated`

Triggered when order or item status changes.

**Business Use Case:**
Ensures inventory accuracy throughout the order lifecycle:
- **Cancellation**: Restores stock to available inventory for resale
- **Return**: Adds returned items back to sellable inventory
- **Delivery failure**: Restores stock when delivery cannot be completed
- **Status recovery**: Deducts stock if canceled order is reactivated
- Enables accurate inventory reporting and forecasting

**Payload:**
```json
{
  "event": "order.status_updated",
  "event_id": "evt_def456uvw",
  "timestamp": "2024-12-09T14:00:00+06:00",
  "data": {
    "order_id": 987654,
    "order_number": "2024120901234",
    "previous_statuses": ["pending"],
    "new_statuses": ["canceled"],
    "reason": "customer_request",
    "reason_detail": "Customer found better price elsewhere",
    "updated_at": "2024-12-09T14:00:00+06:00",
    "items": [
      {
        "order_item_id": 1234567,
        "sku": "FL-NAVY-TS-M",
        "shop_sku": "PARTNER-FL-NAVY-TS-M",
        "quantity": 1,
        "previous_status": "pending",
        "new_status": "canceled"
      },
      {
        "order_item_id": 1234568,
        "sku": "FL-BLACK-TS-L",
        "shop_sku": "PARTNER-FL-BLACK-TS-L",
        "quantity": 1,
        "previous_status": "pending",
        "new_status": "canceled"
      }
    ]
  }
}
```

**Stock Impact Matrix:**

| Previous Status | New Status | Stock Action |
|-----------------|------------|--------------|
| `pending` | `canceled` | **Restore** stock |
| `pending` | `failed` | **Restore** stock |
| `confirmed` | `canceled` | **Restore** stock |
| `ready_to_ship` | `canceled` | **Restore** stock |
| `shipped` | `returned` | **Restore** stock |
| `shipped` | `failed` | **Restore** stock |
| `delivered` | `returned` | **Restore** stock |
| `canceled` | `pending` | **Deduct** stock |
| `failed` | `pending` | **Deduct** stock |
| Any | `delivered` | No change |
| Any | `shipped` | No change |

**Expected Response:**
```json
{
  "code": 0,
  "message": "Webhook received successfully",
  "received_event_id": "evt_def456uvw",
  "stock_updated": true,
  "updated_skus": [
    {"sku": "FL-NAVY-TS-M", "quantity_restored": 1, "new_stock": 151},
    {"sku": "FL-BLACK-TS-L", "quantity_restored": 1, "new_stock": 76}
  ]
}
```

---

### 6.4 Order Item Removed Webhook

**Event:** `order.item_removed`

Triggered when a specific item is removed from an order (partial cancellation).

**Business Use Case:**
Handles partial order modifications without affecting entire order:
- Customer removes one item but keeps others in the order
- Item becomes unavailable after order placement
- Customer service removes item due to pricing error
- Restores only the removed item's stock, not the entire order
- Maintains accurate order totals and item counts

**Payload:**
```json
{
  "event": "order.item_removed",
  "event_id": "evt_ghi789rst",
  "timestamp": "2024-12-09T12:00:00+06:00",
  "data": {
    "order_id": 987654,
    "order_number": "2024120901234",
    "removed_item": {
      "order_item_id": 1234568,
      "sku": "FL-BLACK-TS-L",
      "shop_sku": "PARTNER-FL-BLACK-TS-L",
      "quantity": 1,
      "item_price": 699.00,
      "reason": "out_of_stock",
      "reason_detail": "Size L currently unavailable"
    },
    "remaining_items": [
      {
        "order_item_id": 1234567,
        "sku": "FL-NAVY-TS-M",
        "shop_sku": "PARTNER-FL-NAVY-TS-M",
        "quantity": 1,
        "status": "pending"
      }
    ],
    "updated_totals": {
      "items_count": 1,
      "price": 699.00,
      "shipping_fee": 60.00,
      "voucher": 50.00,
      "total": 709.00
    },
    "updated_at": "2024-12-09T12:00:00+06:00"
  }
}
```

**Fabrilife Action:**
- Restore stock for removed item
- Update local order record

---

### 6.5 Order Item Quantity Changed Webhook

**Event:** `order.item_quantity_changed`

Triggered when item quantity is modified.

**Business Use Case:**
Handles quantity adjustments within an order:
- Customer reduces quantity (e.g., ordered 3, wants only 2)
- Partial fulfillment when full quantity unavailable
- Customer increases quantity before shipment
- Ensures precise stock adjustment matching the quantity change
- More granular than full item removal for partial changes

**Payload:**
```json
{
  "event": "order.item_quantity_changed",
  "event_id": "evt_jkl012mno",
  "timestamp": "2024-12-09T12:30:00+06:00",
  "data": {
    "order_id": 987654,
    "order_number": "2024120901234",
    "changed_item": {
      "order_item_id": 1234567,
      "sku": "FL-NAVY-TS-M",
      "shop_sku": "PARTNER-FL-NAVY-TS-M",
      "previous_quantity": 2,
      "new_quantity": 1,
      "quantity_change": -1,
      "reason": "customer_request"
    },
    "updated_totals": {
      "items_count": 1,
      "price": 699.00,
      "shipping_fee": 60.00,
      "voucher": 50.00,
      "total": 709.00
    },
    "updated_at": "2024-12-09T12:30:00+06:00"
  }
}
```

**Stock Action:**
- If `quantity_change` is negative: **Restore** that quantity
- If `quantity_change` is positive: **Deduct** that quantity

---

### 6.6 Bulk Order Update Webhook

**Event:** `orders.bulk_updated`

For efficiency, send bulk updates when multiple orders change.

**Business Use Case:**
Optimizes webhook delivery during high-volume operations:
- End-of-day bulk status updates from logistics provider
- Batch delivery confirmations
- Mass cancellation processing
- Reduces webhook overhead during bulk operations
- Improves processing efficiency for Fabrilife's systems

**Payload:**
```json
{
  "event": "orders.bulk_updated",
  "event_id": "evt_pqr345stu",
  "timestamp": "2024-12-09T18:00:00+06:00",
  "data": {
    "update_type": "status_change",
    "orders": [
      {
        "order_id": "ORD-2024120901234",
        "previous_status": "shipped",
        "new_status": "delivered",
        "items": [
          {"sku": "FL-NAVY-TS-M", "quantity": 1, "status": "delivered"}
        ]
      },
      {
        "order_id": "ORD-2024120901235",
        "previous_status": "shipped",
        "new_status": "delivered",
        "items": [
          {"sku": "FL-WHITE-TS-S", "quantity": 2, "status": "delivered"}
        ]
      },
      {
        "order_id": "ORD-2024120901236",
        "previous_status": "shipped",
        "new_status": "returned",
        "items": [
          {"sku": "FL-BLACK-TS-L", "quantity": 1, "status": "returned"}
        ]
      }
    ],
    "summary": {
      "total_orders": 3,
      "delivered": 2,
      "returned": 1
    }
  }
}
```

---

### 6.7 Inventory Sync Request Webhook

**Event:** `inventory.sync_request`

Partner can request current stock levels. Use sparingly (max once per hour).

**Business Use Case:**
Enables inventory reconciliation and verification:
- Daily inventory sync to catch any missed webhook events
- Verification after system maintenance or outages
- Debugging inventory discrepancies between systems
- Initial inventory load for new product listings
- Audit trail for inventory accuracy monitoring

**Payload:**
```json
{
  "event": "inventory.sync_request",
  "event_id": "evt_vwx678yza",
  "timestamp": "2024-12-09T08:00:00+06:00",
  "data": {
    "request_type": "specific_skus",
    "seller_skus": [
      "FL-NAVY-TS-S",
      "FL-NAVY-TS-M",
      "FL-NAVY-TS-L",
      "FL-BLACK-TS-S",
      "FL-BLACK-TS-M"
    ]
  }
}
```

**Alternative - Full Sync Request:**
```json
{
  "event": "inventory.sync_request",
  "event_id": "evt_vwx678yzb",
  "timestamp": "2024-12-09T08:00:00+06:00",
  "data": {
    "request_type": "full_catalog",
    "updated_after": "2024-12-08T00:00:00+06:00"
  }
}
```

**Response:**
```json
{
  "code": 0,
  "message": "Success",
  "received_event_id": "evt_vwx678yza",
  "data": {
    "inventory": [
      {
        "seller_sku": "FL-NAVY-TS-S",
        "available_quantity": 150,
        "reserved_quantity": 5,
        "total_quantity": 155
      },
      {
        "seller_sku": "FL-NAVY-TS-M",
        "available_quantity": 200,
        "reserved_quantity": 10,
        "total_quantity": 210
      },
      {
        "seller_sku": "FL-NAVY-TS-L",
        "available_quantity": 180,
        "reserved_quantity": 3,
        "total_quantity": 183
      },
      {
        "seller_sku": "FL-BLACK-TS-S",
        "available_quantity": 0,
        "reserved_quantity": 0,
        "total_quantity": 0
      },
      {
        "seller_sku": "FL-BLACK-TS-M",
        "available_quantity": 95,
        "reserved_quantity": 5,
        "total_quantity": 100
      }
    ],
    "synced_at": "2024-12-09T08:00:05+06:00",
    "next_sync_allowed_at": "2024-12-09T09:00:05+06:00"
  }
}
```

---

## 7. Webhooks (Fabrilife → Partner)

Fabrilife sends webhooks to partner when inventory or product changes occur on Fabrilife's side.

**Why These Webhooks Are Essential:**
Fabrilife sells through multiple channels (own website, other marketplaces). When inventory or pricing changes on Fabrilife's end, partners must be notified to:
- Prevent overselling by updating stock on partner platform
- Maintain consistent pricing across all channels
- React to product availability changes in real-time

### 7.1 Webhook Endpoint (Partner Must Implement)

**Business Use Case:**
Partner must implement a webhook receiver to:
- Accept real-time updates from Fabrilife
- Process inventory changes immediately
- Update product listings on partner platform
- Maintain sync even when partner doesn't initiate the change

**Partner Webhook Endpoint:**
```
POST https://api.partner.com/webhooks/fabrilife
```

**Headers Fabrilife Will Send:**
```
Content-Type: application/json
X-Webhook-Signature: sha256={hmac_sha256_signature}
X-Webhook-Timestamp: {unix_timestamp}
X-Source: fabrilife
```

---

### 7.2 Stock Updated Webhook

**Event:** `stock.updated`

Sent when Fabrilife inventory changes (due to other sales channels, manual adjustments, returns, etc.)

**Business Use Case:**
The most frequent webhook from Fabrilife. Keeps partner inventory accurate when:
- Orders placed on Fabrilife website reduce available stock
- Orders canceled on other channels restore stock
- Manual inventory adjustments (damage, receiving, transfers)
- Returns processed on other channels
- Partner must update their listings to reflect new availability

**Payload:**
```json
{
  "event": "stock.updated",
  "event_id": "evt_fab001abc",
  "timestamp": "2024-12-09T15:30:00+06:00",
  "data": {
    "updates": [
      {
        "seller_sku": "FL-NAVY-TS-M",
        "shop_sku": "PARTNER-FL-NAVY-TS-M",
        "previous_quantity": 200,
        "new_quantity": 195,
        "change": -5,
        "available_for_sale": 195,
        "reason": "order_placed",
        "reason_detail": "Order from Fabrilife website",
        "reference_type": "order",
        "reference_id": "FL-ORD-98765"
      },
      {
        "seller_sku": "FL-BLACK-TS-L",
        "shop_sku": "PARTNER-FL-BLACK-TS-L",
        "previous_quantity": 75,
        "new_quantity": 80,
        "change": 5,
        "available_for_sale": 80,
        "reason": "order_cancelled",
        "reason_detail": "Customer cancelled order",
        "reference_type": "order",
        "reference_id": "FL-ORD-98760"
      }
    ]
  }
}
```

**Reason Codes:**

| Reason | Description |
|--------|-------------|
| `order_placed` | Stock deducted for new order |
| `order_cancelled` | Stock restored from cancelled order |
| `order_returned` | Stock restored from return |
| `manual_adjustment` | Manual stock correction |
| `inventory_received` | New inventory added |
| `inventory_damaged` | Stock removed due to damage |
| `inventory_transfer` | Stock moved between warehouses |

---

### 7.3 Price Updated Webhook

**Event:** `price.updated`

Sent when product pricing changes on Fabrilife.

**Business Use Case:**
Ensures pricing consistency across all sales channels:
- Regular price changes due to cost adjustments
- Sale/promotional pricing campaigns
- End of sale - removing special pricing
- Partner must update listings to maintain price parity
- Prevents customer confusion from price differences

**Payload:**
```json
{
  "event": "price.updated",
  "event_id": "evt_fab002def",
  "timestamp": "2024-12-09T16:00:00+06:00",
  "data": {
    "updates": [
      {
        "seller_sku": "FL-NAVY-TS-M",
        "shop_sku": "PARTNER-FL-NAVY-TS-M",
        "previous_price": 850.00,
        "new_price": 899.00,
        "previous_special_price": 699.00,
        "new_special_price": 749.00,
        "special_from_date": "2024-12-10",
        "special_to_date": "2024-12-31",
        "effective_immediately": true
      },
      {
        "seller_sku": "FL-NAVY-TS-L",
        "shop_sku": "PARTNER-FL-NAVY-TS-L",
        "previous_price": 850.00,
        "new_price": 899.00,
        "previous_special_price": 699.00,
        "new_special_price": null,
        "special_from_date": null,
        "special_to_date": null,
        "effective_immediately": true
      }
    ]
  }
}
```

**Partner Action:**
- Update product prices on partner platform
- Apply/remove sale pricing as specified

---

### 7.4 Product Status Updated Webhook

**Event:** `product.status_updated`

Sent when product availability changes.

**Business Use Case:**
Handles product lifecycle changes:
- Product discontinued - partner should delist
- Temporary unavailability - partner should hide listing
- Product reactivated - partner can relist
- Prevents selling products that are no longer available
- Maintains catalog consistency across platforms

**Payload:**
```json
{
  "event": "product.status_updated",
  "event_id": "evt_fab003ghi",
  "timestamp": "2024-12-09T17:00:00+06:00",
  "data": {
    "item_id": "PRD-001",
    "product_name": "Premium Cotton T-Shirt - Navy Blue",
    "previous_status": "live",
    "new_status": "inactive",
    "reason": "discontinued",
    "affected_skus": [
      {
        "seller_sku": "FL-NAVY-TS-S",
        "shop_sku": "PARTNER-FL-NAVY-TS-S",
        "status": "inactive"
      },
      {
        "seller_sku": "FL-NAVY-TS-M",
        "shop_sku": "PARTNER-FL-NAVY-TS-M",
        "status": "inactive"
      },
      {
        "seller_sku": "FL-NAVY-TS-L",
        "shop_sku": "PARTNER-FL-NAVY-TS-L",
        "status": "inactive"
      },
      {
        "seller_sku": "FL-NAVY-TS-XL",
        "shop_sku": "PARTNER-FL-NAVY-TS-XL",
        "status": "inactive"
      }
    ],
    "reactivation_expected": false,
    "replacement_item_id": null
  }
}
```

**Product Status Values:**

| Status | Description | Partner Action |
|--------|-------------|----------------|
| `live` | Product available for sale | Activate listing |
| `inactive` | Temporarily unavailable | Deactivate listing |
| `deleted` | Permanently removed | Remove listing |

---

### 7.5 New Product Added Webhook

**Event:** `product.created`

Sent when new products are added to the catalog.

**Business Use Case:**
Enables automatic catalog expansion on partner platform:
- New product launches automatically notified to partner
- Partner can quickly list new products without manual discovery
- Reduces time-to-market for new products on partner platform
- Ensures partner doesn't miss new inventory opportunities

**Payload:**
```json
{
  "event": "product.created",
  "event_id": "evt_fab004jkl",
  "timestamp": "2024-12-09T18:00:00+06:00",
  "data": {
    "item_id": "PRD-002",
    "status": "live",
    "attributes": {
      "name": "Premium Cotton Polo - Forest Green",
      "description": "Classic polo shirt in premium cotton",
      "brand": "Fabrilife",
      "category": "Men > Clothing > Polo Shirts"
    },
    "skus": [
      {
        "seller_sku": "FL-GREEN-POLO-S",
        "size": "S",
        "color": "Forest Green",
        "quantity": 100,
        "price": 1199.00,
        "special_price": 999.00
      },
      {
        "seller_sku": "FL-GREEN-POLO-M",
        "size": "M",
        "color": "Forest Green",
        "quantity": 150,
        "price": 1199.00,
        "special_price": 999.00
      }
    ],
    "images": [
      "https://cdn.fabrilife.com/products/green-polo-1.jpg",
      "https://cdn.fabrilife.com/products/green-polo-2.jpg"
    ]
  }
}
```

---

## 8. Error Handling

### 8.1 Error Response Format

All error responses follow this structure:

```json
{
  "code": "ERROR_CODE",
  "message": "Human readable error message",
  "request_id": "req_xyz789abc",
  "timestamp": "2024-12-09T10:30:00+06:00",
  "errors": [
    {
      "field": "seller_sku",
      "message": "SKU not found in catalog",
      "value": "INVALID-SKU-123"
    },
    {
      "field": "quantity",
      "message": "Quantity must be non-negative",
      "value": -5
    }
  ]
}
```

### 8.2 Error Codes

| Code | HTTP Status | Description | Resolution |
|------|-------------|-------------|------------|
| 0 | 200 | Success | - |
| 1001 | 401 | Invalid access token | Refresh token or re-authenticate |
| 1002 | 401 | Expired access token | Use refresh token |
| 1003 | 401 | Invalid signature | Check webhook signature calculation |
| 1004 | 403 | Insufficient permissions | Contact Fabrilife for access |
| 2001 | 400 | Invalid request format | Check JSON syntax |
| 2002 | 400 | Missing required field | Include all required fields |
| 2003 | 400 | Invalid field value | Check field value constraints |
| 2004 | 400 | Invalid date format | Use ISO 8601 format |
| 3001 | 404 | Order not found | Verify order_id |
| 3002 | 404 | Product/SKU not found | Verify seller_sku |
| 3003 | 404 | Order item not found | Verify order_item_id |
| 4001 | 409 | Invalid status transition | Check allowed transitions |
| 4002 | 409 | Insufficient stock | Check available inventory |
| 4003 | 409 | Order already processed | Order cannot be modified |
| 4004 | 409 | Duplicate request | Request already processed |
| 5001 | 429 | Rate limit exceeded | Reduce request frequency |
| 5002 | 500 | Internal server error | Retry with exponential backoff |
| 5003 | 503 | Service temporarily unavailable | Retry later |

### 8.3 Rate Limits

| Endpoint Category | Limit | Window |
|-------------------|-------|--------|
| Authentication | 10 requests | per minute |
| Read operations (GET) | 1000 requests | per minute |
| Write operations (POST/PUT) | 100 requests | per minute |
| Bulk operations | 20 requests | per minute |
| Webhook sends | 1000 events | per minute |
| Inventory sync requests | 1 request | per hour |

**Rate Limit Headers:**
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1702108260
```

### 8.4 Retry Strategy

For failed requests, implement exponential backoff:

```
Attempt 1: Immediate
Attempt 2: Wait 1 second
Attempt 3: Wait 2 seconds
Attempt 4: Wait 4 seconds
Attempt 5: Wait 8 seconds
Max attempts: 5
```

For 429 (Rate Limit) errors, use the `X-RateLimit-Reset` header to determine wait time.

---

## 9. SKU Mapping

SKU mapping links Fabrilife product variants to partner platform SKUs.

**Why SKU Mapping Is Essential:**
Fabrilife and partner platforms use different SKU systems. Mapping ensures:
- Correct inventory updates reach the right products
- Order items are correctly identified for fulfillment
- No confusion between similar products across platforms
- Accurate reporting and reconciliation

### 9.1 Get SKU Mappings

**Business Use Case:**
Retrieve existing mappings for verification and management:
- Audit current mappings during integration review
- Export mappings for offline analysis
- Verify mapping exists before processing orders
- Identify unmapped products that need attention

**Endpoint:** `GET /sku-mappings`

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| limit | int | No | Results per page (default: 100) |
| offset | int | No | Pagination offset |
| product_id | int | No | Filter by Fabrilife product ID |
| updated_after | datetime | No | Filter by update time |

**Response:**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "total_mappings": 1250,
    "mappings": [
      {
        "mapping_id": "MAP-001",
        "fabrilife_sku": "FL-NAVY-TS-M",
        "partner_sku": "PARTNER-FL-NAVY-TS-M",
        "fabrilife_product_id": 12345,
        "product_name": "Premium Cotton T-Shirt - Navy Blue",
        "color": "Navy Blue",
        "size": "M",
        "barcode": "8901234567891",
        "status": "active",
        "created_at": "2024-01-15T10:00:00+06:00",
        "updated_at": "2024-12-01T14:00:00+06:00"
      },
      {
        "mapping_id": "MAP-002",
        "fabrilife_sku": "FL-NAVY-TS-L",
        "partner_sku": "PARTNER-FL-NAVY-TS-L",
        "fabrilife_product_id": 12345,
        "product_name": "Premium Cotton T-Shirt - Navy Blue",
        "color": "Navy Blue",
        "size": "L",
        "barcode": "8901234567892",
        "status": "active",
        "created_at": "2024-01-15T10:00:00+06:00",
        "updated_at": "2024-12-01T14:00:00+06:00"
      }
    ]
  }
}
```

### 9.2 Create/Update SKU Mappings

**Business Use Case:**
Establish and maintain SKU relationships:
- Initial mapping setup during integration
- Adding mappings for new products
- Updating mappings if partner changes their SKU system
- Bulk mapping creation for efficient onboarding

**Endpoint:** `POST /sku-mappings`

**Request:**
```json
{
  "mappings": [
    {
      "fabrilife_sku": "FL-NAVY-TS-M",
      "partner_sku": "PARTNER-FL-NAVY-TS-M"
    },
    {
      "fabrilife_sku": "FL-NAVY-TS-L",
      "partner_sku": "PARTNER-FL-NAVY-TS-L"
    },
    {
      "fabrilife_sku": "FL-BLACK-TS-S",
      "partner_sku": "PARTNER-FL-BLACK-TS-S"
    }
  ]
}
```

**Response:**
```json
{
  "code": 0,
  "message": "Success",
  "data": {
    "created": 2,
    "updated": 1,
    "results": [
      {
        "fabrilife_sku": "FL-NAVY-TS-M",
        "partner_sku": "PARTNER-FL-NAVY-TS-M",
        "status": "created"
      },
      {
        "fabrilife_sku": "FL-NAVY-TS-L",
        "partner_sku": "PARTNER-FL-NAVY-TS-L",
        "status": "created"
      },
      {
        "fabrilife_sku": "FL-BLACK-TS-S",
        "partner_sku": "PARTNER-FL-BLACK-TS-S",
        "status": "updated"
      }
    ]
  }
}
```

### 9.3 Delete SKU Mapping

**Business Use Case:**
Remove outdated or incorrect mappings:
- Product discontinued and removed from partner platform
- Incorrect mapping created that needs removal
- Partner platform SKU changed and old mapping obsolete
- Cleaning up test mappings from sandbox

**Endpoint:** `DELETE /sku-mappings/{mapping_id}`

**Response:**
```json
{
  "code": 0,
  "message": "Mapping deleted successfully",
  "data": {
    "mapping_id": "MAP-001",
    "deleted_at": "2024-12-09T10:00:00+06:00"
  }
}
```

---

## 10. Implementation Checklist

### Partner Must Implement

#### APIs to Expose:

- [ ] `POST /auth/token/create` - Create access token from authorization code
- [ ] `POST /auth/token/refresh` - Refresh access token (requires current access_token in header)
- [ ] `GET /products` - Product listing with pagination and filters
- [ ] `GET /products/{item_id}` - Get single product by ID with all variants
- [ ] `GET /products/sku/{seller_sku}` - Get product by seller SKU
- [ ] `POST /products/price-quantity/update` - Bulk update stock and prices
- [ ] `GET /orders` - Order listing with time and status filters
- [ ] `GET /orders/{order_id}/items` - Get items for specific order
- [ ] `POST /orders/items/get` - Bulk get items for multiple orders
- [ ] `POST /orders/status/update` - Update order/item status
- [ ] `POST /orders/tracking/update` - Set tracking information
- [ ] `GET /sku-mappings` - Get SKU mappings
- [ ] `POST /sku-mappings` - Create/update SKU mappings

#### Webhooks to Send:

- [ ] `order.created` - When customer places order
- [ ] `order.status_updated` - When order status changes
- [ ] `order.item_removed` - When item removed from order
- [ ] `order.item_quantity_changed` - When item quantity modified
- [ ] `orders.bulk_updated` - For bulk status changes
- [ ] `inventory.sync_request` - Request current stock levels

#### Webhooks to Receive:

- [ ] `stock.updated` - Fabrilife inventory changes
- [ ] `price.updated` - Fabrilife price changes
- [ ] `product.status_updated` - Product availability changes
- [ ] `product.created` - New products added

### Data Requirements

#### Required Fields for Orders:

- Order ID (unique identifier)
- Order number (display number)
- Created timestamp
- Updated timestamp
- Status (must use defined statuses)
- Payment method
- Customer name and contact
- Billing address (all fields)
- Shipping address (all fields)
- Item details with SKU mapping
- Pricing breakdown (subtotal, shipping, discounts)

#### Required Fields for Products:

- Item ID (unique identifier)
- Product name
- SKU details (seller_sku, quantity, price)
- Size and color variants
- Images

---

## 11. Testing

### 11.1 Sandbox Environment

```
Base URL: https://sandbox-api.partner.com/v1
Webhook Target: https://sandbox.fabrilife.com/webhooks/partner/{partner_id}
```

### 11.2 Test Credentials

Partner should provide:
- Sandbox API credentials
- Test OAuth authorization flow
- Sample product catalog
- Webhook secret for signature verification

### 11.3 Test Scenarios

#### Authentication Tests:
1. Create token with valid authorization code
2. Create token with invalid/expired code (expect error)
3. Refresh token with valid access_token and refresh_token
4. Refresh token with expired access_token (expect error)
5. API call with invalid token (expect 401)

#### Product Tests:
1. Get products with pagination (limit=10, offset=0, then offset=10)
2. Get products filtered by status
3. Update stock for single SKU
4. Update stock for multiple SKUs
5. Update price with sale pricing
6. Update non-existent SKU (expect error)

#### Order Tests:
1. Get orders by created_after
2. Get orders by update_after
3. Get orders filtered by status
4. Get order items
5. Update order status through valid transitions
6. Update order with invalid transition (expect error)
7. Set tracking information

#### Webhook Tests:
1. Send order.created and verify stock deduction
2. Send order.status_updated (pending → canceled) and verify stock restoration
3. Send order.item_removed and verify partial stock restoration
4. Verify webhook signature validation
5. Test webhook retry on failure

### 11.4 Webhook Testing Tools

Use these tools for webhook development:
- [webhook.site](https://webhook.site) - Inspect incoming webhooks
- [ngrok](https://ngrok.com) - Expose local server for testing
- [Postman](https://postman.com) - API testing and documentation

### 11.5 Sample cURL Commands

**Create Token:**
```bash
curl -X POST https://api.partner.com/v1/auth/token/create \
  -H "Content-Type: application/json" \
  -d '{"code": "auth_code_here"}'
```

**Refresh Token:**
```bash
curl -X POST https://api.partner.com/v1/auth/token/refresh \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer current_access_token" \
  -d '{"refresh_token": "refresh_token_here"}'
```

**Get Products:**
```bash
curl -X GET "https://api.partner.com/v1/products?filter=live&limit=50&offset=0" \
  -H "Authorization: Bearer access_token"
```

**Get Product by ID:**
```bash
curl -X GET "https://api.partner.com/v1/products/PRD-001" \
  -H "Authorization: Bearer access_token"
```

**Get Product by SKU:**
```bash
curl -X GET "https://api.partner.com/v1/products/sku/FL-NAVY-TS-M" \
  -H "Authorization: Bearer access_token"
```

**Get Orders:**
```bash
curl -X GET "https://api.partner.com/v1/orders?created_after=2024-12-01T00:00:00%2B06:00&status=pending" \
  -H "Authorization: Bearer access_token"
```

**Update Stock:**
```bash
curl -X POST https://api.partner.com/v1/products/price-quantity/update \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer access_token" \
  -d '{
    "products": [
      {"seller_sku": "FL-NAVY-TS-M", "quantity": 100}
    ]
  }'
```

---

## Appendix A: Data Type Definitions

| Type | Format | Example |
|------|--------|---------|
| datetime | ISO 8601 with timezone | `2024-12-09T10:30:00+06:00` |
| date | YYYY-MM-DD | `2024-12-09` |
| price | float, 2 decimal places | `699.00` |
| quantity | integer, non-negative | `150` |
| sku | string, alphanumeric with hyphens | `FL-NAVY-TS-M` |
| order_id | string | `ORD-2024120901234` |
| phone | string, E.164 format preferred | `+8801712345678` |

---

## Appendix B: Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | December 2024 | Initial release |

---

**Document End**

For questions or support, contact: api-support@fabrilife.com
