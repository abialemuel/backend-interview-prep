# Round 2: Deep Technical Interview (45 min live + 90 min take-home)

This file is dedicated to one specific round: **"Deep Technical Interview" — a 45-minute live technical conversation on Google Meet covering your engineering experience and approach, paired with a 90-minute take-home exercise completed on your own time.** Everything below is either (a) sourced from real, dated candidate reports for Partly's engineering interviews, clearly marked as such, or (b) general best-practice for this exact round shape (live-experience-conversation + solo take-home) at a senior level, applied to Go and to Partly's own domain. Where something is inference rather than a confirmed fact about *this specific round*, it says so — don't over-trust anything below that isn't labeled as sourced. A second research pass (Coderbyte's assessment specifics, Partly's own hiring-process posts, live API-doc examples) turned up nothing further — what's below is the ceiling of what's publicly findable, not an incomplete draft.

## Do these first, in order — the priority checklist

If tonight is short, work top to bottom; everything here links to the fuller section below it.

1. **[Say the three CS-fundamentals answers out loud, cold](#a2-cs-fundamentals-rapid-fire-the-reported-real-questions-answered-tight)** — stack vs. heap, concurrency vs. parallelism, compiler vs. interpreter. These are the highest-confidence, actually-reported questions in this entire pack. Non-negotiable.
2. **[Rehearse your two lead stories three layers deep](#a1-narrating-your-experience-a-structure-that-lands-for-this-jd)** — the UAE government API reliability rebuild and the Telkom AI Proxy/MCP Orchestrator. Not the summary sentence — the follow-up layer, since the Firebase/Stripe precedent means they will dig into specifics.
3. **[Run the Go take-home skeleton locally once](#b2-practice-exercise-domain-realistic-timeboxed)** (`go test ./...`) so the project-structure and table-driven-test muscle memory is warm before the real 90 minutes starts.
4. **[Re-read the main file's end-to-end scenario](README.md#4-scenario-question-design-the-whole-thing-end-to-end)** once — the take-home may well be a smaller, concrete version of exactly that fitment/supersession/confidence design.
5. **[Skim the Go-specific senior questions](#a3-go-specific-senior-questions-likely-in-scope)** once — goroutine judgment, memory/escape analysis, context propagation, error-wrapping, interfaces.
6. **Have two or three [questions to ask them](README.md#7-questions-worth-asking-them-directly) ready** — the AI-eval/confidence-scoring one and the "where does this role sit relative to AI Research" one fit a *technical* interviewer best.

The sections below are the full detail behind each of those six — read top to bottom if you have time, or jump straight to whichever one you're least confident on.

## What's actually known about Partly's technical interviews

From Glassdoor's Partly Group interview reports (aggregated across roles, most recent data points from 2026):

- **Interview difficulty**: 3.18–3.4 / 5 average across reported interviews; company-wide interview positivity is mixed (30–45% rate it positive), and multiple reviewers independently describe the process as "long," spanning multiple rounds and time zones, sometimes bleeding into evenings/weekends. Average time-to-hire for Software Engineer roles specifically is reported around 16–17 days — plan for the possibility of a slower turnaround after tomorrow, not a fast one.
- **An initial timed technical test** (reported at 15–20 minutes, "mostly back-end/database questions," explicitly **no AI tools or Googling allowed**) is standard before a human round — you've evidently already cleared this stage if you're at Round 2.
- **The technical/hiring-manager round** (45–60 minutes reported) shifts from resume/project questions (one candidate specifically mentioned being asked about **Firebase authentication** and **Stripe integration** — i.e., they will ask about the *actual* systems on your resume, not generic trivia) into **computer-science fundamentals**: reported real questions include **stack vs. heap**, **concurrency vs. parallelism**, and **compiler vs. interpreter** (a fittingly on-brand one, given the flagship product is literally called Interpreter).
- **Later rounds** for some roles have included meeting multiple engineers and the CEO — a signal that senior hires get broad organizational exposure, consistent with a ~160-person company that's tripled in 18 months.
- One reviewer flagged that a role's terms shifted mid-process (full-time framed initially, contract-to-start revealed later) — not a claim about *your* offer, just a reason to confirm terms explicitly if anything about scope/duration/comp hasn't been said plainly by this stage.
- **No public reports describe the specific 45-live + 90-take-home structure for the Senior Software Engineer role** — this looks like a newer or level-specific format not yet reflected in public reviews. Everything from here on for the take-home specifically is best-practice, not a leaked prompt — treat it as calibration, not a spoiler.

Sources: [Partly Group Interview Questions — Glassdoor](https://www.glassdoor.com/Interview/Partly-Group-Interview-Questions-E5331701.htm), [Partly Group Software Engineer Interview Questions — Glassdoor](https://www.glassdoor.com/Interview/Partly-Group-Software-Engineer-Interview-Questions-EI_IE5331701.0,12_KO13,30.htm), [Working at Partly Group — Glassdoor](https://www.glassdoor.com/Overview/Working-at-Partly-Group-EI_IE5331701.11,23.htm).

## Part A — The 45-minute live conversation

Treat this as two blocks, roughly 20/25: **your experience and approach**, then **technical depth probing** that will likely include the CS-fundamentals questions above plus Go-specific senior judgment questions. Interviewers at this level are evaluating *judgment*, not typing speed or trivia recall — the strongest signal is naming trade-offs and failure modes before being asked, not just describing the happy path.

One fact worth carrying into every answer in this block: Partly's own engineering blog states outright that the team runs **"no sprints or scrums,"** flat structure, engineers handed **problems rather than specs**, expected to go talk to the customer directly rather than wait for a written requirement (main file Section 0). "Your approach" isn't a throwaway phrase in the round's title — this is a company that explicitly hires and evaluates for how you behave with ambiguity, not how well you follow process. Every "walk me through how you approached X" answer should show you diagnosing the actual problem yourself, not executing someone else's ticket.

### A.1 Narrating your experience — a structure that lands for this JD

Given Section 1/3 of the main prep file (fault-tolerant + accurate APIs, distributed systems, AI-infrastructure interest), lead with the stories that map most directly, in this order of relevance:

1. **The UAE government API reliability rebuild** (Careem) — this is your single best story for "fault-tolerant and accurate," the JD's exact framing. Structure it as: *symptom* (regulated events silently dropping under lossy retry logic) → *root cause* (no error classification, retries treated all failures identically) → *fix* (a proper error classifier + an ordered retry queue) → *why it matters* (regulated data can't just get "eventually consistent," it has to be provably not lost). This is a near-perfect analogue to Partly's "an available-but-wrong-or-lost answer is worse than a slow one" problem (Section 1 of the main file).
2. **The Telkom AI Proxy / MCP Orchestrator** — your strongest story for the "AI Research & Software Engineering" department fit. Be ready to go deep: multi-tenant isolation model (how you kept business units from seeing/affecting each other's traffic, quota, and cost), what "proper observability and cost tracking" concretely meant (per-tenant token/cost attribution, not just aggregate metrics), and how you handled multiple model types (LLMs, vision, embeddings) behind one gateway — this is directly relevant if this role touches the platform layer around Interpreter (Section 0/3.4 of the main file).
3. **Restaurant network integration at Careem** — your best story for the "Integrations" domain half and for messy, heterogeneous external data (onboarding, catalog sync, order lifecycle) — a close structural cousin to Partly's 20,000+-supplier ingestion problem (Section 3.6).
4. **Bukalapak (100M+ users) and RRQ Guild (first engineer)** — use these for scale and ownership/ambiguity questions respectively, not as your opener.

If asked directly to "walk through your architecture" for the AI Proxy, structure the answer the way a senior distributed-systems engineer should: components → data flow → failure modes → what you'd change with more time. Don't just describe what it does — name what breaks it and how you defended against that, unprompted.

### A.2 CS-fundamentals rapid-fire — the reported real questions, answered tight

Answer these in under 60-90 seconds each, out loud, before tomorrow. The interviewer is checking for a clear mental model, not a textbook definition.

**Stack vs. heap.** The stack holds function-call frames — local variables, return addresses — allocated and freed automatically as functions call and return, fast because it's just a pointer bump, and fixed in size (hence stack overflow on deep/infinite recursion). The heap holds dynamically-allocated memory with a lifetime not tied to any one function's execution, managed by the allocator (and in Go, garbage-collected) — slower to allocate, but necessary for anything that outlives the function that created it or whose size isn't known at compile time. In Go specifically: the compiler performs **escape analysis** to decide stack vs. heap per-variable — a value "escapes to the heap" if a pointer to it outlives the function (e.g., returned, or captured by a closure, or its size isn't known at compile time), and `go build -gcflags="-m"` shows you exactly which variables escaped and why. Senior-level add-on: this is *why* passing large structs by pointer isn't automatically faster — it can force a heap allocation and a GC-tracked pointer where passing by value would have stayed on the stack.

**Concurrency vs. parallelism.** Concurrency is about *structure*: dealing with multiple things at once — tasks that can be in progress simultaneously, potentially interleaved on a single core. Parallelism is about *execution*: multiple things physically happening at the same instant, which requires multiple cores. Rob Pike's framing (fitting, given the interviewer works in Go's ecosystem-adjacent world): "concurrency is about dealing with lots of things at once; parallelism is about doing lots of things at once." Go's goroutines give you concurrency by default (cheap, cooperatively/preemptively scheduled by the Go runtime onto OS threads); whether that concurrency becomes parallelism depends on `GOMAXPROCS` and whether the work is actually CPU-bound across multiple cores versus I/O-bound (where concurrency alone — overlapping waits — already gets you the win, no parallelism required).

**Compiler vs. interpreter.** A compiler translates source code into another form (machine code, bytecode, or another source language) *ahead of time*, producing an artifact that then runs independently — errors surface at compile time, and the translated output typically runs faster since translation cost is paid once. An interpreter executes source (or an intermediate representation) *directly*, line by line or instruction by instruction, at run time — no separate build artifact, generally slower per-execution since translation happens every run, but a faster edit-run loop. Go actually blurs this in an interesting way worth mentioning if you want to show depth: `go build` fully compiles to a native binary (no interpreter involved at runtime), but `go run` compiles to a temp binary and executes it in one step, which *feels* interpreted from the developer's chair without actually being one — a good example of the compiler/interpreter line being about developer experience packaging as much as runtime mechanics.

**Be ready for follow-ups on your own resume verbatim.** The Firebase-auth and Stripe-integration precedent means: expect them to pick a specific system you named (the AI Proxy's tenant-isolation mechanism, the retry queue's ordering guarantee, the restaurant integration's catalog-sync conflict handling) and go two or three layers deeper than your summary sentence. Don't over-polish the summary at the expense of being ready for "okay, how exactly did the error classifier decide retryable vs. not?"

### A.3 Go-specific senior questions likely in scope

Given the JD's explicit fundamentals list (concurrency, architecture, APIs, testing, design patterns) and that Go is very likely (though not confirmed — see main file Section 1) the primary language:

- **When would you *not* reach for a goroutine?** Senior signal: concurrency has a real cost in readability, testability, and debuggability — a senior engineer doesn't reach for `go func(){}()` reflexively. Answer: when the work is small/fast enough that goroutine scheduling overhead and synchronization complexity outweigh any parallelism gained, or when it introduces a race that a simpler sequential path wouldn't have.
- **How do you reason about memory in a hot path?** Escape analysis (above), avoiding unnecessary allocations in a loop (e.g., pre-sizing slices with `make([]T, 0, n)` when the size is known, reusing buffers via `sync.Pool` for high-churn allocations), and profiling with `pprof` rather than guessing.
- **Context propagation and cancellation** — a near-certain topic given "reliable, distributed systems" in the JD. Know `context.Context` as the standard mechanism for carrying deadlines/cancellation/request-scoped values across API boundaries and goroutines, and the idiom of always taking `ctx context.Context` as a handler's first parameter.
- **Error handling philosophy** — Go's explicit multi-return errors vs. exceptions; wrapping with `%w` and unwrapping with `errors.Is`/`errors.As`; when a sentinel error, a custom error type, or just a wrapped string is the right call.
- **Interfaces and design patterns** — small, consumer-defined interfaces (Go's idiom: accept interfaces, return structs) as the natural home for dependency injection and testability; be ready to name the repository/adapter pattern specifically, since it maps directly onto Partly's own many-supplier-data-sources problem (main file Section 3.6).

## Part B — The 90-minute take-home

### B.1 What's actually being evaluated (general senior-backend take-home consensus, not Partly-specific)

- **Trade-off articulation over volume of code.** A working, well-reasoned partial solution with clearly stated trade-offs beats a feature-complete solution with no commentary on what you'd do differently with more time.
- **Failure modes named unprompted.** What happens on a malformed input, a downstream timeout, a duplicate request, an empty result set — a strong submission handles or at least explicitly calls these out; a weak one only handles the happy path.
- **Testing as a first-class deliverable, not an afterthought** — the JD names "testing" explicitly as a fundamental. A few well-chosen table-driven tests covering edge cases signal more than 100% line coverage of the happy path.
- **Idiomatic code over clever code.** Simple, readable Go — small interfaces, clear naming, standard project layout — reads as more senior than a dense one-file trick.
- **A short README/notes explaining your decisions** is almost always worth the five minutes it takes: what you'd do differently with more time, what you deliberately skipped and why, and any assumptions you made about an ambiguous spec.

### B.2 Practice exercise — domain-realistic, timeboxed

Since no leaked prompt exists publicly, the highest-value use of your remaining prep time is a **domain-realistic practice run**, not guessing the literal prompt. Given 90 minutes and Partly's actual problem shape (main file Sections 0 and 3.2), a very plausible exercise shape is: *"Given a small in-memory catalog of parts and their fitment/supersession data, build a service that returns the correct current part(s) for a vehicle + damage description, handling at least one supersession case."* Below is a compact, idiomatic Go skeleton you can actually run and extend live tonight as a rehearsal — not to memorize, but to have the *shape* of a clean solution already in your hands so the real exercise is about the specific spec, not about remembering how to structure a Go service under time pressure.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"encoding/json"
)

// --- Domain types -----------------------------------------------------

type Vehicle struct {
	Make, Model string
	Year        int
	Trim        string
}

type Part struct {
	PartNumber  string
	Description string
	// SupersededBy holds the current replacement part number(s).
	// Empty means this part number is still current.
	// Modeling it as a slice handles "grouped supersession" (one old
	// number splitting into several current ones) from day one.
	SupersededBy []string
}

// FitmentQuery is what a client asks for.
type FitmentQuery struct {
	Vehicle Vehicle
	Damage  string // free-text damage description, e.g. "front bumper cracked"
}

type Recommendation struct {
	PartNumber string
	Confidence float64
	Note       string // e.g. "resolved from superseded part P-100"
}

// --- Interfaces (small, consumer-defined — idiomatic Go) --------------

// CatalogRepository abstracts the data source so it can be swapped
// (in-memory for the exercise, a real DB in production) without
// touching resolution logic — the repository/adapter pattern named
// in Part A.3, applied directly.
type CatalogRepository interface {
	FindCandidateParts(ctx context.Context, v Vehicle, damage string) ([]Part, error)
	ResolvePart(ctx context.Context, partNumber string) (Part, error)
}

// Resolver contains the actual business logic: fitment matching plus
// supersession resolution. Kept separate from the repository so it's
// unit-testable with a fake repository, no network/DB involved.
type Resolver struct {
	repo CatalogRepository
}

func NewResolver(repo CatalogRepository) *Resolver {
	return &Resolver{repo: repo}
}

// Resolve returns current, non-superseded part recommendations for a query.
func (r *Resolver) Resolve(ctx context.Context, q FitmentQuery) ([]Recommendation, error) {
	candidates, err := r.repo.FindCandidateParts(ctx, q.Vehicle, q.Damage)
	if err != nil {
		return nil, fmt.Errorf("finding candidate parts: %w", err)
	}
	if len(candidates) == 0 {
		// Explicit empty-result handling, not a silent nil slice —
		// a take-home grader will look for exactly this.
		return nil, ErrNoFitmentMatch
	}

	var recs []Recommendation
	for _, p := range candidates {
		resolved, note, err := r.resolveSupersession(ctx, p)
		if err != nil {
			return nil, fmt.Errorf("resolving supersession for %s: %w", p.PartNumber, err)
		}
		for _, rp := range resolved {
			recs = append(recs, Recommendation{
				PartNumber: rp.PartNumber,
				Confidence: 0.9, // a real system would score this; stubbed for the exercise
				Note:       note,
			})
		}
	}
	return recs, nil
}

// resolveSupersession follows a supersession chain (including grouped
// supersessions) to the current part(s). Guards against a cyclical
// chain, which is exactly the kind of edge case worth naming even if
// the data won't actually contain one — it signals you're thinking
// about data integrity, not just the happy path.
func (r *Resolver) resolveSupersession(ctx context.Context, p Part) ([]Part, string, error) {
	if len(p.SupersededBy) == 0 {
		return []Part{p}, "", nil
	}
	seen := map[string]bool{p.PartNumber: true}
	var current []Part
	queue := p.SupersededBy
	for len(queue) > 0 {
		next := queue[0]
		queue = queue[1:]
		if seen[next] {
			return nil, "", fmt.Errorf("supersession cycle detected at %s", next)
		}
		seen[next] = true
		np, err := r.repo.ResolvePart(ctx, next)
		if err != nil {
			return nil, "", err
		}
		if len(np.SupersededBy) == 0 {
			current = append(current, np)
		} else {
			queue = append(queue, np.SupersededBy...)
		}
	}
	return current, fmt.Sprintf("resolved from superseded part %s", p.PartNumber), nil
}

var ErrNoFitmentMatch = errors.New("no fitment match for given vehicle and damage description")

// --- HTTP layer ---------------------------------------------------------

func (r *Resolver) HandleResolve(w http.ResponseWriter, req *http.Request) {
	var q FitmentQuery
	if err := json.NewDecoder(req.Body).Decode(&q); err != nil {
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}

	recs, err := r.Resolve(req.Context(), q)
	switch {
	case errors.Is(err, ErrNoFitmentMatch):
		w.WriteHeader(http.StatusNotFound)
		json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
		return
	case err != nil:
		// A real service logs this with a request ID here before
		// returning a generic 500 — worth saying out loud even if
		// you don't wire up real logging in the 90 minutes.
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(recs)
}

func main() {
	// Swap inMemoryRepo for a real database-backed implementation —
	// that's the entire point of depending on the CatalogRepository
	// interface instead of a concrete type.
	r := NewResolver(&inMemoryRepo{})
	http.HandleFunc("/resolve", r.HandleResolve)
	http.ListenAndServe(":8080", nil)
}

// inMemoryRepo is the simplest possible CatalogRepository — enough to
// run the service end-to-end in the 90-minute window before swapping
// in something backed by real data.
type inMemoryRepo struct{}

func (f *inMemoryRepo) FindCandidateParts(_ context.Context, _ Vehicle, _ string) ([]Part, error) {
	return nil, nil
}

func (f *inMemoryRepo) ResolvePart(_ context.Context, _ string) (Part, error) {
	return Part{}, nil
}
```

And the test file — writing at least this much in a real take-home is what separates "I ran out of time for tests" from "testing wasn't a priority":

```go
package main

import (
	"context"
	"testing"
)

type fakeRepo struct {
	candidates map[string][]Part // keyed by damage description, for the test's sake
	parts      map[string]Part
}

func (f *fakeRepo) FindCandidateParts(_ context.Context, _ Vehicle, damage string) ([]Part, error) {
	return f.candidates[damage], nil
}

func (f *fakeRepo) ResolvePart(_ context.Context, partNumber string) (Part, error) {
	p, ok := f.parts[partNumber]
	if !ok {
		return Part{}, errNotFound(partNumber)
	}
	return p, nil
}

func errNotFound(id string) error { return &notFoundError{id} }

type notFoundError struct{ id string }

func (e *notFoundError) Error() string { return "part not found: " + e.id }

func TestResolve_TableDriven(t *testing.T) {
	tests := []struct {
		name        string
		damage      string
		candidates  []Part
		parts       map[string]Part
		wantCount   int
		wantErr     bool
	}{
		{
			name:   "direct match, no supersession",
			damage: "front bumper cracked",
			candidates: []Part{{PartNumber: "P-1", Description: "Front bumper"}},
			wantCount: 1,
		},
		{
			name:   "single supersession resolves to current part",
			damage: "front bumper cracked",
			candidates: []Part{{PartNumber: "P-OLD", SupersededBy: []string{"P-NEW"}}},
			parts: map[string]Part{"P-NEW": {PartNumber: "P-NEW"}},
			wantCount: 1,
		},
		{
			name:   "grouped supersession splits into multiple current parts",
			damage: "front bumper cracked",
			candidates: []Part{{PartNumber: "P-OLD", SupersededBy: []string{"P-LEFT", "P-RIGHT"}}},
			parts: map[string]Part{
				"P-LEFT":  {PartNumber: "P-LEFT"},
				"P-RIGHT": {PartNumber: "P-RIGHT"},
			},
			wantCount: 2,
		},
		{
			name:      "no candidates returns explicit error, not empty success",
			damage:    "unrecognized damage",
			wantErr:   true,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			repo := &fakeRepo{
				candidates: map[string][]Part{tt.damage: tt.candidates},
				parts:      tt.parts,
			}
			r := NewResolver(repo)
			recs, err := r.Resolve(context.Background(), FitmentQuery{Damage: tt.damage})

			if tt.wantErr {
				if err == nil {
					t.Fatalf("expected error, got none")
				}
				return
			}
			if err != nil {
				t.Fatalf("unexpected error: %v", err)
			}
			if len(recs) != tt.wantCount {
				t.Fatalf("got %d recommendations, want %d", len(recs), tt.wantCount)
			}
		})
	}
}
```

**Why this skeleton is worth rehearsing tonight, even though it's not the real prompt:** it exercises exactly the fundamentals the JD names — a small consumer-defined interface for testability, explicit error handling (including a domain-specific sentinel error, not just a bare `error`), a genuinely tricky piece of business logic (supersession chains, including the grouped case) handled with a cycle guard, table-driven tests covering the edge cases rather than just the happy path, and an HTTP layer kept thin and separate from business logic. If the real take-home is *this exact domain*, you've already rehearsed the hard part; if it's a different domain entirely, you've rehearsed the *shape* of a strong Go submission under time pressure, which transfers regardless.

### B.3 Time-boxing a 90-minute take-home

A structure that avoids the single most common take-home failure — running out of time with nothing working end-to-end:

1. **0-10 min**: read the spec twice, write down your assumptions and open questions (even if you can't ask them live), sketch the interfaces on paper/in comments before writing implementation code.
2. **10-55 min**: build the smallest end-to-end working version first — one path through the system, hardcoded/simplified where the spec is ambiguous, clearly noted as such. Resist the urge to handle every edge case before something runs.
3. **55-75 min**: tests — at minimum the happy path plus the two or three edge cases most likely to be graded on (empty input, not-found, the domain-specific tricky case like supersession above).
4. **75-90 min**: a short README covering what you'd do with more time (the honest, senior move — naming what you *didn't* get to and why is a stronger signal than silently omitting it), any assumptions you made, and how to run it.

## Last check before tomorrow

Back to the [priority checklist](#do-these-first-in-order-the-priority-checklist) at the top — if every box there is genuinely done, you're ready. Sleep beats one more read-through at this point.
