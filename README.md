# Placement Notes: Master Index and Revision Guide

One page to find the right section fast. Section ids repeat across parts (for example S and G), so always read them as **Part + section**.

## The eight parts (Parts 7 and 8 are in the addendum at the end)

| Part | Open | What it holds |
| --- | --- | --- |
| 1 | [DSA](https://claude.ai/artifact/FXMz2QgejyBA35S7boi79h) | 25 pattern sections with Python code, loopholes, practice lists; **Appendix A** has the same templates in C++; **section 25** is the pattern cheat sheet |
| 2 | [OS, DBMS, Networks, systems basics](https://claude.ai/artifact/X4WNqh4PoDWFcvtJ1DhieM) | A1 to A9 (OS), B1 to B11 (DBMS), C1 to C12 (networks), D1 to D6 (C++/Linux/architecture), **E = rapid-fire sheet** |
| 3 | [SQL practice and Git](https://claude.ai/artifact/LV6RXhUSVCcZWJ8o31Cg83) | S0 to S6 (sample schema, joins, windows, practice set), G1 to G8 (Git, GitHub Actions, scenarios) |
| 4 | [OOP, patterns, LLD](https://claude.ai/artifact/LmYDdgmKYuBYhdMmbVempX) | O1 to O8 (OOP, SOLID, testing), P1 to P11 (patterns), U1 (UML), L0 to L13 (12 LLD problems + revision sheet) |
| 5 | [System design (HLD)](https://claude.ai/artifact/HPrq11dFMXv5MR5rVeYke8) | H0 to H30 (fundamentals), S1 to S10 (worked designs), R1 to R5 (revision sheet) |
| 6 | [Backend, ML, GenAI, projects, hiring](https://claude.ai/artifact/Fby4QQi49vnmFN15najpTd) | B1 to B10 (backend/deploy), M1 to M11 (ML), G1 to G7 (GenAI), P1 to P9 (projects), J1 to J13 (job hunt) |

## What to open for each situation

| Situation | Read first | Then |
| --- | --- | --- |
| **Online assessment tomorrow** | Part 1 section 25 (patterns), Part 1 section 1 (constraints to complexity), Part 6 J11 (OA strategy + aptitude formulas) | Part 3 S4 (window functions) if SQL appears |
| **DSA round** | Part 1 section 25, then the *Recognize* and *Loopholes* lines of the pattern sections | Part 1 Appendix A if you code in C++ |
| **OS / DBMS / CN round** | Part 2 E (rapid-fire sheet) | Part 2 A4 (scheduling numericals), A6 (Banker's), A8 (page replacement), B4 (normalization), B7 (isolation levels), C3 (subnetting), C12 (URL flow) |
| **SQL round** | Part 3 S2 (joins and NULL traps), S4 (windows), S6 (practice set) | Part 2 B6 and B10 (indexes, optimization) |
| **Git / CI questions** | Part 3 G3 (undo toolbox), G4 (merge vs rebase), G8 (scenario Q&A) | Part 3 G7 (GitHub Actions) |
| **OOP round** | Part 4 O1 to O4 (pillars, SOLID) | Part 4 P1 to P11 (patterns) |
| **LLD round** | Part 4 L0 (framework), then the matching problem in L1 to L12; L13 is the revision table | Part 4 O7 (thread safety), U1 (UML) |
| **HLD round** | Part 5 H0 (framework), R1 to R5 (revision), then the matching system in S1 to S10 | H7 (caching), H9 (DB scaling), H10 (CAP), H14 (queues/Kafka) |
| **Backend / deployment round** | Part 6 B1 to B9 | Part 5 H18 to H22 (jobs, idempotency, retries), H26 to H27 (auth, security) |
| **ML / GenAI role** | Part 6 M2 to M4 (bias-variance, leakage, metrics), G3 to G6 (RAG, eval, injection) | Part 6 M8 to M10, G7 |
| **Project deep-dive** | Part 6 P1 to P9 | Part 6 B9 (verify live links), Part 5 H0 (scale your own project) |
| **HR / behavioural round** | Part 6 J3 to J6 (intro, STAR, HR bank, questions to ask) | Part 6 J7 (CTC and negotiation) |
| **Applying and referrals** | Part 6 J1, J2, J8, J9, J12 | Part 6 J10 (mocks and mistake log) |

## Fast topic lookup

| Topic | Where |
| --- | --- |
| Time complexity, constraints to algorithm | Part 1 section 1 |
| Sliding window, two pointers, prefix sums, binary search on answer | Part 1 sections 4 to 7 |
| Monotonic stack/deque, heap and Top-K | Part 1 sections 10 to 12 |
| Graphs: BFS/DFS, topo, DSU, Dijkstra, MST, SCC | Part 1 section 17 |
| Backtracking, greedy, DP (1D, 2D, knapsack, bitmask) | Part 1 sections 18 to 20 |
| Process vs thread, scheduling, deadlock, paging | Part 2 A2, A4, A6, A7 |
| ACID, keys, normalization, indexes, isolation | Part 2 B2 to B4, B6, B7 |
| TCP vs UDP, DNS, HTTP, TLS, status codes | Part 2 C2, C5 to C8 |
| C++ pointers, RAII, smart pointers, STL | Part 2 D1 to D3 |
| Window functions, nth highest, gaps and islands | Part 3 S4 |
| SOLID with violation and fix | Part 4 O4 |
| Singleton, Factory, Strategy, Observer, State | Part 4 P1, P2, P4, P5, P10 |
| LRU cache, TTL cache, parking lot, elevator | Part 4 L1, L3, L4, L5 |
| Capacity estimation, latency numbers | Part 5 H3, H4 |
| Caching problems, Redis | Part 5 H7 |
| CAP, PACELC, quorum | Part 5 H10 |
| Consistent hashing, Snowflake IDs | Part 5 H11, H12 |
| Rate limiter algorithms | Part 5 H19, S2 |
| Saga, outbox, idempotency | Part 5 H20, H24 |
| Docker, Compose, deployment troubleshooting | Part 6 B2, B8 |
| Confusion matrix, F1, ROC-AUC | Part 6 M4 |
| RAG pipeline, hallucination, prompt injection | Part 6 G3, G5, G6 |

## How to revise in the last 24 hours

1. Skim the **Loopholes** of the topics your interview is likely to cover; these are the follow-up traps.
2. Re-read your project walkthroughs and STAR stories (Part 6 P1, J4).
3. Redo two problems you previously failed, from your mistake log (Part 6 J10).
4. Read the matching revision sheet: Part 2 E, Part 4 L13, Part 5 R1 to R5.
5. Sleep.

## Known caveats

- The code in all parts was not compiled or run during writing; test snippets before relying on them.
- Capacity numbers, latencies and engine-specific behaviour (PostgreSQL, Redis, Kafka, TLS 1.3, cloud services) should be checked against current documentation.
- Project walkthroughs, resume bullets, STAR stories and referral messages are templates with sample details; replace them with your own true facts.

---

## Addendum: Parts 7 and 8

| Part | Open | What it holds |
| --- | --- | --- |
| 7 | [Aptitude](https://claude.ai/artifact/76FqqWpTe9uzeofEY4GoYp) | A1 to A16: mental maths, number system, percentages, profit and loss, interest, ratios, averages, speed and distance, time and work, combinatorics, probability, series, mensuration, data interpretation, logical reasoning, verbal; **A17 = 30-question timed practice set with answers** |
| 8 | [Linux in depth](https://claude.ai/artifact/Ec1zy2qWjEJ8w3QvdE6Ydm) | L1 to L12: architecture and boot, filesystem and links, permissions, processes and signals, pipes and text tools, shell scripting, systemd and cron, networking and SSH, troubleshooting playbook, security, containers on Linux, **L12 = 40 rapid-fire questions, scenarios, cheat sheet** |

**Extra rows for the situation table**

| Situation | Read first | Then |
| --- | --- | --- |
| **Aptitude test or aptitude section in an OA** | Part 7 A1 (mental maths), then the topic sections you are weakest in | Part 7 A17 (timed practice set), Part 6 J11 (OA strategy) |
| **Linux / shell / DevOps screening** | Part 8 L12 (40 questions and scenarios) | Part 8 L3 (permissions), L4 (processes), L5 (text tools), L6 (scripting), L9 (troubleshooting) |
| **Backend role asking about servers and deployment** | Part 8 L7 (systemd, cron), L8 (networking, SSH), L9 | Part 6 B2 and B8 (Docker, cloud) |

**Extra rows for the topic lookup**

| Topic | Where |
| --- | --- |
| Percentages, profit and loss, interest | Part 7 A3 to A5 |
| Time and work, speed and distance | Part 7 A8, A9 |
| Permutations, combinations, probability | Part 7 A10, A11 |
| File permissions, umask, setuid and sticky bits | Part 8 L3 |
| Hard vs symbolic links, inodes, disk full | Part 8 L2, L9 |
| Log analysis one-liners (grep, awk, sort, uniq) | Part 8 L5 |
| Shell scripting pitfalls and templates | Part 8 L6 |
| systemd unit files, cron, journalctl | Part 8 L7 |
| Connection refused vs timed out, SSH keys and tunnels | Part 8 L8 |
| Namespaces, cgroups, exit code 137 | Part 8 L11 |
