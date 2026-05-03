# AddressIQ: LLM-Based Address Matching for Mortgage Document Review

## Problem Overview

When a borrower submits a purchase agreement, someone has to verify that the address on that document actually matches the subject property on the loan. Sounds easy. In practice it's a constant source of flags and delays because the same address can look completely different depending on who typed it and where — `123 Main Street` vs `123 Main St`, `Texas` vs `TX`, a ZIP+4 on one doc and a plain 5-digit on another.

The typical fix is either a regex (which breaks constantly) or just having a processor eyeball it (slow, inconsistent). This project uses an LLM to do the comparison instead — not just returning a yes/no, but breaking it down by street, city, state, and ZIP separately, and writing out a short explanation of its reasoning.

The output is a JSON object the loan origination system can consume directly. Matches get auto-approved. Only real mismatches go to a human reviewer, who can read the `reasoning` field to immediately understand what's wrong.

---

## The Problem in More Detail

The address comparison problem is deceptively annoying. Some real examples of things that should match but break string-matching:

- `Apt 4B` vs `Unit 4B`
- `78701` vs `78701-2345`
- `FL` vs `Florida`
- `St` vs `Street` vs `Street.`

And some things that look close but genuinely shouldn't match:
- Same street, different unit number
- Same street name, different street number (typo on one doc)

The goal is a system that handles the first category automatically and correctly flags the second, with enough context that the reviewer doesn't have to re-read both documents to figure out what's wrong.

---

## Workflow

The upstream process (outside this project's scope) handles OCR and pulls the property address out of the purchase agreement PDF. AddressIQ takes it from there:

1. LOS sends two address strings: the loan's subject property address and the one extracted from the purchase agreement.
2. AddressIQ builds a prompt, optionally pulls similar historical comparisons from the vector store as context, and sends it to the LLM.
3. The LLM returns structured JSON.
4. AddressIQ validates the schema and returns the result to the LOS.
5. If `matching: true`, the step auto-clears. If false, it routes to a processor with the `reasoning` already populated.

---

## AI Features

### Prompt Engineering

The whole thing lives in the prompt. The main things it has to handle:

- Break the comparison into components (street, city, state, ZIP) rather than treating the address as a single string
- Know which variations are acceptable (street type abbreviations, state abbreviations, ZIP+4 extensions) and which aren't (different unit numbers, different street numbers)
- Apply two matching rules: street + ZIP is enough even if city/state differ, and street + city + state is enough even if ZIP differs
- Return only valid JSON, no extra text

The hardest part was getting the unit number logic right — if either address has a unit number, both need to match on it. That required a few iterations to get the model to not just ignore unit numbers when the rest of the address matched.

### Structured Outputs

Every response follows this exact shape:

```json
{
  "matching": true,
  "reasoning": "Street address and ZIP code match; state abbreviation vs full name difference is acceptable.",
  "streetAddressMatch": true,
  "cityMatch": true,
  "stateMatch": true,
  "postalCodeMatch": true
}
```

The schema is enforced via Zod on the way out. If the model returns malformed JSON (rare but happens), the request retries once before erroring. The component-level fields are what make this actually useful — without them, a `matching: false` result just tells the processor "something's wrong" and they have to figure out what.

### Retrieval-Augmented Generation (RAG) and Vector Databases

The plan is to build a vector store of past address comparisons that had human-verified verdicts. When a new comparison comes in, retrieve a few similar historical pairs and inject them into the prompt as examples. This is especially useful for weird regional address formats that are hard to describe in rules but easy to demonstrate with examples.

Tool-wise, Pinecone for the store and `text-embedding-3-small` for the embeddings. This part isn't built yet — it's the main thing I'd tackle after getting the base prompt solid.

### Evaluation

The goal is a labeled dataset of address pairs pulled from real loan files, each tagged with the correct verdict. I'd want a few hundred examples covering clean matches, format variations that should match, genuine mismatches, and edge cases like PO boxes and rural routes.

For evaluation tooling, `promptfoo` can run the dataset against the prompt and report accuracy per field. The main thing I'd watch is false negatives — a missed mismatch is worse than a false flag in this context, because a false flag just slows things down, but a missed mismatch is a compliance problem.

### Observability

Every LLM call gets logged through LangSmith: both input addresses, the full rendered prompt, the raw response, and latency. The main reason is auditability — if a processor disputes a verdict, I need to be able to pull up the exact call that produced it.

Beyond that, I'd track a few metrics over time: overall request volume, the rate of `matching: true` vs `false`, and schema validation errors. A sudden shift in the matching rate is usually a signal that something changed — either the model was updated or the upstream OCR started producing different output.

---

## LLM Configuration

```json
{
  "type": "text",
  "model": "claude-haiku-4-5-20251001",
  "maxTokens": 2000,
  "temperature": 0.8
}
```

I would suggest Haiku because the task is narrow enough that a bigger model doesn't add much. The output is short (~100–150 tokens of JSON), so maxTokens: 2000 is just headroom. Temperature at 0.8 is a deliberate choice — it produces more natural-sounding reasoning strings, which matters since humans read them. The structured fields themselves aren't really affected by temperature. If I start seeing inconsistent verdicts on similar inputs, I'd dial it down.

---

## Example Prompts & Outputs

### Abbreviation variation — should match

Input:
```
<Address 1>123 Oak Street, Austin, Texas 78701</Address 1>
<Address 2>123 Oak St, Austin, TX 78701-2345</Address 2>
```

Output:
```json
{
  "matching": true,
  "reasoning": "Street name and number match with acceptable abbreviation difference (Street vs St). City matches. State matches allowing for TX vs Texas. ZIP codes match allowing for the 4-digit extension on Address 2.",
  "streetAddressMatch": true,
  "cityMatch": true,
  "stateMatch": true,
  "postalCodeMatch": true
}
```

### Unit number mismatch — should not match

Input:
```
<Address 1>500 River Rd, Unit 12, Denver, CO 80203</Address 1>
<Address 2>500 River Rd, Unit 14, Denver, CO 80203</Address 2>
```

Output:
```json
{
  "matching": false,
  "reasoning": "Street, city, state, and ZIP all match, but the unit numbers differ (Unit 12 vs Unit 14). When a unit number is present in either address, both must have a matching unit number.",
  "streetAddressMatch": false,
  "cityMatch": true,
  "stateMatch": true,
  "postalCodeMatch": true
}
```

---

## Evaluation

I'd build a labeled test set from historical loan files — a mix of clean matches, format variations, real mismatches, and edge cases. Run it through `promptfoo` on every prompt change. The threshold I care most about is false negatives (missed mismatches), since those are the ones with real consequences. I'd also track schema validity separately — if the model starts returning malformed JSON, that's usually a sign the prompt needs attention.

---

## Observability

LangSmith for tracing individual calls. Datadog for aggregate metrics: request volume, matching rate over time, schema errors, and p95 latency. The matching rate baseline is the most important alert — if it drifts more than ~10% from the 7-day average, something changed and I need to know about it before a processor notices.

---

## Open Questions

A few things I haven't fully figured out yet:

- **OCR errors in the extracted address.** If the upstream extraction garbles part of the address (common with handwritten docs or bad scans), the model might make a confident wrong call. Not sure yet whether to handle this in the prompt, add a confidence score, or flag low-quality inputs before they even reach AddressIQ.
- **Temperature tuning.** 0.8 feels right for reasoning quality but I haven't tested it at scale. Might need to come down.
- **RAG cold start.** The vector store needs a decent amount of labeled history to be useful. For the first few months it's basically just fancy few-shot examples.

---

## Repository Structure

```
addressiq/
├── README.md
├── src/
│   ├── api/            
│   ├── prompt/         
│   ├── rag/            
│   └── validation/     
├── evals/
│   ├── golden_dataset.json
│   └── promptfoo.yaml
└── observability/
    └── dashboards/
```
