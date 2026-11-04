# Chapter 04 — UltraChat-Style Multi-Turn Conversation Generation

## Why this chapter exists

Self-Instruct and Evol-Instruct produce single-turn `(instruction, response)` pairs. A chat model trained on single-turn SFT alone will be reasonable at one-shot requests and brittle at multi-turn behaviour: carrying context across turns, revising a previous response in light of new user information, following instructions that unfold across several exchanges.

UltraChat (Ding et al. 2023) is the recipe that fills this gap. It generates **multi-turn conversations** by structuring the generation around a topic tree rather than a seed pool, and by casting a *second* generator in the role of the user across turns. The result is a corpus of 10^5–10^6 multi-turn dialogues that teach a student model conversational behaviour at scale, without a human in the inner loop.

The original paper: Ding, Chen, Xu, Qin, Zheng, Hu, Liu, Sun, Zhou. *Enhancing Chat Language Models by Scaling High-Quality Instructional Conversations.* arXiv 2305.14233 (EMNLP 2023). <https://arxiv.org/abs/2305.14233>. The authors' released corpus (UltraChat) is public on Hugging Face: `stingning/ultrachat`.

## The recipe at a glance

Three sectors, each with its own generation structure:

- **Sector 1: Questions about the World.** Topic tree derived from a public knowledge taxonomy (Wikipedia-derived in the paper). Two generators converse: one plays the user asking about a topic, the other plays an assistant answering.
- **Sector 2: Writing and Creation.** Writing-task prompts generated from a task taxonomy (e.g., "write an email", "write a product description"). Two-generator conversation iterates on the writing artifact.
- **Sector 3: Assistance on Existing Materials.** The user turn provides an existing passage (drawn from a real corpus or generated as context); the assistant turn answers questions or performs transformations on it.

The three sectors produce qualitatively different data. In production you will usually run all three; for a targeted student capability, you may run only the sector that matches.

## Sector 1 — the topic tree

The topic tree is the generation backbone. A good tree is a two-to-three-level taxonomy rooted at a domain and branching into topics and sub-topics. The paper used a Wikipedia-derived taxonomy with around 30 top-level categories and ~1,000 sub-topics.

Shape of a tree node:

```yaml
- category: Natural Sciences
  subcategories:
    - name: Astronomy
      meta_topics:
        - origin and evolution of the universe
        - exoplanets and the search for life
        - black holes and relativistic astrophysics
    - name: Chemistry
      meta_topics:
        - periodic trends
        - ...
```

The tree is pinned as an artifact (`seeds/topic_tree.yaml`, hashed into the manifest). Replacing or extending the tree is a run-boundary event, not an in-run change.

### Where the tree comes from

Three sources, in descending order of practice:

1. **A public knowledge taxonomy.** Wikipedia's main-page categories, Library of Congress Subject Headings, DMOZ (archived), or Common Crawl's topic classifiers. Pros: broad, auditable, free. Cons: topic drift relative to your student model's intended domain.
2. **A curated internal taxonomy.** A smaller, hand-authored tree representing the topic surface the student should learn. Pros: matched to intended use. Cons: inherits the authors' blind spots.
3. **A generator-produced taxonomy.** Ask the generator to emit a two-level taxonomy over a named domain. Cheap and fast, but the taxonomy inherits the generator's biases and is not auditable without manual review.

Combinations are common: a public taxonomy, culled to the subset in-scope for the student, and extended with 20% generator-emitted sub-topics that fill specific gaps.

### Meta-topic expansion

For each sub-topic, generate ~30 *meta-topics* — concrete conversational angles on the sub-topic. A sub-topic like "black holes" expands to meta-topics like "how is an event horizon different from an inner horizon", "why do stellar-mass black holes have accretion disks", "what is the information paradox". The meta-topic is the seed for a single conversation.

The paper generates meta-topics with the generator itself, with the prompt shape:

> Give me 30 diverse questions / topics about {sub_topic}. Each should be a complete sentence or phrase that a curious person might ask or want to discuss.

Filter meta-topics for length, diversity (ROUGE-L against the growing list), and refusal patterns, exactly as in Self-Instruct (Chapter 02).

## Sector 1 — the two-generator conversation

For each meta-topic, run a multi-turn conversation. The paper uses *two* generator instances, prompted asymmetrically:

- **User-generator**: prompted to play a curious user. System prompt includes the meta-topic and a persona hint ("You are a curious person asking about X. Ask follow-up questions that build on the assistant's previous answer."). Temperature 0.8–1.0 for user-side diversity.
- **Assistant-generator**: prompted to play a knowledgeable assistant. System prompt is the usual helpful-assistant frame. Temperature 0.5–0.7 for assistant coherence.

The conversation alternates for a fixed number of turns (3–7 is typical) or until a stop condition triggers (one side refuses, the user generator declares the topic exhausted, a hard turn cap).

Conversation-level sanity checks:

- **Turn-topic drift.** Score the final user turn against the meta-topic. If the conversation has drifted off topic (embedding cosine < threshold), drop the conversation.
- **User-turn similarity.** The user generator can fall into a pattern of "That's interesting. What else?" — track the user turns' ROUGE-L against each other and drop conversations with too much intra-user repetition.
- **Assistant refusal.** Drop conversations with a refusal turn from the assistant unless the refusal is itself on-policy content you want in the SFT corpus.
- **Length distribution.** Assistant turns below a floor are usually degenerate; above a ceiling are usually runaway continuations.

## Sector 2 — Writing and Creation

A writing-task meta-topic ("write a product description for a new smart lamp") seeds a conversation where the user iterates on an artifact: initial request, first draft, revision request, revised draft, further revision, etc.

Structural differences from Sector 1:

- The conversation state is an *artifact* (the document being iterated on), not just the dialogue. Track the artifact explicitly so the user generator can ask the assistant to revise a specific section.
- The user-generator prompt is more constrained — persona-shaped around what a human editor would ask for. The paper sets the user-generator to simulate specific editor archetypes (clarity reviewer, concision reviewer, tone reviewer).
- Revision loops bound at 2–4 turns. Longer loops drift into the generator's self-dialogue bias.

## Sector 3 — Assistance on Existing Materials

A passage is the context. Tasks on the passage include summarisation, translation, extraction, rewriting, Q&A.

Two sources for the passage:

- **Real text.** Drawn from your existing acquisition slice ([`mod-103`](../mod-103-web-scale-acquisition-and-curation/README.md)). Licence travels with the passage; respect per-source licence constraints.
- **Generator-produced text.** Prompt the generator to emit a passage first, then conversation. Avoids licence propagation at the cost of amplifying generator biases.

The paper used both; most modern pipelines prefer real text where available.

### Filter stack for Sector 3

- **Passage fidelity.** The assistant turn must stay faithful to the passage. Score with an NLI-style judge or a reference-based metric on a sample.
- **Passage leakage.** The assistant must not quote long spans of the passage verbatim when summarising or rewriting; track quote length and ratio.
- **Task coverage.** Across a corpus, ensure the task types (summarise, translate, extract, rewrite, Q&A) are represented in the intended proportions.

## The three-sector manifest

```yaml
# recipes/ultrachat_config.yaml
seeds:
  topic_tree_path: seeds/topic_tree.yaml
  topic_tree_sha256: c4a8...
  meta_topics_per_subtopic: 30

generators:
  user_generator:
    model_id: claude-3.5-haiku-2024-10-22    # cheaper for the user role
    temperature: 0.9
    max_tokens: 512
  assistant_generator:
    model_id: claude-3.5-sonnet-2024-10-22
    temperature: 0.6
    max_tokens: 2048

sector_1_questions_about_the_world:
  turns_per_conversation: 5
  stop_on_refusal: true
  drift_threshold: 0.6

sector_2_writing_and_creation:
  tasks_taxonomy_path: seeds/writing_tasks.yaml
  revisions_per_task: 3

sector_3_assistance_on_existing_materials:
  passage_source: mod_103_curated_slice
  passage_source_licence_ref: generator/licence_ledger.yaml
  tasks: [summarise, translate_en_to_fr, extract_entities, rewrite_formal, qa]
  passage_leak_max_quote_ratio: 0.2

filters:
  user_turn_rouge_l_threshold: 0.8
  assistant_refusal_action: drop
  min_assistant_turn_tokens: 20
  max_assistant_turn_tokens: 2000

targets:
  sector_1_count: 300000
  sector_2_count: 100000
  sector_3_count: 100000

licence:
  teacher_licence_ref: generator/licence_ledger.yaml
```

## UltraChat is multi-generator

The paper is explicit that the user and assistant generators can be — and typically are — the same model. In practice, teams often use a cheaper or smaller model for the user role (where diversity is the priority) and a stronger model for the assistant role (where correctness is the priority). This is a cost lever; document the asymmetry in the manifest, because it has downstream training implications — the student learns to converse with a user whose voice is set by the user-generator's style.

## Where UltraChat breaks

### 1. Topic-tree drift

Symptom: the generated conversations cover only the top 20% of the tree. Sub-topics in the tail have no coverage.

Catch: track per-sub-topic conversation counts. A sub-topic with zero or near-zero conversations is a coverage failure.

Fix: enforce per-sub-topic quotas at the meta-topic expansion step. Round-robin through the tree rather than sampling proportionally to tree density.

### 2. User-generator refusal

Symptom: the user generator (especially a safety-tuned model) refuses to play its role on specific topics, dropping conversation counts on those topics without a signal.

Catch: separate user-side refusal rate from assistant-side refusal rate. Track per-topic refusal counts.

Fix: strengthen the user-persona system prompt. If the refusal rate stays high, consider a smaller or less safety-tuned user generator (acceptable because the user turns are *inputs* to the student, not training targets).

### 3. Assistant-turn self-consistency drift

Symptom: in a 5-turn conversation, the assistant's turn 2 contradicts its turn 1 (different claim, different formatting, different structure). The student model learns inconsistency as a pattern.

Catch: score multi-turn consistency with a judge model on a sample. A sudden drop in multi-turn consistency is a generator-health signal.

Fix: pin the generator version (same as Chapter 02 — version is a run boundary). Reduce max turn count. Restart the conversation if the consistency judge flags it.

### 4. Hidden contamination via the topic tree

Symptom: a topic in the tree is phrased such that the meta-topic expansion generates near-copies of benchmark items. The student model ends up contaminated via the synthetic surface. The mod-106 firewall on the pretrain corpus catches nothing because the path is through synthetic generation.

Catch: run the Chapter 10 firewall on the synthetic output corpus, not just the pretrain corpus.

Fix: enforce the generator-certification contract (Chapter 10); do not use a generator that has seen the benchmark unless you treat its output as untrusted.

## UltraChat variants and relatives

Two closely related recipes you will encounter:

- **Baize (Xu et al. 2023, arXiv 2304.01196)** — self-chat around a seed question drawn from Quora / StackOverflow. Single generator plays both sides; conversation terminates on exhaustion or N turns.
- **GPT-4 Alpaca / LongForm / OpenAssistant conversations datasets** — conversation generation with GPT-4 or community annotators. Not strictly UltraChat-shaped but consumed similarly downstream.

The licence posture is substantially different per recipe; Chapter 05's licence ledger applies here too.

## Summary

- UltraChat generates **multi-turn conversations** structured around a topic tree, with asymmetric user and assistant generators.
- Three sectors (Questions about the World, Writing and Creation, Assistance on Existing Materials) produce qualitatively different corpora; most pipelines run all three.
- The **topic tree** is the generation backbone. A good tree is broad, auditable, and pinned by SHA. Tree changes are run-boundary events.
- **Two-generator conversations** cast the user role to a cheaper model and the assistant role to a stronger one. The asymmetry is a cost lever; document it.
- Conversation-level filters (drift, intra-user repetition, refusal, length) are additive to the Self-Instruct heuristic filters. Multi-turn consistency is its own failure mode.
- Contamination can enter via the topic tree; the Chapter 10 firewall on the synthetic output corpus is enforced here.

Chapter 05 picks up with teacher-to-student distillation — the recipe where the generator's identity and licence become release-blocking artifacts.
