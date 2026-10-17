# iPhone on-device AI (Apple Intelligence / local LLM apps) for private "share with spouse?" judgment, incl. Shortcuts

Research date: 2026-10-12. Limited depth (about 14 tool calls). Search-snippet-level evidence for many items; flagged where so.

## 1. Shortcuts "Use Model" action (Apple Intelligence)

### Takeaway
Use Model exists since iOS 26 (WWDC25) with On-Device, Private Cloud Compute (PCC), and ChatGPT options, accepts Shortcuts variables and returns structured output (Text/Number/Date/Boolean/List/Dictionary). The on-device model is small (about 3B, small context), so long chat logs need chunking or PCC. Whether it runs unattended in automations, and Japanese quality, were NOT confirmed by sources.

### Cited Findings
- iOS 26 (public release): Use Model prompts the on-device model, the larger PCC model, or ChatGPT (article says GPT-5; an earlier MacStories beta note guessed GPT-4 Turbo, so the version is unclear/conflicting). Output can be Automatic or Text, Number, Date, Boolean, List, Dictionary; third-party app entity types can serve as schemas. — [MacStories iOS 26 review](https://www.macstories.net/stories/ios-and-ipados-26-the-macstories-review/8); [MacStories Q&A](https://www.macstories.net/notes/i-have-many-questions-about-apples-updated-foundation-models-and-the-great-use-model-action-in-shortcuts/)
- On-device model is about 3B parameters, 4,096-token context window (per MacStories review, iOS 26), no vision in iOS 26, "limited subset of languages", heavy guardrails (sometimes refuses), suited to narrow tasks like summarization/tagging. — [MacStories iOS 26 review](https://www.macstories.net/stories/ios-and-ipados-26-the-macstories-review/8)
- Anecdote (iOS 26 beta): on-device model failed on 43 unread RSS items due to context length; ChatGPT handled them. PCC model size about 17B is the author's estimate only, not documented by Apple. — [MacStories Q&A](https://www.macstories.net/notes/i-have-many-questions-about-apples-updated-foundation-models-and-the-great-use-model-action-in-shortcuts/)
- Apple said models are "designed to support 15 languages" (Japanese not explicitly named in that article). — [MacStories Q&A](https://www.macstories.net/notes/i-have-many-questions-about-apples-updated-foundation-models-and-the-great-use-model-action-in-shortcuts/)
- iOS 27 Apple support page (documented): model choices are On-Device (offline, simple requests), Cloud, Cloud Pro (PCC, extended context), Extension Model (ChatGPT). Requires iPhone 15 Pro / 15 Pro Max / any iPhone 16 or later. Siri+device language must be one of 16 supported languages including Japanese. Server-side models (including AFM 3 Cloud models in Shortcuts) have daily usage limits; paid expanded access "may be offered in future". — [Apple Support: Use Apple Intelligence in Shortcuts](https://support.apple.com/guide/shortcuts/tpg3vrvwmclv)
- Same Apple page "doesn't cover using Use Model in automations" (my fetch result). — same link
- WWDC26 (iOS 27): on-device model rebuilt from scratch, gains vision; guardrail false positives reduced in iOS 26.4 with more in 27. Code sample prints contextSize 8192 (iOS 26.4+ API) while a lab Q&A reportedly says 4096 on-device and 32K for PCC, shared between input/output — conflicting. — [WWDC26 session 241](https://developer.apple.com/videos/play/wwdc2026/241) (search snippet for the Q&A numbers: [session page](https://developer.apple.com/videos/play/wwdc2026/241?time=166))
- iOS 27 / WWDC26: Shortcuts can build automations from plain-English prompts. — [TechCrunch](https://techcrunch.com/2026/06/08/apple-will-let-you-build-workflows-using-ai-in-its-new-shortcuts-app)
- Third-party apps (Lire, Chronicling, CARROT, Reminders etc.) already use these models; the on-device model is shared and free to developers. — [MacStories iOS 26 review](https://www.macstories.net/stories/ios-and-ipados-26-the-macstories-review/8)

### Inferences
- A "should I share this?" classifier returning a Boolean + reason Dictionary is technically supported by Use Model's output types; on-device with a 4K–8K token budget means only short excerpts (roughly some hundred Japanese characters to a couple thousand) per call.
- Because on-device input is shared and nothing leaves the phone, it fits the privacy requirement; PCC is Apple-attested but still leaves the device, and has daily caps.
- Unattended automation operation is plausible (Use Model is an ordinary action) but unverified; needs a real test, esp. with the phone locked.

### Gaps
- Japanese-specific quality/benchmarks of the on-device model: not found.
- Explicit confirmation Use Model runs in personal automations while locked: not found.
- Exact on-device context in Shortcuts on iOS 27 (4096 vs 8192): conflicting.
- iOS 26 hardware list text on Apple page for iOS 26 specifically: not fetched (iOS 27 page used).

## 2. Foundation Models framework and App Intents (third-party Shortcuts actions)

### Takeaway
Foundation Models gives third-party apps the same on-device model (Swift API), and apps can expose App Intents that Use Model can consume as entities. No specific app exposing a "classify this text locally" action was verified.

### Cited Findings
- Apple states App Intents actions/entities work with Use Model in Shortcuts. — [Apple developer](https://developer.apple.com/apple-intelligence/?cid=ht-ai) (search snippet)
- Foundation Models uses its own Tool protocol, parallel to App Intents (opinion). — [Blake Crosley](https://blakecrosley.com/blog/app-intents-are-apples-new-api-to-your-app) (snippet)
- Example apps using Use Model-style features: Lire (summarization), Reminders, Awake, etc. — [MacStories iOS 26 review](https://www.macstories.net/stories/ios-and-ipados-26-the-macstories-review/8)
- Sindre Sorhus's "AI Actions" app appeared in results as a Shortcuts-related AI actions app; content not verified. — [sindresorhus.com/ai-actions](https://sindresorhus.com/ai-actions)
- WWDC26: framework opened to a model abstraction layer and works with cloud models such as Claude and Gemini (per Apple's Japanese iOS 27 page, snippet). — [Apple JP iOS 27](https://developer.apple.com/jp/ios/whats-new/)

### Inferences
- Writing a tiny own app with an App Intent "Classify text" using Foundation Models is feasible by design (inference), giving a custom local Shortcuts action; but Use Model itself already covers this without coding.

### Gaps
- Concrete third-party apps with a local-classify Shortcuts action: not verified.
- Japanese support status of Foundation Models per language list: not found.

## 3. Notification summaries / priority notifications

### Takeaway
No evidence that summaries can be consumed by automations. Quality history is poor for news; no LINE/Japanese-specific information found.

### Cited Findings
- Apple disabled summaries for News & Entertainment apps in iOS 18.3 after inaccurate headlines; iOS 26 restored with per-app opt-in/out, labels and disclaimer "Summarization may change the meaning". — [Tom's Guide](https://www.tomsguide.com/ai/apple-intelligence/apple-intelligence-will-summarize-news-again-in-ios-26-after-false-headline-debacle); [9to5Mac](https://9to5mac.com/2025/07/24/heres-what-apple-intelligence-summarized-news-notifications-look-like-on-ios-26)
- A Japanese Apple Community thread says LINE showed no Shortcuts actions on iOS 18 (LINE didn't support Shortcuts then). — [Apple Community JP](https://discussionsjapan.apple.com/thread/255771191)

### Inferences
- Summaries are display-only (no documented API/trigger), so they cannot be a pipeline input.

### Gaps
- LINE notification summary behavior in Japanese; any API for summary text; Shortcuts "notification received" trigger content access for LINE: not found.

## 4. Third-party local LLM apps and Shortcuts/URL schemes

### Takeaway
Private LLM is the only app with a reported Shortcuts integration (vendor claim). Locally AI's Shortcuts action opens Voice Mode only. PocketPal: nothing found. Model size/Japanese/battery data: not found.

### Cited Findings
- Private LLM's App Store listing claims iOS Shortcuts integration for workflows like summarization (vendor claim). — [Private LLM listing](https://mwm.ai/>/apps/private-llm-local-ai-chat/6448106860); vendor comparison [privatellm.app](https://privatellm.app/compare/pocketpal-ai-vs-private-llm.md)
- Locally AI updated for iOS 26, supports Apple Foundation model, adds a Shortcut to open Voice Mode (launcher, not prompt action). — [App Store: Locally AI](https://apps.apple.com/app/locally-ai-private-ai-chat/id6741426692)
- PocketPal AI: free, open source, any GGUF from Hugging Face; no Shortcuts mention found. — [promptquorum guide](https://www.promptquorum.com/power-local-llm/best-local-llm-apps-iphone-2026)

### Inferences
- Even with Shortcuts, 3–4B quantized models on 8GB iPhones likely won't beat Apple's on-device model for Japanese judgment (inference, not tested).

### Gaps
- LLM Farm, MLC Chat, Enchanted, Apollo URL schemes/actions; RAM limits; Japanese model recommendations; speed/battery: not researched/found.

## 5. Mac mini Ollama via "Get contents of URL" over Tailscale

### Takeaway
Technically straightforward and documented in pieces: Ollama exposes an HTTP API callable from Get contents of URL; Tailscale supports Shortcuts actions and Wi-Fi-triggered automations. Reliability of tailnet connection from Shortcuts is a caveat. Chat text leaves phone (to your own Mac), so privacy is user-controlled.

### Cited Findings
- A 2024 walkthrough calls Ollama's generate endpoint via POST JSON from Shortcuts (localhost on Mac). — [Sansan Builders Box](https://buildersbox.corp-sansan.com/entry/2024/05/07/110000) (snippet)
- Tailscale v1.36+ has Shortcuts actions (Toggle, Get Status, etc.) and can be automated. — [Tailscale docs](https://tailscale.com/kb/1233/mac-ios-shortcuts/)
- Tailscale forum: Shortcuts cannot easily detect tailnet connection status on iOS (workaround hacky). — [Tailscale forum](https://forum.tailscale.com/t/shortcuts-support/3039)

### Inferences
- Pros: far larger model (Qwen/Gemma class), good Japanese, no iPhone battery/RAM; Cons: Mac must be on, VPN reachability failure, text traverses the tailnet (WireGuard-encrypted); bind Ollama to tailnet interface only, no public exposure.
- OpenAI-compatible /v1/chat/completions on Ollama: known from general knowledge, not source-verified here.

### Gaps
- No verified hands-on report of iPhone Shortcut -> Tailscale -> Ollama.

## 6. Background execution / hourly runs

### Takeaway
Time of Day personal automations are the most reliable built-in scheduler, but anecdotal reports show failures after iOS 26.x updates, "Ask Before Running" resets, and Low Power Mode interference. A server-side scheduler is the dependable alternative.

### Cited Findings
- Time of Day trigger supports fixed times and daily/weekly/monthly repeats (no documented hourly option in snippet; hourly would need multiple automations). — [Apple Shortcuts guide](https://support.apple.com/en-ca/guide/shortcuts/apd690170742/9.0/ios/26)
- Vendor blog: time triggers relatively reliable, but updates can reset Ask Before Running; Low Power Mode interferes. — [getfinny](https://getfinny.app/blog/ios-shortcuts-automations-flaky-fix)
- Japanese blog (unverified): more time-trigger failures in 2026 after iOS 26.1/26.2. — [minto.tech](https://minto.tech/ios26-iphone-shortcut-personal-automation-time-of-day-not-firing-fix/)
- Apple forum (iOS 16.1.2, old): time automation didn't run while idle overnight; some actions don't run when locked. — [Apple Dev Forums](https://developer.apple.com/forums/thread/665845)

### Inferences
- Any hourly scheme must tolerate missed runs (idempotent, catch-up logic). Running Use Model while locked is the biggest unverified risk.

### Gaps
- No Apple-documented guarantee for background execution; hourly-trigger specifics not verified.
