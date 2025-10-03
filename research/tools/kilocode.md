# My experiences on using tool

## Step to be friend with it

### Config rules

### Config workflows

### Setting Providers

1. Add new profile with your name.
2. Select OpenRouter as API Provider.
3. Go to [OpenRounter](https://github.com/ZigLost/journal-to-ai.git), signin and creat API key (for free - you can set the usage limit) **Keep your key secretely**.
4. Paste your key in kilo code's OpenRounter API Key.
5. Choose free AI model (As of 2025-10-03, I think `x-ai/grok-4-fast:free` is good enough for help you coding, documenting, and reviewing).
6. In Advanced settings, don't forget to Enable todo list tool. This will help you see the big picture step-by-step before kilo code will take action.

### Index your codebase

1. Download and install [Ollama](https://ollama.com/download).
2. Run command `ollama pull bge-m3:latest` for downloading embedding model used for indexing your codebase files.
3. Download and install [Docker Desktop](https://www.docker.com/get-started/).
4. Run command `docker run -d --name qdrant -p 6333:6333 qdrant/qdrant` to start vector database for storing embedding index of you codebase.
5. Goto Codebase Indexing in your kilo code and configure:

```text
Embedder Provider: Ollama
Ollama Base URL: http://localhost:11434     // default
Model: bge-m3:latest                        // your model choice from step 2
Model Dimension: 1024                       // token that the model can support, for bge-m3, it support upto 8192 tokens
Qdrant URL: http://localhost:6333           // default
Qdrant API Key:                             // leave blank due to running locally in docker
```

6. Hit the button: Start Indexing, kilo code will make your entire codebase as embedding indexed and store them to Qdrant (vector database).
