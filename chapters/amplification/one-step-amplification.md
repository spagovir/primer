---
description: Answering given subquestion answers
---

# One-step amplification

We need an equivalent of `make_qa_prompt` that optionally takes a list of subquestions and answers and provides those in the prompt. Let's introduce a type `Subs` for pairs of questions and answers and extend `make_qa_prompt` to use it if given:

```python
Question = str
Answer = str
Subs = list[tuple[Question, Answer]]


def render_background(subs: Subs) -> str:
    if not subs:
        return ""
    subs_text = "\n\n".join(f"Q: {q}\nA: {a}" for (q, a) in subs)
    return f"Here is relevant background information:\n\n{subs_text}\n\n"


def make_qa_prompt(question: str, subs: Subs) -> str:
    background_text = render_background(subs)
    return f"""{background_text}Answer the following question, using the background information above where helpful:

Question: "{question}"
Answer: "
""".strip()
```

Now we can render prompts like this:

{% code overflow="wrap" %}
```
Here is relevant background information:
Q: What is creatine?
A: Creatine is a nitrogenous organic acid that helps supply energy to cells, primarily in the muscles.

Q: What is cognition?
A: Cognition is the mental action or process of acquiring knowledge and understanding through thought, experience, and the senses.

...

Q: What are the side effects of creatine on cognition?
A: Creatine has been shown to improve cognitive function in people with certain medical conditions, but it is not known to have any side effects on cognition.

Answer the following question, using the background information above where helpful:

Question: "What is the effect of creatine on cognition?"
Answer: "
```
{% endcode %}

With this in hand, we can write the one-step amplified Q\&A recipe:

```python
class AmplifiedQA(Recipe):
    async def run(self, question: str = "What is the effect of creatine on cognition?"):
        subs = await self.get_subs(question)
        answer = await self.answer(question=question, subs=subs)
        return answer

    async def get_subs(self, question: str) -> Subs:
        subquestions = await Subquestions().run(question=question)
        subanswers = await map_async(subquestions, self.answer)
        return list(zip(subquestions, subanswers))

    async def answer(self, question: str, subs: Subs = []) -> str:
        prompt = make_qa_prompt(question, subs=subs)
        answer = (await self.agent().answer(prompt=prompt, max_tokens=100)).strip('" ')
        return answer
```

If we run it with

```shell
scripts/run-recipe.sh -r amplified_qa.py -t
```

we get:

{% code overflow="wrap" %}
```
The effect of creatine on cognition is mixed. Some studies have found that creatine can help improve memory and reaction time, while other studies have found no significant effects. It is possible that the effects of creatine on cognition may vary depending on the individual.
```
{% endcode %}

Compare with the unamplified answer:

{% code overflow="wrap" %}
```
Creatine has been shown to improve cognition in people with Alzheimer's disease and other forms of dementia.
```
{% endcode %}