# Chain of Thought

## 背景

CoT（思维链）是帮助大型语言模型解决如数学问题和关系推理问题等复杂问题的有效方式，在OpenCompass中，我们支持多种类型的CoT方法。

## 1. 零样本思维链

可以通过在数据集配置中简单地添加 “Let's think step by step"，来更改数据集配置的 PromptTemplate，从而实现 零样本 CoT prompt 以进行评估：

```python
qa_infer_cfg = dict(
    prompt_template=dict(
        type=PromptTemplate,
        template="Answer the question:\nQ: {question}?\nLet's think step by step:\n"
    ),
    retriever=dict(type=ZeroRetriever)
)
```

## 2. 小样本思维链

小样本思维链可以使大型语言模型更容易跟随预设的指示并得到更好的答案。对于小样本思维链，按照以下配置将思维链模板添加到 `PromptTemplate` 中，可以创建一个 one-shot prompt：

```python
qa_infer_cfg = dict(
    prompt_template=dict(
        type=PromptTemplate,
        template=
'''Question: Mark's basketball team scores 25 2 pointers, 8 3 pointers and 10 free throws.  Their opponents score double the 2 pointers but half the 3 pointers and free throws.  What's the total number of points scored by both teams added together?
Let's think step by step
Answer:
Mark's team scores 25 2 pointers, meaning they scored 25*2= 50 points in 2 pointers.
His team also scores 6 3 pointers, meaning they scored 8*3= 24 points in 3 pointers
They scored 10 free throws, and free throws count as one point so they scored 10*1=10 points in free throws.
All together his team scored 50+24+10= 84 points
Mark's opponents scored double his team's number of 2 pointers, meaning they scored 50*2=100 points in 2 pointers.
His opponents scored half his team's number of 3 pointers, meaning they scored 24/2= 12 points in 3 pointers.
They also scored half Mark's team's points in free throws, meaning they scored 10/2=5 points in free throws.
All together Mark's opponents scored 100+12+5=117 points
The total score for the game is both team's scores added together, so it is 84+117=201 points
The answer is 201

Question: {question}\nLet's think step by step:\n{answer}
'''),
    retriever=dict(type=ZeroRetriever)
)
```

## 3. Self-Consistency

SC (Self-Consistency) 方法是在 [此文章](https://arxiv.org/abs/2203.11171) 中提出的，该方法会为问题生成多个不同的推理路径，并对生成的答案进行众数投票。这种方法在复杂推理任务中表现出了显著的能力，但由于需要推理多次来采样多条推理链，所以可能会消耗很多的时间和资源。在 OpenCompass 中，您可以在数据集配置中简单地设置 SC 方法，例如：

```python
gsm8k_infer_cfg = dict(
    inferencer=dict(
        type=SCInferencer,
        generation_kwargs=dict(do_sample=True, temperature=0.7, top_k=40),  # 设置采样参数以确保模型生成不同的输出
        infer_type='SC',
        sc_size = SAMPLE_SIZE
    )
)
gsm8k_eval_cfg = dict(sc_size=SAMPLE_SIZE)
```

```{note}
注意，OpenCompass 默认使用默认使用 argmax 的方式采样下一个 token，因此若不指定采样参数，模型每次的推理结果将会是完全一致的，多轮评测将会失效。
```

其中 `SAMPLE_SIZE` 是推理路径的数量，较高的值通常会带来更高的性能。文章中展示了不同推理任务间推理路径数量与性能之间的关系：
![image](https://github.com/InternLM/opencompass/assets/28834990/05c7d850-7076-43ca-b165-e6251f9b3001)
从图中可以看出，在不同的推理任务中，随着推理路径数量的增加，性能呈现出增长的趋势。但是，对于某些任务，增加推理路径的数量可能达到一个极限，进一步增加推理路径的数量可能不会带来更多的性能提升。因此，需要在具体任务中进行实验和调整，找到最适合任务的推理路径数量。