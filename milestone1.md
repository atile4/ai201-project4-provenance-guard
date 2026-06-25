A creator submits their text to POST /submit. The text is run through two signals — one checking if it's structured like AI writing (vocabulary diversity, sentence structure), the other checking if it sounds like AI (tone, wording). Each gives a score, and the two are combined into a single confidence score. That score maps to one of three outcomes: likely AI, likely human, or uncertain (when the signals disagree or are weak, it lands in uncertain). The result becomes a transparency label shown to the reader, and the full decision — both signal scores, the combined score, and the verdict — is written to the audit log before the response returns to the creator.

Signal 1: Holistic Check

- This signal checks the tone and wording of the text. AI tends to be more flat, vague, and generic when it writes, while human writing has more meaning and has a certain style.
- However, a blind spot comes from the fact that the system is essentially going off the vibe of the text, so it can't be quantitavely measured. This makes it difficult to verify that text is AI generated.
- Another blind spot comes from the way people can write; there are people who can write clean, formal, and fluent prose, but these are also qualities of AI-generated text.

Signal 2: Statistic Check

- This signal checks for the length and structure of the text, checking things like sentence-length variance, vocabulary diversity, and punctuation density. AI tends to have more uniform-length sentences and rhythm, while human writing can vary significantly more, being more irregular.
- A blind spot that comes from this is that naturally uniform writers could be flagged for AI, and smaller pieces of text would give more unreliable statistics.

If a false positive is found, instead of immediately flagging a user of AI-generated work, which would ruin our credibilty, it would instead use the "uncertain" label. The creator should appeal by showing proof.

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
        |  (combine + calibrate)|
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
                   | label text + verdict
                   +-------------------+
                   |                   |
                   v                   v
        +-----------------+   +-------------------------+
        |   Audit Log     |   |   Response to creator   |
        | (scores, verdict|   | { content_id,           |
        |  signals, time) |   |   attribution,          |
        +-----------------+   |   confidence, label }   |
                              +-------------------------+

```

```
APPEAL FLOW
   Creator
     |
     |  POST /appeal  { content_id, creator_reasoning }
     v
+--------------------+
|   /appeal route    |
+--------------------+
     |
     |  content_id + reasoning
     v
+----------------------------+
|   Status Update            |
|   status -> "under review" |
+----------------------------+
     |
     |  appeal + original decision
     v
+----------------------------+
|   Audit Log                |
|  (logs appeal next to the  |
|   original classification) |
+----------------------------+
     |
     |  confirmation
     v
+----------------------------+
|   Response to creator      |
|  { status: under_review,   |
|    message }               |
+----------------------------+
```
