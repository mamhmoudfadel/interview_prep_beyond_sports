# Beyond Sports — Technical Delivery Lead Interview Prep
Interview date: Aug 20 · Panel: Edsart Boelens (Tech Lead), Thomas Meijles (Team Lead)

---

## 1. Project overview (Thomas)
**Q: Walk me through a recent project you're proud of, and what your role was on it.**

I work on a healthcare project built on a microservices architecture. My team has two seniors and two juniors, and we own two backend services that expose APIs to the frontend and receive sensitive clinical data from an external client over a critical endpoint. My split is roughly 60% hands-on coding, plus code review, pairing sessions, running technical deep-dive sessions on new features or tech we adopt, testing with xUnit, and working closely with QA. One thing I'm proud of is a performance issue we solved under load using async processing — happy to go into detail if useful.

*(Keep the opener under ~90 seconds — project, role, team shape, one teaser line. Let them pull the thread rather than diving into the full technical story unprompted.)*

---

## 2. Technical deep-dive: the backpressure fix (Edsart)
**Q: Walk me through the actual technical problem. What was breaking, and how did you first realize it?**

We had a backend service accepting a high volume of measurements via API, running blood pressure calculations, then exporting the results to Azure Blob Storage. Under high concurrent load, that flow was getting overwhelmed — the export step started failing, and I caught it through our alerts: 10–15 dropped export exceptions a day. I traced the root cause to the export step failing under load — the process couldn't keep up with concurrent throughput, and export attempts were failing without being retried, so files were silently lost.

I proposed decoupling the export with an async channel rather than doing it inline. I ran a POC comparing an in-memory channel (`System.Threading.Channels`) against a message broker (RabbitMQ), my team helped with unit and integration tests, and we shipped the in-memory channel. After deployment, export failures dropped to zero, and the API endpoint got faster too, since it was no longer blocked on the export step.

**Follow-up: What's your exposure if the process crashes mid-flight with messages still in the channel?**

If the process crashed with messages in the channel, we'd lose them completely — that's a real risk. We discussed mitigating it with a retry mechanism or a database-backed durable buffer, but decided against building an interim layer since we were already planning to migrate to RabbitMQ, which gives us durability and retries natively. Message volume was low enough and crashes rare enough that the risk was acceptable short-term, so we prioritized getting the broker migration scheduled rather than building throwaway durability.

**Follow-up: Where does the RabbitMQ migration stand today?**

It's on our backlog — I proactively flagged it to our PM once we saw the channel fix working, because we expect throughput to grow significantly over the coming months. It hasn't started yet, but it's prioritized ahead of becoming urgent.

---

## 3. Concurrency (Edsart)
**Q: How do you handle thread safety when multiple requests hit a shared resource? Give a real example.**

We had a background service processing data with calculations based on an external API. Without a coordination mechanism, we found more than one thread ended up calculating on the same piece of data — a real race condition. We fixed it by routing work through a channel-based queue: since we had a single consumer reading from the channel, each item was processed by only one thread at a time, which eliminated the race condition entirely.

*(If pushed on "why does a channel guarantee safety": it's single-consumer semantics, not magic — with one consumer, only one thread pulls and processes each item. Multiple consumers would need partitioned or idempotent work to stay safe.)*

Other tools I use for concurrency: `ConcurrentDictionary`/`ConcurrentBag` for shared state instead of manual locking, `SemaphoreSlim` to cap concurrent work in background services, and async/await throughout to avoid blocking threads unnecessarily.

---

## 4. Live system design under pressure (Edsart)
**Q: Design an ingestion pipeline for high-frequency sensor data — hundreds of events/sec. What's in v1, what do you deliberately leave out?**

**v1:** Ingest → message broker (Kafka or RabbitMQ, using an outbox pattern for delivery guarantees) → consumer → persistent storage. Kafka's built-in partitioning gives natural ordering and retry semantics per partition.

**Deliberately deferred:** auto-scaling consumers — start with a fixed pool and scale manually once real throughput data justifies it.

**Kept in v1, not deferred, with reasoning:**
- Schema versioning — needed early since message formats will evolve and backward compatibility matters from day one.
- Real-time alerting — core to the product; users depend on real-time notification.
- Multi-region — needed given the product tracks live sporting events globally (Europe, US, potentially Middle East).

*(Structure tip: state the split up front — "here's what I'd build in v1, then what I'd defer" — before diving in.)*

---

## 5. Match-day incident response (Edsart)
**Q: Viewer numbers are 5x expected on match day and frames are lagging. What do you check first, and what are your options live?**

First, I'd check the monitoring dashboard — event throughput per second, bandwidth, which service is generating alerts/exceptions — to identify whether the bottleneck is ingestion or fan-out to viewers. Fan-out (Web PubSub/consumer capacity) is usually easier and faster to scale horizontally than the ingestion path, which is fixed by the sensor feed rate. If it's consumer lag, I'd scale out consumer instances immediately — that's a fast, low-risk lever I can act on without waiting for approval. As a last resort mid-match, I'd consider graceful degradation — e.g., dropping update frequency from 10Hz to 5Hz for viewers — better to stay live and coherent than fall further behind trying to hold full fidelity.

---

## 6. Mentoring (Thomas)
**Q: Tell me about someone you helped grow technically. What did you actually do?**

I built and ran a 6-month development plan for Sara, a junior engineer on my team. Weekly sessions combined a technical deep dive with a hands-on POC: complex task breakdown, EF Core internals and SQL query profiling, async/await in depth, RESTful API design and versioning, messaging/broker concepts, authentication/authorization protocols, and xUnit testing. Beyond the technical side, I also prepared her for client-facing work — joining client calls and eventually running demos herself.

**Q: Tell me about a time you had to give an engineer difficult feedback.**

*(Note: keep this framed as feedback that helped growth, not necessarily "difficult" — the strongest real example is low-friction.)* Sara wrote a complex SQL query that worked but wasn't efficient. In code review I gave her specific, concrete feedback — change the join, project only needed columns, add pagination — then followed up with a dedicated SQL pairing session on the reasoning behind it. That led to her learning to profile queries and read execution plans herself, and she got noticeably better at handling complex queries going forward.

---

## 7. Disagreement with an architect (Edsart)
**Q: Tell me about a time you disagreed with an architectural decision. What happened?**

On a monolithic project, our architect proposed building a new feature as an independent microservice — partly to train the team on working with a microservices architecture ahead of a broader migration. I disagreed: the new feature was closely dependent on the existing monolith's data model, and I felt splitting it out would add deployment and communication complexity without real microservices benefits yet. We debated it over several discussions — his reasoning was genuinely about building team capability for the future; mine was about near-term delivery efficiency. His decision stood. I fully committed: built the service, set up its own repo and CI/CD, and taught the team how to develop and deploy it independently.

**Q: Describe a time you were wrong and had to change your position.**

*(Same story, told from the outcome side)* In practice, it played out close to what I'd worried about — we duplicated logic between the services, and communication overhead (HTTP calls, retry handling, separate logging/monitoring) made root-causing issues take noticeably longer than it would have in the monolith. Looking back, we didn't see a business benefit that justified that cost at that stage. We haven't rolled it back, but it's an open question for the team.

*(A separate, smaller "wrong" story if needed: choosing to embed full payloads inside broker messages instead of using a lightweight notification + API pull pattern — caused broker bandwidth/throughput issues, corrected by redesigning to notification + API.)*

---

## 8. Stakeholder pushback (Thomas)
**Q: Tell me about a time a client pushed for something you knew was technically risky.**

I was tech lead on an investment mapping platform — a frontend map showing investment points, backed by our API. The client wanted every project loaded on the map at once. I explained the risk: a massive API payload and real performance problems on both backend and rendering. After a few days of discussion, I proposed batched loading with a configurable threshold (e.g. 50–100 points at a time) instead of a hard cap. We agreed on that approach — the client still saw everything eventually, just loaded progressively rather than all at once.

---

## 9. Stakeholder communication on trade-offs (Thomas)
**Q: How do you explain a technical trade-off to a non-technical stakeholder?**

After the channel fix proved out — zero export failures, confirmed via App Insights — I flagged to our PM that the in-memory channel wouldn't hold up against expected throughput growth over the coming months. I framed it as a clear trade-off: migrate to a message broker now while it's low-risk and non-urgent, or risk having to do it reactively under pressure once volume causes a real problem. That "cost now vs. cost under pressure later" framing got it prioritized onto the backlog.

---

## 10. Testing / TDD (Edsart)
**Q: Walk me through how you actually apply TDD in practice.**

It's pragmatic, not dogmatic. For features with clear, well-understood scenarios, I write happy-path tests first, true TDD-style, and let that drive the implementation. For more complex logic, I write the code first, then add negative and edge-case scenarios to the suite — xUnit for both unit and integration tests. More recently I've also used AI tooling, like Claude, to help generate test scenarios — sometimes running one agent to write tests and another to write the implementation — then reviewing and adjusting by hand and adding any missing edge cases myself.

---

## 11. Delivery under a hard deadline with multiple clients (Thomas)
**Q: Tell me about a time you had to deliver under real pressure, with a deadline you couldn't move.**

We had two features due for two different clients around the same time. Partway through development, it became clear we wouldn't hit both deadlines at full scope. I brought it to the PM and stakeholders early rather than waiting — we categorized the work into must-have versus nice-to-have, and decided to deliver the priority feature in full and a reduced scope of the second, which protected time for QA rather than cutting testing to make the date. We also brought the client into that conversation directly, so it wasn't a surprise on delivery day — they understood and agreed to the trade-off.

*(Have a closing beat ready: how the client received the reduced-scope feature, and whether the rest shipped in the next iteration as planned.)*

---

## 12. Cloud flexibility — Azure vs AWS (Edsart)
**Q: If a client insisted on AWS when your team's expertise is mainly Azure, how would you approach it?**

It wouldn't be trivial, but it's manageable. I'd be upfront with the client and management that it's a real shift, not a seamless one, and ask for spike/training time for the team to get comfortable with the AWS equivalents — IAM instead of Azure RBAC, S3 instead of Blob Storage, SQS/SNS or Lambda instead of Service Bus/Event Hubs or Azure Functions. Then we'd migrate gradually, service by service, rather than a big-bang cutover. The upside is that most of the architectural thinking — async processing, message-driven design, partitioning, channels vs brokers — is cloud-agnostic; it's really the specific SDKs and managed services that need relearning, not the underlying design judgment.

---

## 13. Why Beyond Sports (Thomas / culture fit)
**Q: Why Beyond Sports, and why this role, at this point in your career?**

A few things drew me here specifically. Technically, this is exactly the kind of problem I find most interesting — high-frequency, real-time data at scale, where the engineering challenge is genuinely hard: latency, backpressure, reliable delivery under load. That's the work I've spent the last few years getting good at, and here it's not a side concern, it's the core of the product. Sport is also something I'm personally passionate about, so being able to work on systems that directly shape how fans experience the game matters to me. And at this stage in my career, moving into a Technical Delivery Lead role that combines architecture ownership, hands-on engineering, and mentoring is exactly what I want next — Beyond Sports, working with partners like the NFL, NHL and FIFA under Sony, is a serious environment to do that at scale.

---

## 14. Reliability vs. scale trade-off (Edsart)
**Q: Reliability first, or scale first, for a new service?**

Reliability first. A feature that's fully working but only serves a small load is still valuable and trustworthy; a feature that's fast and scaled but unreliable just fails more people, faster. Early on I'd invest in making the core behavior correct and stable — solid unit and integration test coverage, performance testing to know the actual limits — before investing in scaling infrastructure. Once reliability is proven, scaling becomes safer and more predictable, because you're scaling something you already trust.

*(If pushed — "what if the client needs both from day one for a live match deadline?": reliability and scale aren't fully sequential in practice; design for scale from the start — stateless services, async processing, partitioning — but validate reliability before investing heavily in scaling infrastructure. It's about sequencing effort, not ignoring scale.)*

---

## 15. Leading through a teammate's performance dip (Thomas)
**Q: Tell me about a time someone on your team wasn't performing at the level you needed. How did you handle it?**

I had a senior engineer who was consistently strong — good code quality, on-time delivery, solid technical input with the team. About three months in, I noticed a real drop — recurring issues on things we'd already aligned on, less engagement generally. I started with a private one-to-one, not to correct him but to understand what was going on, staying open to a few possibilities — personal issues, burnout, feeling over- or under-loaded, or not feeling enough ownership. It turned out he was dealing with some personal issues outside work that were weighing on him. I told him he could talk to me as a friend, not just as his lead, and made sure to check in regularly so he didn't feel like he was carrying it alone. Over time, his performance and engagement came back, and he's fully re-engaged with the team now.

---

## Delivery notes for the 20th
- **Structure**: Situation (2 sentences) → Task (1 sentence) → Action (concrete: "I did X, then Y") → Result (1–2 sentences, numbers where you have them). Land each story under ~90 seconds.
- **Don't lead with your general philosophy** ("I always give healthy feedback...") — go straight to the one specific instance. If they want the general approach, they'll ask.
- **Pause before answering open design questions** — 5–10 seconds of silence is normal and expected at this level.
- **State your structure up front** on multi-part questions ("Let me give you what I'd build in v1, then what I'd defer") to avoid drifting.
- If asked whether TDD is "really" TDD given your mixed approach — own it plainly rather than getting defensive.
- On the microservices disagreement: frame it as two people optimizing for different things (his: team capability; yours: delivery efficiency) rather than "I was right, he was wrong."
