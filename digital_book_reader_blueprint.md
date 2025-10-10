# Windows Digital Library & Listening Room — MVP Blueprint

**What you’re building (one sentence)**
A Windows desktop app that auto-builds a personal library from PDFs/EPUBs, delivers a Kindle-grade reading view, and offers natural language TTS plus an AI librarian for organization and insights.

---

## User experience (first-run to daily use)
**Entry point:** Windows installer → first launch onboarding wizard.
**User provides:** folders to watch, sign-in (optional) for cloud sync, preferred voice/language, accessibility settings.
**Core surfaces:**
- **Home / Shelves:** Recently added, In Progress, Finished, AI Picks, Custom Collections.
- **Reader:** Book canvas (single/double page toggle), navigation pane, highlights/notes drawer, TTS mini-player, search.
- **Assistant:** Chat-style panel docked in reader + floating “Ask” button on Home.
- **Settings:** Library folders, voice packs, shortcuts, theme, privacy controls.

Key flows:
1. **Library ingestion:** App scans selected folders, normalizes metadata (title, author, cover), deduplicates, tags format.
2. **Shelf organization:** Drag-and-drop to collections; AI auto-shelves by genre/priority; manual override always available.
3. **Reading & listening:** Seamless switch between visual reading and continuous TTS playback (per paragraph, section, full book).
4. **Assistant interactions:** Ask for summaries, study guides, cross-book connections, due-date reminders, reading goals.
5. **Note-taking:** Highlight text, add sticky notes, export to Markdown/OneNote, sync across devices if cloud enabled.

---

## System architecture (modular, Windows-native)

**1) Desktop Shell (UI layer)**
- Windows Presentation Foundation (WPF) or WinUI 3 with MVVM.
- Hosts navigation frame, library shelves, reader canvas, assistant pane.
- Integrates native accessibility (Narrator, high-contrast, keyboard navigation).

**2) Library Service**
- Background watcher (FileSystemWatcher) + manual “Scan Now”.
- Extraction pipeline: format detection (PDF, EPUB, MOBI future), metadata extraction, cover rendering, text indexing.
- Persists catalog to SQLite (local-first) with optional cloud sync module (OneDrive/Dropbox API wrappers).

**3) Content Pipeline**
- PDF parsing via PDFium/MuPDF; EPUB via EpubSharp; fallback to Calibre CLI for edge cases.
- Text segmentation service builds chapter/section tree, paragraph spans, and canonical IDs for sync.
- Stores normalized text for search, quoting, and TTS.

**4) Reading Engine**
- Layout renderer (pagination, reflow for EPUB, fixed layout for PDF if user prefers).
- Theme engine (sepia, dark, high-contrast), typography controls, zoom.
- Annotation layer (highlights, notes, bookmarks) stored in SQLite with delta sync.

**5) TTS & Audio Service**
- Local voices via Windows Speech SDK; optional neural voices via Azure Cognitive Services / ElevenLabs API.
- Handles text chunking, SSML, caching of generated audio snippets for offline replay.
- Mini-player with speed control, sleep timer, background playback when minimized.

**6) AI Assistant Orchestrator**
- Embedding-based retrieval (FAISS/SQLite FTS) over user library + notes.
- Conversation manager (LLM front-end) with policy guardrails; optionally uses Azure OpenAI or local GGUF model via llama.cpp.
- Tasks: summarization, recommendations, reading schedule suggestions, note synthesis.

**7) Recommendation Engine**
- Signals: reading progress, completion history, highlight density, genre tags.
- Produces shelf sections (Next Up, Because you liked…, Academic focus) with transparent explanations.

**8) Settings & Sync**
- Profiles stored in JSON; roaming via OneDrive optional.
- Backup/restore of library metadata; export highlights/notes to user-selected folders.

---

## Data flow (scan → read → assist)

1. **Folder selection** stored; watcher kicks off initial scan.
2. **Ingestion pipeline** detects format → extracts metadata → generates cover thumbnails → indexes text.
3. **Library DB** updated; UI shelves refresh via observable collections.
4. **Reader launch:** fetch paginated content, apply user theme, load annotations.
5. **TTS request:** reading engine sends paragraph IDs → TTS service produces/streams audio → caches segments.
6. **Assistant query:** takes current context (book, selection, notes) → runs retrieval → sends prompt to LLM → returns actionable response (summary, tasks, citation references) → optionally writes new notes/goals.
7. **Sync (optional):** diff-based updates uploaded; remote changes merged on startup.

---

## Core interfaces (keeps modules decoupled)

### `DocumentDescriptor`
```json
{
  "id": "uuid",
  "path": "string",
  "title": "string",
  "author": "string",
  "format": "pdf | epub | mobi",
  "cover_thumbnail": "file://...",
  "tags": ["history", "research"],
  "reading_state": "not_started | in_progress | finished",
  "progress_pct": 42.5,
  "last_opened": "2024-05-01T12:32:00Z"
}
```

### `ReadingSessionState`
```json
{
  "document_id": "uuid",
  "location": { "chapter": 3, "paragraph": 12, "char_offset": 250 },
  "tts_speed": 1.2,
  "theme": "dark",
  "queued_actions": [
    { "type": "highlight", "range": {"start": "p12", "end": "p13"} },
    { "type": "assistant_prompt", "prompt": "Summarize this section in 3 bullets" }
  ]
}
```

### `AssistantTask`
```json
{
  "conversation_id": "uuid",
  "user_query": "string",
  "context": {
    "active_document": "uuid",
    "selection_text": "string",
    "notes": ["..."],
    "reading_goals": ["finish by Friday"]
  },
  "required_capabilities": ["summarize", "recommend", "categorize"],
  "safety_level": "default | offline_only"
}
```

---

## Assistant prompt patterns

**Summarize selection**
```
You are the in-app librarian. Summarize the highlighted passage for a busy reader.
- Tone: neutral, academic-friendly.
- Length: <= 120 words.
- Include 2 follow-up reading suggestions drawn from the current book or library metadata.
```

**Recommend next read**
```
You are the user's reading coach.
Input: profile summary, recent completions, available library metadata.
Output: 3 recommendations with rationale, estimated commitment (hours), and mood tags.
Highlight if any recommendation requires purchase/outside source.
```

**Organize library**
```
You are the librarian assistant.
Given document metadata + existing shelves, propose shelf assignments (genre, priority, research project).
Return JSON with shelves[], each entry {name, reason, confidence}.
Respect user's manual overrides list.
```

---

## Minimal tech stack (shipping in 8–10 weeks)
- **UI:** WinUI 3 (Desktop) with MVVM Toolkit and Fluent design controls.
- **Library & Parsing:** .NET 8, PdfPig/PDFium bindings, EpubSharp, AngleSharp for EPUB HTML, Calibre CLI as fallback.
- **Persistence:** SQLite + EF Core for catalog/annotations; LiteDB or blob table for cached audio.
- **Search & Embeddings:** SQLite FTS5 initially, upgrade path to FAISS shards; orchestration via Semantic Kernel.
- **TTS:** Windows Speech SDK locally; optional Azure Cognitive Services / ElevenLabs for premium voices.
- **Assistant:** Azure OpenAI / OpenAI API with retrieval plugins; feature-flagged local model via llama.cpp.
- **Sync & Cloud:** MS Graph / OneDrive API wrappers, Dropbox SDK optional, background diff merge service.
- **Observability:** AppCenter or Sentry for crash reporting, Serilog + OpenTelemetry exporters for structured logs.

---

## Upgrade execution plan (deliver suggestions in staged increments)

### Phase 0 — Foundation Hardening (Weeks 1-3)
- Implement telemetry/observability baseline (AppCenter + OpenTelemetry exporters).
- Stand up automated PDF/EPUB ingestion regression suite covering edge-case fixtures.
- Instrument feature flags for DRM, offline cache, and assistant providers using LaunchDarkly SDK.
- Deliver compliance checklist (GDPR, CCPA, SOC2 controls) with owner assignments.

### Phase 1 — DRM & Content Governance (Weeks 3-7)
- Integrate Readium LCP for EPUB; wire Adobe Content Server SDK for PDF where licensing permits.
- Add secure license store encrypted with DPAPI + optional Azure Key Vault for enterprise tenants.
- Extend ingestion pipeline with license validation hooks and fallback messaging when DRM blocks features.
- Update reader UX to surface rights metadata (print/copy limits, expiry) and audit log export.

### Phase 2 — Scale-out Library & Search (Weeks 5-9, overlaps Phase 1)
- Swap file watcher to use USN journal (for NTFS) to prevent missed events on large corpora.
- Introduce background indexing queue (System.Threading.Channels) with backpressure metrics.
- Externalize embeddings/indexes to SQLite/FAISS shard per 10k documents; add nightly compaction job.
- Benchmark ingestion throughput with 100k mixed documents; capture flamegraphs and remediate hotspots.

### Phase 3 — Offline-first Experience (Weeks 8-11)
- Create offline profile with user-defined cache quota; persist synthesized audio chunks and key assets.
- Implement sync conflict resolver UI for highlights/notes edits performed offline.
- Add service worker–style queue for assistant requests to retry when connectivity resumes.

### Phase 4 — Privacy, Retention & Governance (Weeks 10-14)
- Ship in-app data lifecycle dashboard (export/delete per book, per note, per telemetry stream).
- Add configurable retention policies with scheduled purge job (Quartz.NET) and audit trail.
- Deliver enterprise admin REST API + PowerShell module for policy automation.

### Phase 5 — Extensibility & Ecosystem (Weeks 12-18)
- Publish plugin manifest schema (JSON) and sandboxed execution host (AppDomain or WASM runtime).
- Provide sample extensions: custom shelf classifier, research export to Obsidian, LMS integration hook.
- Add marketplace surfacing with trust score, telemetry opt-ins, and auto-update pipeline.

### Phase 6 — Accessibility & Inclusive Interactions (Ongoing)
- Conduct WCAG 2.2 AA audit; schedule quarterly re-validation.
- Localize assistant prompts + reader chrome for top 8 locales; ensure TTS voice availability parity.
- Partner with blind/low-vision beta group for feedback loops; integrate quick toggle for dyslexia-friendly fonts.

---

## Delivery governance & success metrics
- **Program cadence:** Bi-weekly release trains with feature toggles controlling GA vs. beta exposure.
- **KPIs:** Library scan success rate (>99%), average TTS start latency (<1.5s), daily active listeners, crash-free sessions (>99.5%), assistant helpfulness CSAT (>4/5).
- **Quality gates:** Each feature must meet automated ingestion tests, reader rendering golden comparisons, and accessibility smoke tests before GA.
- **Rollout strategy:** Use staged flighting (internal → beta ring → GA) with automatic rollback triggered by crash spike thresholds.

---

## Reference pseudocode (C#-style)
```csharp
public async Task InitializeAsync()
{
    await libraryWatcher.StartAsync(settings.LibraryFolders);
    await recommendationEngine.RefreshAsync();
}

public async Task OpenDocumentAsync(Guid documentId)
{
    var doc = await libraryRepository.GetDocumentAsync(documentId);
    var layout = await contentPipeline.LoadAsync(doc);
    readerView.Bind(layout, userPreferences.CurrentTheme);
}

public async Task PlayTtsAsync(ParagraphSpan span)
{
    var audio = await ttsService.GetOrGenerateAsync(span, userPreferences.VoiceProfile);
    audioPlayer.Play(audio.Stream);
}

public async Task<AssistantResponse> AskAssistantAsync(string prompt, Context ctx)
{
    var retrieved = await retrievalService.FetchAsync(ctx);
    return await assistantClient.RespondAsync(prompt, retrieved);
}
```

---

## MVP cut (first public beta)
1. **Formats:** Support PDF + EPUB, basic metadata extraction, manual cover upload.
2. **Shelves:** Recently Added, In Progress, Finished, Favorites; manual tagging.
3. **Reader:** Reflowed EPUB + simple PDF viewer, highlights, bookmarks.
4. **TTS:** Local Windows voices with adjustable speed; no neural cloud dependency yet.
5. **Assistant:** Basic summarization + recommendations using hosted LLM; no offline mode.
6. **Sync:** Export/import library JSON; cloud sync flagged “coming soon”.
7. **Telemetry & crash reporting** in place.

---

## V2+ roadmap (post-launch)
- **Advanced parsing:** Academic PDF cleanup (two-column detection, figure extraction), OCR for scanned docs.
- **Voice packs:** Premium neural voices, multilingual speech synthesis, Whisper-based narration for user-recorded notes.
- **Collaboration:** Shared annotations, study groups, classroom mode.
- **Mobile companion:** Lightweight iOS/Android app for progress sync and quick listening.
- **Goal tracking:** Streaks, reading challenges, integrations with Todoist/Notion.
- **Privacy modes:** Completely offline LLM + encryption at rest for sensitive libraries.
- **Marketplace:** Optional plug-ins (e.g., citation generators, language learning overlays).

---

## Production hardening & compliance
- **Security:** Windows Credential Locker for tokens, full-disk encryption guidance, secure API storage.
- **Privacy controls:** Toggle for “never upload content”; transparent logs of assistant prompts/responses; easy data wipe.
- **Performance:** Incremental folder scanning, multi-threaded parsing, lazy-loading thumbnails.
- **Reliability:** Background worker retry queue, health diagnostics page, auto-backup of SQLite DB.
- **Accessibility:** WCAG-compliant color contrast, keyboard shortcuts, screen reader labels, dyslexia-friendly fonts.
- **Localization:** Resource files for UI strings, right-to-left layout support.
- **Testing:** Unit tests for ingestion/parser, integration tests for TTS playback, UI automation via WinAppDriver.

---

## Analysis of current blueprint

### Strengths already covered
- **End-to-end narrative:** The blueprint spans onboarding, ingestion, consumption, and assistance, giving engineering, design, and PM teams shared language for the MVP path.
- **Clear modularization:** Eight service boundaries (UI, Library, Content, Reading, TTS, Assistant, Recommendation, Settings) outline ownership seams that map well to a multi-pod delivery plan.
- **Balanced AI integration:** Retrieval-augmented assistant and recommendation models are scoped with both cloud-hosted and local-first options, keeping the MVP feasible while hinting at premium upsells.
- **Compliance awareness:** Early references to accessibility, localization, telemetry, and privacy controls ensure production-readiness discussions start on day one.

### Gaps & risks to track
- **DRM & proprietary formats:** Current scope omits Adobe DRM or Kindle proprietary formats; without a mitigation plan, customers with purchased catalogs may churn.
- **Large library performance:** SQLite + FileSystemWatcher will need stress testing for multi-terabyte research libraries; indexing and thumbnail generation could become bottlenecks.
- **Data governance:** Assistant prompts/responses are logged, but there is no retention or redaction policy—important for academic or enterprise adopters handling sensitive manuscripts.
- **Offline resilience:** Several flows (assistant, neural TTS, sync) assume connectivity; fallback behaviors and user messaging are not specified.
- **QA coverage:** Testing guidance lists frameworks but lacks metrics (e.g., parsing accuracy targets, TTS latency SLAs) to gauge readiness.
- **Extensibility hooks:** Plugin/marketplace ideas exist in V2, yet no extension API contracts or sandboxing strategies are defined.

---

## Upgrade recommendations

1. **Architect for DRM & vendor connectors**  
   - Investigate integrating Adobe Content Server SDK and explore partnerships for Kobo/Kindle APIs where licensing allows.  
   - Build an abstraction in the Content Pipeline for decryptors, isolating DRM secrets from general parsing logic.

2. **Scale-out library indexing**  
   - Introduce a background job queue (e.g., System.Threading.Channels) to batch metadata extraction, thumbnails, and embedding generation.  
   - Benchmark SQLite FTS vs. LiteDB vs. a local ElasticSearch instance; document thresholds for switching engines.

3. **Offline-first UX playbook**  
   - Define degradations when cloud services fail (e.g., cache last five TTS chapters, fall back to on-device summarization via small models).  
   - Add explicit user notifications and retry policies for assistant interactions and sync jobs.

4. **Data residency & retention controls**  
   - Provide per-profile settings for prompt log retention, export, and wipe.  
   - Document encryption-in-use strategy for embeddings and notes, and evaluate Windows DPAPI for secrets at rest.

5. **Quality benchmarks dashboard**  
   - Set MVP success metrics: ingestion success rate (>98%), TTS start latency (<1.5s), assistant response quality via human eval rubric.  
   - Instrument telemetry dashboards (AppCenter, Azure Monitor) with these KPIs and gate releases on trend lines.

6. **Extension & automation API**  
   - Design a sandboxed scripting interface (e.g., WebView2 + JavaScript, or .NET MEF plugins) with permission prompts for library access.  
   - Expose events (onDocumentImported, onTtsStarted) enabling academic tooling, citation generators, or study workflows.

7. **Inclusive interaction upgrades**  
   - Prototype dyslexia-friendly modes (OpenDyslexic font, adjustable line spacing) and haptic feedback integration with Xbox Adaptive Controller.  
   - Expand TTS voices to include regional accents and pronunciations, leveraging SSML lexicons users can curate.

8. **Enterprise readiness track**  
   - Offer deployment via MSIX with Intune policies, enable single sign-on (Azure AD/MSA), and integrate with enterprise logging (Event Tracing for Windows).  
   - Conduct threat modeling and document compliance posture (SOC 2 roadmap, GDPR data processing agreements).

These upgrades provide a sequenced evolution path: shore up compliance and performance gaps for large collections, enhance trust for privacy-conscious users, and open avenues for power-user extensibility and enterprise distribution.

