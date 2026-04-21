# Hugo Personal Assistant Plan

## Summary
Build a `Gi` personal-site assistant as a `Hugo + Azure Functions + OpenAI` system, with no Microsoft Graph integration and no lead storage. The assistant will appear on all public pages, answer only from site-owned context, qualify booking intent lightly, and then redirect users to the existing `business` or `coffee` Microsoft booking links already defined in `hugo.toml`.

Chosen defaults:
- Default behavior: bilingual, replies in the user's language
- Widget scope: all pages
- Booking flow: light qualification
- Scope: strict site context only
- Conversion goal: balanced between booking and content navigation
- Data handling: no persistence or email notifications in v1

## Implementation Changes
### 1. Backend: Azure Function chat API
Create a small `functions/` app in Node.js/TypeScript hosted on Azure Functions.

Primary endpoint:
- `POST /api/chat`
  - Input:
    - `messages`: array of chat messages
    - `pageContext`: current page hint such as `home`, `about`, `blog`, `blog:<slug>`
    - `locale`: optional, inferred from frontend if available
  - Output:
    - `reply`: assistant text
    - `actions`: optional CTA actions array
    - `intent`: one of `about`, `latest_blog`, `booking_business`, `booking_coffee`, `social`, `fallback`
    - `collectedFields`: optional partial booking intake such as `meetingType`, `topic`, `email`

Optional support endpoint:
- `GET /api/context`
  - Returns a normalized public context payload for debugging or frontend bootstrapping if desired
  - Can be omitted if context is assembled server-side only

Backend responsibilities:
- Load trusted context from a normalized source derived from:
  - `content/about.md`
  - latest blog post metadata/content summary
  - social links from `hugo.toml`
  - booking links from `hugo.toml`
- Call OpenAI with a constrained system prompt
- Enforce response schema so frontend can render CTAs deterministically
- Reject non-site claims and avoid hallucinated facts
- Add CORS for the site origin only

Environment/config:
- `OPENAI_API_KEY`
- `OPENAI_MODEL`
- `SITE_BASE_URL`
- `ALLOWED_ORIGIN`
- booking and social links should ideally stay sourced from site config, not duplicated in code unless build/runtime separation forces a small mirrored config file

### 2. Assistant behavior and prompt contract
Define the assistant as a personal concierge, not a general chatbot.

Behavior rules:
- Answer only using known site context and explicitly configured links
- Reply in Turkish or English based on the user's last message
- Keep responses short, polished, and action-oriented
- If unsure, say that the site assistant only answers questions about Gizem, her work, content, and booking options
- Never claim calendar access or booking confirmation
- Never invent experience, services, availability, or personal details not present in the site context

Intent handling:
- `about`
  - Answer who Gizem is, what she works on, focus areas, and where to find more
- `latest_blog`
  - Return a short intro plus CTA to open the latest blog post
- `social`
  - Offer direct CTA links for GitHub, LinkedIn, YouTube, Instagram, email
- `booking`
  - Ask lightly qualifying questions before redirect:
    - meeting type if unclear: `business` or `coffee`
    - topic
    - optional email if the user volunteers or agrees
  - After qualification, return the correct booking CTA
- `fallback`
  - Offer one of: ask about Gizem, read latest blog, book a call, open YouTube

Booking routing rules:
- Business-oriented questions, consulting, architecture, SAP BTP, project collaboration, speaking, partnerships:
  - route to `business`
- Casual intro, networking, getting to know each other:
  - route to `coffee`
- If intent is ambiguous:
  - ask one clarifying question before showing a link

### 3. Hugo integration
Add a floating chat launcher and panel to the site's shared layout so it renders on all pages.

Frontend behavior:
- Floating launcher in bottom-right
- Lightweight modal or slide-up panel
- Starter chips:
  - `About Gizem`
  - `Latest blog`
  - `Book a business call`
  - `Coffee chat`
- Page-aware hints:
  - on home: promote about/latest/booking
  - on about: promote booking/social
  - on blog: promote latest content, YouTube, booking
- Render assistant replies plus CTA buttons from backend response
- CTA button types:
  - `open_url`
  - `open_latest_blog`
  - `open_booking_business`
  - `open_booking_coffee`
  - `open_social`

Suggested integration points:
- shared partial for widget markup/script
- inject into main templates so home/about/blog pages all receive it
- use existing site visual language, but make the assistant visually distinct enough to feel intentional and modern

### 4. Repo additions
Planned additions:
- `functions/` Azure Functions app
- one shared site-context normalization module or generated JSON source for assistant content
- one Hugo partial for chat widget UI
- one small frontend script for widget state, API calls, and CTA handling
- optional assistant stylesheet if current inline-template CSS becomes too fragmented

Planned reuse:
- booking links from `hugo.toml`
- social links from `hugo.toml`
- about content from `content/about.md`
- latest blog from existing blog content and front matter

## Test Plan
### Functional scenarios
- User asks in Turkish who Gizem is
- User asks in English what Gizem works on
- User asks for latest post and receives latest blog CTA
- User asks to book time for SAP architecture discussion and gets `business` flow
- User asks for a casual intro call and gets `coffee` flow
- User asks an unrelated general knowledge question and gets a bounded fallback response
- User switches language mid-conversation and assistant follows the latest user language

### Booking scenarios
- Ambiguous booking request triggers one clarification question
- Clear business request goes directly to business CTA after light qualification
- Clear coffee request goes directly to coffee CTA after light qualification
- User declines to share email and still gets redirected
- No booking link available in config returns a graceful fallback message

### Frontend scenarios
- Widget loads on home, about, and blog pages
- Mobile open/close behavior works cleanly
- CTA buttons open the intended links
- Chat failure state shows retry/fallback text without breaking layout

## Assumptions
- Azure Function remains necessary to protect the OpenAI API key
- No Microsoft Graph integration in v1
- No user data persistence, CRM sync, or email notification in v1
- Only current site content and configured links are trusted sources
- Latest blog means the latest post present in Hugo content, which currently appears to be a single blog post
- The implementation can introduce a small mirrored assistant-context config if reading Hugo config directly from the Function runtime is awkward
