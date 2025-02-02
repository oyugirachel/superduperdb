# Sentiment analysis with transformers

In this notebook we implement a classic NLP use-case using Hugging Face's `transformers` library.
We show that this use-case may be implementing directly in the SuperDuperDB `Datalayer` using MongoDB as the
data-backend. 


```python
!pip install datasets
```


```python
from datasets import load_dataset, load_metric
import numpy
import pymongo
from transformers import AutoTokenizer, AutoModelForSequenceClassification

import superduperdb
from superduperdb.misc.superduper import superduper
from superduperdb.container.document import Document as D
from superduperdb.db.mongodb.query import Collection
from superduperdb.ext.transformers.model import TransformersTrainerConfiguration, Pipeline
from superduperdb.container.dataset import Dataset
```

SuperDuperDB supports MongoDB as a databackend.
Correspondingly, we'll import the python MongoDB client pymongo and "wrap" our database to convert it 
to a SuperDuper Datalayer:


```python
db = pymongo.MongoClient().documents
db = superduper(db)
collection = Collection('imdb')
```

We use the IMDB dataset for training the model:


```python
data = load_dataset("imdb")

db.execute(collection.insert_many([
    D({'_fold': 'train', **data['train'][int(i)]}) for i in numpy.random.permutation(len(data['train']))[:4]
]))

db.execute(collection.insert_many([
    D({'_fold': 'valid', **data['test'][int(i)]}) for i in numpy.random.permutation(len(data['test']))[:4]
]))
```

Check a sample from the database:


```python
r = db.execute(collection.find_one())
r
```

Create a tokenizer and use it to provide a data-collator for batching inputs:


```python
tokenizer = AutoTokenizer.from_pretrained("distilbert-base-uncased")
model = AutoModelForSequenceClassification.from_pretrained("distilbert-base-uncased", num_labels=2)
model = Pipeline(
    identifier='my-sentiment-analysis',
    task='text-classification',
    preprocess=tokenizer,
    object=model,
    preprocess_kwargs={'truncation': True},
)
```

We'll evaluate the model using a simple accuracy metric. This metric gets logged in the
model's metadata during training:


```python
training_args = TransformersTrainerConfiguration(
    identifier='sentiment-analysis',
    output_dir='sentiment-analysis',
    learning_rate=2e-5,
    per_device_train_batch_size=2,
    per_device_eval_batch_size=2,
    num_train_epochs=2,
    weight_decay=0.01,
    save_strategy="epoch",
    use_mps_device=False,
    evaluation_strategy='epoch',
    do_eval=True,
)
```

Now we're ready to train the model:


```python
from superduperdb.container.metric import Metric

model.fit(
    X='text',
    y='label',
    db=db,
    select=collection.find(),
    configuration=training_args,
    validation_sets=[
        Dataset(
            identifier='my-eval',
            select=collection.find({'_fold': 'valid'}),
        )
    ],
    data_prefetch=False,
    metrics=[Metric(
        identifier='acc',
        object=lambda x, y: sum([xx == yy for xx, yy in zip(x, y)]) / len(x)
    )]
)                                                                            
```

We can verify that the model gives us reasonable predictions:


```python
model.predict("This movie sucks!", one=True)
```
