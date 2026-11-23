# ExamPilot

I had three finals in one week (SWE on Dec 16, AI on Dec 17, DAA on Dec 21) and the
material was spread over 44 PDFs plus my own notes. ExamPilot reads all of it, answers
questions with a citation to the exact file and page, quizzes me from the same material,
and keeps track of which topics I keep getting wrong so the next quiz leans on those.

![Ask](docs/screenshots/1-ask.png)

| Quiz | Library | Progress |
|---|---|---|
| ![](docs/screenshots/2-quiz.png) | ![](docs/screenshots/3-library.png) | ![](docs/screenshots/4-progress.png) |

## Try the UI without setting anything up

```sh
cd web
npm install
npm run demo        # http://localhost:5173
```

Demo mode runs on sample data right in the browser, so you don't need a database, Redis
or an API key.

## How it works

```
 Browser (React) ──► FastAPI ──► Postgres 16 + pgvector + full-text
                        │   └──► Claude API (chat agent, quiz gen, grading)
                        ▼
                     Redis queue ──► 3 workers: PDF → pages → chunks → embeddings → topic tags
```

Uploads are hashed with SHA-256 and the hash becomes the document id, so uploading the
same file twice still gives you one document. Each upload goes onto a Redis queue. A worker
pulls the text out page by page, strips repeated headers and footers, cuts each page into
chunks of at most 300 tokens, embeds them with `bge-small-en-v1.5`, tags each one with a
topic using a small PyTorch model, and writes it all to Postgres. A chunk never spans two
pages, so every citation points at exactly one page.

I wrote the queue myself on Redis lists with `BLMOVE`. Delivery is at-least-once, with
heartbeats, a reaper that hands a dead worker's job to another worker, exponential backoff,
and a dead-letter list after three failures. Processing is idempotent (content-hash ids
plus `ON CONFLICT DO NOTHING`), so a job that runs twice does no harm.

Search runs vector search and Postgres full-text search and merges the two with reciprocal
rank fusion. Vectors catch paraphrases, and keywords catch exact terms like "Bellman-Ford"
and "CAP theorem".

Chat is a tool loop I wrote on the Messages API. The model gets `search_notes`,
`weakest_topics` and `recent_mistakes`. Search hits go back as `search_result` blocks with
citations turned on, so the model can only cite pages it was actually shown.

Quizzes don't use the tool loop. Generating a quiz is one structured-output call and grading
is another. If a generated question cites a chunk id we didn't send, it gets dropped.
Multiple-choice answers are graded locally, and short answers go to Claude along with the
reference answer and the source page.

The scheduler picks topics with discounted Thompson sampling. It goes after questions you're
likely to get wrong, weighted by how much each topic counts on the exam, and old evidence
fades by 15% a day. Spaced-repetition reviews that are due come first, and nothing gets
scheduled after the exam date.

## Running it for real

Needs Python 3.12, Docker, and an Anthropic API key with credits.

```sh
uv venv --python 3.12 .venv && source .venv/bin/activate
uv pip install -e ".[dev]"
cp .env.example .env                 # add ANTHROPIC_API_KEY and EXAMPILOT_TOKEN

docker compose up -d db redis
python -m exampilot.cli ingest ~/Desktop/"Sem 1"/"Design & Analysis Algorithms" --course daa --inline
python -m exampilot.cli ask "why does Dijkstra fail with negative edges?"
```

No course files handy? `make sample` loads the [sample notes](samples/README.md) for all
three courses and two weeks of made-up quiz history, without any API calls.

To run the full stack (the API plus three workers):

```sh
make web            # builds web/dist, the API serves it
docker compose up -d --build
open http://localhost:8000
```

Other CLI commands: `chat`, `quiz daa`, `search "..."`, `status`, `requeue-stuck`,
`requeue-dead`, `retag`.

I don't commit the course PDFs (`data/` is gitignored), because the slides are NYU material
and the textbook is copyrighted. CI uses the sample notes in [samples/](samples/README.md)
instead, with the questions in [eval/fixtures](eval/fixtures/README.md).

## Tests and evals

```sh
pytest -q                                          # unit tests + Redis queue tests
python -m eval.retrieval eval/questions.jsonl --all-modes
python -m eval.answers eval/questions.jsonl        # calls the API, costs ~$4 a run
python -m sim.compare_policies --seeds 500 --plot bench/results/bandit.png
python bench/embed_bench.py chunks.jsonl mps
```

CI lints, runs the tests, and fails the build if recall@5 on the sample notes drops below 0.80.

Here's how retrieval does on the sample notes. It's a small, easy set (19 answerable
questions), but it still caught a real bug: keyword search used to AND every word of the
question together and found almost nothing.

| Retrieval (samples) | recall@5 | MRR@10 |
|---|---|---|
| Vector only | 1.000 | 0.939 |
| Keyword only (before the fix) | 0.105 | 0.105 |
| Keyword only | 1.000 | 0.912 |
| Hybrid (RRF) | 1.000 | 0.947 |

The LLM-judged answer eval doesn't run on every push. You trigger it from the Actions tab.

## Results

To fill in once I've run everything on my real material:

| Retrieval | recall@5 | MRR@10 |
|---|---|---|
| Vector only | | |
| Keyword only | | |
| Hybrid (RRF) | | |

| Topic classifier | macro-F1 | course acc. |
|---|---|---|
| Zero-shot (cosine to topic names) | | |
| Logistic regression | | |
| TopicNet (multi-task MLP) | | |

## Layout

```
src/exampilot/    api, worker, queue, retrieval, agent, quiz, bandit, review, classifier
db/               schema.sql, topics.sql
web/              Vite + React + Tailwind + Recharts
eval/ bench/ sim/ retrieval + answer evals, GPU benchmark, bandit simulation
samples/          study notes + quiz questions for demos and CI
k8s/              kind manifests with HPA (see k8s/README.md)
```

## What I'd do next

Signed upload URLs (Cloud Run caps request bodies at 32 MiB), KEDA to scale workers on
queue length instead of CPU, a cross-encoder re-ranker if the eval says it's worth the
latency, streaming answers, and real auth if anyone other than me ever uses it.
