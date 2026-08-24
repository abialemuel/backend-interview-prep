# Round 4: Loop Interview — Values (and a live-coding contingency)

Per Partly's official published hiring journey, the stage after the Deep Technical Interview (which you've passed) is the **Loop Interview: Values** — three separate 45-minute Google Meet conversations, each with a different Partly team member, each anchored to one of the company's six named values. If your recruiter also told you this round is "live code," the most likely reading is that **one (or more) of these three sessions pairs a values conversation with a live-coding segment**, run by an engineer rather than a non-technical teammate — a common real pattern even in "values"-titled loops. This file preps both halves so you're covered either way: don't skip the values prep to over-index on coding, since the round's own name says values is the primary axis being graded.

## The six values, and a real story mapped to each

These are Partly's own published values (main file, Section 0), each below with a story pulled from your actual background — pick the three most relevant if that's what a given interviewer probes, but have all six ready since you don't know in advance which three (of six) values this loop's three sessions will cover.

### Ownership & Initiative
*"Care deeply, take responsibility, look for solutions, take initiative, don't wait for instructions, never say 'that's not my job.'"*

**Your story: first engineer at RRQ Guild.** There was no backlog, no senior engineer to ask, no existing pattern to copy — you helped shape the backend strategy from a blank page and built the community platform from zero. Close the story by naming the value directly: *"That's the posture I bring by default — I'd rather diagnose the actual problem myself than wait to be told what to build, which is part of why 'no sprints or scrums, problems not specs' is something I read on your engineering blog and thought, that's how I already work."*

### Ingenuity & Resourcefulness
*"Can-do attitude, practical problem-solving, build with very little."*

**Your story: the Telkom network monitoring redesign that cut memory usage by ~80%.** Frame it as: existing system worked but was resource-heavy at a national-telco scale; rather than throwing more infrastructure at it, you redesigned the approach itself. Name the specific trade-off you made (e.g., what you gave up — polling frequency, in-memory retention window, a less granular data structure — for the memory win) so it reads as engineering judgment, not just "I optimized it."

### Buyer-Focused
*"Obsess over the buyer, work backwards, build trust, put yourself in their shoes."*

**Your story: the Saudi restaurant network integration at Careem.** Partner onboarding, catalog sync, order lifecycle, delivery tracking — this is a story about a *business* (the restaurant partner) as your actual customer, not an abstract API consumer. Emphasize any point where you made a call that traded engineering convenience for the partner's actual experience (e.g., how catalog sync handled a partner's messy/inconsistent menu data without pushing that mess back onto them).

### Trust & Communication
*"Reliable, respectful, admit mistakes, value thoughtful feedback."*

**Your story: the UAE government API reliability rebuild.** This is your strongest story for this value specifically because it's fundamentally about *admitting something was broken* (regulated events silently dropping) and fixing it properly (an error classifier + an ordered retry queue) rather than patching around it quietly. If asked "tell me about a mistake or a system you inherited that wasn't working," lead with the honesty of naming the failure mode plainly, not just the fix.

### Bias Toward Action
*"Calculated risk, speed is critical, deliver results."*

**Your story: building the Telkom AI Proxy and MCP Orchestrator from scratch.** Frame the speed angle explicitly — you didn't wait for a fully-specified platform strategy before shipping something real; you built production infrastructure (multi-tenant isolation, multiple model types) that then organically became the company's internal AI platform across business units, rather than a slow, over-planned rollout.

### Intellectual Rigor
*"Care about detail, rethink fundamentals, skeptical when data and anecdotes differ, extraordinary claims need extraordinary evidence."*

**Your story: infrastructure at Bukalapak serving 100M+ users.** This value rewards precision over confidence — have a specific example ready where you didn't trust an assumption (a metric, a "this should scale fine" claim, a partner's stated data format) and verified it directly before building on it. If you don't have a crisp Bukalapak-specific instance in mind, the government-feed story also fits here: rethinking the *fundamental* assumption that retries alone were sufficient, rather than patching around symptoms.

## How to actually run each 45-minute session

- **Assume one clear behavioral question per value, with real follow-up depth** — these interviewers are trained to dig, not just collect an anecdote. Have the STAR shape ready (Situation, Task, Action, Result) but be ready to go two levels deeper than your opening answer on any of them, the same way Round 2 taught you to expect follow-ups on your resume verbatim.
- **Name the value explicitly at least once per session**, the way the six story closers above do — it's a small thing, but it signals you did the homework rather than happening to have relevant experience.
- **Ask each interviewer one real question**, not a generic "what do you like about working here." A strong, values-loop-appropriate question: *"Which of the six values do you personally find hardest to live up to day-to-day, and why?"* — genuinely interesting, hard to fake an answer to, and shows you take the values as operating principles rather than a poster on a wall.

## Live-coding contingency (if a session pairs values with code)

If one of the three sessions does include live coding, the skill being tested is different from the take-home you already passed: **communication while coding**, under direct observation, in real time. The three things that separate a strong live-coding performance from a weak one at senior level, regardless of the specific problem:

1. **Clarify before coding.** Restate the problem in your own words, ask about edge cases and constraints (input size, duplicates, empty input, ordering guarantees) *before* writing a line — this is exactly the "problems, not specs" posture Partly says it hires for, demonstrated live.
2. **Narrate your plan before your fingers move.** State the approach and its complexity out loud, then implement — silence while typing reads poorly in a live/remote format specifically, because the interviewer has nothing to evaluate except a blank pause.
3. **Talk through the test cases you'd want, even if you don't have time to write them all.** Naming "I'd want a test for the empty-input case and for duplicate keys" costs ten seconds and signals the same testing discipline the take-home round already rewarded.

Three warm-up problems below, each with the narration script alongside the code — rehearse saying the italicized lines out loud, not just reading the code silently.

### Two Sum — the standard opener, rarely skipped even at senior level

*"I'll start with the brute force to make sure I have the problem right, then optimize — brute force is O(n²) checking every pair, but I can trade space for time with a hash map to get O(n)."*

```go
func twoSum(nums []int, target int) []int {
	seen := make(map[int]int, len(nums)) // value -> index
	for i, n := range nums {
		if j, ok := seen[target-n]; ok {
			return []int{j, i}
		}
		seen[n] = i
	}
	return nil // explicit: no pair found, caller must handle this case
}
```

*"One thing worth naming: this assumes exactly one solution exists, per the classic problem statement — if that's not guaranteed here, I'd return `(result, bool)` or an error instead of a bare nil so the caller can't silently treat 'not found' as a valid answer."*

### Merge Intervals — tests handling of edge cases and sort-based reasoning out loud

*"The key insight is that if I sort by start time first, any intervals that overlap must be adjacent in that sorted order — so I only ever need to compare against the last merged interval, not all of them."*

```go
import "sort"

type Interval struct{ Start, End int }

func mergeIntervals(intervals []Interval) []Interval {
	if len(intervals) == 0 {
		return nil
	}
	sort.Slice(intervals, func(i, j int) bool {
		return intervals[i].Start < intervals[j].Start
	})

	merged := []Interval{intervals[0]}
	for _, cur := range intervals[1:] {
		last := &merged[len(merged)-1]
		if cur.Start <= last.End { // overlaps (or touches) the last merged interval
			if cur.End > last.End {
				last.End = cur.End
			}
		} else {
			merged = append(merged, cur)
		}
	}
	return merged
}
```

*"Edge case worth calling out: `cur.Start <= last.End`, not `<` — I need to decide out loud whether touching intervals (end of one equals start of the next) should merge, since that's a real ambiguity in the spec, not just an implementation detail."*

### Worker pool with cancellation — the Go-concurrency live-coding staple, given the JD's emphasis

*"I'll use a fixed pool of goroutines reading from a shared job channel, with a `context.Context` so the caller can cancel outstanding work — this is the pattern I'd reach for any time I need bounded concurrency rather than spawning one goroutine per item, which doesn't scale under load."*

```go
func processJobs(ctx context.Context, jobs []int, workers int, process func(int) (int, error)) ([]int, error) {
	jobCh := make(chan int)
	resultCh := make(chan int, len(jobs))
	errCh := make(chan error, 1)

	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				select {
				case <-ctx.Done():
					return
				case j, ok := <-jobCh:
					if !ok {
						return
					}
					res, err := process(j)
					if err != nil {
						select {
						case errCh <- err:
						default: // don't block if another worker already reported an error
						}
						return
					}
					resultCh <- res
				}
			}
		}()
	}

	go func() {
		defer close(jobCh)
		for _, j := range jobs {
			select {
			case jobCh <- j:
			case <-ctx.Done():
				return
			}
		}
	}()

	go func() {
		wg.Wait()
		close(resultCh)
	}()

	var results []int
	for res := range resultCh {
		results = append(results, res)
	}
	select {
	case err := <-errCh:
		return results, err
	default:
		return results, ctx.Err()
	}
}
```

*"Two things I'd flag unprompted: the error channel is buffered size 1 with a non-blocking send, specifically so one worker's failure can't deadlock the others still draining `jobCh` — and I'm returning partial results alongside the error, because in most real systems 'some work succeeded before the failure' is more useful to the caller than throwing it all away."* This is a denser answer than the JD strictly requires for a warm-up, but volunteering the deadlock-avoidance reasoning and the partial-results design choice — both real senior judgment calls, not syntax — is exactly the kind of unprompted depth that separates a senior live-coding answer from a mid-level one.

## Final checklist before this round

- [ ] Say each of the six value-closer lines above out loud once, in your own words — not memorized verbatim, or it'll read as scripted.
- [ ] Confirm with the recruiter (a short message is fine) whether "live code" means a coding component inside the Values Loop, or something separate — removing the ambiguity costs nothing and a same-day reply is common.
- [ ] Rehearse the worker-pool problem specifically once, narrating out loud the whole time, since it's the one most likely to actually come up given the JD's concurrency/distributed-systems emphasis.
- [ ] Have your one real question ready for all three interviewers (the "which value is hardest to live up to" one, or your own variant) — asking the *same* thoughtful question of all three, and noting how their answers differ, is itself a small signal of intellectual rigor.
