# together-restaurantrecs-overview
An overview of a RAG restaurant recommender I built on Together AI’s API! Here is an overview of the project, and [some of] my results.


My parents were traveling to Santa Barbara CA for Halloween weekend, so I rebuilt a restaurant recommender I first made at a hackathon in Oct 2024 for them to use on their trip, but this time on Together AI's API. This was solely a personal, educational project created on October 30th, 2026. This page covers the approach, platform measurements, and developer feedback. The code and the full evaluation results are private for now, pending approval from Yelp for a public release. The project uses the Yelp Open Dataset, and I made sure no data or review text appears here!

## The problem
A user (e.g., My dad) asks for restaurants in natural language ("quiet spot for date night", "cheap ramen open past 9pm") and the system retrieves relevant businesses and answers using only what it retrieved. The hard part is constraints like price and opening hours, which an embedding barely captures, so a naive system recommends places that are closed or too expensive, and sounds confident doing it.

## How it works
1. **Data prep:** open restaurants in the Santa Barbara metro (my parents were visiting, and the dataset has no Bay Area coverage). One document per business, built from its categories, price, hours, attributes, and a few reviews.
2. **Embeddings and index:** `BAAI/bge-small-en-v1.5` run locally, because Together's serverless catalog currently lists no embedding models (see the friction log). FAISS for exact vector search.
3. **Retrieval and generation:** top-5 retrieval, then a Together chat model answers and must cite each recommendation by business ID. Code checks that every cited ID was actually retrieved, and the model replies `NO_GOOD_MATCH` when nothing fits.
4. **Hard-constraint handling:** a rule-based parser pulls category, price, hours, and attributes out of the query, and a metadata filter keeps only matching businesses before semantic ranking.
5. **Model comparison:** `Qwen/Qwen3.5-9B` (thinking disabled) vs `openai/gpt-oss-120b` (`reasoning_effort: low`), both on Together serverless.

## Evaluation
16 test queries across four types: vague vibe, single hard constraint, multi-constraint, and "no good answer exists" to test for hallucination. Retrieval was scored automatically against hand-written constraint labels, and every answer was hand-scored 0 to 2. The full eval was 40 Together calls for $0.014 in total.

Retrieval accuracy and answer-quality results will be published with the code once Yelp approves. The Together platform measurements are below.

| Model | Median latency | Max latency | Mean input tokens | Mean output tokens | Mean cost per query |
|---|---|---|---|---|---|
| `Qwen/Qwen3.5-9B` | 1.58s | 3.72s | 1,849 | 114 | $0.000343 |
| `openai/gpt-oss-120b` | 2.30s | 4.00s | 1,734 | 145 | $0.000347 |

Prices from Together's serverless docs on Oct 30, 2026.

## What I found
1. **In RAG, input price matters more than model size.** Prompts were 12 to 16 times longer than answers, so input tokens drove cost. gpt-oss-120b's lower input price offset its higher output price, and a 120B model cost the same per query as a 9B one. The real trade-off was latency: Qwen was about 0.7s faster at the median.
2. **gpt-oss-120b carries hidden token overhead.** In a setup check asking each model to reply with the single word "OK," gpt-oss-120b used 76 input and 28 output tokens. Qwen3.5-9B with thinking off used 19 and 2. At "low" effort this didn't change cost per query, but it's worth knowing before choosing reasoning settings at scale.
3. **My own citation checker failed open (what didn't work).** gpt-oss-120b once wrote citations as `[ id: ... ]` with spaces, which my pattern didn't recognize. The checker found zero citations, reported zero invalid ones, and passed the answer. I fixed the pattern and made any recommendation without a readable citation fail. Separately, some format breaks only surfaced in hand scoring, never in automated checks. Lesson: validators should fail closed, and output format belongs in code (structured outputs), not just the prompt.

## Developer friction log (Together AI)
Notes from building on the platform, written as feedback a customer team might send:
- **The serverless catalog API omitted the models I used.** `client.models.list_supported(product="PRODUCT_SERVERLESS")` returned 13 chat models and left out both `Qwen/Qwen3.5-9B` and `openai/gpt-oss-120b`, though both answered calls fine. About 8 models on the docs' serverless page were missing from the API list. I got unstuck by trusting a live test call over the catalog. A customer building a model picker from this API would hide models that work.
- **No serverless embedding models.** The model older Together RAG tutorials use returned a 400: "Unable to access non-serverless model BAAI/bge-base-en-v1.5. Please visit ... to create and start a new dedicated endpoint." The error is clear, but I only learned it at call time. I embedded locally and kept all generation on Together. My suggestion: flag serverless vs. dedicated status at the top of the embeddings and RAG docs.
- **One bad API key, two error formats.** The catalog endpoint returned a Google-style error (`'code': 16` with a `ProblemDetail` block), while chat returned an OpenAI-style one (`invalid_api_key`). I handled it by checking HTTP status codes instead of parsing error bodies. One schema across endpoints would mean one error handler.
- **Docs lag the SDK.** Installed `together` 3.0.0 differs from the docs' v2 examples: `client.models.list()` now returns your own project's models, and the public catalog moved to `list_supported()`.

## What I would do for a real customer
- Define success metrics with the customer before building, and grow the eval past 16 queries with a second, blind scorer.
- Plan for data freshness: connect a live listings source, or at least check freshness before recommending.
- Add a location filter, so "near downtown" is a distance check rather than a hope that reviews mention it.
- Enforce output format with structured outputs, and have validators fail closed.
- Benchmark a dedicated Together embedding endpoint against the local model at the customer's query volume.

## Notes
- Built using the Yelp Open Dataset. This is a personal educational project, not affiliated with or endorsed by Yelp or Together AI.
- No dataset files, review text, or code are included in this repository.

Kirin Desai | UC Berkeley, Data Science and Economics | [LinkedIn](https://www.linkedin.com/in/kirindesai/) | https://substack.com/@kirindesai
