# Natural Language Processing (NLP) — Summary Notes

Reference covering the NLP pipeline: text cleaning → representing text as numbers → classical vs. deep learning approaches → using pretrained models from **Hugging Face** → evaluation → saving.

---

## 1. What Is NLP (Quick Framing)

NLP is about getting models to work with human language — text or speech — for tasks like classification (spam detection, sentiment), generation (chatbots, summarization), translation, and information extraction (named entities, keywords). The core challenge: language is unstructured, so before any model can use it, text has to be converted into numbers.

---

## 2. Text Preprocessing (Cleaning)

```python
text = text.lower()
```
- Lowercasing — so "Apple" and "apple" aren't treated as different tokens.

```python
import re
text = re.sub(r'[^a-zA-Z\s]', '', text)
```
- Removes punctuation/numbers/special characters, keeping only letters and spaces.

```python
from nltk.tokenize import word_tokenize
tokens = word_tokenize(text)
```
- **Tokenization** — splits text into individual words (or sub-word units, depending on the tokenizer). The foundational step every other technique below builds on.

```python
from nltk.corpus import stopwords
tokens = [t for t in tokens if t not in stopwords.words('english')]
```
- **Stopword removal** — filters out very common, low-information words ("the", "is", "and") that usually don't help distinguish meaning.

```python
from nltk.stem import PorterStemmer
stemmer = PorterStemmer()
tokens = [stemmer.stem(t) for t in tokens]
```
- **Stemming** — crudely chops words down to a root form (e.g. "running" → "run", but sometimes "studies" → "studi"). Fast but sometimes produces non-words.

```python
from nltk.stem import WordNetLemmatizer
lemmatizer = WordNetLemmatizer()
tokens = [lemmatizer.lemmatize(t) for t in tokens]
```
- **Lemmatization** — reduces words to their proper dictionary root form using actual language rules (e.g. "better" → "good"). Slower than stemming but more accurate.

> **Note:** For modern transformer-based models (§6), most of this manual cleaning is **not needed** — their built-in tokenizers handle raw text directly, including punctuation and casing, often better than manual preprocessing would.

---

## 3. Text Representation (Turning Words Into Numbers)

### Bag of Words (BoW)
```python
from sklearn.feature_extraction.text import CountVectorizer
vectorizer = CountVectorizer()
X = vectorizer.fit_transform(corpus)
```
- Represents each document as a vector of raw word counts, ignoring grammar and word order entirely. Simple baseline, but treats "great, not bad" and "bad, not great" identically.

### TF-IDF (Term Frequency–Inverse Document Frequency)
```python
from sklearn.feature_extraction.text import TfidfVectorizer
vectorizer = TfidfVectorizer(max_features=5000)
X = vectorizer.fit_transform(corpus)
```
- Like BoW, but downweights words that appear in *many* documents (less informative) and upweights words that are frequent in one document but rare overall. A stronger classical baseline than plain BoW.

### Word Embeddings (Word2Vec / GloVe)
```python
from gensim.models import Word2Vec
model = Word2Vec(sentences=tokenized_corpus, vector_size=100, window=5, min_count=1)
vector = model.wv['king']
```
- Represents each word as a dense vector (e.g. 100 dimensions) learned so that words with similar meanings end up close together in vector space (famous example: `king - man + woman ≈ queen`). A step up from BoW/TF-IDF since it captures semantic meaning, not just word counts.

### Contextual Embeddings (Transformers)
```python
from transformers import AutoTokenizer, AutoModel
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
model = AutoModel.from_pretrained("bert-base-uncased")
```
- Unlike Word2Vec (one fixed vector per word regardless of context), transformer models generate a **different** vector for the same word depending on its surrounding sentence (e.g. "bank" in "river bank" vs. "savings bank"). This is the representation modern NLP is built on — covered in detail in §6.

---

## 4. Classical ML for NLP

Once text is vectorized (BoW/TF-IDF), it becomes standard tabular data — any classical ML model works.

```python
from sklearn.linear_model import LogisticRegression
model = LogisticRegression()
model.fit(X_train, y_train)
```
- **Logistic Regression / Naive Bayes / SVM** are common lightweight baselines for text classification (spam detection, sentiment) — see the Supervised ML notes file for details on each. Fast, interpretable, and surprisingly competitive on simpler tasks.

---

## 5. Deep Learning for NLP (Pre-Transformer Era)

```python
keras.layers.Embedding(input_dim=vocab_size, output_dim=64)
keras.layers.LSTM(64)
```
- **RNN/LSTM** — processes text word-by-word, retaining context across the sequence (see the Deep Learning notes file, §4). Was the standard approach for text before transformers took over — still useful for smaller-scale sequence tasks.

---

## 6. Transformers & Hugging Face

Transformers are the current standard for NLP — they process an entire sequence at once (not word-by-word) using an **attention mechanism** that lets the model weigh how relevant every other word is to each word it's processing. **Hugging Face** is the most widely used library for accessing pretrained transformer models without training from scratch.

```bash
pip install transformers
```

### Fastest way in: the `pipeline` API

```python
from transformers import pipeline

classifier = pipeline("sentiment-analysis")
classifier("I really loved this movie!")
# [{'label': 'POSITIVE', 'score': 0.9998}]
```
- **What it does:** `pipeline()` downloads a sensible **default pretrained model** for the given task and wraps preprocessing + inference + postprocessing into one call. The fastest way to try a task with zero setup.

Other common tasks with `pipeline`:

```python
summarizer = pipeline("summarization")
summarizer("Long article text here...", max_length=60, min_length=20)
```
- **Summarization** — condenses a long text into a shorter version.

```python
translator = pipeline("translation_en_to_fr")
translator("Hello, how are you?")
```
- **Translation** — converts text from one language to another (model name pattern depends on language pair).

```python
generator = pipeline("text-generation", model="gpt2")
generator("Once upon a time", max_length=50)
```
- **Text generation** — continues/completes a prompt.

```python
ner = pipeline("ner", grouped_entities=True)
ner("Elon Musk founded SpaceX in California.")
```
- **Named Entity Recognition (NER)** — identifies and labels entities (people, organizations, locations) in text.

```python
qa = pipeline("question-answering")
qa(question="Who founded SpaceX?", context="Elon Musk founded SpaceX in 2002.")
```
- **Question Answering** — extracts an answer span from a given context passage.

```python
zero_shot = pipeline("zero-shot-classification")
zero_shot("This is a course about the Roman Empire", candidate_labels=["history", "cooking", "sports"])
```
- **Zero-shot classification** — classifies text into custom labels **without needing training examples** for those specific labels.

### Loading a specific model by name

Every task above used a *default* model. To use a **specific** model from the [Hugging Face Hub](https://huggingface.co/models), just pass its name:

```python
classifier = pipeline("sentiment-analysis", model="distilbert-base-uncased-finetuned-sst-2-english")
```
- Browse available models at huggingface.co/models, filtered by task, language, or dataset — copy the model's name (e.g. `"username/model-name"`) and pass it directly.

### Manual access (tokenizer + model separately)

For more control than `pipeline` gives you (e.g. custom preprocessing, accessing raw model outputs):

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

model_name = "distilbert-base-uncased-finetuned-sst-2-english"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name)

inputs = tokenizer("I love this course!", return_tensors="pt", padding=True, truncation=True)
outputs = model(**inputs)
predictions = torch.softmax(outputs.logits, dim=-1)
```
- `AutoTokenizer.from_pretrained(model_name)` — downloads and loads the **exact tokenizer** that matches the model (critical — using the wrong tokenizer for a model silently breaks results).
- `AutoModelForSequenceClassification.from_pretrained(model_name)` — downloads the pretrained model weights. Swap the class for the task: `AutoModelForCausalLM` (text generation), `AutoModelForTokenClassification` (NER), `AutoModelForQuestionAnswering`, `AutoModelForSeq2SeqLM` (translation/summarization), or the generic `AutoModel` (raw embeddings only, no task head).
- `return_tensors="pt"` — returns PyTorch tensors (`"tf"` for TensorFlow instead).
- `padding=True, truncation=True` — pads shorter sequences and cuts off longer ones so all inputs are a uniform length the model can process in a batch.

### Fine-tuning a pretrained model (brief — commands only)

```python
from transformers import TrainingArguments, Trainer

training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=16,
    learning_rate=2e-5
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset
)
trainer.train()
```
- **Fine-tuning** — continues training a pretrained model on your own labeled data, adapting it to your specific task instead of training from scratch. `Trainer` handles the training loop; `TrainingArguments` configures it (epochs, batch size, learning rate — same concepts as the Deep Learning notes file, just applied to a pretrained starting point).

---

## 7. Evaluation

```python
from sklearn.metrics import accuracy_score, f1_score
```
- **Text classification** — same metrics as any classical classification task (see Supervised ML notes, §8).

```python
from evaluate import load
bleu = load("bleu")
bleu.compute(predictions=predictions, references=references)
```
- **BLEU score** — measures overlap between generated and reference text (n-grams matched); standard for translation quality.

```python
rouge = load("rouge")
rouge.compute(predictions=predictions, references=references)
```
- **ROUGE score** — similar idea, standard for summarization quality (measures recall-oriented overlap).

```python
import math
perplexity = math.exp(loss)
```
- **Perplexity** — measures how "surprised" a language model is by the actual next word; lower is better. Common metric for text generation models.

---

## 8. Save & Load a Model

```python
model.save_pretrained("./my_model")
tokenizer.save_pretrained("./my_model")

# reload later
model = AutoModelForSequenceClassification.from_pretrained("./my_model")
tokenizer = AutoTokenizer.from_pretrained("./my_model")
```
- Saves both the model weights and the matching tokenizer together locally — always save/load them as a pair.

```python
model.push_to_hub("your-username/your-model-name")
tokenizer.push_to_hub("your-username/your-model-name")
```
- Uploads your fine-tuned model directly to the Hugging Face Hub, making it loadable by anyone (including yourself, elsewhere) via `from_pretrained("your-username/your-model-name")`. Requires `huggingface-cli login` first.

---

## Summary — Key Hyperparameters (Quick Reference)

| Hyperparameter | What it controls | Default | Typical range |
|---|---|---|---|
| `max_length` (tokenizer) | Maximum number of tokens per input sequence | model-dependent (often `512`) | `128`–`512` common for classification |
| `truncation` | Whether to cut off text longer than `max_length` | `False` | `True`/`False` |
| `padding` | Whether to pad shorter sequences to a uniform length | `False` | `True`, `False`, or `'max_length'` |
| `learning_rate` (fine-tuning) | Step size for weight updates during fine-tuning | task-dependent (commonly `2e-5`) | `1e-5`–`5e-5` — much smaller than training from scratch, since the model is already pretrained |
| `num_train_epochs` | Number of passes over the fine-tuning dataset | `3` (Hugging Face `TrainingArguments` default) | `2`–`5` — fine-tuning needs far fewer epochs than training from scratch |
| `per_device_train_batch_size` | Samples per training step | `8` | `8`–`32`, limited by GPU memory |
| `window` (Word2Vec) | Number of surrounding words considered as context | `5` | `2`–`10` |
| `vector_size` (Word2Vec) | Dimensionality of each word embedding | `100` | `50`–`300` |

---

## Quick Reference — Cheat Sheet

| Task | Command |
|---|---|
| Quick sentiment/task inference | `pipeline("sentiment-analysis")(text)` |
| Load specific model | `pipeline(task, model="model-name")` |
| Load tokenizer manually | `AutoTokenizer.from_pretrained(name)` |
| Load model manually | `AutoModelForSequenceClassification.from_pretrained(name)` |
| Fine-tune | `Trainer(model, args, train_dataset, eval_dataset).train()` |
| Save locally | `model.save_pretrained(path)` |
| Push to Hub | `model.push_to_hub(name)` |
