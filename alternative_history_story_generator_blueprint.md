# Alternative-History Story Generator — Rabbit R1 MVP Blueprint

**What you’re building (one sentence)**  
A routed, multi-agent story generator that:  
1) interprets a user’s hypothetical; 2) dispatches it to a subject-matter “specialist” (history, sports, finance/econ, weather, business, etc.); 3) runs a prediction/simulation where appropriate; 4) researches real-time facts when requested; 5) creates an image prompt and renders art; 6) polishes the result into an award-style briefing; 7) narrates via TTS.

---

## User experience (aligned with your screenshot)
**Entry point:** the “intern” box in Rabbithole → “hey boss! what can I do for you today?”  
**User provides:** a hypothetical + desired lens (history | sports | finance/econ | weather | business | “auto”).  
**Output:**  
- a one-page “Office of the Alternative Historical Society” briefing (hero title, executive summary, methods, results, uncertainties, citations),  
- a generated image that visualizes the counterfactual,  
- optional audio playback of the briefing,  
- toggle to show “how we reasoned” and the specialist(s) consulted.

---

## System architecture (modular, cloud-first, r1-friendly)

**1) Orchestrator / Router**  
- Classifies intent + lens, extracts entities, timeframes, geos.  
- Chooses a *Specialist Chain* from a registry (history, sports, finance/econ, weather, business).  
- Decides whether “live research” is required (based on user toggle or if request references *now, latest, current*).

**2) Specialist Chains (pluggable)**  
Each chain is a pipeline with:  
- **Parser** (normalize the hypothetical),  
- **Reasoner/Model** (domain logic; may call a simulator),  
- **Researcher** (optional web/data calls),  
- **Result composer** (domain-specific sections: e.g., for sports, odds & rosters; for finance, indicators & scenarios),  
- **Artifacts** (image prompt, key numbers, citations).

Specialists you’ll start with:
- **History** (counterfactual analysis; causal narrative, plausibility grading)  
- **Sports** (matchup model, Elo/Bayesian win probs, injury reports)  
- **Finance/Econ** (macro indicators, market reaction scenarios, Monte Carlo ranges)  
- **Weather** (historical climatology vs. present forecast deltas)  
- **Business** (industry analysis, 5-forces snapshot, TAM/SAM/SOM sketches)

**3) Prediction Model Array**  
- A registry of “calculators” the chains can call:  
  - Time-series forecaster (Prophet/NeuralProphet),  
  - Monte Carlo simulator (NumPy/Pandas),  
  - Elo/Bayesian sports engine,  
  - Event impact model (simple causal graph + priors),  
  - Economic shock model (VAR-lite),  
  - Weather anomaly comparator (climatology baseline vs. forecast deltas).  

**4) Research Agent (optional, on-demand)**  
- Hits your approved APIs for fresh facts (sports injuries, economic releases, weather forecasts, news headlines).  
- Normalizes and caches results (timestamped).  
- Always returns provenance (source, retrieved_at).

**5) Image Prompt Generator**  
- Turns the distilled scenario into a vivid, style-controlled prompt (and a negative prompt if your image model supports it).  
- Injects era, mood, medium, and key visual elements inferred from the hypothetical.

**6) Briefing/Grammar Agent**  
- Converts raw results into a newsroom-grade “Polished Briefing” in a consistent voice, with sections, plain-English methods, confidence, and caveats.  
- Adjusts tone for the selected lens (financial vs. sports vs. historical).

**7) TTS Layer**  
- Streams audio of the briefing with SSML (pauses for numbers, spells tickers, handles foreign names).  
- Exposes play/pause/seek + download.

**8) Audit & Safety Layer**  
- Flags speculative claims, injects disclaimers where needed, and shows sources.  
- Red-teams prompts for sensitive content; rate-limits “trending” live queries.

---

## Data flow (step-by-step)

1) **User input** → `Input Manager`  
2) **Router** classifies lens, entities, era, intent, “needs research?”  
3) **Specialist Chain** runs:  
   a. normalize hypothetical;  
   b. (if requested) **Research Agent** pulls fresh data;  
   c. **Prediction Model Array** runs simulations;  
   d. assemble **Findings JSON** (numbers, plots, highlights, sources).  
4) **Image Prompt Generator** produces `image_prompt`.  
5) **Image Service** renders image(s).  
6) **Briefing/Grammar Agent** crafts the final write-up from the Findings + image caption.  
7) **TTS** synthesizes narration.  
8) **Presenter** returns: briefing HTML/Markdown, image(s), audio URL, sources, and the “explain” toggle.

---

## Interfaces (so everything snaps together)

### Request schema
```json
{
  "user_id": "string",
  "hypothetical": "string",
  "lens": "auto | history | sports | finance | economics | weather | business",
  "needs_realtime": true,
  "style": {
    "reading_time": "short | standard | deep",
    "voice": "neutral | investigative | cinematic",
    "image_style": "photo | illustration | retro-newspaper | oil | isometric"
  }
}
```

### Findings schema (internal)
```json
{
  "normalized": { "entities": ["..."], "timeframe": "YYYY..YYYY", "geo": ["..."] },
  "assumptions": ["..."],
  "analyses": [
    {
      "name": "EloMatchup",
      "summary": "Team A 63% over Team B",
      "key_numbers": { "p_win": 0.63, "spread": -4.5 },
      "plots": [],
      "sources": [{"title":"Injury Report","url":"...","retrieved_at":"..."}]
    }
  ],
  "uncertainties": ["..."],
  "image_prompt": "string",
  "citations": [{"title":"...","url":"..."}],
  "disclaimer": "string"
}
```

### Response schema (to UI)
```json
{
  "title": "string",
  "executive_summary": "string",
  "sections": [
    {"heading":"Background","body":"..."},
    {"heading":"Method","body":"..."},
    {"heading":"Results","body":"..."},
    {"heading":"What the picture shows","body":"..."},
    {"heading":"Uncertainties","body":"..."}
  ],
  "image": {"url":"...", "alt":"..."},
  "audio": {"url":"...", "duration_sec": 0},
  "citations": [{"title":"...","url":"..."}]
}
```

---

## Routing logic (lightweight and robust)

**Heuristics + classifier**  
- If the user explicitly picks a lens, honor it.  
- Otherwise, use a small intent model (labels: history, sports, finance, economics, weather, business, mixed).  
- If the prompt contains “today, latest, right now, this week” → set `needs_realtime=true`.  
- If mixed, run top-2 specialists and merge in the Briefing Agent.

---

## Prompt templates (you can paste these today)

### 1) Router (classification & plan)
> You are the Dispatcher for the Office of the Alternative Historical Society.  
> User hypothetical: `{hypothetical}`. Desired lens: `{lens}`.  
> 1) Extract entities, timeframes, locations.  
> 2) Decide the single best specialist (or top-2 if mixed).  
> 3) List required data calls if `needs_realtime=true`.  
> 4) Output a JSON plan with: `specialists[]`, `assumptions[]`, `data_calls[]`, `image_seed`.

### 2) Specialist chain (domain work)
> You are the `{specialist}` analyst. Normalize the hypothetical, state salient assumptions, then run a structured analysis.  
> If data is missing, specify *exactly* what you need.  
> Return a `Findings` JSON with `analyses[]`, `key_numbers`, `uncertainties`, `citations`. Keep it factual and check arithmetic digit-by-digit.

### 3) Image prompt generator
> Create a single best image prompt that visually narrates this scenario. Include subject, setting, era cues, composition, and medium. Avoid text in the image.  
> Output fields: `prompt`, `negative_prompt` (if applicable), `seed_suggestion`, `caption`.

### 4) Briefing/Grammar agent
> Turn the Findings + caption into a polished, journalist-grade briefing for a general audience.  
> Style: concise lead, rich detail where meaningful, clear caveats.  
> Include sections: Background, Method, Results, What the picture shows, Uncertainties, Citations.

### 5) TTS (SSML wrapper)
Wrap the final briefing in SSML: add `<break time="400ms"/>` after headings, expand numbers, spell tickers, and slow down for foreign names using `<prosody rate="85%">`.

---

## Minimal tech stack (drop-in choices)

- **Orchestration & Specialists:** Python (FastAPI) or Node (Express).  
- **LLM(s):** Any that support function calling / tool use.  
- **Image:** Stable Diffusion / SDXL (local or API), or any hosted image generation API.  
- **TTS:** Amazon Polly, Google Cloud TTS, ElevenLabs, or any equivalent API.  
- **Research (live):** plug modules for sports (e.g., Stats APIs), finance (market data/news APIs), weather (forecast APIs), macro calendars (economic releases).  
- **Storage:** Postgres for jobs; S3-compatible blob for images & audio; Redis for caching latest research lookups.  
- **Device:** On Rabbit R1, keep orchestration cloud-side; stream audio and thumbnails back to the device UI.

---

## Reference pseudocode (end-to-end)
```python
# rough, but runnable with proper API stubs

def generate_story(req):
    plan = router(req.hypothetical, req.lens, req.needs_realtime)

    findings = []
    for spec in plan["specialists"]:
        data_bundle = {}
        if req.needs_realtime and plan["data_calls"]:
            data_bundle = research_agent(plan["data_calls"])
        spec_findings = specialist_run(spec, req.hypothetical, data_bundle)
        findings.append(spec_findings)

    fused = fuse_findings(findings)

    img = image_prompt_from_findings(fused, plan["image_seed"])
    image_url = image_service_render(img["prompt"], img.get("negative_prompt"))

    briefing = grammar_agent_briefing(
        findings=fused,
        image_caption=img["caption"],
        voice=req.style.get("voice","neutral"),
        lens=req.lens
    )

    audio_url = tts_ssml(briefing)

    return {
        "title": fused.get("title","Office Briefing"),
        "executive_summary": fused.get("summary",""),
        "sections": to_sections(briefing),
        "image": {"url": image_url, "alt": img["caption"]},
        "audio": {"url": audio_url},
        "citations": fused.get("citations",[])
    }
```

---

## Example: single run (sports lens)

**User:** “If the 1998 Bulls faced the 2017 Warriors in a seven-game series at a neutral site, who wins? Give a sports briefing.”  
**Router:** → `sports` specialist, no realtime.  
**Sports Specialist does:** roster normalizations, apply era-adjusted pace, possession model, Elo, plus Monte Carlo.  
**Findings (condensed):**
- Bulls p(series win) ≈ 0.38, Warriors ≈ 0.62, median series length 6.2 games.  
- Key levers: 3-pt volume, pace inflation, bench depth.  
- Uncertainties: rule enforcement, officiating style.  
**Image Prompt:** “split-frame arena, neon scoreboards, 90s Bulls vs 2017 Warriors, dynamic motion blur, hardwood reflections, stadium crowd bokeh, sports photojournalism, 35mm look.”  
**Briefing + TTS:** delivered as polished newsroom copy; audio plays automatically.

---

## Example: mixed lens with research (finance + business)

**User:** “Suppose OpenAI announced a $50/month personal plan with full office suite—how might it hit Microsoft and Google this quarter? Use real-time comps.”  
**Router:** mixed → finance + business; `needs_realtime=true`.  
**Research Agent pulls:** recent office suite ARPU comps, market share estimates, recent earnings guidance, sentiment snapshots.  
**Prediction Model:** quick VAR-lite shock sim; range of revenue cannibalization vs. expansion.  
**Image Prompt:** “boardroom glass walls, charts glowing, three tech logos implied via abstract shapes, cinematic dusk skyline, magazine illustration.”  
**Briefing:** exec summary + scenario table + caveats; TTS with tickers spelled out.

---

## Image prompt pattern (copy this)
```
ROLE: You are a visual journalist. Turn the scenario into a single arresting image.

SCENARIO: {one-sentence distilled hypothetical}
MUST INCLUDE: {2–3 salient objects or characters}
ERA / STYLE: {time period cues}, {medium: photo | oil | woodcut | isometric | retro newspaper}
COMPOSITION: {wide / portrait / split-frame / overhead}, {lighting}, {background}
MOOD: {tone}, avoid text in-image.
NEGATIVE PROMPT: watermarks, text, distorted faces, low-res.
CAPTION: one sentence that explains what the viewer is seeing.
```

---

## Briefing pattern (keeps tone consistent)
**Title**  
**Lead (2 sentences)**  
**Background** – context and what the hypothetical changes.  
**Method** – how the specialist(s) modeled/estimated.  
**Results** – crisp bullet table and a short paragraph.  
**What the picture shows** – connect the image to the story.  
**Uncertainties** – what could flip the result.  
**Citations** – clearly listed.

---

## Choosing the “type of response”
Expose a simple toggle or chips: **Financial, Weather, Business, Economics, Sports, History, Auto**.  
- If user chooses one → hard-route to that specialist chain.  
- If **Auto** → router decides; show the chosen lens label on the result.

---

## MVP cut (2–3 weeks of build time, small team)
1) **Router** (heuristics + few-shot classifier).  
2) **Two specialists** to start (History + Sports or Finance).  
3) **Image** via a hosted API (reduce ops).  
4) **TTS** via a hosted API (SSML support).  
5) **No-database** first: store assets in object storage; log JSON to a simple table.  
6) **Research** off by default; wire the first live source (e.g., weather or sports) behind a toggle.  
7) **UI**: single page with input, lens chips, result card (briefing, image, play audio).

---

## Production hardening (as you scale)
- **Caching & rate limits:** cache identical research queries; exponential backoff on APIs.  
- **Observability:** log plan → calls → outputs; sample and review briefings.  
- **Evaluation:** golden prompts for each lens; regression tests on math and formatting.  
- **Safety:** forbidden topics list, redaction for PII, disclaimers for speculative/financial claims.  
- **Latency:** parallelize research + model inference; pre-warm image model; stream TTS.  
- **Edge fallback:** when offline, skip research; still generate analysis + image + local TTS if available.

---

## “Is it possible?” (short answer)
Yes. The design above is intentionally straightforward: a router, pluggable specialists, a small prediction toolkit, and two media services (image, TTS). Each piece is independently testable, and you can start simple (no research, one specialist) and grow.

---

## Acceptance criteria for MVP
1) Given any hypothetical + selected lens, the system returns within N seconds:  
   - a structured briefing with Background/Method/Results/Uncertainties;  
   - one 1024×1024 image;  
   - an audio narration in <60s length for “standard” briefs;  
   - a JSON “explain” object listing the chosen specialist and any sources.
2) If `needs_realtime=true`, the response includes a “Data retrieved at: <timestamp>” line and at least one citation.  
3) Arithmetic in Results is correct to two decimals; if a number is derived, the method explains it.  
4) Pressing “Sports” or “Finance” chips forces correct routing every time.

---

## Quick demo prompt you can try now
**Hypothetical:** “If a late-season atmospheric river stalls over the Bay Area during an El Niño winter, what happens to flights and local power grids?”  
**Lens:** Weather  
**System will output:** a forecast-style counterfactual (without scraping live data unless toggled), an image showing saturated radar bands over SFO with grounded aircraft, and a TTS briefing that explains which thresholds (wind gusts, runway crosswinds, soil saturation) force cascading delays.
