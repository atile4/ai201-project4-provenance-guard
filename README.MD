# Provenance Guard

A backend attribution system for creative platforms that classifies submitted text as likely AI-generated, likely human-written, or uncertain — and gives creators a path to appeal.

---

## Architecture Overview

A submitted piece of text takes the following path through the system:

1. A creator sends a `POST /submit` request with their `text` and `creator_id`.
2. The raw text is passed to **both detection signals simultaneously** — the LLM holistic check (Signal 1) and the stylometric heuristics (Signal 2).
3. The two signal scores are fed into the **confidence scorer**, which combines them into a single calibrated score between 0 and 1.
4. The confidence score is passed to the **label generator**, which maps it to one of three plain-language transparency labels.
5. A **structured audit log entry** is written capturing the full decision — both signal scores, the combined confidence, the verdict, and the label.
6. The response is returned to the creator containing the `content_id`, verdict, confidence score, transparency label, and individual signal scores.

If the creator disagrees with the result, they send a `POST /appeal` with their `content_id` and reasoning. The system updates the content's status to `under_review`, records the appeal alongside the original decision in the audit log, and returns a confirmation. No automated re-classification occurs — a human reviewer handles it from there.

```
SUBMISSION FLOW
   Creator
     |
     |  POST /submit  { text, creator_id }
     v
+--------------------+
|   /submit route    |
+--------------------+
     |
     |  raw text (sent to BOTH signals)
     +---------------------------+
     |                           |
     v                           v
+----------------+        +--------------------+
|  Signal 1      |        |  Signal 2          |
|  LLM / holistic|        |  Stylometric /     |
|  (Groq)        |        |  statistical       |
+----------------+        +--------------------+
     |                           |
     | signal_1 score (0-1)      | signal_2 score (0-1)
     +-------------+-------------+
                   |
                   v
        +-----------------------+
        |  Confidence Scorer    |
        |  (weighted average)   |
        +-----------------------+
                   |
                   | combined confidence score (0-1)
                   v
        +-----------------------+
        |   Label Generator     |
        |  (3 thresholds ->     |
        |   AI / uncertain /    |
        |   human)              |
        +-----------------------+
                   |
                   +-------------------+
                   |                   |
                   v                   v
        +-----------------+   +-------------------------+
        |   Audit Log     |   |   Response to creator   |
        | (scores, verdict|   | { content_id,           |
        |  signals, time) |   |   attribution,          |
        +-----------------+   |   confidence, label }   |
                              +-------------------------+

APPEAL FLOW
   Creator
     |
     |  POST /appeal  { content_id, creator_reasoning }
     v
+--------------------+
|   /appeal route    |
+--------------------+
     |
     v
+----------------------------+
|   Status Update            |
|   status -> "under_review" |
+----------------------------+
     |
     v
+----------------------------+
|   Audit Log                |
|  (appeal + original entry) |
+----------------------------+
     |
     v
+----------------------------+
|   Response to creator      |
|  { status: under_review,   |
|    message }               |
+----------------------------+
```

---

## API Endpoints

| Method | Endpoint | Description                                 |
| ------ | -------- | ------------------------------------------- |
| POST   | /submit  | Submit text for attribution analysis        |
| POST   | /appeal  | Contest a classification by content_id      |
| GET    | /log     | Return the 20 most recent audit log entries |

### POST /submit

Request body:

```json
{ "text": "...", "creator_id": "..." }
```

Response:

```json
{
  "content_id": "uuid",
  "attribution": "likely_ai | uncertain | likely_human",
  "confidence": 0.76,
  "label": "...",
  "signals": {
    "llm_score": 0.9,
    "stylometric_score": 0.55,
    "stylometric_breakdown": { ... }
  }
}
```

### POST /appeal

Request body:

```json
{ "content_id": "uuid", "creator_reasoning": "..." }
```

Response:

```json
{
  "status": "under_review",
  "content_id": "uuid",
  "appeal_timestamp": "...",
  "message": "Your appeal has been received and is under review..."
}
```

---

## Detection Signals

### Signal 1 — LLM Holistic Check (Groq)

**What it measures:** The semantic and stylistic feel of the text as a whole. The model assesses whether the writing exhibits patterns typical of AI output — uniform sentence rhythm, generic or hedged phrasing ("it is important to note", "furthermore", "various stakeholders"), polished but impersonal tone, and absence of personal voice or idiosyncratic word choices.

**Why this property differs:** AI models are trained to produce coherent, well-structured text that avoids sounding odd to a broad audience. This produces a characteristic flatness — writing that is technically correct but tonally uniform. Human writing is messier, more personal, and more variable in ways that are hard to fake.

**How it works:** The text is sent to `llama-3.3-70b-versatile` via the Groq API with a prompt specifying what to look for. The model returns a JSON object with a score (0.0 = definitely human, 1.0 = definitely AI) and a one-sentence reasoning explanation. Temperature is set to 0.1 for scoring consistency.

**What it misses:**

- Confident, formal writers whose clean prose happens to resemble AI output
- The score is not fully reproducible — small prompt variations can shift results
- It cannot explain its reasoning in a way that is independently verifiable

---

### Signal 2 — Stylometric Heuristics (pure Python)

**What it measures:** Three structural statistics that differ between AI and human writing at the character and word level, independent of meaning.

| Metric                             | What it captures                | AI pattern                            |
| ---------------------------------- | ------------------------------- | ------------------------------------- |
| Sentence-length standard deviation | How much sentence length varies | Low variance — uniform rhythm         |
| Type-token ratio                   | Unique words ÷ total words      | Lower diversity — common words reused |
| Expressive punctuation density     | `! ? ; : — …` per word          | Low density — sticks to . and ,       |

Each metric is normalised to a 0–1 AI score and the three are averaged equally to produce a single stylometric signal score.

**Why these properties differ:** AI models tend to produce sentences that cluster around a similar length and use high-frequency vocabulary consistently. Human writing is structurally noisier — sentence length swings widely, vocabulary is more idiosyncratic, and punctuation reflects emotional cadence.

**What it misses:**

- Short texts (under 20 words / 2 sentences) produce unreliable statistics; the function returns a neutral 0.5 in this case
- Writers who naturally produce clean, uniform prose — academic authors in particular — score higher than expected on all three metrics

---

## Confidence Scoring

### Combining the signals

The final confidence score is a weighted average:

```
confidence = 0.6 × llm_score + 0.4 × stylometric_score
```

The LLM receives the higher weight (60%) because it captures meaning and tone — properties that statistical metrics cannot access. The stylometric signal (40%) acts as an independent structural cross-check. A creator who writes formally but with genuinely variable structure should not be over-penalised by the LLM signal alone, and vice versa.

### Score thresholds

| Range       | Verdict      |
| ----------- | ------------ |
| 0.00 – 0.40 | likely_human |
| 0.41 – 0.69 | uncertain    |
| 0.70 – 1.00 | likely_ai    |

The threshold asymmetry is intentional: the spec notes that a false positive (labelling a human's work as AI) is worse than a false negative on a writing platform. Setting the AI threshold at 0.70 rather than 0.50 means the system must be reasonably confident before making an accusation. The uncertain band is wide (nearly 30 points) to give borderline cases an honest label rather than forcing a binary guess.

### Validation — two example submissions

**Example 1 — Clearly AI-generated** (confidence: 0.7618)

> "Artificial intelligence represents a transformative paradigm shift in modern society. It is important to note that while the benefits of AI are numerous, it is equally essential to consider the ethical implications. Furthermore, stakeholders across various sectors must collaborate to ensure responsible deployment."

```json
{
  "attribution": "likely_ai",
  "confidence": 0.7618,
  "signals": {
    "llm_score": 0.9,
    "stylometric_score": 0.5544
  }
}
```

The LLM identified the hallmark hedging phrases and impersonal tone immediately (score 0.9). The stylometric signal was moderate (0.55) — sentence lengths were somewhat varied but expressive punctuation was entirely absent.

---

**Example 2 — Clearly human-written** (confidence: 0.1928)

> "ok so i finally tried that new ramen place downtown and honestly? underwhelming. the broth was fine but they put WAY too much sodium in it and i was thirsty for like three hours after. my friend got the spicy version and said it was better. probably wont go back unless someone drags me there"

```json
{
  "attribution": "likely_human",
  "confidence": 0.1928,
  "signals": {
    "llm_score": 0.0,
    "stylometric_score": 0.4819
  }
}
```

The LLM gave this a 0.0 — the informal language, personal voice, and casual structure were unmistakable. The stylometric score (0.48) was moderate rather than low, partly because the short sentences produced a middling variance score. The combined score of 0.19 correctly lands in the likely_human band.

The gap between these two examples — 0.76 vs 0.19 — confirms the scoring produces meaningful variation across clearly different inputs.

---

## Transparency Label

Three label variants are displayed to readers based on the confidence score. The exact text each one displays:

**High-confidence AI (confidence ≥ 0.70):**

```
⚠️ Likely AI-generated. Our analysis suggests this content was probably created with
AI assistance (confidence: high). This is an automated estimate, not a certainty.
If you're the creator and disagree, you can appeal this label.
```

**High-confidence human (confidence ≤ 0.40):**

```
✓ Likely human-written. Our analysis found no strong signs of AI generation
(confidence: high). This is an automated estimate and not a guarantee of authorship.
```

**Uncertain (0.40 < confidence < 0.70):**

```
❓ Inconclusive. Our analysis couldn't confidently determine whether this content was
human- or AI-written. We're showing this honestly rather than guessing.
No attribution claim is being made.
```

The label text is designed to communicate three things to a non-technical reader: what the system concluded, how confident it is, and what the reader can do about it. The AI label explicitly surfaces the appeal path because a false positive on a creative platform is a serious credibility problem.

---

## Appeals Workflow

Any creator can appeal a classification. To do so, they submit a `POST /appeal` request with their `content_id` and `creator_reasoning`.

When an appeal is received, the system:

1. Locates the original log entry by `content_id`
2. Updates the content's status from `"classified"` to `"under_review"`
3. Records `appeal_reasoning` and `appeal_timestamp` into the existing log entry
4. Returns a confirmation to the creator

No automated re-classification occurs. A human reviewer opening the appeal queue via `GET /log` will see:

- The original text's `content_id` and `creator_id`
- The original verdict and combined confidence score
- Both individual signal scores (so they can see if it was a borderline disagreement between signals)
- The `llm_reasoning` explanation from Signal 1
- The creator's stated reasoning
- All timestamps

---

## Rate Limiting

Rate limiting is applied to `POST /submit`:

```
10 requests per minute
100 requests per day
```

**Reasoning:** A real creator submitting their own work — even iterating on multiple drafts — would rarely need more than a few submissions per minute. Ten per minute gives comfortable headroom for legitimate use while immediately blocking rapid-fire automated scripts. The daily limit of 100 reflects the volume a prolific creator might generate across a full working day while still preventing systematic abuse at scale.

**Verified behaviour:** Sending 12 rapid requests produces 10 `200` responses followed by `429 Too Many Requests` on requests 11 and 12:

```
200
200
200
200
200
200
200
200
200
200
429
429
```

---

## Audit Log

Every attribution decision and appeal is written to `audit_log.json` as a structured entry. Sample entries (accessible via `GET /log`):

```json
[
  {
    "content_id": "f78e9861-d00f-4b32-bdef-17b0b7b192d5",
    "creator_id": "test-ai",
    "timestamp": "2025-04-01T14:32:10.123Z",
    "attribution": "likely_ai",
    "confidence": 0.7618,
    "llm_score": 0.9,
    "llm_reasoning": "The text uses generic, impersonal phrasing and hedged language typical of AI-generated content.",
    "stylometric_score": 0.5544,
    "stylometric_breakdown": {
      "sentence_count": 3,
      "word_count": 43,
      "sentence_length_std_dev": 5.44,
      "len_variance_ai_score": 0.547,
      "type_token_ratio": 0.8837,
      "ttr_ai_score": 0.1163,
      "expressive_punct_density": 0.0,
      "punct_ai_score": 1.0
    },
    "status": "classified",
    "appeal_reasoning": null,
    "appeal_timestamp": null
  },
  {
    "content_id": "abc840c9-6cd9-4c25-8dff-1e261a7da417",
    "creator_id": "test-human",
    "timestamp": "2025-04-01T14:33:05.456Z",
    "attribution": "likely_human",
    "confidence": 0.1928,
    "llm_score": 0.0,
    "llm_reasoning": "The text is clearly informal and personal, with colloquial language and casual structure typical of human writing.",
    "stylometric_score": 0.4819,
    "stylometric_breakdown": {
      "sentence_count": 5,
      "word_count": 55,
      "sentence_length_std_dev": 6.72,
      "len_variance_ai_score": 0.4397,
      "type_token_ratio": 0.8727,
      "ttr_ai_score": 0.1273,
      "expressive_punct_density": 0.0182,
      "punct_ai_score": 0.8788
    },
    "status": "classified",
    "appeal_reasoning": null,
    "appeal_timestamp": null
  },
  {
    "content_id": "0f745691-431d-490a-8b3c-157683ae9082",
    "creator_id": "test-border-1",
    "timestamp": "2025-04-01T14:34:22.789Z",
    "attribution": "likely_ai",
    "confidence": 0.7042,
    "llm_score": 0.8,
    "llm_reasoning": "The text reads as formal academic prose with structured argumentation and impersonal phrasing consistent with AI generation.",
    "stylometric_score": 0.5604,
    "stylometric_breakdown": {
      "sentence_count": 2,
      "word_count": 43,
      "sentence_length_std_dev": 5.5,
      "len_variance_ai_score": 0.5417,
      "type_token_ratio": 0.8605,
      "ttr_ai_score": 0.1395,
      "expressive_punct_density": 0.0,
      "punct_ai_score": 1.0
    },
    "status": "under_review",
    "appeal_reasoning": "I wrote this myself from personal experience. I am a non-native English speaker and my writing style may appear more formal than typical.",
    "appeal_timestamp": "2025-04-01T14:35:10.000Z"
  }
]
```

---

## Known Limitations

**Formal human writing is frequently misclassified as AI-generated.** During testing, an academic-style paragraph about monetary policy received a confidence score of 0.70 — just enough to trigger the `likely_ai` label. The LLM signal gave it 0.80 because the precise vocabulary, impersonal tone, and structured argumentation closely resemble AI output patterns. The stylometric signal agreed (0.56) because the two long sentences had low length variance and no expressive punctuation.

This is a fundamental limitation of both signals: they measure properties that correlate with AI writing but are not exclusive to it. A non-native English speaker writing carefully formal prose, an academic writing in their discipline's register, or any writer who has developed a clean and precise style will appear AI-like to this system.

The appeals workflow exists specifically to catch this — but the burden is placed on the creator to dispute the label rather than the system to express genuine uncertainty. Widening the uncertain band or raising the AI threshold further would reduce false positives at the cost of missing genuine AI-generated content.

---

## Spec Reflection

**One way the spec helped:** Writing out the three label variants in planning.md before touching any implementation code made the label generator almost trivially easy to build. Because the exact text was already decided, the code became a simple mapping from score range to pre-written string. Without that step, it would have been tempting to write vague placeholder text and iterate later — which would have produced a weaker label.

**One way implementation diverged from the spec:** The planning.md described Signal 2 as an independent cross-check that would meaningfully influence the final score. In practice, the stylometric scores clustered tightly between 0.45 and 0.56 across all four test inputs — including both clearly AI and clearly human text. This made the stylometric signal a weak differentiator. The LLM signal, with its wider output range (0.0 to 0.9 across tests), became the dominant driver of every verdict. A production version would need to recalibrate the stylometric normalisation ranges or replace individual metrics with ones that discriminate more sharply.

---

## AI Usage

**Instance 1 — Flask app skeleton and Signal 1:** I provided my planning.md detection signals section and architecture diagram and asked the AI to generate a Flask app skeleton with a `POST /submit` stub and the LLM signal function. The generated function returned only a score. I revised it to also return a reasoning string (stored in the audit log for reviewers) and added defensive parsing to strip markdown code fences that the model sometimes wraps its JSON output in despite being told not to.

**Instance 2 — Signal 2 and confidence scoring:** I provided my uncertainty representation section and asked the AI to generate the stylometric function and the weighted average scoring logic. The generated stylometric function used only two metrics. I added a third (expressive punctuation density) after reviewing the output, because the first two alone would not distinguish between a short formal email and AI-generated text. I also added the short-text guard (returning a neutral 0.5 for texts under 20 words) after recognising that dividing by small word counts would produce unreliable TTR values.

**Instance 3 — Appeals endpoint:** I provided the appeals workflow section and asked the AI to generate the `POST /appeal` route. The generated version created a separate appeal log rather than updating the existing submission entry in place. I changed this so the appeal fields (`appeal_reasoning`, `appeal_timestamp`, `status`) are written directly into the original log entry — which meant a reviewer checking `GET /log` sees the full picture in one record rather than having to cross-reference two separate logs.
