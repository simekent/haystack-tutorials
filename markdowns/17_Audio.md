---
layout: tutorial
colab: https://colab.research.google.com/github/deepset-ai/haystack-tutorials/blob/main/tutorials/17_Audio.ipynb
toc: True
title: "Make Your QA Pipelines Talk!"
last_updated: 2023-02-03
level: "intermediate"
weight: 90
description: Convert text Answers into speech.
category: "QA"
aliases: ['/tutorials/audio']
download: "/downloads/17_Audio.ipynb"
---
    


<img style="float: right;" src="https://upload.wikimedia.org/wikipedia/en/d/d8/Game_of_Thrones_title_card.jpg">

Question answering works primarily on text, but Haystack provides some features for audio files that contain speech as well.

In this tutorial, we're going to see how to use `AnswerToSpeech` to convert answers into audio files.

### Prepare environment

#### Colab: Enable the GPU runtime
Make sure you enable the GPU runtime to experience decent speed in this tutorial.
**Runtime -> Change Runtime type -> Hardware accelerator -> GPU**

<img src="https://github.com/deepset-ai/haystack-tutorials/raw/main/tutorials/img/colab_gpu_runtime.jpg">

You can double check whether the GPU runtime is enabled with the following command:


```bash
%%bash

nvidia-smi
```

To start, install the latest release of Haystack with `pip`:


```bash
%%bash

pip install --upgrade pip
pip install git+https://github.com/deepset-ai/haystack.git#egg=farm-haystack[colab,audio]
```

## Logging

We configure how logging messages should be displayed and which log level should be used before importing Haystack.
Example log message:
INFO - haystack.utils.preprocessing -  Converting data/tutorial1/218_Olenna_Tyrell.txt
Default log level in basicConfig is WARNING so the explicit parameter is not necessary but can be changed easily:


```python
import logging

logging.basicConfig(format="%(levelname)s - %(name)s -  %(message)s", level=logging.WARNING)
logging.getLogger("haystack").setLevel(logging.INFO)
```

### Start an Elasticsearch server
You can start Elasticsearch on your local machine instance using Docker. If Docker is not readily available in your environment (eg., in Colab notebooks), then you can manually download and execute Elasticsearch from source.


```python
# Recommended: Start Elasticsearch using Docker via the Haystack utility function
from haystack.utils import launch_es

launch_es()
```

### Start an Elasticsearch server in Colab

If Docker is not readily available in your environment (e.g. in Colab notebooks), then you can manually download and execute Elasticsearch from source.


```bash
%%bash

wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.9.2-linux-x86_64.tar.gz -q
tar -xzf elasticsearch-7.9.2-linux-x86_64.tar.gz
chown -R daemon:daemon elasticsearch-7.9.2

```


```bash
%%bash --bg

sudo -u daemon -- elasticsearch-7.9.2/bin/elasticsearch
```

### Populate the document store with `SpeechDocuments`

First of all, we will populate the document store with a simple indexing pipeline. See [Tutorial 1](https://colab.research.google.com/github/deepset-ai/haystack/blob/main/tutorials/Tutorial1_Basic_QA_Pipeline.ipynb) for more details about these steps.

To the basic version, we can add here a DocumentToSpeech node that also generates an audio file for each of the indexed documents. This will make possible, during querying, to access the audio version of the documents the answers were extracted from without having to generate it on the fly.

**Note**: this additional step can slow down your indexing quite a lot if you are not running on GPU. Experiment with very small corpora to start.


```python
import os
import time

from haystack.document_stores import ElasticsearchDocumentStore
from haystack.utils import fetch_archive_from_http, launch_es
from pathlib import Path
from haystack import Pipeline
from haystack.nodes import FileTypeClassifier, TextConverter, PreProcessor, DocumentToSpeech


# Wait 30 seconds only to be sure Elasticsearch is ready before continuing
time.sleep(30)

# Get the host where Elasticsearch is running, default to localhost
host = os.environ.get("ELASTICSEARCH_HOST", "localhost")

document_store = ElasticsearchDocumentStore(host=host, username="", password="", index="document")

# Get the documents
documents_path = "data/tutorial17"
s3_url = "https://s3.eu-central-1.amazonaws.com/deepset.ai-farm-qa/datasets/documents/wiki_gameofthrones_txt17.zip"
fetch_archive_from_http(url=s3_url, output_dir=documents_path)

# List all the paths
file_paths = [p for p in Path(documents_path).glob("**/*")]

# NOTE: In this example we're going to use only one text file from the wiki, as the DocumentToSpeech node is quite slow
# on CPU machines. Comment out this line to use all documents from the dataset if you machine is powerful enough.
file_paths = [p for p in file_paths if "Stormborn" in p.name]

# Prepare some basic metadata for the files
files_metadata = [{"name": path.name} for path in file_paths]

# Here we create a basic indexing pipeline
indexing_pipeline = Pipeline()

# - Makes sure the file is a TXT file (FileTypeClassifier node)
classifier = FileTypeClassifier()
indexing_pipeline.add_node(classifier, name="classifier", inputs=["File"])

# - Converts a file into text and performs basic cleaning (TextConverter node)
text_converter = TextConverter(remove_numeric_tables=True)
indexing_pipeline.add_node(text_converter, name="text_converter", inputs=["classifier.output_1"])

# - Pre-processes the text by performing splits and adding metadata to the text (Preprocessor node)
preprocessor = PreProcessor(
    clean_whitespace=True,
    clean_empty_lines=True,
    split_length=100,
    split_overlap=50,
    split_respect_sentence_boundary=True,
)
indexing_pipeline.add_node(preprocessor, name="preprocessor", inputs=["text_converter"])

#
# DocumentToSpeech
#
# Here is where we convert all documents to be indexed into SpeechDocuments, that will hold not only
# the text content, but also their audio version.
#
# Note that DocumentToSpeech implements a light caching, so if a document's audio have already
# been generated in a previous pass in the same folder, it will reuse the existing file instead
# of generating it again.
doc2speech = DocumentToSpeech(
    model_name_or_path="espnet/kan-bayashi_ljspeech_vits", generated_audio_dir=Path("./generated_audio_documents")
)
indexing_pipeline.add_node(doc2speech, name="doc2speech", inputs=["preprocessor"])

# - Writes the resulting documents into the document store (ElasticsearchDocumentStore node from the previous cell)
indexing_pipeline.add_node(document_store, name="document_store", inputs=["doc2speech"])

# Then we run it with the documents and their metadata as input
output = indexing_pipeline.run(file_paths=file_paths, meta=files_metadata)
```


```python
from pprint import pprint

# You can now check the document store and verify that documents have been enriched with a path
# to the generated audio file
document = next(document_store.get_all_documents_generator())
pprint(document)

# Sample output:
#
# <Document: {
# 'content': "'Stormborn' received praise from critics, who considered Euron Greyjoy's raid on Yara's Iron Fleet,
#             the assembly of Daenerys' allies at Dragonstone, and Arya's reunion with her direwolf Nymeria as
#             highlights of the episode. In the United States, it achieved a viewership of 9.27 million in its
#             initial broadcast.",
# 'content_type': 'audio',
# 'score': None,
# 'meta': {
#       'content_audio': './generated_audio_documents/f218707624d9c4f9487f508e4603bf5b.wav',
#       '__initialised__': True,
#       'type': 'generative',
#       '_split_id': 0,
#       'audio_format': 'wav',
#       'sample_rate': 22050,
#       'name': '2_Stormborn.txt'},
#       'embedding': None,
#       'id': '2733e698301f8f94eb70430b874177fd'
# }>
```

### Querying
   
Now we will create a pipeline very similar to the basic `ExtractiveQAPipeline` of Tutorial 1,
with the addition of a node that converts our answers into audio files! Once the answer is retrieved, we can also listen to the audio version of the document where the answer came from.


```python
from pathlib import Path
from haystack import Pipeline
from haystack.nodes import BM25Retriever, FARMReader, AnswerToSpeech

retriever = BM25Retriever(document_store=document_store)
reader = FARMReader(model_name_or_path="deepset/roberta-base-squad2-distilled", use_gpu=True)
answer2speech = AnswerToSpeech(
    model_name_or_path="espnet/kan-bayashi_ljspeech_vits", generated_audio_dir=Path("./audio_answers")
)

audio_pipeline = Pipeline()
audio_pipeline.add_node(retriever, name="Retriever", inputs=["Query"])
audio_pipeline.add_node(reader, name="Reader", inputs=["Retriever"])
audio_pipeline.add_node(answer2speech, name="AnswerToSpeech", inputs=["Reader"])
```

## Ask a question!


```python
# You can configure how many candidates the Reader and Retriever shall return
# The higher top_k_retriever, the better (but also the slower) your answers.
prediction = audio_pipeline.run(
    query="Who is the father of Arya Stark?", params={"Retriever": {"top_k": 10}, "Reader": {"top_k": 5}}
)
```


```python
# Now you can either print the object directly...
from pprint import pprint

pprint(prediction)

# Sample output:
# {
#     'answers': [ <SpeechAnswer:
#                       answer_audio=PosixPath('generated_audio_answers/fc704210136643b833515ba628eb4b2a.wav'),
#                       answer="Daenerys Targaryen",
#                       context_audio=PosixPath('generated_audio_answers/8c562ebd7e7f41e1f9208384957df173.wav'),
#                       context='...'
#                       type='extractive', score=0.9919578731060028,
#                       offsets_in_document=[{'start': 608, 'end': 615}], offsets_in_context=[{'start': 72, 'end': 79}],
#                       document_id='cc75f739897ecbf8c14657b13dda890e', meta={'name': '43_Arya_Stark.txt'}}  >,
#                  <SpeechAnswer:
#                       answer_audio=PosixPath('generated_audio_answers/07d6265486b22356362387c5a098ba7d.wav'),
#                       answer="Daenerys",
#                       context_audio=PosixPath('generated_audio_answers/3f1ca228d6c4cfb633e55f89e97de7ac.wav'),
#                       context='...'
#                       type='extractive', score=0.9767240881919861,
#                       offsets_in_document=[{'start': 3687, 'end': 3801}], offsets_in_context=[{'start': 18, 'end': 132}],
#                       document_id='9acf17ec9083c4022f69eb4a37187080', meta={'name': '43_Arya_Stark.txt'}}>,
#                  ...
#                ]
#     'documents': [ <SpeechDocument:
#                        content_type='text', score=0.8034909798951382, meta={'name': '43_Arya_Stark.txt'}, embedding=None, id=d1f36ec7170e4c46cde65787fe125dfe',
#                        content_audio=PosixPath('generated_audio_documents/07d6265486b22356362387c5a098ba7d.wav'),
#                        content='The title of the episode refers to both Daenerys Targaryen, who was born during a  ...'>,
#                    <SpeechDocument:
#                        content_type='text', score=0.8002150354529785, meta={'name': '191_Gendry.txt'}, embedding=None, id='dd4e070a22896afa81748d6510006d2',
#                        content_audio=PosixPath('generated_audio_documents/07d6265486b22356362387c5a098ba7d.wav'),
#                        content='"Stormborn" received praise from critics, who considered Euron Greyjoy's raid on ...'>,
#                    ...
#                  ],
#     'no_ans_gap':  11.688868522644043,
#     'node_id': 'Reader',
#     'params': {'Reader': {'top_k': 5}, 'Retriever': {'top_k': 5}},
#     'query': 'Who was born during a storm?',
#     'root_node': 'Query'
# }
```


```python
from haystack.utils import print_answers

# ...or use a util to simplify the output
# Change `minimum` to `medium` or `all` to raise the level of detail
print_answers(prediction, details="minimum")

# Sample output:
#
# Query: Who was born during a storm
# Answers:
# [   {   'answer_audio': PosixPath('generated_audio_answers/07d6265486b22356362387c5a098ba7d.wav'),
#         'answer': 'Daenerys Targaryen',
#         'context_transcript': PosixPath('generated_audio_answers/3f1ca228d6c4cfb633e55f89e97de7ac.wav'),
#         'context': ' refers to both Daenerys Targaryen, who was born during a terrible storm, and '},
#    {   'answer_audio': PosixPath('generated_audio_answers/83c3a02141cac4caffe0718cfd6c405c.wav'),
#        'answer': 'Daenerys',
#        'context_audio': PosixPath('generated_audio_answers/8c562ebd7e7f41e1f9208384957df173.wav'),
#        'context': 'The title of the episode refers to both Daenerys Targaryen, who was born during a terrible storm'},
#    ...
```


```python
# The document the first answer was extracted from
original_document = [doc for doc in prediction["documents"] if doc.id == prediction["answers"][0].document_id][0]
pprint(original_document)

# Sample output
#
# <Document: {
#   'content': '"'''Stormborn'''" is the second episode of the seventh season of HBO's fantasy television
#               series ''Game of Thrones'', and the 62nd overall. The episode was written by Bryan Cogman,
#               and directed by Mark Mylod. The title of the episode refers to both Daenerys Targaryen,
#               who was born during a terrible storm, and Euron Greyjoy, who declares himself to be "the storm".',
#   'content_type': 'audio',
#   'score': 0.6269117688771539,
#   'embedding': None,
#   'id': '9352f650b36f93ab99684fd4746af5c1'
#   'meta': {
#       'content_audio': '/home/sara/work/haystack/generated_audio_documents/2c9223d47801b0918f2db2ad778c3d5a.wav',
#       'type': 'generative',
#       '_split_id': 19,
#       'audio_format': 'wav',
#       'sample_rate': 22050,
#       'name': '2_Stormborn.txt'}
# }>
```

### Hear them out!


```python
from IPython.display import display, Audio
import soundfile as sf
```


```python
# The first answer in isolation

print("Answer: ", prediction["answers"][0].answer)

speech, _ = sf.read(prediction["answers"][0].answer_audio)
display(Audio(speech, rate=24000))
```


```python
# The context of the first answer

print("Context: ", prediction["answers"][0].context)

speech, _ = sf.read(prediction["answers"][0].context_audio)
display(Audio(speech, rate=24000))
```


```python
# The document the first answer was extracted from

document = [doc for doc in prediction["documents"] if doc.id == prediction["answers"][0].document_id][0]

print("Document: ", document.content)

speech, _ = sf.read(document.meta["content_audio"])
display(Audio(speech, rate=24000))
```

## About us

This [Haystack](https://github.com/deepset-ai/haystack/) notebook was made with love by [deepset](https://deepset.ai/) in Berlin, Germany

We bring NLP to the industry via open source!  
Our focus: Industry specific language models & large scale QA systems.  
  
Some of our other work: 
- [German BERT](https://deepset.ai/german-bert)
- [GermanQuAD and GermanDPR](https://deepset.ai/germanquad)
- [FARM](https://github.com/deepset-ai/FARM)

Get in touch:
[Twitter](https://twitter.com/deepset_ai) | [LinkedIn](https://www.linkedin.com/company/deepset-ai/) | [Discord](https://haystack.deepset.ai/community/join) | [GitHub Discussions](https://github.com/deepset-ai/haystack/discussions) | [Website](https://deepset.ai)

By the way: [we're hiring!](https://www.deepset.ai/jobs)

