# AntiSlop Sampler

## Overview

The AntiSlop sampler uses a backtracking mechanism to go back and retry with adjusted token probabilities when it encounters a disallowed word or phrase. No more testaments or tapestries or other gpt-slop.

Try the sampler here: 

- [Backtracking visualisation notebook](https://colab.research.google.com/drive/1tHS3tHXbZ5PWOsP-Mbk8oK35y6otBZUn?usp=sharing)
- [Generate example notebook](https://colab.research.google.com/drive/1Rd3V4AN31cDytfmY9u80rzHXPD_dS6x9?usp=sharing)


### How to use the sampler

<b>With koboldcpp (new!)</b>

Koboldcpp now includes antislop phrase banning in their latest release:
https://github.com/LostRuins/koboldcpp/releases/tag/v1.76

<b>With a local chat UI (open-webui):</b>

<details>
<summary>[Click to view instructions]</summary>

### if you want to run in an isolated environment (note: open-webui requires python 3.11):
```bash
sudo apt install python3.11 python3.11-venv
python3.11 -m venv open-webui
source open-webui/bin/activate
```

### install open-webui
```bash
pip install open-webui
open-webui serve
```

### start the openai compatible antislop server:
```bash
git clone https://github.com/sam-paech/antislop-sampler.git && cd antislop-sampler
pip install fastapi uvicorn ipywidgets IPython transformers bitsandbytes accelerate
python3 run_api.py --model unsloth/Llama-3.2-3B-Instruct --slop_adjustments_file slop_phrase_prob_adjustments.json
```

### configure open-webui
- browse to http://localhost:8080
- go to admin panel --> settings --> connections
- set the OpenAI API url to http://0.0.0.0:8000/v1
- set api key to anything (it's not used)
- click save (!!)
- click the refresh icon to verify the connection; should see a success message

Now it should be all configured! Start a new chat, select the model, and give it a try.

Note: Only some of the settings in the chat controls will be active. Those are:
- stream chat response
- temperature
- top k
- top p
- min p
- max tokens

![image](https://github.com/user-attachments/assets/8bcc2906-b1e1-4be0-b01b-66cf1b18a9ad)

</details>

Here it is in action (in slow mode so you can see its backtracking & revisions):

https://github.com/user-attachments/assets/e289fb89-d40c-458b-9d98-118c06b28fba

### Disclaimers:

It's recommended that you roll your own slop list! The slop_phrase_prob_adjustments.json file is mostly auto-generated by computing over-represented words in a large LLM-generated story dataset. As such, it's not well optimised or curated. Curated lists will be added in the future.

The api is very minimalist, and doesn't support concurrency. It's more geared for local use, testing & dataset generation than for production.

This is research grade code. Meaning it probably contains bugs & is a work in progress.

### 2024-10-05 Update

Refactored the code, lots of fixes.

- Added an OpenAI compatible API server.
- Now using model.generate with stopping conditions, to generate for multiple tokens instead of just 1 at a time. This is much faster.
- Added a basic JSON validator + enforcement to demonstrate how the sampler can enforce long-range constraints.
- Switch to probs from logits for the cached values, so that down/upregulation works as expected (this was a mistake in the previous implementation).
- Refactored the code for better organisation.

Quick blurb on the JSON validator:

It uses the same backtracking mechanism to retry invalid JSON output. It checks for unintended unescaped quotes in strings, and encourages the model to choose a valid continuation. This is a very common fail mode for JSON outputs. Other kinds of per-token JSON grammars will just terminate the string if they see an unescaped quote, sadly ending the profound thought the LLM was in the middle of expressing. This is better. You can also use it with high temps.

<details>
<summary>### 2024-10-01 Update</summary>

- Squashed vram leaks, fixed bugs. It should work with any transformers model now.
- Support min_p
- Now using slop_phrase_prob_adjustments.json by default, which has a more intuitive probability adjustment per slop phrase (1 == no change; < 1 means probability is reduced by that factor). It looks like this:
```
[
    ["kaleidoscope", 0.5],
    ["symphony", 0.5],
    ["testament to", 0.5],
    ["elara", 0.5],
    ...
]
```
</details>

### chat_antislop
```python
# Chat generation with streaming
messages = [
    {"role": "user", "content": prompt}
]
for token in chat_antislop(
    model=model,
    tokenizer=tokenizer,
    messages=messages,
    max_new_tokens=400,
    temperature=1,
    min_p=0.1,
    # The adjustment_strength param scales how strongly the probability adjustments are applied.
    # A value of 1 means the values in slop_phrase_prob_adjustments (or the defaults) are used unmodified.
    # Reasonable values are 0 (disabled) thru 100+ (effectively banning the list).
    adjustment_strength=100.0,
    # Optional: Provide a list of slop phrases and probability adjustments
    slop_phrase_prob_adjustments=slop_phrase_prob_adjustments,
    enforce_json=False,
    antislop_enabled=True,
    streaming=True
):
    print(tokenizer.decode(token), end='', flush=True)
```

### generate_antislop
```python
# generate without streaming
prompt_with_template = tokenizer.apply_chat_template(messages, tokenize=False)
generated_text = generate_antislop(
    model=model,
    tokenizer=tokenizer,
    prompt=prompt,
    max_length=300,
    temperature=1,
    min_p=0.1,
    adjustment_strength=100.0,
    slop_phrase_prob_adjustments=slop_phrase_prob_adjustments,
    enforce_json=False,
    antislop_enabled=True,
    streaming=False
)        
print(tokenizer.decode(generated_text))
```

## What this does:

You can give it a list of words & phrases to avoid like "a tapestry of", "a testament to", etc., and it will backtrack and try something else if it hits that phrase. It can handle 1000s of slop phrases since the lookups are fast. The phrases and downregulation amounts are user configurable. Previous approaches have done this with per-token logit biasing; but that's quite ineffective since most slop words & phrases are more than one token, and it impairs output quality if we downregulate all those partial-word tokens. So instead, we wait for the whole phrase to appear in the output, then backtrack and downregulate all the tokens that could have produced the slop phrase, and continue from there.

For the default slop list, we computed a large list of words that are over-represented in LLM output compared to normal human writing. This list is supplemented by a list of over-used phrases that are pet peeves of LLM enthusiasts. During generation, if any of the words & phrases in this list are generated, the sampler reduces the probability of the starting tokens that can lead to that phrase, by the factor specified in the config. This way you can lightly de-slop or strongly de-slop. You can of course also specify your own phrase list & weights.

## Why it's interesting:

Samplers typically work at the token level -- but that doesn't work if want to avoid words/phrases that tokenise to >1 tokens. Elara might tokenise to ["El", "ara"], and we don't want to reduce the probs of everything beginning with "El". So, this approach waits for the whole phrase to appear, then backtracks and reduces the probabilities of all the likely tokens that will lead to that phrase being output. ~~Nobody afaik has tried this before.~~ [edit] It turns out exllamav2 has a banned_strings feature with same/similar implementation so we can't claim novelty. [/edit]

* Disclaimers: This is only implemented in Transformers (and now koboldcpp!) thus far. It is not well optimised. Expect research grade code & possibly bugs.


## What you need to implement this

If you'd like to implement this sampler in something other than transformers, here's what you need:

- A loop to manage the state of the sampler, as it backtracks and needs to refer to past logits that it's cached
- Per-token continuation generation (think: completions, not chat.completions)
- Raw logits
- Ability to bias logits when generating

Unfortunately that rules out most commercial APIs since few let you specify logit biases. For inferencing engines, they will likely be a mixed bag in terms of ease of integration, as most/all samplers work per token without this weird backtracking stuff we're doing here.

If you do implement this sampler in your thing, please let me know about it!

## Acknowledgements

Turboderp was the first to implement this mechanism in exllamav2 as the "banned strings" feature. This was unknown to us at the time of creating the AntiSlop sampler, which was birthed independently in a case of convergent evolution. Credit to them for doing it first!

## How to Cite

A paper is in the works, hang tight.

```
@misc{paech2024antislop,
      title={antislop-sampler},
      author={Samuel J. Paech},
      year={2024},
      howpublished={\url{https://github.com/sam-paech/antislop-sampler}},
      note={GitHub repository}
}
```
