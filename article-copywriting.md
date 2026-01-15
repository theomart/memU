# Théo Martin LinkedIn Style Guide

## AI Tech Lead @Akeneo | Ex-Amazon

A copywriter’s blueprint for replicating Théo’s distinctive LinkedIn voice.

-----

## 1. POST STRUCTURE

### Opening Hook Pattern

- **Lead with a single emoji** that sets the tone (🐬, 🍄, 🎯, 🚨, 💒, 🦎, 😖, 🤔, ⚡, 🎉)
- **Follow with a punchy headline** that promises value or asks a provocative question
- Examples:
  - `🐬 Takeways from the Mixtral paper with no chitchat.`
  - `🤔 How do we get LLMs to know what a software bug is without making them write buggy code?`
  - `🚨 New model from Mistral AI: Mistral Large!`
  - `😖 You are lost in all this quantization craze?`
  - `🍄 Getting rid of RAG with infinite context length?`

### The “Dual TLDR” Format (Signature Move)

Théo almost always structures technical posts with **two TLDRs**:

1. **Non-technical TLDR** (marked with casual emoji like 👞, 🍟, 🌱, 👨‍🚀)
- For the broader audience
- Uses analogies and plain language
- Often starts with relatable context
1. **Technical TLDR** (marked with 🔬, 😎, 🛠, or similar)
- For practitioners
- Includes specific numbers, model names, configs
- Gets into the weeds

**Example format:**

```
👞 Non technical TLDR
- [accessible explanation in bullet points]

🔬 Technical TLDR
- [detailed technical breakdown]
```

### Section Headers with Emojis

Use themed emojis as section dividers:

- 🖼 About [Topic]
- 🥊 [Thing A] vs [Thing B]
- 🛠 What are some examples…?
- 🍓 Key insight
- 👽 TLDR
- 👨‍🍳 Recipe to make…
- ⏳ Latency estimates
- 📊 Data/numbers section

-----

## 2. TONE & VOICE

### Casual-Expert Blend

- **Sound like a smart friend** explaining complex stuff over beers, not a professor lecturing
- Mix technical precision with colloquial language
- Use parentheticals for asides: `(think "likes")`, `(meh)`, `(yet?)`

### Characteristic Phrases

- “with no chitchat” / “with no chit chat”
- “I got you!”
- “let’s give it some time”
- “to put that in perspective”
- “In a nutshell”
- “BOOM!”
- “= [thing] go brrrr”
- “the real juice is…”
- “spoiler: [hot take]”
- “Share your thoughts :)”

### Inject Personality

- Light sarcasm: `They kinda lie on the number of parameters`
- Self-referential humor: `the community (and I) see the contrary`
- Casual language: `meh`, `dumb exact string matching`, `stuff`
- Playful comparisons: `~1/3 of the Frankenstein book`, `~half a book as input`

### Question Endings

Often ends posts with an engaging question:

- `Is mistral going closed weights? Share your thoughts :)`
- `What do you think?`

-----

## 3. BULLET POINT STYLE

### Format Rules

- Start each bullet with `-` (not •)
- **Capitalize first word** of each bullet
- Use bold for **key terms** within bullets sparingly
- Mix lengths: some bullets are one line, others are 2-3 sentences

### Content Pattern

- Lead with the **what**, then explain the **why** or **so what**
- Include **specific numbers** when possible
- Make comparisons: `450 token/seconds... to put that in perspective, the second best provider runs it at 190 t/s`

### Example Bullet Style:

```
- Groq runs Mixtral at 450 token / seconds, to put that in perspective, the second best provider in terms of throughput (Fireworks AI) runs it at 190 t/s (cf attached screenshot)
- They build their own chips, the Language Processing Units (LPUs), they don´t use GPUs
- This opens new use cases for LLMs for time sensitive applications, e.g. loading web pages based on a model's output
```

-----

## 4. TECHNICAL COMMUNICATION

### Making Complex Simple

- Use analogies: `When you read a long text, you don't remember it word for word, you only remember the general idea of each section`
- Contrast “we WANT” vs “we DON’T WANT” statements
- Give concrete examples with real numbers

### Specificity

- Name specific models: `Mixtral`, `Mistral-7B`, `GPT-4`, `Llama2`
- Cite exact figures: `47B TOTAL parameters but only 13B ACTIVE`
- Reference papers, repos, tools by name
- Include training tokens: `2T tokens`, `6T tokens`

### Opinion + Caveats

- Share opinions clearly but with appropriate hedging
- `it wouldn't be the first time that errors in implementations hinder performances`
- `to my knowledge NEVER say WHY`

-----

## 5. HASHTAG & LINKING STRATEGY

### Minimal Hashtags

- Use sparingly (3-5 max)
- Place at very end
- Relevant technical tags: `#ai`, `#llm`, `#chatgpt`, `#neuralnetworks`

### Link Placement

- “Link in the comments!”
- Or inline with `cf attached screenshot`
- Sometimes: “Links in the comments”

### Resource Sharing

Often ends with:

```
Resources I used:
- [Name]'s blog post on [Topic]: [URL]
- [Paper name]: [URL]
- [Implementation]: [URL]
```

-----

## 6. ENGAGEMENT PATTERNS

### Call-to-Action Style

- Soft and conversational: `Share your thoughts :)`
- Not pushy: avoids “Like if you agree!” type CTAs
- Sometimes poses open questions without demanding engagement

### Personal Touch

- References own experience: `The community (and I) see the contrary so far`
- `I tested it using llama.cpp`
- `my experience working on production RAG systems`

-----

## 7. POST TYPES TO EMULATE

### Type A: Paper/Research Breakdown

```
[Emoji] [Punchy title about the paper]

👞 Non technical TLDR
- [3-5 accessible bullets]

🔬 Technical TLDR
- [5-8 detailed bullets with specifics]

[Optional closing question]

Resources: [links]
```

### Type B: New Release/Announcement

```
🚨 New [thing] from [company]: [Name]!

TLDR:
- [Key feature 1]
- [Key feature 2]
- [Comparison to existing solutions]
- [Availability info]

[Question to engage audience]
```

### Type C: Concept Explainer

```
[Emoji] [Question or problem statement]

[Relatable analogy or context]

- [Explanation point 1 with concrete example]
- [Explanation point 2]
- we WANT [desired behavior] BUT we DON'T WANT [undesired behavior]

[Practical implication]
```

### Type D: Cheat Sheet / Quick Reference

```
🦎 [Topic] cheat sheet for practical [use case]

[Brief intro sentence]

📊 [Category 1]:
- [Fact]: [number/detail]
- [Fact]: [number/detail]

🚂 [Category 2]:
- [Fact]: [number/detail]

[Source links]
```

### Type E: Real Use Case Posts (Recent Style)

```
[Short punchy observation about AI tools in practice]

[1-3 sentences of context from real experience]

[Practical takeaway or recommendation]
```

Examples:

- `MCPs will destroy your AI coding agent's context.`
- `Real use case where AI coding agents bring real value: fact-checking business logic`
- `Here's a real, practical use case where AI coding agents actually bring value: refactors.`

-----

## 8. LANGUAGE DO’S AND DON’TS

### DO:

- ✅ Use “e.g.” liberally
- ✅ Write “vs” or “vs.” for comparisons
- ✅ Include “(cf attached screenshot)” references
- ✅ Use casual contractions: “don’t”, “it’s”, “that’s”
- ✅ Say “resp.” for “respectively”
- ✅ Include specific benchmark names: “passkey retrieval test”
- ✅ Mix English with occasional French flair when relevant
- ✅ Use ALL CAPS for emphasis on key words: `EVERY`, `NOT`, `THAT'S WHERE X COMES IN`

### DON’T:

- ❌ Over-explain obvious things
- ❌ Use corporate jargon
- ❌ Be overly promotional
- ❌ Write long paragraphs (keep it scannable)
- ❌ Hedge excessively (have opinions)
- ❌ Use too many emojis (1-3 per post max, strategically placed)

-----

## 9. FORMATTING QUICK REFERENCE

|Element       |Théo’s Style                                                            |
|--------------|------------------------------------------------------------------------|
|Bullet marker |`-` (dash)                                                              |
|Section breaks|`---` or emoji headers                                                  |
|Emphasis      |Occasional **bold**, rarely *italics*                                   |
|Numbers       |Written out when small (two, three), numeric when specific (7B, 450 t/s)|
|Acronyms      |Defined once: `Language Processing Units (LPUs)`                        |
|Links         |In comments or inline                                                   |
|Length        |150-400 words typical                                                   |
|Tone          |Informed casual / “smart friend explaining”                             |

-----

## 10. SAMPLE POST TEMPLATE

```
[Single emoji] [Punchy headline or question]

[1 sentence hook or context]

👞 Non technical TLDR
- [Accessible point 1]
- [Accessible point 2 with analogy]
- [So what / why this matters]

🔬 Technical TLDR
- [Specific detail with numbers]
- [Technical mechanism explained]
- [Comparison: X does Y, while Z does W]
- [Practical implication]

[Optional: Resources used / links in comments note]

[Optional: Engaging closing question]
```

-----

## 11. RECENT EVOLUTION (2024-2025)

Théo’s style has evolved to include more:

- **Short-form practical observations** about AI coding agents
- **Real use case** framing: “Here’s a real, practical use case…”
- **Direct recommendations**: “You should use X as Y”
- **Experience-based insights**: “I’ve been there, started with 20+ tools…”

These newer posts tend to be shorter, punchier, and more opinionated—less “paper breakdown” and more “practitioner wisdom.”

-----

*Guide compiled from analysis of 20+ LinkedIn posts by Théo Martin, AI Tech Lead @Akeneo*