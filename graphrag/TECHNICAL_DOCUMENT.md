# GraphRAG Technical Document

## 1. Introduction and Overview

GraphRAG is a Python-based system designed to transform unstructured text data into structured knowledge graphs. Its primary purpose is to enable sophisticated information retrieval and question answering by leveraging Large Language Models (LLMs) over these generated graphs. By structuring data as a graph, GraphRAG allows for nuanced exploration of relationships and themes within text, which traditional methods might miss.

Core capabilities include:

-   **Indexing:** This process involves reading unstructured text documents, identifying and extracting key entities (e.g., people, organizations, locations) and the relationships between them. This structured information is then used to construct a knowledge graph. Alongside graph construction, relevant text segments are converted into vector embeddings to support semantic similarity searches.
-   **Querying:** GraphRAG allows users to ask natural language questions. The system employs various search strategies to find the most relevant information within the indexed knowledge graph and associated text embeddings. This retrieved context is then provided to an LLM, which synthesizes a comprehensive answer.
-   **Prompt Tuning:** To enhance the quality of LLM-driven tasks during indexing (like entity extraction), GraphRAG features a prompt tuning capability. This process automatically generates and optimizes prompts based on the specific characteristics of the input dataset, leading to more accurate and relevant LLM outputs.

## 2. Architecture

GraphRAG's architecture is modular, centered around three main stages: Configuration, Indexing, and Querying. These stages are supported by a common set of components for data handling, LLM interaction, and operational utilities.

-   **Configuration:** Before any processing begins, GraphRAG is configured via YAML or JSON files. This setup defines data sources, LLM models and API keys, storage solutions (for artifacts and vectors), and parameters that control the behavior of the indexing and querying pipelines.

-   **Indexing Pipeline:** This is where the core transformation from unstructured text to a queryable knowledge graph occurs. The pipeline typically involves:
    -   **Data Ingestion & Preparation:** Loading documents and chunking them into manageable `TextUnit`s.
    -   **Information Extraction:** Utilizing LLMs, guided by either predefined or auto-tuned prompts, to identify entities and relationships. NLP-based extraction methods can also be employed.
    -   **Embedding Generation:** Creating vector embeddings for `TextUnit`s and potentially other textual elements (like entity descriptions) using embedding LLMs.
    -   **Graph Construction:** Assembling the extracted entities and relationships into a formal graph structure.
    -   **Graph Enrichment & Analysis:** Performing operations like graph pruning, community detection (clustering nodes into meaningful groups), and generating textual summaries for entities and communities.
    -   **Data Persistence:** Storing the resulting graph artifacts (entities, relationships, communities, reports) and vector embeddings in the configured storage and vector store backends.

-   **Querying Pipeline:** This stage handles user queries and generates answers based on the indexed knowledge graph.
    -   **Query Processing:** Receiving the user's natural language query.
    -   **Context Retrieval:** Based on the chosen search strategy, relevant data (e.g., `TextUnit`s, `Entity` details, `CommunityReport` summaries) is fetched from persistent storage and vector stores. This often involves semantic search using the query's vector embedding.
    -   **Answer Synthesis:** The retrieved context and the original query are formulated into a prompt for a chat LLM, which then generates a textual answer. Different search strategies (Local, Global, etc.) determine how context is selected and utilized.

Shared components like data models, LLM management, caching, logging, and callback systems provide foundational support across both indexing and querying.

## 3. Key Components

This section details the primary modules within the `graphrag` package and their roles.

### 3.1. CLI (`graphrag.cli`)

The Command Line Interface (CLI), built with the Typer library, serves as the main user entry point for GraphRAG operations:

-   **`graphrag init --root <project_path>`**: Initializes a new GraphRAG project by creating a default configuration file (typically `settings.yaml`) in the specified project root directory.
-   **`graphrag index --root <project_path>`**: Executes the indexing pipeline using the configuration found in the project root, building or updating the knowledge graph.
-   **`graphrag query --root <project_path> --method <search_method> "<your_query>"`**: Runs a natural language query against the indexed graph in the project, using a specified search method (e.g., `local`, `global`).
-   **`graphrag prompt-tune --root <project_path>`**: Starts the automated process of generating and optimizing prompts for the indexing phase, based on the project's input data.

### 3.2. Configuration (`graphrag.config`)

GraphRAG's operational parameters are defined through configuration files (YAML or JSON, with `settings.yaml` being the common entry point).

-   **Loading Mechanism:** The `load_config.py` module is responsible for parsing these configuration files. It supports environment variable substitution (e.g., for API keys) and allows overriding specific settings via CLI arguments.
-   **Structure & Validation:** Configurations are mapped to Pydantic models (defined in `graphrag.config.models`). The central `GraphRagConfig` model aggregates all other specialized configuration objects (e.g., `InputConfig`, `LanguageModelConfig`, `StorageConfig`, `VectorStoreConfig`). Pydantic ensures data validation and type correctness.
-   **Defaults:** Default values for many parameters are provided in `defaults.py`, simplifying the initial setup.

### 3.3. Indexing Engine (`graphrag.index`)

This is the heart of GraphRAG, responsible for transforming raw text into a structured and queryable knowledge graph.

-   **Orchestration (`run/run_pipeline.py`):** The `run_pipeline` async function orchestrates the entire indexing process. It dynamically loads and executes a sequence of "workflows" based on the chosen indexing method and configuration. It handles full data builds as well as incremental updates to existing graphs, and integrates storage, caching, and callback mechanisms.
-   **Workflows (`workflows/`):** These are high-level recipes defining the major stages of indexing, composed of multiple "operations." Examples include `extract_graph.py` (for entity/relationship extraction), `create_communities.py` (for graph clustering), and `generate_text_embeddings.py`. The system allows for custom workflows to be defined and registered.
-   **Operations (`operations/`):** These are granular, reusable building blocks that perform specific tasks within a workflow. Examples:
    -   **Text Processing:** `chunk_text/` splits documents into smaller `TextUnit`s.
    -   **Graph Extraction:** `extract_graph/` uses LLMs to identify entities and relationships; `build_noun_graph/` offers an alternative NLP-based approach.
    -   **Embedding:** `embed_text/` generates vector embeddings for textual data; `embed_graph/` can embed graph structural information.
    -   **Graph Algorithms:** `create_graph.py` assembles graph structures; `cluster_graph.py` performs community detection; `prune_graph.py` refines the graph.
    -   **Summarization:** `summarize_descriptions/` (for entities/relationships) and `summarize_communities/` (for community reports) use LLMs to generate concise summaries.
    -   **Covariate Extraction:** `extract_covariates/` pulls out additional structured information, like claims, from text.

### 3.4. Query Engine (`graphrag.query`)

The query engine enables users to ask questions and retrieve answers from the indexed knowledge graph.

-   **Factory (`factory.py`):** Provides utility functions (e.g., `get_local_search_engine`, `get_global_search_engine`) to instantiate and configure different search engine objects based on the active `GraphRagConfig`.
-   **Search Strategies (`structured_search/`):** GraphRAG implements several search strategies to cater to different query needs:
    -   **Local Search:** Focuses on a specific, targeted portion of the graph related to the query, aiming for detailed answers from a narrow context.
    -   **Global Search:** Provides broader answers by synthesizing information from across multiple communities or the entire dataset, often employing a map-reduce pattern with LLMs.
    -   **Basic Search:** A straightforward Retrieval Augmented Generation (RAG) method that performs a vector similarity search over text units and uses the top results as context for an LLM.
    -   **DRIFT Search:** A more sophisticated strategy, potentially involving iterative refinement or multi-step reasoning to explore the graph and data.
-   **Context Building (`context_builder/`):** A critical part of the RAG process. Context builders retrieve relevant data (text units, entity details, community reports, relationships) from persistent storage and vector stores. The selection process is guided by the query and the chosen search strategy, often involving semantic search using query embeddings.
-   **LLM Interaction:** Each search strategy formats the user's query and the retrieved context into a structured prompt, which is then sent to the configured chat LLM to synthesize a final textual answer.

### 3.5. Language Model Management (`graphrag.language_model`)

This component provides a consistent interface for interacting with various LLMs for both text generation (chat) and embedding creation.

-   **Protocols (`protocol/base.py`):** Defines `ChatModel` and `EmbeddingModel` abstract interfaces (Python Protocols), standardizing methods like `achat`, `achat_stream` (for chat models) and `aembed`, `aembed_batch` (for embedding models).
-   **Factory (`factory.py`):** The `ModelFactory` is responsible for creating instances of specific LLM clients (e.g., for OpenAI, Azure OpenAI) based on the `ModelType` enum and the detailed settings in `GraphRagConfig`. It uses a registry to manage different LLM provider implementations.
-   **Manager (`manager.py`):** The `ModelManager` acts as a singleton to manage named instances of chat and embedding models. This allows different parts of GraphRAG to efficiently reuse configured LLM clients.
-   **Providers (`providers/fnllm/`):** Concrete implementations for LLM interactions. Currently, GraphRAG uses the `fnllm` library as an intermediary to communicate with OpenAI and Azure OpenAI services. These providers handle API request formatting, response parsing, configuration mapping, integration with GraphRAG's caching, and callback systems.

### 3.6. Data Models (`graphrag.data_model`)

This module defines the core Python dataclasses that represent the various types of information GraphRAG works with. These structures ensure consistency in how data is handled across the system.

-   **Key Data Structures:**
    -   `Document`: Represents an input source document.
    -   `TextUnit`: A chunk of text derived from a `Document`, often the basic unit for embedding and analysis.
    -   `Entity`: An extracted entity (e.g., person, organization) with attributes like type, description, and embeddings.
    -   `Relationship`: A connection between two `Entity` objects, with attributes like weight and description.
    -   `Community`: A group of related entities or text units, typically identified through graph clustering, with hierarchical information.
    -   `CommunityReport`: A summary and analysis of a `Community`, often generated by an LLM.
    -   `Covariate`: Represents other structured information extracted from text, such as claims or facts.
-   **Base Classes:** Common attributes like `id`, `short_id` (human-readable ID), and `title` are often inherited from base classes like `Identified` and `Named`.
-   **Schemas (`schemas.py`):** This file is crucial for data persistence. It defines string constants for DataFrame column names (e.g., `ID`, `TITLE`, `EDGE_SOURCE`). It also specifies `*_FINAL_COLUMNS` lists, which dictate the exact structure and column order for the Parquet files in which these data models are stored.

### 3.7. Storage (`graphrag.storage`)

This component handles the saving and loading of intermediate and final artifacts generated during the indexing pipeline.

-   **Interface (`pipeline_storage.py`):** The `PipelineStorage` abstract base class defines a standard API for storage operations, including methods like `get`, `set`, `find` (for discovering files/blobs), `delete`, and `child` (for creating logical sub-stores).
-   **Factory (`factory.py`):** The `StorageFactory` instantiates specific storage backend clients (e.g., for local files, Azure Blob Storage, or in-memory storage) based on the `OutputType` enum and settings in `GraphRagConfig`.
-   **Implementations:**
    -   `file_pipeline_storage.py`: Implements storage on the local file system, organizing artifacts in a specified root directory.
    -   `blob_pipeline_storage.py`: Implements storage in Azure Blob Storage, managing data within a specified container and optional path prefix.
-   **Usage:** Artifacts, frequently Pandas DataFrames (saved as Parquet files) or JSON files (like run statistics), are managed through this layer.

### 3.8. Vector Stores (`graphrag.vector_stores`)

This component is dedicated to managing vector embeddings, which are essential for semantic search.

-   **Interface (`base.py`):**
    -   `VectorStoreDocument`: A dataclass representing an item stored in the vector store, containing its ID, original text (optional), vector embedding, and other attributes.
    -   `VectorStoreSearchResult`: A dataclass for results from a similarity search, containing the found `VectorStoreDocument` and its similarity score.
    -   `BaseVectorStore`: An abstract base class defining the standard API for vector store operations, including `connect`, `load_documents`, `similarity_search_by_vector`, and `similarity_search_by_text`.
-   **Factory (`factory.py`):** The `VectorStoreFactory` creates instances of specific vector store clients (e.g., LanceDB, Azure AI Search, Azure Cosmos DB) based on the `VectorStoreType` enum and configuration.
-   **Implementations:**
    -   `lancedb.py`: Provides an implementation for LanceDB, a local, file-based vector database.
    -   `azure_ai_search.py`: Implements support for Azure AI Search (formerly Azure Cognitive Search).
-   **Role:** Vector stores persist the embeddings generated during indexing. The query engine uses them to find documents or text chunks semantically similar to a user's query, forming a key part of the RAG pattern.

### 3.9. API (`graphrag.api`)

The `graphrag.api` module offers a programmatic interface to GraphRAG's main functionalities, facilitating its use as a library within other applications or for scripting purposes.

-   **`index.py` (`build_index` function):** Allows users to trigger the indexing pipeline programmatically, providing configuration and options for full or incremental builds.
-   **`query.py` (e.g., `global_search`, `local_search` functions):** Exposes functions to execute various search strategies against an indexed graph. These functions typically require the caller to load the necessary data (entities, reports, etc.) into Pandas DataFrames. Streaming and multi-index search variants are also available.
-   **`prompt_tune.py` (`generate_indexing_prompts` function):** Provides programmatic access to the automated prompt generation process.

### 3.10. Caching (`graphrag.cache`)

The caching system helps improve performance and reduce costs by storing and reusing the results of expensive operations, particularly LLM API calls.

-   **Interface (`pipeline_cache.py`):** The `PipelineCache` abstract base class defines methods like `get`, `set`, `has`, and `delete`.
-   **Factory (`factory.py`):** The `CacheFactory` creates cache instances based on the `CacheType` enum and configuration (e.g., in-memory, file-based JSON, Azure Blob-based JSON, or a no-op cache).
-   **Implementations:**
    -   `JsonPipelineCache`: Persists cache entries as JSON files, using a `PipelineStorage` backend (like local files or Azure Blob Storage).
    -   `InMemoryCache`: A non-persistent, dictionary-based cache for fast in-session caching.
-   **Usage:** Applied primarily during indexing and prompt tuning to cache LLM responses.

### 3.11. Callbacks (`graphrag.callbacks`)

GraphRAG's callback system allows users or other system components to react to events during indexing and querying.

-   **Protocols:**
    -   `WorkflowCallbacks` (`workflow_callbacks.py`): For events in the indexing pipeline (e.g., `pipeline_start`, `workflow_end`, `progress`, `error`).
    -   `BaseLLMCallback` (`llm_callbacks.py`): For LLM-specific events, notably `on_llm_new_token` for streaming output.
    -   `QueryCallbacks` (`query_callbacks.py`): For events during query processing (e.g., `on_context` after context retrieval, `on_map_response_end` for global search steps).
-   **Role:** Enables custom monitoring, logging, reporting, real-time feedback (like streaming query answers to a UI), and other extensions.

### 3.12. Logging (`graphrag.logger`)

This component manages how GraphRAG reports progress and status messages.

-   **Interfaces (`base.py`):**
    -   `StatusLogger`: For basic status messages (error, warning, log).
    -   `ProgressLogger`: For detailed progress updates, often for display in a CLI.
-   **Factory (`factory.py`):** The `LoggerFactory` creates logger instances based on the `LoggerType` enum (e.g., `RICH` for an enhanced terminal UI with progress bars via the `rich` library, `PRINT` for simple text output, or `NONE` for no logging).
-   **Implementations:** `ConsoleReporter` (basic printing), `RichProgressLogger` (interactive CLI progress).
-   **Role:** Provides crucial feedback to the user, especially during CLI operations, and assists in debugging.

### 3.13. Prompt Tuning (`graphrag.prompt_tune`)

This advanced feature automates the creation of LLM prompts that are specifically tailored to the user's input data, aiming to improve the performance of LLM-driven tasks during indexing.

-   **Process:**
    1.  **Data Loading (`loader/input.py`):** Loads and chunks a representative subset of the input documents.
    2.  **Component Generation (`generator/`):** Uses an LLM to analyze this data subset and generate various components like the data's primary `domain`, `language`, a suitable LLM `persona`, relevant `entity_types`, and few-shot `examples` of entity/relationship extraction.
    3.  **Prompt Assembly:** These generated components are then inserted into more complex prompt `template/` structures (e.g., for entity extraction) to create the final, data-tuned prompts.
-   **Role:** By customizing prompts (e.g., for entity extraction, summarization) to the nuances of the input data, this process aims to achieve higher accuracy and relevance in LLM outputs compared to using generic, static prompts. The `graphrag.api.prompt_tune.generate_indexing_prompts` function orchestrates this.

### 3.14. Prompts (`graphrag.prompts`)

This directory stores a collection of predefined, static prompt templates that serve as defaults or fallbacks for various LLM tasks if custom-tuned prompts are not used.

-   **Structure:** Organized into `index/` (for indexing tasks) and `query/` (for querying tasks) subdirectories.
-   **Examples:**
    -   `index/extract_graph.py`: Contains detailed prompt templates for entity and relationship extraction, including instructions on output format and few-shot examples.
    -   `query/local_search_system_prompt.py`: Provides the system prompt that guides the LLM in generating answers for the local search strategy, including instructions on how to cite data sources.
    -   Similar predefined prompts exist for global search (map and reduce steps), claim extraction, and summarization tasks.
-   **Role:** These prompts provide baseline instructions for LLMs, ensuring a degree of consistency and structure in their outputs. They can be overridden by user-provided prompt files or by the outputs of the `prompt_tune` process.

## 4. Data Flow

### 4.1. Indexing Process

The indexing process transforms raw unstructured documents into a structured, queryable knowledge graph and associated vector embeddings.

1.  **Input & Configuration:** The process starts with user-provided unstructured documents (e.g., text files, CSVs) and a `GraphRagConfig` file detailing all parameters for the run.
2.  **Text Processing:**
    -   Documents are loaded and segmented into smaller, manageable chunks called `TextUnit`s.
    -   These `TextUnit`s are then typically embedded using a configured `EmbeddingModel`. The resulting vector embeddings are stored in the designated `VectorStore`.
3.  **Graph Extraction:**
    -   Using LLM prompts (which can be predefined or auto-tuned via `graphrag.prompt_tune`), a `ChatModel` processes the `TextUnit`s to identify and extract:
        -   `Entity` objects (e.g., persons, organizations, locations) with their types and descriptions.
        -   `Relationship` objects connecting these entities, along with descriptions and strengths of these connections.
    -   Descriptions for extracted entities and relationships might be further summarized by an LLM.
4.  **Graph Construction & Refinement:**
    -   The extracted entities and relationships are used to construct an initial graph (often in-memory using a library like NetworkX, then serialized).
    -   This graph can be refined through pruning (removing less significant nodes or edges based on metrics like frequency or degree) and community detection (clustering related nodes into `Community` hierarchies using algorithms like Leiden).
5.  **Enrichment & Reporting:**
    -   `CommunityReport` objects are generated, often using an LLM to summarize the key themes, entities, and insights within each detected community.
    -   Additional structured information, termed `Covariate`s (e.g., factual claims), may also be extracted from the text and linked to relevant graph elements.
6.  **Persistence:** All generated artifacts—`TextUnit`s, `Entity`s, `Relationship`s, `Community`s, `CommunityReport`s, `Covariate`s, along with their vector embeddings and other metadata—are saved to the configured `PipelineStorage` (typically as Parquet files for tabular data and JSON for other metadata) and the `VectorStore`.

### 4.2. Querying Process

The querying process allows users to ask natural language questions and receive answers synthesized from the indexed knowledge graph.

1.  **User Query:** A user submits a query through the CLI or API.
2.  **Setup:** The query engine loads the `GraphRagConfig` and initializes the necessary components, including the `ModelManager` (for LLMs), `PipelineStorage` (to access indexed data like Parquet files), and the `VectorStore`.
3.  **Search Engine Initialization:** Based on the specified search method (e.g., "local", "global"), the corresponding search engine (e.g., `LocalSearch`, `GlobalSearch`) is instantiated from `graphrag.query.factory`. This involves loading the required indexed data (entities, relationships, reports, etc.) into memory (often as Pandas DataFrames).
4.  **Context Retrieval & Building:** This is a core RAG step.
    -   The user's query is often converted into a vector embedding.
    -   The `ContextBuilder` associated with the chosen search strategy retrieves relevant information. This may involve:
        -   Performing a similarity search in the `VectorStore` using the query embedding to find relevant `TextUnit`s or `Entity` descriptions.
        -   Fetching specific `Entity` details, `Relationship` data, or `CommunityReport` summaries based on keywords, entity mentions in the query, or graph proximity.
    -   The retrieved data is then formatted into "context chunks"—text snippets that will be included in the prompt to the LLM.
5.  **LLM Prompting & Answer Synthesis:**
    -   The search strategy constructs a detailed prompt by combining:
        -   The user's original query.
        -   The retrieved context chunks.
        -   A system prompt (from `graphrag.prompts` or user configuration) that instructs the LLM on its role, the desired answer format, and how to use the provided context and cite sources.
    -   For **Global Search**, this may be a multi-step process:
        -   **Map:** The context is divided, and multiple LLM calls are made in parallel, each processing a part of the context to extract key points.
        -   **Reduce:** The outputs from the map calls are aggregated and sent to a final LLM call to synthesize a single, coherent answer.
    -   The configured `ChatModel` processes the prompt(s) and generates a textual response.
6.  **Output:** The LLM's generated answer is returned to the user. If streaming is enabled, the answer is returned token by token as it's generated. The context data used for the answer may also be provided for transparency.

## 5. How to Use

The primary way to interact with GraphRAG is through its Command Line Interface (CLI).

-   **Initialization:**
    ```bash
    graphrag init --root /path/to/your/project
    ```
    This command creates a default `settings.yaml` configuration file in your project directory. You **must** edit this file to specify your data inputs, LLM API keys and endpoints, desired storage locations, vector store preferences, and other operational parameters.

-   **Indexing:**
    ```bash
    graphrag index --root /path/to/your/project
    ```
    This command runs the full indexing pipeline as defined by the `settings.yaml` in your project's root directory. It will process your input data, build the knowledge graph, generate embeddings, and save all artifacts to your configured storage.

-   **Querying:**
    ```bash
    graphrag query --root /path/to/your/project --method local "Your natural language question here"
    ```
    This command executes a query against the previously indexed graph in your project directory. You can specify different search methods like `local`, `global`, or `basic` using the `--method` option.

For more advanced use cases, GraphRAG's functionalities can be accessed programmatically through the functions exposed in the `graphrag.api` module.

## 6. Extensibility

GraphRAG is designed with several extensibility points to allow users to customize and enhance its behavior:

-   **Callbacks (`graphrag.callbacks`):** Users can implement defined callback interfaces (`WorkflowCallbacks`, `QueryCallbacks`, `BaseLLMCallback`). This allows for custom actions to be triggered at various stages of the indexing and querying pipelines, such_as custom logging, external monitoring, UI updates, or specialized reporting.
-   **Custom Indexing Workflows and Operations:** The indexing engine (`graphrag.index`) supports the registration of custom workflow functions via `graphrag.api.index.register_workflow_function`. This enables developers to define their own data processing steps or algorithms and integrate them into the main indexing pipeline by referencing them in the `workflows` list within the `GraphRagConfig`.
-   **Custom Component Implementations:** The factory patterns used for `PipelineStorage`, `PipelineCache`, `ProgressLogger`, and `BaseVectorStore` allow developers to register and use their own custom implementations for these components if the provided ones do not meet specific needs (e.g., a different type of database or a specialized caching strategy).
-   **Prompt Customization:** While GraphRAG offers predefined prompts and an auto-tuning mechanism (`graphrag.prompt_tune`), users can also provide their own custom prompt files for any LLM-driven task. These custom prompts can be specified in the `GraphRagConfig`, allowing for fine-grained control over LLM behavior.

This technical document provides a comprehensive high-level overview of the GraphRAG system. For more granular details on specific functionalities or configurations, users should consult the relevant source code modules and any accompanying examples or tutorials.
