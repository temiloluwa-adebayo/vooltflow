# VooltFlow

> **AI-powered affiliate automation for WooCommerce. Type a niche — get a live store listing in under 60 seconds.**

[![Stack](https://img.shields.io/badge/Frontend-Next.js%2015-black?style=flat-square&logo=next.js)](https://nextjs.org)
[![AI](https://img.shields.io/badge/AI-GPT--4o-purple?style=flat-square)](https://openai.com)
[![Automation](https://img.shields.io/badge/Automation-n8n-orange?style=flat-square)](https://n8n.io)
[![Database](https://img.shields.io/badge/Database-Supabase-green?style=flat-square&logo=supabase)](https://supabase.com)
[![Status](https://img.shields.io/badge/Status-Production-brightgreen?style=flat-square)]()

---

## Table of Contents

- [Overview](#overview)
- [The Problem It Solves](#the-problem-it-solves)
- [How It Works](#how-it-works)
- [System Architecture](#system-architecture)
- [Workflow 1 — Product Discovery](#workflow-1--product-discovery)
- [Workflow 2 — WooCommerce Publisher](#workflow-2--woocommerce-publisher)
- [Frontend Application](#frontend-application)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)

---

## Overview

VooltFlow is a two-part system — an n8n automation backend and a Next.js frontend — that completely automates the affiliate product research and listing process for WooCommerce store owners.

A user types a product niche (e.g. `"Wireless Earbuds"`). VooltFlow simultaneously searches Amazon, Jumia, and Konga, filters products by quality score, generates proper affiliate links, finds a recent viral YouTube Short proving demand, lets the user preview the product, and publishes it directly to their WooCommerce store with a GPT-4o-written product description — in one click.

**Total time from niche input to live WooCommerce listing: under 60 seconds.**

## Screenshots

![Dashboard](assets/Screenshot%202026-03-19%20223609.png)
![Leads Table](assets/Screenshot%202026-03-19%20225110.png)
![More View](assets/Screenshot%202026-03-19%20225139.png)
![Leads Table](assets/Screenshot%202026-03-19%20225250.png)

---

## The Problem It Solves

Traditional affiliate marketing is a deeply manual, time-consuming process:

1. Browse multiple marketplaces individually to find products
2. Check ratings, reviews, and prices to gauge quality
3. Build affiliate links with correct tags and IDs
4. Search YouTube or TikTok to verify the product is trending
5. Write a compelling product description
6. Manually create a WooCommerce listing

VooltFlow automates all six steps. The only human decision remaining is whether to publish a product or skip it.

---

## How It Works

```
User types a niche
        │
        ▼
n8n Workflow 1: "The Affiliate"
        │
        ├── Search Amazon (parallel)
        ├── Search Jumia (parallel)
        └── Search Konga (parallel)
                │
                ▼
        Quality Filter (rating ≥ 4.0, reviews ≥ 20)
                │
                ▼
        Affiliate Link Generation
                │
                ▼
        YouTube Shorts Search (last 2 months)
                │
                ▼
        Return enriched product list to frontend
                │
                ▼
User previews products + selects one
        │
        ▼
n8n Workflow 2: "The Publisher"
        │
        ├── Download product image
        ├── Upload image to WordPress Media Library
        ├── GPT-4o writes HTML product description
        └── Create WooCommerce external/affiliate product
                │
                ▼
        Live product on WooCommerce store ✓
```

---

## System Architecture

```
┌──────────────────────────────────────┐
│        FRONTEND (Next.js 15)         │
│  Landing · Auth · Dashboard · Products │
└──────────────┬───────────────────────┘
               │ Webhook calls
               ▼
┌──────────────────────────────────────┐
│       AUTOMATION (n8n — self-hosted) │
│  Workflow 1: The Affiliate           │
│  Workflow 2: The Publisher           │
└──────┬───────────────────────────────┘
       │
       ├── Serper.dev API (Google Shopping + YouTube)
       ├── OpenAI GPT-4o (Product Descriptions)
       ├── WordPress REST API (Media Upload)
       └── WooCommerce REST API (Listing Creation)
               │
               ▼
┌──────────────────────────────────────┐
│         DATABASE (Supabase)          │
│  users · saved_products · history    │
└──────────────────────────────────────┘
```

---

## Workflow 1 — Product Discovery

**Endpoint:** `POST /webhook/find-products`  
**Input:** `{ "niche": "Wireless Earbuds", "user_id": "uuid" }`

### Node Pipeline

| Node | Purpose |
|---|---|
| Webhook | Receives niche and user_id from frontend |
| Code (Extract) | Isolates the niche field for clean downstream usage |
| HTTP: Amazon Search | Queries Serper.dev Google Shopping for `"Amazon {niche}"` |
| HTTP: Jumia Search | Queries Serper.dev Google Shopping for `"Jumia {niche}"` |
| HTTP: Konga Search | Queries Serper.dev Google Shopping for `"Konga {niche}"` |
| Parse Amazon / Jumia / Konga | Normalises all results into a consistent product shape |
| Merge → Merge1 | Combines all three platform results into a single stream |
| Quality Filter | Removes products with rating < 4.0 or reviews < 20. Amazon price filter: $20–$80 |
| Affiliate Generator | Builds proper affiliate links: Amazon ASIN links, Jumia/Konga tagged URLs |
| YouTube Search | Searches for recent YouTube Shorts (`site:youtube.com/shorts`) within last 2 months |
| Parse YouTube | Attaches `youtube_link`, `video_title`, and `video_context` to each product |
| Respond to Webhook | Returns the full enriched product array to the frontend |

### Product Data Shape (Returned)
```json
{
  "platform": "Amazon",
  "product_name": "Sony WF-1000XM5 Wireless Earbuds",
  "price": "$279.99",
  "affiliate_link": "https://www.amazon.com/dp/B0C33XXXXX?tag=vooltgroup-20",
  "image_url": "https://...",
  "rating": "4.8",
  "reviews": "12400",
  "youtube_link": "https://www.youtube.com/shorts/abc123",
  "video_title": "This earbuds changed everything 🎧",
  "video_context": "Sony XM5 review — best noise cancelling 2024"
}
```

---

## Workflow 2 — WooCommerce Publisher

**Endpoint:** `POST /webhook/publish-products`

### Node Pipeline

| Node | Purpose |
|---|---|
| Webhook | Receives selected product data from frontend |
| Code (Read Body) | Extracts product fields from webhook body |
| Limit | Caps processing at 10 products per run |
| Download Image | Downloads product image as binary data |
| Format Image | Validates minimum size (500 bytes), renames file |
| Upload Image | POSTs binary to WordPress REST API `/wp-json/wp/v2/media` |
| WooCommerce | Creates external/affiliate product with name, price, description, image, and affiliate URL |
| Respond to Webhook | Returns creation result to frontend |

---

## Frontend Application

Built with **Next.js 15**, **TypeScript**, **Tailwind CSS v4**, and **Supabase Auth**.

### Routes

| Route | Description |
|---|---|
| `/` | Marketing landing page with hero, stats, and features |
| `/auth/login` | Email/password login via Supabase Auth |
| `/auth/signup` | New user registration |
| `/dashboard` | Main app shell with sidebar navigation |
| `/dashboard/products` | Core product discovery and publishing interface |
| `/dashboard/history` | Log of all past automation runs |
| `/dashboard/settings` | WooCommerce and affiliate ID configuration |

### Products Page (Core Interface)
1. User enters a product niche in the search field
2. Clicks "Find Products" — triggers Workflow 1 via webhook
3. Results appear as product cards showing: image, name, price, platform badge, star rating, review count, and embedded YouTube Short
4. User clicks "Publish to Store" on any product — triggers Workflow 2 via webhook
5. Success state shows a direct link to the live WooCommerce listing

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (App Router), TypeScript |
| Styling | Tailwind CSS v4 |
| Authentication | Supabase Auth |
| Database | Supabase (PostgreSQL) |
| Automation | n8n (self-hosted on Hostinger) |
| Product Search | Serper.dev Google Shopping API |
| Video Discovery | Serper.dev Videos API (YouTube Shorts) |
| AI Descriptions | OpenAI GPT-4o |
| Store Integration | WooCommerce REST API + WordPress REST API |
| Hosting | Vercel |

---

## Getting Started

### Prerequisites
- Node.js 18+
- Supabase project
- n8n instance (self-hosted or cloud)
- Serper.dev API key
- OpenAI API key
- WooCommerce store with REST API enabled

### Installation

```bash
git clone https://github.com/temiloluwa-adebayo/vooltflow.git
cd vooltflow
npm install
cp .env.example .env.local
# Fill in your environment variables
npm run dev
```

---

## Environment Variables

```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

NEXT_PUBLIC_N8N_AFFILIATE_WEBHOOK=https://your-n8n.cloud/webhook/find-products
NEXT_PUBLIC_N8N_PUBLISHER_WEBHOOK=https://your-n8n.cloud/webhook/publish-products
```

WooCommerce credentials and affiliate IDs are configured per user through the Settings page inside the application.

---

## Author

**Temiloluwa Adebayo** — AI Software Engineer  
[GitHub](https://github.com/temiloluwa-adebayo) · [LinkedIn](https://linkedin.com/in/temiloluwa-adebayo-4843ba377)
