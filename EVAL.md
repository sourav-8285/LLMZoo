# LLM Evaluation 🧐

## Code will be released soon

## Data Format
To gain a better grasp of our evaluation framework or participate in the evaluation process, please familiarize yourself with the data format we used for evaluation.

Our evaluation data are encoded with **JSON Line** files.

### Questions

`questions/` contains 140 questions we used for evaluation in English and in Chinese. We follow [Vicuna](https://github.com/lm-sys/FastChat) to build our test sets. Specifically, we remove 10 questions from math/code category and translate the other questions into Chinese to build our test sets. 
_Apart from that, we believe a multilingual chat model should not only be proficient in multiple languages, but also have a comprehensive understanding of diverse cultural information. To that end, we have undertaken additional experimental work on a set of manually crafted questions in Chinese, related to various perspectives of Chinese culture. We will provide updates on this experiment soon._

* `question_id` (int): A unique integer for a question. Questions with different IDs is supposed to be different.
* `text` (str): The question content.
* `category` (str): The category of the question. Questions with the same category are supposed to be similar or originate from the same source.
* `lang`(str): The language of the question.
* `meta_data`(Optional): In case more information given.

### Universal Unique IDentifier (UUID) Generation
We use the `shortuuid` Python library for generating 22-digit random UUIDs, with a mix of uppercase letters, lowercase letters and digits. This is useful when running a model multiple times on the same dataset. Modify `answer_id: auto` to `answer_id: question_id` in `utils/config.yaml` to set the answer id to its corresponding question_id.

```python
import shortuuid
shortuuid.uuid() -> str
```

### Models

`utils/config.yaml` contains model information we used for generating answers.

Each entry contains the configuration for model loading, decoding hyperparameters and output path.

For example:

```yaml
chimera-chat-7b:                             
   config_dir: /dir/to/config             # Directory with model configuration information
   model_id: 'chimera-chat-7b'            # Model ID, must be the same as the key above
   min_gen_len: 100                       # Minimum length of generated texts
   max_gen_len: 150                       # Maximum length of generated texts
   top_k: 30                              # Top k filtering
   top_p: 0.85                            # Top p filtering
   temperature: 0.35                      # Sampling temperature
   repetition_penalty: 1.2                # Repetition penalty for generation
   batch_size: 8                          # Batch size
   answer_id: 'auto'                      # Answer IDs
   input_pth: 'data/eval_dataset.jsonl'   # Path to input dataset
   output_dir: 'output'                   # Directory of output files

```
We store prompts in `prompts/prompts_all.json`. The prompts within are carefully crafted for assessing overall performance on multiple perspectives including **helpfulness, relevance, accuracy, and level of detail** (from vicuna), assessing a single perspective from **relevance, diversity, and coherence**, or evaluating a specific type of questions such as **role playing**.

For example:

```json
{
    "relevance":{
		"role": "Assistant",
		"prompt": "Relevance: The response should be closely related to the question and answer the question accurately with sufficient details without repetition or redundancy. The more relevant they are, the better.\nPlease evaluate the relevance of {num_str} AI assistants in response to the user question displayed above.\nPlease first clarify how each response addresses the question and whether it is accurate respectively.\nThen, provide a comparison on relevance among Assistant 1 - Assistant {num}, and you need to clarify which one is more relevant than or equal to another. Avoid any potential bias and ensuring that the order in which the responses were presented does not affect your judgment.\nIn the last line, order the {num_str} assistants. Please output a single line ordering Assistant 1 - Assistant {num}, where '>' means 'is better than' and '=' means 'is equal to'. The order should be consistent to your comparison. If there is not comparision that one is more relevant, it is assumed they have equivalent relevance ('=').",
		"description": "Prompt for the relevance evaluation. Relevance: address the question closely, accurately, detailedly, without repetition or redundancy."
	},
    "diversity":{
        ...
    }
}
```


NB: After conducting multiple tests on the automatic evaluation provided by ChatGPT, we found that rating the order of candidates is a more robust approach compared to directly assigning scores using ChatGPT.

### Reviews
Refer to `review_output/` for reviews of different granularity where we assign the directory as follows:
```
reveiw_output/
    en/
        coherence/
            1/
                metric.json
                review_gpt35_cot.jsonl
                ordering.txt
            2/
                ...
            ...
            final_metrics.json
           
        diversity/
            ...
        immersion/
        relevance/
        general/
    zh/
    ...
    model.txt
```


The naming of the directory is based on the evaluated perspective. The `k/`-dir within them denotes the `k`-th evaluation (if needed). 
For example, we obtain the **first** evaluation of the models' outputs from **Coherence** perspective on **English** datasets using ChatGPT (**gpt-3.5-turbo**) api in `review_output/en/coherence/1/review_gpt35_cot.jsonl`. 

```json
{
  "reviewer_id": "gpt-3.5-turbo",
  "lang": "en",
  "question_id": 1,
  "answer_ids": [
    "[uuid_answer_1]",
    "[uuid_answer_2]",
    "[uuid_answer_3]",
    "[uuid_answer_4]"
  ],
  "category": "generic",
  "metadata": {
    "question": "How to improve ...",
    "answers": [
      "Answer_from_model_1",
      "Answer_from_model_2",
      "Answer_from_model_3",
      "Answer_from_model_4"
    ],
    "model_ids": [
      "gpt-3.5-turbo",
      "Phoenix-chat-7b",
      "Chimera-chat-7b"
      "Chimera-chat-13b",
    ]
  },
  "text": "Assistant 1: The response flows smoothly.......Assistant 2 > Assistant 1 = Assistant 3 > Assistant 4",
  "order": [
    2,
    1,
    2,
    4
  ]
}
```
* `reviewer_id` (str): ID of the reviewer model or api.
* `answer_ids` (list): A list of UUIDs for answers generated by candidate models.
* `question_id` (int): ID of the question.
* `category`: Category of the question.
* `model_ids` (list): A list of IDs of the candidate models from which the answers are generated.
* `text` (str): Review given by reviewer.
* `metadata` (dict)(Optional): Any metadata of the answer.

We then assign different scores based on the performance of different models on each question. The performance is quantified by the ranking of models showed in the _order_ entry above. Each ranking is assigned different scores according to the following rule:

|Ranking|Assigned score|
|:-:|:-:|
1|10|
2|7.5|
3|5|
4|2.5|

Then we compute the overall scores by averaging over total questions.


If you prefer relative orders of the candidate models, you can directly see the ranking results in `review_output/en/general/ordering.txt`. For example:
```
Model                         Ranking Score                         Rank                          
turbo                         1.0285714285714285                      1                             
Phoenix-chat-7b               1.9714285714285715                      2                             
Chimera-chat-7b               2.585714285714286                       3                             
Chimera-chat-13b              2.8714285714285714                      4  
```
where _Ranking Score_ denotes the average rank of each model and _Rank_ indicates their final ranks.



### Answers

`output/answer_{model_name}.jsonl` contains answers generated by different models. Each row contains a record of an answer with the following field:

* `answer_id` (str): A unique UUID for an answer. Answers with different IDs is supposed to be different.
* `question_id` (int): The ID of the question the answer is generated for.
* `category`: The category of the question.
* `model_id` (str): The ID of the model the answer is generated by.
* `text` (str): The answer text.
* `metadata` (dict)(Optional): Any metadata of the answer.

Example:

```json
{
 "question_id": 1,
 "text": "There are several ways you can ...",
 "answer_id": "[uuid]",
 "model_id": "Chimera-chat-7b",
 "category": "generic",
 "lang": "en",
 "metadata": {}
}
```
