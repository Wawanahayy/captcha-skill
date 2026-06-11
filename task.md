```

┌─ Request ──────────────────────────────────┐
│  GET /turnstile?url=...&sitekey=...         │
│  GET /recaptchaV3?url=...&sitekey=...       │
│  GET /clearance?url=...                     │
│  GET /aws-token?url=...                     │
└────────────────┬───────────────────────────┘
                 │
                 ▼
      ┌──────────────────┐
      │  Return task_id   │  ← 202 Accepted
      │  (async, no blok) │
      └────────┬─────────┘
               │
               ▼
      ┌──────────────────┐
      │  Pool Manager     │  ←  page pool
      │  - thread × page  │     (1×1 = 1 slot default)
      │  - proxy rotation │
      │  - periodic cleanup│    tiap 10 menit
      └────────┬─────────┘
               │
               ▼
      ┌──────────────────┐
      │  Camoufox Browser │  ← Playwright-based
      │  - stealth        │     fingerprint randomization
      │  - headless       │
      │  - no uBlock      │
      └────────┬─────────┘
               │
         solver logic:
               │
   ┌───────────┼───────────┐
   ▼           ▼           ▼
┌──────┐ ┌──────────┐ ┌─────────┐
│Turnstile│ │reCAPTCHAv3│ │Clearance│
│        │ │          │ │+ AWS   │
│ inject │ │ inject   │ │navigate │
│ widget │ │ api.js   │ │→ poll   │
│ click  │ │execute() │ │cookies  │
│ poll   │ │get token │ │         │
│ value  │ │          │ │         │
└───────┘ └──────────┘ └─────────┘
               │
               ▼
      ┌──────────────────┐
      │  save  ke        │
      │  self.results[]  │
      └────────┬─────────┘
               │
         client polling:
         GET /result?id=xxx
               │
               ▼
          ✅ 200 /token/
          ❌ 422 error
          ⏳ 202 still processing
```
