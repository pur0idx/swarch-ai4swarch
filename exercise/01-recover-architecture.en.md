**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| checkoutservice → paymentservice | `checkoutservice.yaml` (PAYMENT_SERVICE_ADDR) | ✅ | Env var is in the YAML and is read in `src/checkoutservice/main.go` |
| checkoutservice → emailservice | `checkoutservice.yaml` (EMAIL_SERVICE_ADDR) | ✅ | Env var is in the YAML and is read in `src/checkoutservice/main.go` |
| frontend → adservice | `frontend.yaml` (AD_SERVICE_ADDR) | ✅ | Env var is in the YAML and is read in `src/frontend/main.go` |
| recommendationservice → productcatalogservice | `recommendationservice.yaml` (PRODUCT_CATALOG_SERVICE_ADDR) | ✅ | Env var is in the YAML and is read in `src/recommendationservice/recommendation_server.py` |
| frontend → checkoutservice | `frontend.yaml` (CHECKOUT_SERVICE_ADDR) | ✅ | Env var is in the YAML and is read in `src/frontend/main.go` |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question): `loadgenerator → frontend` is real, but the AI's evidence is weak. It cites the init container's `wget` ping (`loadgenerator.yaml` line 49), which only waits for the frontend to start; the actual traffic comes from `src/loadgenerator/locustfile.py`. The shopping assistant is a container to question: `frontend.yaml` (line 83) sets `SHOPPING_ASSISTANT_SERVICE_ADDR` and `src/frontend/main.go:139` requires it via `mustMapEnv`, and `src/frontend/handlers.go:464` calls it over HTTP, but no manifest deploys that service. The AI was right not to draw an undeployed container, but listing this real HTTP dependency only under Gaps hides it from readers of the diagram. `redis-cart` (`REDIS_ADDR`) was not checked.

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?

   Run A could list the nine gRPC services and their RPCs from `demo.proto`, but it could not tell which services call each other, and it missed the `frontend` and the `redis-cart` data store entirely, because neither is defined in the proto. A `.proto` file only describes what each service offers, not who uses it or how it is deployed. Run B's Kubernetes manifests filled this gap through the `*_SERVICE_ADDR` environment variables, which I confirmed are read in each caller's source code.

2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?

   The diagram shows no link between the `frontend` and a shopping assistant, yet `src/frontend/handlers.go:464` sends HTTP requests to `SHOPPING_ASSISTANT_SERVICE_ADDR`, which `main.go:139` requires at startup, and no manifest deploys that service. A new team member would assume the frontend depends only on the gRPC services shown, and could be surprised when that feature fails in a deployment. The AI was right not to draw an undeployed service, but the dependency deserves a visible note on the diagram rather than only a line under Gaps.
