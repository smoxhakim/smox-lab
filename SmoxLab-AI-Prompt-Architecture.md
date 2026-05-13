# Smox Lab — AI Prompt Architecture

**Version:** 1.0  
**Status:** Draft  
**Last Updated:** 2026-05-03  
**Owner:** Smox Lab AI Team

---

## Table of Contents

1. [Overview](#overview)
2. [AI Use Cases in Smox Lab](#ai-use-cases-in-driplab)
3. [Prompt Pipeline Architecture](#prompt-pipeline-architecture)
4. [Prompt Templates](#prompt-templates)
   - [Design Generation (DALL·E 3)](#1-design-generation-dalle-3)
   - [Design Enhancement (GPT-4o Vision)](#2-design-enhancement-gpt-4o-vision)
   - [Style Suggestion (GPT-4o-mini)](#3-style-suggestion-gpt-4o-mini)
   - [Content Moderation](#4-content-moderation)
   - [Design Description Generator](#5-design-description-generator)
5. [Prompt Enhancer Module](#prompt-enhancer-module)
6. [Style Profiles](#style-profiles)
7. [Provider Configuration](#provider-configuration)
8. [Credit System & Cost Control](#credit-system--cost-control)
9. [Caching Strategy](#caching-strategy)
10. [Safety & Guardrails](#safety--guardrails)
11. [Evaluation & Quality Control](#evaluation--quality-control)

---

## Overview

This document defines the AI prompt strategies, templates, processing pipelines, and safety guardrails used in Smox Lab. The AI layer is responsible for:

- **Generating** original apparel designs from text prompts.
- **Enhancing** user-created designs using visual AI.
- **Suggesting** styles, colors, and layouts intelligently.
- **Moderating** all user-generated content before printing or publishing.
- **Describing** designs for marketplace SEO and discoverability.

All prompts are processed through a **Prompt Pipeline** inside the AI Service before being dispatched to any external provider. This pipeline ensures consistency, safety, quality, and cost efficiency.

---

## AI Use Cases in Smox Lab

| Use Case | Trigger | Provider | User-Facing |
|---|---|---|---|
| Design generation from text | Customer types a prompt on kiosk or web | OpenAI DALL·E 3 | Yes — image options shown on canvas |
| Design style enhancement | Customer clicks "Enhance" on a design | GPT-4o Vision | Yes — refined image returned |
| Color palette suggestion | Customer selects a style preference | GPT-4o-mini (text) | Yes — color swatches shown |
| Layout suggestion | AI recommends design placement on garment | GPT-4o-mini (text) | Yes — placement guide overlay |
| Background removal | Image uploaded or AI-generated | rembg (self-hosted) | Transparent, automatic |
| Content moderation | Every design before order / publish | OpenAI Moderation API | No — silent gate |
| Design description | Creator publishes a design | GPT-4o-mini | Yes — auto-filled description field |
| Trend-based recommendations | Web homepage / account dashboard | GPT-4o-mini + internal data | Yes — "You might like" section |

---

## Prompt Pipeline Architecture

Every user-facing AI request passes through a sequential pipeline inside the AI Service before reaching an external API.

```
[Raw User Input]
        │
        ▼
┌──────────────────────────────────────┐
│  Step 1: INPUT SANITIZER             │
│  - Remove HTML/injection patterns    │
│  - Normalize whitespace              │
│  - Strip profanity (optional soft)   │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Step 2: SAFETY PRE-FILTER           │
│  - Check against banned keyword list │
│  - Detect PII (names, faces, brands) │
│  - Flag for moderation if uncertain  │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Step 3: PROMPT ENHANCER             │
│  - Detect style intent               │
│  - Append style modifiers            │
│  - Add print-quality suffixes        │
│  - Apply product-context injection   │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Step 4: CACHE CHECK                 │
│  - Hash enhanced prompt              │
│  - Check Redis for cached result     │
│  - Return cached result if hit       │
└──────────────┬───────────────────────┘
               │ cache miss
               ▼
┌──────────────────────────────────────┐
│  Step 5: API DISPATCH                │
│  - Route to correct provider         │
│  - Set model, size, quality params   │
│  - Attach system instructions        │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Step 6: POST-PROCESSOR              │
│  - Download result images            │
│  - Run background removal (rembg)    │
│  - Validate image dimensions / DPI   │
│  - Store in S3                       │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  Step 7: MODERATION CHECK            │
│  - Run OpenAI Moderation on result   │
│  - Reject if flagged                 │
│  - Log incident                      │
└──────────────┬───────────────────────┘
               │
               ▼
        [Return to Client]
```

---

## Prompt Templates

### 1. Design Generation (DALL·E 3)

**Used when:** A customer enters a text prompt to generate a custom design.

#### System Context (sent as `system` message or prepended)
```
You are a professional graphic design assistant specializing in 
print-on-demand apparel. Your role is to generate clean, 
print-ready artwork suitable for DTG (Direct-to-Garment) printing.
```

#### Base Prompt Template
```
{USER_PROMPT}, {STYLE_MODIFIER}, 
on a pure white or transparent background, 
no gradients unless specified, 
centered composition, 
high contrast edges, 
{PRODUCT_TYPE} graphic design, 
vector-like illustration, 
professional print quality
```

#### Variable Substitution Examples

| Variable | Example Values |
|---|---|
| `{USER_PROMPT}` | "a wolf howling at the moon" |
| `{STYLE_MODIFIER}` | "in a minimalist line art style" (from style selector) |
| `{PRODUCT_TYPE}` | "t-shirt" / "hoodie" / "cap" |

#### Full Composed Prompt Example
```
a wolf howling at the moon, in a minimalist line art style, 
on a pure white or transparent background, 
no gradients unless specified, 
centered composition, 
high contrast edges, 
t-shirt graphic design, 
vector-like illustration, 
professional print quality
```

#### DALL·E 3 API Parameters
```json
{
  "model": "dall-e-3",
  "prompt": "<composed_prompt>",
  "n": 1,
  "size": "1024x1024",
  "quality": "hd",
  "style": "vivid",
  "response_format": "url"
}
```

> Note: DALL·E 3 only supports `n=1`. To get 3 options, make 3 sequential calls with slight style variation on each. Use the **Style Variants** system below.

#### Style Variants (3 Options Strategy)
To give customers 3 design choices from a single prompt, dispatch 3 calls with varied style suffixes:

```
Call 1 → append: "bold graphic style, strong outlines, street fashion aesthetic"
Call 2 → append: "watercolor texture, soft artistic style, hand-drawn feel"
Call 3 → append: "geometric abstract style, modern, clean, minimal"
```

Run all 3 calls in parallel with `Promise.all()`.

---

### 2. Design Enhancement (GPT-4o Vision)

**Used when:** A customer uploads a rough sketch or basic design and clicks "Enhance my design".

#### System Prompt
```
You are an expert fashion design illustrator. 
A customer has provided a rough design or sketch for a custom apparel piece.
Your task is to describe a refined, professional version of this design 
that would look great when printed on a garment.
Be specific about style, colors, composition, and visual elements.
Output only the enhanced image description — no explanations or caveats.
```

#### User Message Template
```
Here is the customer's current design: [IMAGE_ATTACHED]

Product: {PRODUCT_TYPE}
Color of garment: {GARMENT_COLOR}
Customer's style preference: {STYLE_PREFERENCE}

Please generate a refined, professional version of this design 
suitable for DTG printing on a {GARMENT_COLOR} {PRODUCT_TYPE}.
```

#### API Call
```json
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "system",
      "content": "<system_prompt>"
    },
    {
      "role": "user",
      "content": [
        {
          "type": "image_url",
          "image_url": { "url": "<S3_SIGNED_URL>", "detail": "high" }
        },
        {
          "type": "text",
          "text": "<user_message_template>"
        }
      ]
    }
  ],
  "max_tokens": 500
}
```

The response (text description) is then passed to DALL·E 3 as the generation prompt.

---

### 3. Style Suggestion (GPT-4o-mini)

**Used when:** The customer selects a vibe/mood and wants AI to suggest a color palette and layout.

#### System Prompt
```
You are a fashion and streetwear design consultant. 
Your role is to suggest color palettes, typography styles, and layout 
recommendations for custom apparel designs.
Always respond in valid JSON. No explanations.
```

#### User Message Template
```
The customer wants a design with the following vibe: "{USER_VIBE}"
Product: {PRODUCT_TYPE}
Base garment color: {GARMENT_COLOR}

Suggest:
1. A 5-color palette (hex codes)
2. A font style description
3. A layout recommendation (where to place the design on the garment)
4. 3 example design prompt ideas based on this vibe

Respond in this exact JSON format:
{
  "palette": ["#hex1", "#hex2", "#hex3", "#hex4", "#hex5"],
  "font_style": "description of font mood",
  "layout": "description of placement",
  "prompt_ideas": ["idea 1", "idea 2", "idea 3"]
}
```

#### Response Handling
```typescript
interface StyleSuggestion {
  palette: string[];      // 5 hex colors
  font_style: string;
  layout: string;
  prompt_ideas: string[]; // 3 clickable prompt starters on kiosk
}
```

---

### 4. Content Moderation

**Used on:** Every design before order confirmation, and every creator design before marketplace listing.

#### Primary: OpenAI Moderation API

```typescript
const moderation = await openai.moderations.create({
  input: userPrompt,                     // Text prompt check
  // For image moderation, use GPT-4o Vision (see below)
});

if (moderation.results[0].flagged) {
  throw new ForbiddenException('Design content is not allowed.');
}
```

#### Categories Checked

| Category | Action |
|---|---|
| `hate` | Hard block |
| `sexual` | Hard block |
| `violence` | Hard block |
| `self-harm` | Hard block |
| `harassment` | Hard block |
| `illicit` | Hard block |

#### Image Moderation (GPT-4o Vision Fallback)

When a generated image must be moderated visually:

```
System: You are a content moderation system for a fashion retail platform.
Your job is to evaluate whether a clothing design image is appropriate 
for all-ages retail display.

User: [IMAGE_ATTACHED]
Does this image contain any of the following? Answer only with JSON.
- Explicit sexual content
- Graphic violence or gore  
- Hate symbols or discriminatory imagery
- Real identifiable people
- Copyrighted brand logos or characters

{
  "flagged": true | false,
  "reasons": ["reason1", "reason2"]
}
```

#### Brand/Logo Detection

Before every print, run a secondary check for recognized brand logos:

```
System: You are a trademark protection system.
User: [IMAGE_ATTACHED]
Does this image contain any recognizable brand logos, 
trademarks, or copyrighted characters (e.g. Nike, Adidas, 
Disney, sports team logos)?

Respond: { "detected": true | false, "brands": ["brand1"] }
```

---

### 5. Design Description Generator

**Used when:** A creator publishes a design to the marketplace. Auto-generates the listing description.

#### System Prompt
```
You are a copywriter for a fashion design marketplace.
Write compelling, SEO-friendly product descriptions for 
custom apparel designs that will be printed on-demand.
Keep descriptions between 60-100 words.
Use a confident, creative tone. No bullet points.
Output only the description text, nothing else.
```

#### User Message Template
```
Design image: [IMAGE_ATTACHED]
Style tags chosen by creator: {TAGS}
Product type: {PRODUCT_TYPE}

Write a marketplace description for this design.
```

#### Usage
```typescript
const description = await aiService.generateDesignDescription({
  imageUrl: design.previewUrl,
  tags: design.tags,
  productType: 'tshirt'
});
// Auto-fills the description field in creator publish form
// Creator can edit before submitting
```

---

## Prompt Enhancer Module

The Prompt Enhancer is a critical internal module that transforms raw user input into optimized, print-ready prompts. It runs as Step 3 in the pipeline.

### Implementation

```typescript
// apps/ai-service/src/prompt-enhancer/enhancer.service.ts

export class PromptEnhancerService {
  
  enhance(input: PromptEnhancerInput): string {
    const parts: string[] = [input.rawPrompt];
    
    // 1. Apply style modifier from user's selection
    if (input.styleId) {
      parts.push(STYLE_PROFILES[input.styleId].modifier);
    }

    // 2. Apply garment color context
    parts.push(this.getColorContext(input.garmentColor));

    // 3. Apply product-specific composition rules
    parts.push(PRODUCT_COMPOSITION_RULES[input.productType]);

    // 4. Always append print-quality base
    parts.push(PRINT_QUALITY_BASE);

    return parts.join(', ');
  }

  private getColorContext(garmentColor: string): string {
    const isDark = this.isColorDark(garmentColor);
    return isDark
      ? 'high contrast design optimized for dark fabric, bright or light colored elements'
      : 'design optimized for light fabric, bold outlines, strong contrast';
  }
}

const PRINT_QUALITY_BASE = 
  'centered composition, no gradients unless watercolor style, ' +
  'clean edges, professional print quality, ' +
  'no text unless requested, no borders or frames';

const PRODUCT_COMPOSITION_RULES: Record<string, string> = {
  tshirt:  't-shirt chest graphic, square or portrait format, max 30cm print area',
  hoodie:  'hoodie center chest or full back graphic, bold and visible',
  cap:     'embroidery-style patch design, simple icon or monogram, small format',
  tote:    'tote bag print, landscape or square format, statement graphic',
};
```

---

## Style Profiles

Style profiles are pre-defined design aesthetics that users can select on the kiosk. Each profile injects a specific style modifier into the prompt.

```typescript
export const STYLE_PROFILES: Record<string, StyleProfile> = {

  minimalist: {
    label: 'Minimalist',
    description: 'Clean lines, simple shapes, less is more',
    modifier: 'minimalist style, single color or duotone, clean geometric shapes, negative space',
    examplePrompt: 'a mountain silhouette',
  },

  streetwear: {
    label: 'Streetwear',
    description: 'Urban, bold, graphic — inspired by NYC and Tokyo street fashion',
    modifier: 'streetwear aesthetic, bold typography optional, urban graphic style, high contrast, graffiti or sticker art influence',
    examplePrompt: 'a city skyline at night',
  },

  vintage: {
    label: 'Vintage',
    description: 'Retro feel, worn textures, classic Americana or 70s style',
    modifier: 'vintage retro style, distressed texture, muted warm tones, 1970s or 1980s aesthetic, badge or crest composition',
    examplePrompt: 'a classic motorcycle',
  },

  anime: {
    label: 'Anime / Manga',
    description: 'Japanese animation-inspired, bold lines, expressive style',
    modifier: 'anime illustration style, bold black outlines, cel-shaded coloring, dynamic composition, manga-inspired',
    examplePrompt: 'a samurai in the rain',
  },

  abstract: {
    label: 'Abstract',
    description: 'Non-representational, shapes, colors, and forms',
    modifier: 'abstract art style, geometric or fluid shapes, bold color blocking, modern art aesthetic',
    examplePrompt: 'energy and motion',
  },

  watercolor: {
    label: 'Watercolor',
    description: 'Soft, artistic, hand-painted feel',
    modifier: 'watercolor painting style, soft edges, translucent layered colors, artistic brush textures, gentle gradients',
    examplePrompt: 'a blooming flower',
  },

  glitch: {
    label: 'Glitch / Cyber',
    description: 'Digital distortion, neon, futuristic',
    modifier: 'glitch art aesthetic, neon colors, digital distortion effects, cyberpunk style, dark background elements',
    examplePrompt: 'a face fragmenting into pixels',
  },

  nature: {
    label: 'Nature & Botanical',
    description: 'Plants, animals, organic forms',
    modifier: 'botanical illustration style, nature-inspired, detailed linework, earthy or vivid natural colors',
    examplePrompt: 'tropical leaves and flowers',
  },
};
```

---

## Provider Configuration

### OpenAI Configuration

```typescript
// apps/ai-service/src/config/ai.config.ts

export const AI_CONFIG = {
  openai: {
    apiKey: process.env.OPENAI_API_KEY,
    models: {
      imageGeneration: 'dall-e-3',
      vision:          'gpt-4o',
      text:            'gpt-4o-mini',
      moderation:      'omni-moderation-latest',
    },
    limits: {
      maxTokens: {
        vision:  500,
        text:    300,
      },
      imageSize:    '1024x1024',
      imageQuality: 'hd',
      imageStyle:   'vivid',
    },
    timeout: 30_000,  // 30 seconds
    retries: 2,
  },

  stableDiffusion: {
    // Phase 2 — self-hosted SDXL endpoint
    endpoint: process.env.SD_ENDPOINT,
    model:    'stable-diffusion-xl-base-1.0',
    enabled:  process.env.SD_ENABLED === 'true',
  },

  rembg: {
    // Self-hosted Python microservice
    endpoint: process.env.REMBG_ENDPOINT || 'http://rembg-service:7000',
    timeout:  15_000,
  },
};
```

### Fallback Strategy

```
Primary: OpenAI DALL·E 3
    │
    ├── Timeout (>30s) or API error
    │
    └── Fallback: Stable Diffusion (if self-hosted and enabled)
            │
            ├── Also fails
            │
            └── Return error to user:
                "Design generation is temporarily unavailable.
                 Please use a template or upload your own image."
```

---

## Credit System & Cost Control

### Credit Model

| Action | Credits Consumed |
|---|---|
| AI design generation (1 image set of 3) | 3 credits |
| Design enhancement (GPT-4o Vision) | 2 credits |
| Style suggestion (GPT-4o-mini) | 0 credits (free) |

### Credit Allocation

| User Type | Credits |
|---|---|
| Anonymous kiosk session | 2 free credits |
| Registered customer (free) | 5 credits/month |
| Premium customer | 30 credits/month |
| Creator account | 20 credits/month |
| Add-on purchase | Sold in bundles of 10/25/50 |

### Cost Estimation (OpenAI)

| Model | Cost | Per Action |
|---|---|---|
| DALL·E 3 HD 1024×1024 | $0.080 / image | $0.24 for 3 variants |
| GPT-4o Vision | ~$0.01 / call | $0.01 for enhancement |
| GPT-4o-mini | ~$0.001 / call | Negligible |

> **Target margin:** Each AI credit costs Smox Lab ~$0.08. Sell credits at $0.50 each → ~6× margin on AI cost, covering infrastructure and overhead.

---

## Caching Strategy

Identical prompts should not trigger duplicate API calls. Use Redis with a content hash as the key.

```typescript
async generateDesign(input: GenerateInput): Promise<GenerateResult> {
  const enhancedPrompt = this.enhancer.enhance(input);
  const cacheKey = `ai:design:${createHash('sha256').update(enhancedPrompt).digest('hex')}`;

  // Check cache
  const cached = await this.redis.get(cacheKey);
  if (cached) {
    await this.metricsService.recordCacheHit();
    return JSON.parse(cached);
  }

  // Generate
  const result = await this.callDallE(enhancedPrompt);
  
  // Store in cache (24h TTL)
  await this.redis.set(cacheKey, JSON.stringify(result), 'EX', 86_400);
  
  return result;
}
```

### Cache TTLs

| Cache Type | TTL | Notes |
|---|---|---|
| Design generation result | 24 hours | Same prompt → same result |
| Style suggestions | 6 hours | Palette + prompt ideas |
| Design descriptions | 7 days | Marketplace listings |
| Moderation results | 1 hour | For repeated uploads |

---

## Safety & Guardrails

### Banned Keywords List (Pre-filter)

```typescript
export const BANNED_KEYWORDS = [
  // Brand trademarks
  'nike', 'adidas', 'gucci', 'louis vuitton', 'supreme',
  'disney', 'marvel', 'dc comics', 'pokemon', 'nike swoosh',
  
  // Violence / harm
  'kill', 'murder', 'suicide', 'bomb', 'weapon',
  
  // Hate content
  'nazi', 'kkk', 'swastika',
  
  // NSFW
  'nude', 'naked', 'sex', 'porn',
  
  // Real people
  // (handled separately by entity detection)
];
```

> Note: This is a soft pre-filter. The OpenAI Moderation API is the authoritative gate. The keyword list reduces unnecessary API calls and latency.

### Real Person Detection

Prompts containing proper nouns that resemble person names are flagged and checked:

```
Heuristic: if prompt contains a proper noun AND 
           ("face", "portrait", "photo", "wearing") 
           → send to GPT-4o-mini for person-detection check before generation
```

### Logging & Incidents

- All flagged prompts are logged to a `moderation_incidents` table (prompt text, timestamp, flag reason, userId/sessionId).
- Repeated violations from the same session/user trigger a session ban.
- Incident log is reviewed weekly by the Smox Lab trust & safety team.

---

## Evaluation & Quality Control

### Prompt Quality Metrics

Track these metrics per AI generation in the analytics pipeline:

| Metric | Description |
|---|---|
| `generation_accepted_rate` | % of generated images that the customer uses in their design |
| `regeneration_rate` | % of sessions that request >1 generation |
| `style_distribution` | Which style profiles are most used |
| `cache_hit_rate` | % of requests served from cache (cost efficiency) |
| `moderation_flag_rate` | % of prompts blocked by moderation |
| `avg_generation_latency` | Time from prompt submission to image display |

### A/B Testing Prompts

The Prompt Enhancer supports A/B testing of suffix variants:

```typescript
const variant = this.abTest.getVariant(sessionId, 'print_quality_suffix');
// variant: 'control' | 'variant_a' | 'variant_b'

const suffix = PRINT_QUALITY_VARIANTS[variant];
```

Test generation acceptance rate per variant to continuously improve the default suffix.

### Monthly Prompt Review

- Review top 50 most-used prompts monthly.
- Identify patterns in rejected/regenerated outputs.
- Update style profiles and enhancer rules based on findings.
- Document changes in this document with version bump.
