# LLM As A Judge Evaluation

## Background

Current knowledge evaluation in InsturctLab, MMLU-Branch, evaluates models on their answers to multiple choice questions.
In MMLU-Branch the model either gets the answer right or wrong, there's no inbetween to give the model credit on answers that are less than perfect. 

Additionally there is no way to bring a custom evaluation data set Training to do evaluation on Instructlab trained models.

The LLM as a Judge (LLMaaJ) evaluation proposed in this document gives users a way for users to bring custom evaluation questions and score the answers to those questions on a sliding scale.

## Feature Description

LLMaaJ provides a flexible way for users to bring custom evaluation questions and compare any number of different models on those questions.
The answers to each of questions the models being evaluated on is given a score from 1-5 (1 being in accurate, 5 being completely accurate) by a judge model.

Any number of different judges can be provided. A model to be evaluated or judge can either be a model served at a remote OpenAI compatible endpoint or a model hosted locally where `ilab` is running.

### Inputs

The two inputs to the LLMaaJ evaluation are the evaluation data set, and models.
Models can either be local, existing on the filesystem, or OpenAI API compatible endpoints.
API keys for endpoints, and custom prompts for each model can be specified.

The evaluation dataset consists of the question and the ground truth answer.
The data set could either be a `.csv` file, `.jsonl` file, or the `qna.yaml` file that is a leaf node in the taxonomy.

Here is an example of a 3 evaluation dataset in `.csv` format.

```csv
question,ground_truth
What is the Phoenix constellation?,Phoenix is a minor constellation in the southern sky.
Who charted the Phoenix constellation?,The Phoenix constellation was charted by french explorer and astronomer Nicolas Louis de Lacaille.
How far does the Phoenix constellation stretch?,The phoenix constellation stretches from roughly −39° to −57° declination, and from 23.5h to 2.5h of right ascension.
```

### What's going under the hood

Each of the questions from the evaluation data set is asked to a model, one model at a time. A model's chat template can be configured to be anything in order to optimize the response from the model.

For example here is the proposed default chat template for a model:

```shell
<|system|>
You are a Red Hat Instruct Model based on Granite 7B,
an AI language model developed by Red Hat and IBM Research,
based on the Granite-7b-base language model.
Your primary function is to be a chat assistant.
<|user|>
Answer the following question.
Question: {question}
Answer:
<|assistant|>
```

The answers to these questions is recorded per model and then compared with the ground truth provided in the evaluation data set. The judges chat template includes the evaluation criteria for comparing the model answer with the ground truth.

Here is the default chat template for the judge.

```shell
You are an evaluation system tasked with assessing the answer quality of a AI generated response in relation to the posed question and reference answer. Assess if the response is correct, accurate, and factual based on the reference answer. Evaluate the answer_quality as:
- Score 1: The response is completely incorrect, inaccurate, and/or not factual.
- Score 2: The response is mostly incorrect, inaccurate, and/or not factual.
- Score 3: The response is somewhat correct, accurate, and/or factual.
- Score 4: The response is mostly correct, accurate, and factual.
- Score 5: The response is completely correct, accurate, and factual.
Here is the question: \n ------- \n {question} \n -------
Here is model answer: \n ------- \n {answer} \n -------
Here is the reference answer(may be very short and lack details or indirect, long and extractive):  \n ------- \n {reference_answer} \n ------- \n
Assess the quality of model answer with respect to the Reference Answer, but do not penalize the model answer for adding details or give a direct answer to user question. Provide the quality level as a JSON object with two keys: 'reasoning' and 'answer_quality'.
```

### Outputs

Scores for the models are output in files. These files can be in `.jsonl`, `.csv`, of `.xlsx` format.
Output files will be created for each output file specified.

The files will have columns for `question`, `ground_truth`, `answer`, `answer_score`, `answer_score_reasoning`

```csv
question,ground_truth,model-1 answer,answer_score,answer_score_reasoning
What is the Phoenix constellation?,Phoenix is a minor constellation in the southern sky.,The Pheonix consetllation is a minor constellation in the southern sky.,5,The model answer is completely correct, accurate, and factual.
```

The evaluation also outputs an intermediate files that just has the responses from the model for the judge model to judge againist.

```csv
question,ground_truth,model-1 answer
What is the Phoenix constellation?,Phoenix is a minor constellation in the southern sky.,The Phoenix consetllation is a minor constellation in the southern sky.,5,The model answer is completely correct, accurate, and factual.
```

## Feature Interface

The LLMaaJ evaluation can do one of three things.
1. Ask the model(s) the questions from the evaluation data set and then score the responses with a judge model(s).
2. Ask models the questions from the evaluation data set(s) and aggregate the responses, skipping the judgement step.
3. Score file(s) with model answers and ground trith generated by #2 using the judge model(s), skipping the asking the models question.

There are two ways to configure the `ilab` CLI to run the LLMaaJ evaluation, either through flags on the CLI or in the ilab configuration file

### Configuration file

Here is an example of the configuration for the LLMaaJ evaluation, that goes in the `evaluate` section of the file.

```yaml
llm_aaj:
  judges:
    - model: www.openai.com/v1
      name: gpt4
      api_key: path_to_secret_file
      chat_template: None # if this is not specified it defaults to judge chat template shown above
  models:
    - model: https://model1.endpoint.com/v1 # defaults to a local model
      name: model1 # defaults to None
      api_key: 'abcde12345' # defaults to None
      chat_template: # defaults to model template below
    - model: /home/ali/model2_dir
      chat_template: None # if this is not specified it defaults to model chat template shown above
      name: model2
  score_output_format:
    - csv
    - jsonl
    - xlsx
  input_responses:
    - /path/to/a/model1-responses-1_1_24_1_11_43.csv
  input_qna:
    - /path/to/a/qna.yaml
    - /path/to/a/qna.csv
    - /path/to/a/qna.jsonl
    - /path/to/a/dir # this would glob all .yaml, csv, and jsonl files here
```

In the items in the `models` and `judges` sections are the same format that represents a model.
The `model` can either be an endpoint to an OpenAI API compatible model or a model on the filesystem that will be served by `ilab`.
The `name` is the optional name of the model for the output files.
The `api_key` is the optional API key to be able to inference with the OpenAI API compatible endpoint. These values are hashed to ensure that keys don't live directly in the config.
The `chat_template` is chat template for the model or the judge. This is an optional field and the defaults will be similar to those given above.

The `score_output_format` sections are a list of the file formats for the output scores.
The `input_qna` section is a list of sources of evaluation datasets. The output scores are for all of the questions in the `input_qna` section to a single file for each output format.
The `input_responses` section is a list of files with the question, ground truth, and a answer from a model. These files will be directly judged. Scores for these judgements will be given for each output format specified.

If everything is properly configured in the configuration file then the user just needs to run:

```shell
$ ilab model evaluate --benchmark llmaaj
```

### `ilab` CLI

Configuration for LLMaaJ evaluation can be configured as long as the `--benchmark` is set to `llmaaj`.

Here are the flags to configurate the evaluation after `ilab model evaluate --benchmark llmaaj`. All lists are delimited by commas.

```shell
--llmaaj-models # A list of models to be evaluated where a model can be an endpoint or the path to a model on the file system.
--llmaaj-model-api-keys # A list of hashed keys for models represented as endpoints to be evaluated
--llmaaj-model-names # A list of model names for each of the models

--llmaaj-judges # A list of models to judge where a model can be an endpoint or the path to a model on the file system.
--llmaaj-judge-api-keys # A list of hashed keys for the judge models that are the api keys for the models that are endpoints
--llmaaj-judge-names # A list of model names for each of the judge models

--llmaaj-score-output-format # A list of output formats
--llmaaj-input-qna # A list of evaluation datasets
--llmaaj-resp-path # A list of files that are responses to evaluation data sets
```

A flag for chat templates for the models to be evaluated and the judge models is not included.

### CLI examples

The following command evaluates a local model based with a local judge with the questions in qna.yaml.

```shell
$ ilab model evaluate --benchmark llmaaj --llmaaj-models ~/.cache/instructlab/models/instructlab/granite-7b-lab --llmaaj-judges ~/.cache/instructlab/models/Mixtral-8x7b --llmaaj-input-qna qna.yaml
```

The following command evaluates an endpoint and a local model with a judge model that is also an endpoint with the questions in questions.csv

```shell
$ ilab model evaluate --benchmark llmaaj \
    --llmaaj-models https://model1.endpoint.com/v1/,~/.cache/instructlab/models/instructlab/granite-7b-lab,https://model2.endpoint.com/v1/
    --llmaaj-model-api-keys abcdefg12345,defghi54321 \
    --llmaaj-model-names model1,granite-7b-lab,model2 \
    --llmaaj-judges https://judge.endpoint.com/v1 \
    --llmaaj-judge-names judge1 \
    --llmaaj-input-qna questions.csv \
```

The api keys flag is a list of api keys for an endpoint. In the previous example the first and the third model in the list of models are endpoints
The first api key in the list of api keys would be for the first model in the list, `model1`.
The second api key in the list of api keys would be for the third model in the list, `model2`.

If the user does not want to run the judgement but just get responses from the models they can exclude the flags with `judge` in them.

```shell
$ ilab model evaluate --benchmark llmaaj \
    --llmaaj-models https://model1.endpoint.com/v1/ \
    --llmaaj-model-api-keys abcdefg12345 \
    --llmaaj-model-names model1 \
    --llmaaj-input-qna questions.csv \
```

This will output a file called `model1-responses-{timestamp}.csv`.

If the user wants to just run the judgement from an intermediate response file, like the one created above, they can exclude the flags with `model` in them.

```shell
$ ilab model evaluate --benchmark llmaaj \
    --llmaaj-judges https://judge.endpoint.com/v1 \
    --llmaaj-judge-names judge1 \
    --llmaaj-resp-path model1-responses-1-1-24-12-02-03.csv
    --llmaaj-score-output-format csv,jsonl
```
