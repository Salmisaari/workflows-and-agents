---
name: voice-dna
description: Clone a user's communication style by analyzing their real writing across channels (Slack, email, WhatsApp, texts, long-form). Use this skill whenever the user wants to draft messages, emails, Slack messages, or any written communication in their own voice. Also triggers when the user asks to "write like me," "match my style," "clone my voice," asks for voice/style analysis, or shares writing samples for analysis. Use for any message composition task where the user's authentic voice matters.
---

# Voice DNA

A system for cloning someone's real communication style from their actual writing. Not a style guide with rules someone wrote down. A learned model built from what they actually do.

## How It Works

The user shares real writing samples (Slack messages, emails, texts, investor updates, customer replies, personal messages) and Claude analyzes them to build a voice profile. Claude then writes in that voice, including the quirks.

## Step 1: Collect Samples

Ask the user to share writing samples. The more channels, the better. Each channel reveals different layers of the same voice.

Prompt the user like this:

> To clone your voice I need real samples of your writing. Share whatever you've got: copy-paste Slack threads, emails, WhatsApp messages, investor updates, customer service replies, personal texts, LinkedIn posts, long-form writing. Mix of contexts is ideal. Even 5-10 messages per channel gives me something to work with. More is better.

If the user has connected tools (Gmail, Slack, Google Drive), offer to pull samples directly:

> I can also search your Gmail, Slack, or Drive for recent messages if you'd rather not copy-paste. Want me to pull some?

For Gmail: search recent sent messages across different recipients (investors, team, customers, personal).
For Google Drive: search for docs, investor updates, or memos the user authored.

**Critical: preserve everything exactly as written.** Typos, punctuation shortcuts, incomplete sentences, emoji usage, capitalization quirks. These are signal, not noise.

## Step 2: Analyze the Voice

Once you have samples, analyze across all 18 dimensions below. Don't list every dimension mechanically. Write a natural voice profile that covers what's distinctive. Focus on what makes this person's writing *theirs*, not what's generic.

### Surface Dimensions (how they write)

**1. Structural patterns**
Sentence length distribution. Paragraph length. How they open and close messages. Ratio of short punchy lines vs. longer explanatory ones. Fragments vs. complete sentences. Transition habits. Comma splicing. Dash usage (or avoidance). Parenthetical asides.

**2. Lexical fingerprint**
Vocabulary range and register (formal, casual, mixed, shifting). Pet words and phrases they default to. Filler patterns ("like," "basically," "I mean," "kinda"). Domain jargon bleed (startup language in personal messages, meditation language in business contexts). Words they consistently avoid.

**3. Typo and error signature**
Which typos they make (swapped letters, missing letters, autocorrect artifacts, consistently misspelled words). Capitalization habits. Punctuation shortcuts (dropping periods, commas instead of periods). Whether they correct themselves or let it ride. This dimension gets reproduced in output when writing casual messages. Clean it up for formal contexts.

**4. Rhythm and musicality**
Cadence patterns. Staccato vs. flowing. How they use repetition for emphasis. Sentence-ending energy (trail off or land hard). Overall tempo of a message.

### Interpersonal Dimensions (how they relate)

**5. Pragmatic style**
How directly they make requests ("would you mind" vs. "can you" vs. just stating it). How they soften (or don't). How they disagree. How they deliver bad news. How they give praise. Hedging patterns. Confidence signaling.

**6. Emotional texture**
Warmth markers (greetings, sign-offs, use of names). Humor style (dry, self-deprecating, absurdist, specificity-based). How they express frustration. Enthusiasm markers. Empathy language. Vulnerability willingness.

**7. Attachment and relational style**
How they build trust in text. How much they self-disclose. How they handle conflict and repair. Distance vs. closeness calibration. How they respond to vulnerability from others.

**8. Meta-communication habits**
How they frame conversations ("quick thing," "thinking out loud," "fyi"). How they signal importance. How they manage expectations. How they follow up.

**9. Audience adaptation**
How voice shifts between contexts: investor emails vs. team Slack vs. friends vs. suppliers vs. customer service. The delta between registers is itself a signature. Map which dimensions shift and which stay constant.

### Cognitive Dimensions (how they think)

**10. Cognitive patterns**
Do they think in analogies, frameworks, lists, narratives? How they structure arguments. Whether they lead with the conclusion or build to it. How they handle ambiguity (resolve it or sit with it). Context-switching speed between topics.

**11. Cognitive style**
Field-dependent vs. independent thinking. Tolerance for ambiguity. Abstraction level preference (jump to principles or stay concrete). Speed of closure (decide fast or keep options open). Pattern-matching tendencies. How they handle cognitive load (simplify vs. embrace complexity).

**12. Epistemic style**
How they handle not knowing something. Do they speculate freely, hedge carefully, defer to authority, go empirical? How they update beliefs when challenged. Certainty calibration. How they signal the difference between "I know" and "I think."

### Depth Dimensions (who they are)

**13. Attribution patterns**
How they explain success and failure. Internal vs. external locus. Do they credit the team, the system, luck, effort? This shows up in language constantly: "we nailed it" vs. "it worked out" vs. "I pushed hard on this."

**14. Motivational signature**
Achievement vs. affiliation vs. autonomy orientation. What energizes them in language (solving, connecting, building, understanding). Approach vs. avoidance framing: do they move toward goals or away from problems? "Let's capture this opportunity" vs. "let's make sure we don't miss this."

**15. Regulatory style**
How they manage their own emotions in writing. Do they process out loud or arrive pre-processed? How they handle uncertainty. Stress markers. Self-regulation strategies visible in text (reframing, humor, distancing). What their writing looks like when they're tired vs. energized.

**16. Identity narratives**
Recurring stories they tell about themselves. Metaphors they use for life/work. How they position themselves relative to others (underdog, builder, outsider, leader). Consistency of self-concept across contexts.

**17. Defense mechanisms in language**
Intellectualization, humor as deflection, rationalization patterns, minimizing, compartmentalizing. Not pathologizing: these are deeply characteristic and often invisible to the person themselves. Note them neutrally.

**18. Values leaking through language**
What they emphasize repeatedly. What they assume doesn't need stating. What they challenge vs. defer to. How they talk about time, money, people, effort. These reveal priority hierarchies that should bleed through in generated output.

## Step 3: Build the Voice Profile

After analysis, produce a **Voice Profile** document. Structure it as:

1. **Voice summary** (2-3 sentences capturing the essence)
2. **Constants** (what stays the same across all contexts)
3. **Adaptations** (how the voice shifts by audience/channel, mapped as deltas from baseline)
4. **Signature patterns** (the 5-10 most distinctive, reproducible quirks)
5. **Typo/error profile** (specific patterns to reproduce in casual contexts)
6. **Calibration table** (see below)
7. **Growth edges** (see below)

### Calibration Table

For every pattern identified, classify it into one of three columns:

| Pattern | Classification | Reasoning |
|---|---|---|
| *e.g. dropping articles in Slack* | **Keep as-is** | Signals speed and authenticity |
| *e.g. burying the ask in investor emails* | **Soften** | Reduce frequency, don't eliminate |
| *e.g. clear analogies when explaining technical concepts* | **Amplify** | Correlates with highest-quality communication |

**Keep as-is:** The pattern is characteristic and either neutral or positively received. Reproducing it makes output feel authentic. Some "imperfect" patterns belong here: typos that signal urgency, bluntness that reads as confidence, humor-as-deflection that's actually charming.

**Amplify:** The pattern shows up in the user's best communication: when they're clear, warm, decisive, creative. Weight these higher in generated output. These are the user's voice at its sharpest.

**Soften:** The pattern shows up under stress, fatigue, or habit, and the user would probably prefer less of it. Don't eliminate (that would break the voice). Reduce frequency. Examples: over-hedging, terseness that reads as curt, burying key information.

The key insight: some "negative" patterns are actually part of the charm. Stripping them out makes it feel less like the person. The framework needs nuance, not just positive/negative.

### Best-Self Model

The calibration table builds toward a "best self" model. Identify which samples represent the user's highest-quality communication (clear, warm, decisive, creative, whatever their specific version of "best" looks like). Weight those patterns higher. Then flag patterns that correlate with stress/fatigue as things the model acknowledges but doesn't amplify.

The result: output that sounds like them on a good day, not a sanitized version of them.

### Growth Edges

While analyzing the voice, identify patterns the user likely wants to improve. Don't assume. Frame as observations:

> "You tend to soften requests heavily in team messages ('if you get a chance,' 'no rush,' 'whenever works') even when the task is urgent. This might be intentional warmth or it might be diluting urgency. Worth flagging."

> "Your investor emails front-load context and bury the ask. The ask usually lands in the last 2 sentences. Some investors prefer the ask up top."

> "You drop articles ('need update on this' vs. 'need an update on this') in Slack but not email. Consistent pattern, probably speed-typing."

> "When delivering bad news you tend to intellectualize first and empathize second. The information is always accurate but the emotional landing could be warmer."

These observations help the user see their own patterns. They choose what to keep and what to sand down.

## Step 4: Generate in Voice

When writing a message for the user, follow this process:

1. **Identify the channel and audience.** Slack to a teammate? Email to an investor? Text to a friend? This selects the right register from the voice profile.
2. **Apply constants.** The patterns that never change regardless of context.
3. **Apply adaptations.** Shift register based on audience.
4. **Include signature patterns.** The quirks that make it sound like them.
5. **Apply calibration.** Amplify the "amplify" patterns. Keep the "keep" patterns. Reduce (don't eliminate) the "soften" patterns.
6. **Calibrate typo/error level.** Casual Slack: include their natural typo patterns. Investor email: clean it up. Customer service: somewhere in between. When in doubt, ask.

### Channel Defaults

- **Slack (team):** Casual register. Include typo patterns, abbreviations, emoji habits. Fragments OK. Short.
- **Slack (cross-functional):** Slightly more structured. Still casual but clearer.
- **Email (investor):** Formal register. Clean copy. But keep the voice (sentence rhythm, cognitive patterns, how they frame things). Don't make it sound like someone else wrote it.
- **Email (supplier/partner):** Professional but direct. Match their natural level of warmth or bluntness.
- **Email (customer service):** Match their CS tone. Some people are warm, some are efficient, some are both.
- **Text/WhatsApp:** Most casual. Closest to raw voice. Typos, shortcuts, lowercase, all of it.
- **Long-form (blog, update, memo):** Full voice with structure. This is where cognitive patterns, depth dimensions, and rhythm matter most.

## Updating the Profile

The voice profile should evolve. When the user shares new samples or corrects a generated message, note what changed. Over time the profile gets sharper.

If the user edits a draft you generated, pay attention to *what they changed*. That's direct signal about where the profile is off. Especially note:
- Words they swapped (lexical preference signal)
- Sentences they restructured (rhythm signal)
- Things they deleted (over-generation signal)
- Things they added (missing pattern signal)
- Softening or sharpening edits (calibration signal)

## When No Profile Exists Yet

If the user asks you to write something but hasn't shared samples yet, say so:

> I don't have samples of your writing yet, so I can't clone your voice. Want to share some messages so I can build a profile? Even 5-10 across a couple channels gives me a starting point.

Don't guess. Don't apply a generic "casual" or "professional" tone and call it theirs. The whole point is specificity.
