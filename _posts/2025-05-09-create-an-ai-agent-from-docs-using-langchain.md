---
layout: post
title: Creating your own AI agent/bot using a local folder as input
date: 2025-05-09
tags: [ AI, Java ]
---

This is my first time posting about artificial intelligence, even though I've been digging into this topic a lot lately. 
My idea is to build an AI agent / bot that uses a set of folders with documents, and then be able to reply questions using these documents as root of information.  
The genesis of this concept arose from a collaborative engagement with another team within my organization. Specifically, upon the posting of a message to the mentioned team's Slack channel, an automated bot responded with a proposed solution. So, I wanted to understand how this system was built and working. 

Let's start first with a very high level introduction about AI services which tooks inspiration from modern RAG (Retrieval Augmented Generation) architectures. 

# Introduction to AI services

In a high level overview, AI services do a couple of actions:

- Embeddings
Embeddings convert text into vectors (lists of numbers) that capture the semantic meaning of the content. 
Example: "The capital of France is Paris." is embedded as [0.13, -0.22, ..., 0.08] (typical dimensions: 384, 768, 1024...)
This allows us to compare texts by semantic (not exact) similarity, using metrics such as cosine similarity.
In this project, we'll divide the local files into smaller chunks of text that will be added to the AI service for embedding

- LLM (large language model)
Its goal is to understand natural language and generate coherent responses.
Examples of LLMs: GPT-3, GPT-4 → from OpenAI, LLaMA, LLaMA2, Mistral, Phi-2 → from Meta, and others, available locally Gemini, Claude, Falcon, etc.

To sum up:
1. Someone asks a question
2. The AI service use embeddings to find the most relevant pieces of your documents which is then used as a "context"
3. This "context" plus the question from the user then goes to the LLM that will generate the final answer in natural language. 

# Architecture

Let's enumerate the components that we'll need:

- **Input**: 
  We'll use local files from different folders.

- **Embeddings service**: 
  Converts document text chunks into high-dimensional vectors (embeddings).
  We'll use `nomic-embed-text` served via Ollama because it works locally without any other third-party services dependencies.

- **LLM (Large Language Model)**: 
  Runs locally using Ollama (e.g., llama2).

- **Vector Store**
  Stores embeddings + text as JSON in `/app/vectorstore/store.json` into into memory at startup.

- **LangChain4j**:
  This is the Java framework that integrates with the embeddings service, the Ollama (for LLM) and provides the interfaces for managing them cleanly.

- **UI**:
  Using Vaadin to generate the HTML files.

- **API**:
  Quarkus WebSocket service.

To be precise, I will be using the Quarkiverse LangChain4j extension which makes all the above integration super easy to follow and understand.

# Getting Started

You can start by cloning the GitHub repository: [https://github.com/Sgitario/docbot](https://github.com/Sgitario/docbot) which is a Maven project using the following extensions:
- quarkus-websockets-next: to connect the UI and Langchain4j.
- quarkus-langchain4j-ollama: to automatically configure the ollama model with langchain4j. 
- langchain4j-embeddings-bge-small-en-q: to embed external files using the [bge-small-en-q](https://huggingface.co/RedHatAI/bge-small-en-v1.5-quant) embedding model by huggingface.
- quarkus-langchain4j-pgvector: to store the embeddings (we'll use a postgresql instance).
- importmap + vaadin: for UI

Credits to [https://quarkus.io/quarkus-workshop-langchain4j](https://quarkus.io/quarkus-workshop-langchain4j).

The first step is to pull the ollama container:
```
docker pull ollama/ollama:latest
```

When starting the Quarkus service, this is the image that will be used to interact with the Ollama model. 

And looks like the AI service? Here, it is:

```java
@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {
    @SystemMessage("""
            Use a super informal language.
            """)
    Multi<String> chat(String userMessage);
}
```

Simple like that. The Quarkus Langchain4j extension will auto-generate the actual service that internally uses Langchain4j with Ollama. 
Note that the annotation `@SystemMessage` allows us to configure the tone and general directives of the AI bot. 

Now, let's configure the folder where will be adding the files:

```
# use the bge-small-en-q embedding model:
quarkus.langchain4j.embedding-model.provider=dev.langchain4j.model.embedding.onnx.bgesmallenq.BgeSmallEnQuantizedEmbeddingModel
# size of the embeddings that will be stored:
quarkus.langchain4j.pgvector.dimension=384
```

## Ingestor

Ok, so, we have configured the hugging face embedding for ingesting the files, but what files? This is what we need a new component:

```java
@ApplicationScoped
public class RagIngestion {

    /**
     * Ingests the documents from the given location into the embedding store.
     *
     * @param ev             the startup event to trigger the ingestion when the application starts
     * @param store          the embedding store the embedding store (PostGreSQL in our case)
     * @param embeddingModel the embedding model to use for the embedding (BGE-Small-EN-Quantized in our case)
     * @param documents      the location of the documents to ingest
     */
    public void ingest(@Observes StartupEvent ev,
                       EmbeddingStore store, EmbeddingModel embeddingModel,
                       @ConfigProperty(name = "rag.location") Path documents) {
        store.removeAll(); // cleanup the store to start fresh (just for demo purposes)
        List<Document> list = FileSystemDocumentLoader.loadDocumentsRecursively(documents);
        EmbeddingStoreIngestor ingestor = EmbeddingStoreIngestor.builder()
                .embeddingStore(store)
                .embeddingModel(embeddingModel)
                .documentSplitter(recursive(100, 25,
                        new HuggingFaceTokenizer()))
                .build();
        ingestor.ingest(list);
        Log.info("Documents ingested successfully");
    }
}
```

And now, we can configure the documents we want to ingest using the property `rag.location`. 

## Retrieval

Now, we have all the embeddings in our vector store (postgres), we need to retriever which is the responsible for finding the most relevant segments for a given text:

```java
public class RagRetriever {

    @Produces
    @ApplicationScoped
    public RetrievalAugmentor create(EmbeddingStore store, EmbeddingModel model) {
        var contentRetriever = EmbeddingStoreContentRetriever.builder()
                .embeddingModel(model)
                .embeddingStore(store)
                .maxResults(3)
                .build();

        return DefaultRetrievalAugmentor.builder()
                .contentRetriever(contentRetriever)
                .build();
    }
}
```

## Start application

Using `mvn quarkus:dev`, it will pull the ollama model, start up the postgresql database and then our service that is running at `http://localhost:8080` waiting for you to make questions!
Take into account that we can also check how the embedding store looks in http://localhost:8080/q/dev-ui. 

# What's next

Note that we can add as many other retrieval augmentors as we want and also that we can define even a repository that our model could use to make queries directly to our database, for example:

```java
@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {

    @SystemMessage("""
            You are a customer support agent of a car rental company 'Miles of Smiles'.
            You are friendly, polite and concise.
            If the question is unrelated to car rental, you should politely redirect the customer to the right department.

            Today is {current_date}.
            """)
    @ToolBox(BookingRepository.class)
    String chat(String userMessage);
}
```

where BookingRepository is a regular Panache repository:

```java
@ApplicationScoped
public class BookingRepository implements PanacheRepository<Booking> {

    @Tool("List booking for a customer")
    @Transactional
    public List<Booking> listBookingsForCustomer(String customerName, String customerSurname) {
        var found = Customer.findByFirstAndLastName(customerName, customerSurname);

        return found
          .map(customer -> list("customer", customer))
          .orElseThrow(() -> new CustomerNotFoundException(customerName, customerSurname));
    }


    // ...
}
```

Note that annotations `@ToolBox` and `@Tool`. 

# Conclusion

All the credits to the Quarkus community and specially to this blog [https://quarkus.io/quarkus-workshop-langchain4j](https://quarkus.io/quarkus-workshop-langchain4j) and [https://developers.redhat.com/articles/2025/04/07/how-build-ai-ready-applications-quarkus](https://developers.redhat.com/articles/2025/04/07/how-build-ai-ready-applications-quarkus), where you can find much more information about how this internally works.
