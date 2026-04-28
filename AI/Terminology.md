# Terminology

## Context Window: 
LLMs don't retain any context between prompts and completions. It's necessary to send some portion of the chat history to the LLM when sending the latest prompt to be processed to give it context. This mechanism is often referred to as a context window, 
This extra text increases the compute cost, or tokens, required to process the request. Additionally, as the amount of text increases, so too does the latency for generating the completion.
