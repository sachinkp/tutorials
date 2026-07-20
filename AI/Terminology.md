# Terminology
<img width="1762" height="796" alt="image" src="https://github.com/user-attachments/assets/f5f7b5df-ab2b-47d7-9305-b7423db9d286" />

## Why Agents
<img width="1561" height="548" alt="image" src="https://github.com/user-attachments/assets/533c3635-5589-4348-9ff1-d43d5363180a" />

## MCP
<img width="1797" height="742" alt="image" src="https://github.com/user-attachments/assets/4693b0ea-474e-4777-8b6b-078e44cd3035" />

## LLM
large language models (LLMs) - and their more compact relations, small language models (SLMs) - encapsulate the linguistic and semantic relationships between the words and phrases in a vocabulary. The model can use these relationships to reason over natural language input and generate meaningful and relevant responses

## Vector Embeddings
We need to transform the initial vector representations of the tokens into new vectors with linguistic and semantic characteristics embedded in them, based on the contexts in which they appear in the training data. Because the new vectors have semantic values embedded in them, we call them embeddings.

For example,
given the sequence "When my dog was a ...", the model will evaluate the tokens in the sequence so far, use attention to assign weights, and predict that the next most probable token is "puppy" rather than, say, "cat" or "skateboard".

https://learn.microsoft.com/en-us/training/modules/fundamentals-generative-ai/3-language-models?pivots=text


## Context Window: 
LLMs don't retain any context between prompts and completions. It's necessary to send some portion of the chat history to the LLM when sending the latest prompt to be processed to give it context. This mechanism is often referred to as a context window, 
This extra text increases the compute cost, or tokens, required to process the request. Additionally, as the amount of text increases, so too does the latency for generating the completion.

## Prompt 
Be clear and specific – prompts with explicit instructions or questions work better than vague language.
Add context - mention the topic, audience, or format you want.
Use examples, If you want a certain style, provide an example of what you mean.
Ask for structure, Like bullet points, tables, or numbered lists.

## Speech Recognition
### Pre-processing: Extract meaningful features
Raw audio samples contain too much information for efficient pattern recognition. Pre-processing transforms the waveform into a compact representation that highlights speech characteristics while discarding irrelevant details like absolute volume.

### Acoustic modeling: Recognize phonemes
Acoustic models learn the relationship between audio features and phonemes—the smallest units of sound that distinguish words. English uses about 44 phonemes; for example, the word "cat" comprises three phonemes: /k/, /æ/, and /t/.

### Prosody generation: Determine pronunciation
Prosody refers to the rhythm, stress, and intonation patterns that make speech sound natural. Prosody generation determines how to say words, not just which sounds to produce.

## Agent Orchestration Patterns
- Concurrent orchestration - Broadcast the same task to multiple agents at once and collect their results independently. Useful for parallel analysis, independent subtasks, or ensemble decision making.
- Sequential orchestration - Pass the output from one agent to the next in a fixed order. Ideal for step-by-step workflows, pipelines, and progressive refinement.
- Handoff orchestration - Dynamically transfer control between agents based on context or rules. Great for escalation, fallback, and expert routing where one agent works at a time.
- Group chat orchestration - Coordinate a shared conversation among multiple agents (and optionally a human), managed by a chat manager that chooses who speaks next. Best for brainstorming, collaborative problem solving, and building consensus.
- Magentic orchestration - A manager-driven approach that plans, delegates, and adapts across specialized agents. Suited to complex, open-ended problems where the solution path evolves.
