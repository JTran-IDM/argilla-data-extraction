---
title: migrate_from_legacy_datasets
---
# Migrate your legacy datasets to Argilla V2

This guide will help you migrate task specific datasets to Argilla V2. These do not include the `FeedbackDataset` which is just an interim naming convention for the latest extensible dataset. Task specific datasets are datasets that are used for a specific task, such as text classification, token classification, etc. If you would like to learn about the backstory of SDK this migration, please refer to the [SDK migration blog post](https://argilla.io/blog/introducing-argilla-new-sdk/).

!!! note Legacy Datasets include: `DatasetForTextClassification`, `DatasetForTokenClassification`, and `DatasetForText2Text`.

```
`FeedbackDataset`'s do not need to be migrated as they are already in the Argilla V2 format.
```

To follow this guide, you will need to have the following prerequisites:

- An argilla 1.\* server instance running with legacy datasets.
- An argilla >=1.29 server instance running. If you don't have one, you can create one by following the <SwmLink doc-title="installation" repo-id="Z2l0aHViJTNBJTNBZXh0cmFsaXQlM0ElM0FleHRyYWxpdA==" repo-name="extralit" path="/.swm/installation.kjpwua04.sw.md">[installation](https://app.swimm.io/repos/Z2l0aHViJTNBJTNBZXh0cmFsaXQlM0ElM0FleHRyYWxpdA%3D%3D/docs/kjpwua04)</SwmLink>.
- The `argilla` sdk package installed in your environment.

If your current legacy datasets are on a server with Argilla release after 1.29, you could chose to recreate your legacy datasets as new datasets on the same server. You could then upgrade the server to Argilla 2.0 and carry on working their. Your legacy datasets will not be visible on the new server, but they will remain in storage layers if you need to access them.

## Steps

The guide will take you through three steps:

1. **Retrieve the legacy dataset** from the Argilla V1 server using the new `argilla` package.
2. **Define the new dataset** in the Argilla V2 format.
3. **Upload the dataset records** to the new Argilla V2 dataset format and attributes.

### Step 1: Retrieve the legacy dataset

Connect to the Argilla V1 server via the new `argilla` package. First, you should install an extra dependency:

```bash
pip install "argilla[legacy]"
```

Now, you can use the `v1` module to connect to the Argilla V1 server.

```python
import argilla.v1 as rg_v1

# Initialize the API with an Argilla server less than 2.0
api_url = "<your-url>"
api_key = "<your-api-key>"
rg_v1.init(api_url, api_key)
```

Next, load the dataset settings and records from the Argilla V1 server:

```python
dataset_name = "news-programmatic-labeling"
workspace = "demo"

settings_v1 = rg_v1.load_dataset_settings(dataset_name, workspace)
records_v1 = rg_v1.load(dataset_name, workspace)
hf_dataset = records_v1.to_datasets()
```

Your legacy dataset is now loaded into the `hf_dataset` object.

### Step 2: Define the new dataset

Define the new dataset in the Argilla V2 format. The new dataset format is defined in the `argilla` package. You can create a new dataset with the `Settings` and `Dataset` classes:

First, instantiate the `Argilla` class to connect to the Argilla V2 server:

```python
import argilla as rg

client = rg.Argilla()
```

Next, define the new dataset settings:

=== "For single-label classification"

````
```python
settings = rg.Settings(
    fields=[
        rg.TextField(name="text"), # (1)
    ],
    questions=[
        rg.LabelQuestion(name="label", labels=settings_v1.label_schema),
    ],
    metadata=[
        rg.TermsMetadataProperty(name="split"), # (2)
    ],
    vectors=[
        rg.VectorField(name='mini-lm-sentence-transformers', dimensions=384), # (3)
    ],
)
```

1. The default field in `DatasetForTextClassification` is `text`, but make sure you provide all fields included in `record.inputs`.

2. Make sure you provide all relevant metadata fields available in the dataset.

3. Make sure you provide all relevant vectors available in the dataset.
````

=== "For multi-label classification"

````
```python
settings = rg.Settings(
    fields=[
        rg.TextField(name="text"), # (1)
    ],
    questions=[
        rg.MultiLabelQuestion(name="labels", labels=settings_v1.label_schema),
    ],
    metadata=[
        rg.TermsMetadataProperty(name="split"), # (2)
    ],
    vectors=[
        rg.VectorField(name='mini-lm-sentence-transformers', dimensions=384), # (3)
    ],
)
```

1. The default field in `DatasetForTextClassification` is `text`, but we should provide all fields included in `record.inputs`.

2. Make sure you provide all relevant metadata fields available in the dataset.

3. Make sure you provide all relevant vectors available in the dataset.
````

=== "For token classification"

````
```python
settings = rg.Settings(
    fields=[
        rg.TextField(name="text"),
    ],
    questions=[
        rg.SpanQuestion(name="spans", labels=settings_v1.label_schema),
    ],
    metadata=[
        rg.TermsMetadataProperty(name="split"), # (1)
    ],
    vectors=[
        rg.VectorField(name='mini-lm-sentence-transformers', dimensions=384), # (2)
    ],
)
```

1. Make sure you provide all relevant metadata fields available in the dataset.

2. Make sure you provide all relevant vectors available in the dataset.
````

=== "For text generation"

````
```python
settings = rg.Settings(
    fields=[
        rg.TextField(name="text"),
    ],
    questions=[
        rg.TextQuestion(name="text_generation"),
    ],
    metadata=[
        rg.TermsMetadataProperty(name="split"), # (1)
    ],
    vectors=[
        rg.VectorField(name='mini-lm-sentence-transformers', dimensions=384), # (2)
    ],
)
```

1. We should provide all relevant metadata fields available in the dataset.

2. We should provide all relevant vectors available in the dataset.
````

Finally, create the new dataset on the Argilla V2 server:

```python
dataset = rg.Dataset(name=dataset_name, settings=settings)
dataset.create()
```

!!! note If a dataset with the same name already exists, the `create` method will raise an exception. You can check if the dataset exists and delete it before creating a new one.

````
```python
dataset = client.datasets(name=dataset_name)

if dataset is not None:
    dataset.delete()
```
````

### Step 3: Upload the dataset records

To upload the records to the new server, we will need to convert the records from the Argilla V1 format to the Argilla V2 format. The new `argilla` sdk package uses a generic `Record` class, but legacy datasets have specific record classes. We will need to convert the records to the generic `Record` class.

Here are a set of example functions to convert the records for single-label and multi-label classification. You can modify these functions to suit your dataset.

=== "For single-label classification"

````
```python
def map_to_record_for_single_label(data: dict, users_by_name: dict, current_user: rg.User) -> rg.Record:
    """ This function maps a text classification record dictionary to the new Argilla record."""
    suggestions = []
    responses = []

    if prediction := data.get("prediction"):
        label, score = prediction[0].values()
        agent = data["prediction_agent"]
        suggestions.append(
            rg.Suggestion(
                question_name="label", # (1)
                value=label,
                score=score,
                agent=agent
            )
        )

    if annotation := data.get("annotation"):
        user_id = users_by_name.get(data["annotation_agent"], current_user).id
        responses.append(
            rg.Response(
                question_name="label", # (2)
                value=annotation,
                user_id=user_id
            )
        )

    return rg.Record(
        id=data["id"],
        fields=data["inputs"],
        # The inputs field should be a dictionary with the same keys as the `fields` in the settings
        metadata=data["metadata"],
        # The metadata field should be a dictionary with the same keys as the `metadata` in the settings
        vectors=data.get("vectors") or {},
        suggestions=suggestions,
        responses=responses,
    )
```

1. Make sure the `question_name` matches the name of the question in question settings.

2. Make sure the `question_name` matches the name of the question in question settings.
````

=== "For multi-label classification"

````
```python
def map_to_record_for_multi_label(data: dict, users_by_name: dict, current_user: rg.User) -> rg.Record:
    """ This function maps a text classification record dictionary to the new Argilla record."""
    suggestions = []
    responses = []

    if prediction := data.get("prediction"):
        labels, scores = zip(*[(pred["label"], pred["score"]) for pred in prediction])
        agent = data["prediction_agent"]
        suggestions.append(
            rg.Suggestion(
                question_name="labels", # (1)
                value=labels,
                score=scores,
                agent=agent
            )
        )

    if annotation := data.get("annotation"):
        user_id = users_by_name.get(data["annotation_agent"], current_user).id
        responses.append(
            rg.Response(
                question_name="labels", # (2)
                value=annotation,
                user_id=user_id
            )
        )

    return rg.Record(
        id=data["id"],
        fields=data["inputs"],
        # The inputs field should be a dictionary with the same keys as the `fields` in the settings
        metadata=data["metadata"],
        # The metadata field should be a dictionary with the same keys as the `metadata` in the settings
        vectors=data.get("vectors") or {},
        suggestions=suggestions,
        responses=responses,
    )
```

1. Make sure the `question_name` matches the name of the question in question settings.

2. Make sure the `question_name` matches the name of the question in question settings.
````

=== "For token classification"

````
```python
def map_to_record_for_span(data: dict, users_by_name: dict, current_user: rg.User) -> rg.Record:
    """ This function maps a token classification record dictionary to the new Argilla record."""
    suggestions = []
    responses = []

    if prediction := data.get("prediction"):
        scores = [span["score"] for span in prediction]
        agent = data["prediction_agent"]
        suggestions.append(
            rg.Suggestion(
                question_name="spans", # (1)
                value=prediction,
                score=scores,
                agent=agent
            )
        )

    if annotation := data.get("annotation"):
        user_id = users_by_name.get(data["annotation_agent"], current_user).id
        responses.append(
            rg.Response(
                question_name="spans", # (2)
                value=annotation,
                user_id=user_id
            )
        )

    return rg.Record(
        id=data["id"],
        fields={"text": data["text"]},
        # The inputs field should be a dictionary with the same keys as the `fields` in the settings
        metadata=data["metadata"],
        # The metadata field should be a dictionary with the same keys as the `metadata` in the settings
        vectors=data.get("vectors") or {},
        # The vectors field should be a dictionary with the same keys as the `vectors` in the settings
        suggestions=suggestions,
        responses=responses,
    )
```

1. Make sure the `question_name` matches the name of the question in question settings.

2. Make sure the `question_name` matches the name of the question in question settings.
````

=== "For text generation"

````
```python
def map_to_record_for_text_generation(data: dict, users_by_name: dict, current_user: rg.User) -> rg.Record:
    """ This function maps a text2text record dictionary to the new Argilla record."""
    suggestions = []
    responses = []

    if prediction := data.get("prediction"):
        first = prediction[0]
        agent = data["prediction_agent"]
        suggestions.append(
            rg.Suggestion(
                question_name="text_generation", # (1)
                value=first["text"],
                score=first["score"],
                agent=agent
            )
        )

    if annotation := data.get("annotation"):
        # From data[annotation]
        user_id = users_by_name.get(data["annotation_agent"], current_user).id
        responses.append(
            rg.Response(
                question_name="text_generation", # (2)
                value=annotation,
                user_id=user_id
            )
        )

    return rg.Record(
        id=data["id"],
        fields={"text": data["text"]},
        # The inputs field should be a dictionary with the same keys as the `fields` in the settings
        metadata=data["metadata"],
        # The metadata field should be a dictionary with the same keys as the `metadata` in the settings
        vectors=data.get("vectors") or {},
        # The vectors field should be a dictionary with the same keys as the `vectors` in the settings
        suggestions=suggestions,
        responses=responses,
    )
```

1. Make sure the `question_name` matches the name of the question in question settings.

2. Make sure the `question_name` matches the name of the question in question settings.
````

The functions above depend on the `users_by_name` dictionary and the `current_user` object to assign responses to users, we need to load the existing users. You can retrieve the users from the Argilla V2 server and the current user as follows:

```python
users_by_name = {user.username: user for user in client.users}
current_user = client.me
```

Finally, upload the records to the new dataset using the `log` method and map functions.

```python
records = []

for data in hf_records:
    records.append(map_to_record_for_single_label(data, users_by_name, current_user))

# Upload the records to the new dataset
dataset.records.log(records)
```

You have now successfully migrated your legacy dataset to Argilla V2. For more guides on how to use the Argilla SDK, please refer to the [How to guides](index.md).

<SwmMeta version="3.0.0"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
