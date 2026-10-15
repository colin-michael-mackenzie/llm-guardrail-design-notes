Guardrailing an LLM Health Coach: Design Notes from a Production Prototype

Colin Mackenzie, October 2026

Earlier this year I was the sole engineer on an AI health coach prototype for a client (details anonymized). The product ingests a user's Apple Health data and uses an LLM (Anthropic's Claude) to surface potential health concerns. Health is about the least forgiving domain you can point a language model at: the data is sensitive, the stakes are personal, and a confident wrong answer is worse than no answer. This post is a summary of the safety-relevant design decisions I made, what worked, and what I'd push on further. None of this is novel research. It's the view from the implementation seat, which I think is where a lot of alignment theory either becomes real or quietly gets skipped.

The core stance: the model proposes, it does not decide

The most important decision was architectural, made before any prompt was written: the model's output never reaches the end user directly. Claude's job is to read compacted health data and produce candidate concerns. Those concerns land in a review queue for a human coach, classified by severity, with the model's explanation and recommendation attached. The human decides what, if anything, the user sees.

This one decision changed the risk profile of everything downstream. The failure mode of the system is no longer "the model told a user something dangerous." It is "the model wasted a reviewer's time," which is a cost problem rather than a harm problem. I think of this as choosing where the blast radius lands. In a health product, it must land on a professional whose job is to catch errors, not on the person the data describes.

Structured output as a safety boundary, not a convenience

Every model response is constrained to a JSON schema: an array of typed concern objects, each with a severity level, a plain-language explanation, and a recommendation field. Anything that does not parse against the schema is rejected and retried, and never enters the pipeline.

Teams usually adopt structured output for engineering convenience. I'd argue it is underrated as a safety mechanism:

It bounds what the model can say. There is no field for medical diagnosis, so the model cannot deliver one. The schema is a hard ceiling on the shape of the output, enforced by a parser rather than by politeness.
It makes review tractable. A coach triaging typed, severity-classed objects can apply consistent judgment. A coach reading free-form model prose cannot.
It makes failure visible. A model gone off the rails produces parse failures, which are loggable and countable. Free text that is subtly wrong produces nothing measurable at all.
A persona and off-topic sanitizer in front of the model

The system includes a guardrail layer that constrains the model to its health-coach persona and deflects off-topic or adversarial input. A health app holding personal medical data should not be jailbreakable into a general-purpose assistant, both because the context window holds sensitive data and because the product's authority ("my health app said...") must not be lendable to arbitrary content. The sanitizer treats user-influenced text as untrusted input rather than as instructions, which is the same posture I apply to prompt injection generally: data is data, not commands.

Context compression as attack-surface reduction

Raw HealthKit exports are enormous, so I compress samples into compact per-tile payloads before they reach the model, to fit output budgets. The safety side effect turned out to matter as much as the cost side: the less raw, unconstrained text you put in front of a model, the less surface there is for injection and the more predictable the outputs become. Curating the model's input is cheap, boring, and effective, and I now treat it as a first-class guardrail rather than an optimization.

Testing against synthetic clinical scenarios

You cannot responsibly test a health AI against real user data, so I built a mock data source that speaks the authentic HealthKit schema across twelve quantity, category, and correlation types, with four compound clinical scenarios layered on top. Compound scenarios are the interesting part: single-signal cases are easy, but the risky behaviors show up when multiple plausible signals interact and the model is tempted to over-infer. Building deliberate stress scenarios felt like a small, applied cousin of the model-organisms idea: construct the situation where the failure should appear, and check whether it does.

The same pattern in a lower-stakes domain

I used the same grounding discipline on a commerce product I co-founded (matrixxyz.com): an AI product finder where the model is grounded in the live catalog and constrained to structured output, so it can only recommend SKUs that actually exist and are in stock. Different domain, same principle: the model's expressive range is deliberately narrower than its capability, and the narrowing is enforced outside the model.

What I take from this

The through-line of all five decisions is that none of them rely on the model behaving well. They assume a capable model that will sometimes be wrong, sometimes be overconfident, and sometimes be adversarially prompted, and they arrange the system so those failures are caught, bounded, or rendered harmless. Prompting for good behavior is the start, not the mechanism.

What I find compelling about AI safety research, and the reason I'm writing this up, is that the production version of these questions keeps pointing at the research version. Schema constraints bound outputs, but what is the model doing when it strains against the schema? Sanitizers catch persona breaks, but what makes a persona stable in the first place? Human review catches bad outputs at prototype scale, but scalable oversight is exactly the question of what happens when it can't. After twenty-five years of shipping systems, these are the problems I most want to work on from the inside.

I'm a principal-level software engineer in Calgary. More at stackpartner.ca, code at github.com/colin-michael-mackenzie.
