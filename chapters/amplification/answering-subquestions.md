# Answering subquestions



Now we want to use the subquestions recipe to help a question-answerer like the one we built early on in this tutorial. We can start with the question-answerer we built earlier and modify it as follows:

1. Add a call to the subquestions recipe to generate subquestions
2. Use `map_async` to answer all the subquestions in parallel
3. Provide answers from subquestions as advice in the overall question-answering prompt

Let's start with (1) and (2), reusing the subquestions subrecipe:

```python
from ice.recipe import Recipe
from ice.utils import map_async
from subquestions import Subquestions


def make_qa_prompt(question: str) -> str:
    return f"""Answer the following question:

Question: "{question}"
Answer: "
""".strip()


class AmplifiedQA(Recipe):

    async def answer(self, question: str) -> str:
        prompt = make_qa_prompt(question)
        answer = (await self.agent().answer(prompt=prompt, max_tokens=100)).strip('" ')
        return answer

    async def run(self, question: str = "What is the effect of creatine on cognition?"):
         subquestions = await Subquestions().run(question=question)
         subanswers = await map_async(subquestions, self.answer)
         return list(zip(subquestions, subanswers))
```

If we run this, we get back a list of subquestions and their answers:

{% code overflow="wrap" %}
```python
[
    (
        'What is creatine?',
        'Creatine is a nitrogenous organic acid that helps supply energy to cells, primarily in the muscles.'
    ),
    (
        'What is cognition?',
        'Cognition is the mental action or process of acquiring knowledge and understanding through thought, experience, and the senses.'
    ),
    (
        'How does creatine affect cognition?',
        'Creatine is a dietary supplement that is often used by athletes to improve their performance. Some research has suggested that it may also improve cognitive function, but the evidence is mixed. Some studies have found that creatine can improve memory and reaction time, while others have found no significant effects.'
    ),
    (
        'What are the benefits of creatine on cognition?',
        'Creatine has been shown to improve cognitive function in people with certain medical conditions, such as Parkinson’s disease and Alzheimer’s disease. It has also been shown to improve cognitive function in healthy adults.'
    ),
    (
        'What are the side effects of creatine on cognition?',
        'There is no definitive answer to this question as the research on the topic is inconclusive. Some studies suggest that creatine may improve cognitive function, while other studies have found no significant effects. More research is needed to determine the potential cognitive effects of creatine.'
    )
]
```
{% endcode %}