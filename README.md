# Second_Brain_AI_Assistant
- This is an end to end project where I build a second brain(a knowledge base of my personal documents) AI assistant.

Here's the system architecture for the project -
![image](https://github.com/user-attachments/assets/57cbbbf6-80b2-43c9-a9c5-773f4c6f75c3)

The following concepts, algorithms and tools will be used in this project:
- Architecting an AI system using the FTI architecture.
- Using MLOps best practices such as data registries, model registries, and experiment trackers.
- Crawling over 700 links and normalizing everything into Markdown using Crawl4AI.
- Computing quality scores using LLMs.
- Generating summarization datasets using distillation.
- Fine-tuning a Llama model using Unsloth and Comet.
- Deploying the Llama model as an inference endpoint to Hugging Face serverless Dedicated Endpoints.
- Implement advanced RAG algorithms using contextual retrieval, hybrid search and MongoDB vector search.
- Build an agent that uses multiple tools using Hugging Face’s smolagents framework.
- Using LLMOps best practices such as prompt monitoring and RAG evaluation using Opik.
- Integrate pipeline orchestration, artifact and metadata tracking using ZenML.
- Manage the Python project using uv and ruff.
- Apply software engineering best practices.

UNDERSTANDING OUR DATA - 
- 8 Notion DBs(which contain various resources on topics such as GenAI, information retrieval, MLOps and system design)
- these DBs have  “Node” type data entries that contain high-level links, such as blogs, benchmarks, or awesome lists, while a “Leaf” contains super-specific resources or tools.
- "Notes" pages have comments while "Toggle" elements have llinks to articles, tools and blogs.
- all of these pages need to be converted into markdown format(easier for llms to parse)
- some of these have links with more insightful notes which also need to be converted to Markdown and seperate the ones that only have links to non valuable documents
- this data is stored on a public S3 bucket

FLOW OF DATA
- We collect raw Notion documents in Markdown format.
- We crawl each link in the Notion documents and normalize them in Markdown.
- We store a snapshot of the data in a document database.
- For fine-tuning, we filter the documents more strictly to narrow the data to only high-quality samples.
- We use the high-quality samples to distillate a summarization instruction dataset, which we store in a data registry.
- Using the generated dataset, we fine-tune an open-source LLM, which we save in a model registry.
- In parallel, we use a different filter threshold for RAG to narrow down the documents to medium to high-quality samples (for RAG, we can work with more noise).
- We chunk, embed, plus other advanced RAG preprocessing steps to optimize the retrieval of the documents.
- We load the embedded chunks and their metadata in a vector database.
- Leveraging the vector database, we use semantic search to retrieve the top K most relevant chunks relative to a user query.

![image](https://github.com/user-attachments/assets/0e0e2845-6bb6-4e04-9ba2-742ca9458627)

FTI ARCHITECTURE
- Feature pipeline takes raw data as input and output features and labels to train our model(The feature pipeline takes in data and outputs features & labels saved to the feature store)
- The training pipeline takes the features and labels from the feature stored as input and outputs our trained model(s)(The training pipelines query the features store for features & labels and output a model to the model registry).
- The inference pipeline inputs the features and labels from the feature store and the trained model(s) from the model registry. Using these two components, we can make predictions in batch or real-time mode and serve them to the client(The inference pipeline uses the features from the feature store and the model from the model registry to make predictions).

![image](https://github.com/user-attachments/assets/ffbaa686-0375-4c3b-bf7e-44573b7ec5d9)

ARCHITECTURE OF SECOND BRAIN ASSISTANT
- The data pipelines: Collect data from Notion, clean it, standardize it and load it to a document DB(collections of key-value pairs like MongoDB)
  a: Data collection pipeline -  Uses Notion API to fetch entries, converts them to Markdown, adds metadata, and stores them as JSON in a public S3 bucket
  b: ETL pipeline -  Pulls documents from S3, extracts and crawls embedded links, standardizes content to Markdown, scores document quality with LLMs, and saves everything to MongoDB
- The feature pipelines:
  a: RAG Pipeline (Retrieval-Augmented Generation):
    *Uses distillation (LLM-generated summaries) to build a custom instruction dataset
    *Implements Contextual Retrieval for smarter results (requires summarization + hybrid search).
  b: Summarization Dataset Generation(fine tune an LLM):
    *Chunks documents → generates embeddings → stores them in MongoDB vector DB.
    *Saves it to Hugging Face Dataset Hub
- The training pipeline:
  a: Fine-tunes a Llama 3.1 8B model using the summarization dataset via Unsloth.
  b: Tracks experiments using Comet.
  c: Pushes best model to Hugging Face Model Hub.
  d: Uses ZenML to orchestrate and manage training workflows.
- The inference pipelines:
  a: Summarization Inference: Deploys the fine-tuned LLM as a real-time API using Hugging Face’s endpoints.
  b: Agentic Inference: The AI assistant that uses:
    *A retriever (talks to vector DB).
    *A summarizer (improves response).
    *Uses smolagents + Gradio UI for interaction
- The observability pipeline: Opik to monitor - Prompt traces (see every step the LLM takes), Evaluate LLMs like experiments (track accuracy, consistency, etc.)

![image](https://github.com/user-attachments/assets/1c922445-3a94-4fe7-9ece-861701d46f48)

OFFLINE VS ONLINE PIPELINE
- Offline pipeline: not user-facing and can run in the background. They prepare all the data and models ahead of time.
  *Data collection pipeline
  *ETL data pipeline
  *RAG feature pipeline
  *Dataset generation feature pipeline
  *Training pipeline
- Online pipeline: user-facing and must respond in real-time or near-real-time. Triggered when a user interacts with your system.
  *Agentic inference pipeline
  *Summarization inference pipeline
  *Observability pipeline
  
a. Offline work is heavy (embedding, training) and it is not suited for real-time. Doing it ahead of time saves time during queries.
b. Online pipelines stay fast because they use precomputed results.
c. You don’t want to re-train or re-embed every time a user asks a question.
