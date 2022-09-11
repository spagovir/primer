# Q\&A without context

Let's make our first recipe `qa.py` that calls out to an agent:

```python
from ice.recipe import Recipe


def make_qa_prompt(question: str) -> str:
    return f"""Answer the following question:

Question: "{question}"
Answer: "
""".strip()


class QA(Recipe):
    async def run(self, *, question: str = "What is happening on 9/9/2022?"):
        prompt = make_qa_prompt(question)
        answer = (await self.agent().answer(prompt=prompt)).strip('" ')
        return answer
```

We can run recipes in different modes, which controls what type of agent is used. Some examples:

* `machine`: Use an automated agent (usually GPT-3 if no hint is provided in the agent call). This is the default mode.
* `human`: Elicit answers from you using a command-line interface.
* `augmented`: Elicit answers from you, but providing the machine-generated answer as a default.

You specify the mode like this:

```shell
scripts/run-recipe.sh -r qa.py -t -m human
```

Try running your recipe in different modes.

{% hint style="info" %}
Because the agent's `answer` method is async, we use `await` when we call it.
{% endhint %}