# New Features in RHOAI 2.25

## Feature Store

### Description 

In the RHOAI UI there is a new section where you can view all registered "Feature Store" objects, including: features, data sources, entities, and feature services, as well as their relationships with each other

### Is it useful to anyone in the MOC?

Not sure, I don't fully understand this feature yet.

## Model Registry and Model Catalog

### Description

The Model Registry acts as a repository for users to register, version, and manage the lifecycle of their AI models.

The Model Catalog is a curated library of validated AI Models (validated by Red Hat) that users can go through and easily spin up. 

### Is it useful to anyone in the MOC?

I can see the Model Registry feature being useful for researchers that are doing iterations of fine tuning on their specific model. They can version each model iteration and store it within the internal model registry. 

The Model Catalog could be useful for research projects that are investigating different models as it will allow them to easily spin them up and evaluate if it fits their specific project needs. But I don't think this is super useful, since researchers typically have an idea of what model they will be using already. 

## LLM Compressor Library 

### Description

The LLM Compressor Library is available as a standard RHOAI image and pipeline job. 

Using this image and library you can apply a compression algorithm to your LLMs to reduce its size and optomize for inference, particularly with deployment on vLLM. 

### Is it useful to anyone in the MOC?

I see this being very useful. Applying model compression can significantly reduce hardware costs and improve inference speeds. It seems relatively easy to do, as you can run the model compression as a notebook task or a job within a pipeline. So users can integrate it into their current workflows.