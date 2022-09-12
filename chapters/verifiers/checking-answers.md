# Checking answers

Let's start with the simplest possible way of verifying an answer—just ask the model whether's it's correct. Our recipe `verify_answer.py`:

```python
from ice.recipe import Recipe


def make_verification_prompt(question: str, answer: str) -> str:
    return f"""Consider this question: "{question}"

Potential answer: "{answer}"

Q: Is the potential answer above correct? Say "A: Yes" or "A: No".
A:"""


class Verifier(Recipe):
    async def run(self, question: str, answer: str) -> float:
        prompt = make_verification_prompt(question=question, answer=answer)
        answer, answer_p, _ = await self.agent().classify(
            prompt=prompt, choices=[" Yes", " No"]
        )
        p_correct = answer_p if answer == " Yes" else 1 - answer_p
        return p_correct
```

The interesting bit here is that we don't just want a boolean Yes/No answer from the model, but that we want the probability of the "Yes" answer to the correctness question. This way, we get a more graded signal that we can use, e.g. to only show or use model responses when they exceed a threshold.

## Sanity checks

Let's test it:

{% code overflow="wrap" %}
```shell
scripts/run-recipe.sh -r verify_answer.py -t --args '{"question": "What is 2 + 2?", "answer": "4" }'
```
{% endcode %}

```
0.9948396822920341
```

Good.

{% code overflow="wrap" %}
```
scripts/run-recipe.sh -r verify_answer.py -t --args '{"question": "What is 2 + 2?", "answer": "5" }'
```
{% endcode %}

```
0.0010152581398344962
```

Basic sanity checks pass.

{% code overflow="wrap" %}
```shell
run-recipe.sh -r verify_answer.py -t --args '{"question": "What is the capital of Germany?", "answer": "Munich" }'
```
{% endcode %}

```
0.0005455832226911594
```

Also correct.&#x20;

## A math problem

Let's try something harder: A problem from the GSM8K math problems dataset:

> Beth bakes 4x 2 dozen batches of cookies in a week. If these cookies are shared amongst 16 people equally, how many cookies does each person consume?

The correct answer is 6, but it takes a few steps of reasoning to work that out.

```shell
scripts/run-recipe.sh -r verify_answer.py -t --args '{"question": "Beth bakes 4x 2 dozen batches of cookies in a week. If these cookies are shared amongst 16 people equally, how many cookies does each person consume?", "answer": "6" }'
```

```
0.06723949284762187
```

The model can't see that the answer is correct.

What if we also give the reasoning steps?

```shell
scripts/run-recipe.sh -r verify_answer.py -t --args '{"question": "Beth bakes 4x 2 dozen batches of cookies in a week. If these cookies are shared amongst 16 people equally, how many cookies does each person consume?", "answer": "Beth bakes 4x 2 dozen batches of cookies for a total of 4*2 = 8 dozen cookies. There are 12 cookies in a dozen and she makes 8 dozen cookies for a total of 12*8 = 96 cookies. She splits the 96 cookies equally amongst 16 people so they each eat 96/16 = 6 cookies. So, the final answer is 6 cookies per person." }'
```

```
0.3231381082881086
```

Now the answer is judged to be more likely to be correct, but still less than 50% correct. What if we check the answer step by step?