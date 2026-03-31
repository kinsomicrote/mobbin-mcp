# Mobbin API Discovery Notes

## Architecture Overview

Mobbin is a **Next.js app** that uses **Supabase** as its backend. Unlike the old Swift wrapper's approach (direct Supabase RPC calls), the current site uses **Next.js API routes** (`/api/...`) that proxy to Supabase server-side.

- **Supabase project**: `ujasntkfphywizsdaapi.supabase.co`
- **Auth**: Google OAuth -> Supabase callback -> cookie-based session
- **Media CDN**: `bytescale.mobbin.com` (videos), Supabase Storage (images)
- **Notifications**: `api.knock.app`
- **Feature flags**: `cdn.growthbook.io`
- **Payments**: Stripe
- **Analytics**: Google Analytics, Pinterest, Sentry (`/monitoring`)

---

## Authentication

### Flow
1. User clicks "Continue with Google" on `/login`
2. Redirects to Google OAuth with `client_id=672621582021-a5cmbeo4rjqqj0tqo6u2ff614lmnjh2s.apps.googleusercontent.com`
3. Google redirects to `https://ujasntkfphywizsdaapi.supabase.co/auth/v1/callback`
4. Supabase redirects to `mobbin.com/api/auth/authenticate?redirect_to=/`
5. Auth token stored in cookies (NOT localStorage)

### Token Storage
Cookies named `sb-ujasntkfphywizsdaapi-auth-token.0` and `.1` (split across two cookies due to size).

Contains JSON with:
```json
{
  "access_token": "eyJ...<JWT>",
  "token_type": "bearer",
  "expires_in": 3600,
  "expires_at": 1774919235,
  "refresh_token": "<short_token>",
  "user": {
    "id": "<uuid>",
    "aud": "authenticated",
    "role": "authenticated",
    "email": "...",
    "app_metadata": { "provider": "google", "providers": ["google"] },
    "user_metadata": { "avatar_url": "...", "full_name": "...", "email": "..." }
  },
  "provider_token": "ya29...<google_oauth_token>"
}
```

### JWT Claims
```json
{
  "iss": "https://ujasntkfphywizsdaapi.supabase.co/auth/v1",
  "sub": "<user_uuid>",
  "aud": "authenticated",
  "exp": 1774919235,
  "iat": 1774915635,
  "email": "...",
  "role": "authenticated",
  "session_id": "<uuid>"
}
```

### Email OTP (alternative)
Login page also has an email field. Likely sends OTP via Supabase `/auth/v1/otp`. Not tested yet.

---

## Discovered API Endpoints

### GET Endpoints

#### `GET /api/searchable-apps/{platform}`
Platforms: `ios`, `android`, `web`

Returns array of all searchable apps for autocomplete:
```json
[
  {
    "id": "0038a133-fa16-4f6d-a453-da7885432b36",
    "platform": "ios",
    "appName": "Disney+",
    "appLogoUrl": "https://ujasntkfphywizsdaapi.supabase.co/storage/v1/object/public/content/app_logos/<uuid>.webp",
    "appTagline": "Unlimited entertainment",
    "keywords": ["streaming", "movies", "tv shows", "entertainment"],
    "previewScreens": [
      {
        "id": "<uuid>",
        "screenUrl": "https://ujasntkfphywizsdaapi.supabase.co/storage/v1/object/public/content/app_screens/<uuid>.png"
      }
    ]
  }
]
```

### POST Endpoints

#### `POST /api/filter-tags/fetch-dictionary-definitions`
Body: `{}` (no params needed)

Returns all filter categories with definitions and content counts:
```json
{
  "value": [
    {
      "id": "<uuid>",
      "slug": "appCategories",
      "displayName": "App Categories",
      "order": 0,
      "experience": "web",
      "subCategories": [
        {
          "id": "<uuid>",
          "entries": [
            {
              "id": "<uuid>",
              "displayName": "AI",
              "definition": "Artificial Intelligence (AI) apps...",
              "synonyms": ["Artificial Intelligence"],
              "hidden": false,
              "contentCounts": {
                "apps": { "web": 89 },
                "flows": { "web": 12485 },
                "screens": { "web": 49163 }
              },
              "exampleScreens": ["<supabase_storage_url>"]
            }
          ]
        }
      ]
    }
  ]
}
```

**Known category slugs**: `appCategories`, plus likely `screenPatterns`, `screenElements`, `flowActions`

#### `POST /api/popular-apps/fetch-popular-apps-with-preview-screens`
Needs specific body params (empty `{}` returns error). TODO: capture actual request body.

#### `POST /api/search-bar/fetch-trending-apps`
Needs specific body params. TODO: capture actual request body.

#### `POST /api/search-bar/fetch-trending-filter-tags`
Needs specific body params. TODO: capture actual request body.

#### `POST /api/search-bar/fetch-trending-text-in-screenshot-keywords`
TODO: capture request/response.

#### `POST /api/search-bar/fetch-searchable-sites`
TODO: capture request/response.

#### `POST /api/churnkey/fetch-churnkey-hash`
Internal billing/subscription management.

---

## URL Patterns

### App Pages
```
/apps/{app-slug}-{platform}-{app-uuid}/{version-uuid}/screens
/apps/{app-slug}-{platform}-{app-uuid}/{version-uuid}/flows
```
Example: `/apps/e-bay-ios-d09e2237-abe4-4090-b19c-abd9258bfaa1/68f1d5e4-fd92-4aff-94c7-2e7cd74d195e/screens`

### Discover/Browse
```
/discover/apps/{platform}/{tab}    # tab = latest|popular|top|animations
/discover/sites/{tab}              # tab = latest
```

### Search
```
/search/apps/{platform}?content_type={type}&sort={sort}&filter={filter}
```
- content_type: `apps`, `screens`, `ui-elements`, `flows`
- sort: `publishedAt`, `trending`
- filter examples:
  - `appCategories.AI`
  - `screenPatterns.Login`
  - `screenElements.Card`
  - `flowActions.Filtering+%26+Sorting`

### Saved/Collections
```
/saved/mobile/screens
/community/mobile/featured
```

---

## Image/Media URLs

### App Logos
```
https://ujasntkfphywizsdaapi.supabase.co/storage/v1/object/public/content/app_logos/{uuid}.webp
```

### App Screens
```
https://ujasntkfphywizsdaapi.supabase.co/storage/v1/object/public/content/app_screens/{uuid}.png
```

### Flow Videos
```
https://bytescale.mobbin.com/FW25bBB/video/mobbin.com/prod/content/app_flow_videos/{uuid}.mp4?f=mp4-h264&w=1920&hp=1920&...
```

### Dictionary Example Screens
```
https://ujasntkfphywizsdaapi.supabase.co/storage/v1/object/public/dictionary/web_images/{category}/{slug}-{index}-{uuid}.png
```

**Note**: Images appear to be publicly accessible (no auth required for Supabase storage public bucket).

---

## Still TODO
- [ ] Capture POST body params for trending/popular endpoints (intercept actual page requests)
- [ ] Search endpoint - trigger search and capture API calls
- [ ] App detail page - get screens/flows API shape
- [ ] Collections CRUD endpoints
- [ ] Token refresh mechanism
- [ ] Pagination patterns
