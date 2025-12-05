
🧬 Skin Cancer Mutation RAG System
📘 Project Overview
This project implements a Retrieval-Augmented Generation (RAG) system designed to act as an intelligent assistant for clinical and molecular oncology. It answers questions specifically related to skin cancer protein mutations (e.g., BRAF, NRAS, TP53).
The system combines:
    1. Knowledge Retrieval: Fetches relevant scientific instructions from the "Mol-Instructions" dataset.
    2. External Knowledge Base: Real-time integration with the UniProt API to validate protein function and structure.
    3. Generative AI: Uses a 4-bit quantized Llama-3.2-1B model to synthesize answers based on the retrieved context.
🛠 Tech Stack & Dependencies
    • Language: Python 3.10+
    • LLM: unsloth/Llama-3.2-1B-Instruct (Quantized via bitsandbytes)
    • Embeddings: sentence-transformers/all-MiniLM-L6-v2
    • Vector Database: FAISS (Facebook AI Similarity Search)
    • Data Sources: HuggingFace Datasets, UniProt API
    • Frameworks: PyTorch, Transformers, Gradio
📦 Installation
code Bash
downloadcontent_copy
expand_less
    # Install core dependencies for RAG and LLM inference
pip install -q datasets transformers sentence-transformers faiss-cpu
pip install -q bitsandbytes accelerate
pip install -q gradio requests
  

📂 Code Structure & API Reference
This section details the classes and functions implemented in the system.
1. Class: MolInstructionsFilter
Purpose: Manages the ingestion and filtering of raw scientific text data. It ensures the system only learns from relevant skin-cancer data rather than general chemistry.
    • __init__(self, cache_dir="./data")
        ◦ Sets up the directory structure for caching downloaded datasets to avoid redundant downloads.
    • download_and_filter(self, max_samples=5000)
        ◦ Streams the "zjunlp/Mol-Instructions" dataset from HuggingFace.
        ◦ Filters entries based on a specific keyword list (e.g., 'melanoma', 'BRAF', 'V600E').
        ◦ Saves the filtered dataset as a JSON file (cancer_filtered.json).
2. Class: UniProtCache
Purpose: Acts as a bridge to the UniProt Knowledgebase. It provides ground-truth biological data about specific genes to hallucination-proof the LLM.
    • __init__(self, cache_dir="./data")
        ◦ Initializes the local JSON cache to store protein data.
        ◦ Defines a target list of high-priority skin cancer proteins (BRAF, TP53, NRAS, etc.).
    • _load_cache(self) / _save_cache(self)
        ◦ Helper methods to read from and write to the local JSON storage.
    • fetch_protein_info(self, gene_name: str)
        ◦ Queries the UniProt REST API for a specific gene.
        ◦ Extracts key details: Protein Name, Biological Function, Accession ID, and Sequence Length.
        ◦ Returns a dictionary of the protein's metadata.
    • preload_cancer_proteins(self)
        ◦ Iterates through the priority list of cancer proteins and pre-fetches their data into the cache during system startup.
3. Class: CancerRAGRetriever
Purpose: Handles the semantic search engine. It converts text into vectors and finds the most relevant scientific contexts for a user's question.
    • __init__(self, cache_dir="./data")
        ◦ Loads the SentenceTransformer model (all-MiniLM-L6-v2) for generating embeddings.
    • build_index(self, data: List[Dict])
        ◦ Takes the filtered text data and converts it into vector embeddings.
        ◦ Builds a FAISS Index (IndexFlatIP) for efficient similarity searching.
        ◦ Normalizes vectors to allow for Cosine Similarity search.
    • retrieve(self, query: str, top_k: int = 3)
        ◦ Converts the user's question into a vector.
        ◦ Searches the FAISS index for the top_k most similar documents.
        ◦ Returns a list of relevant text snippets with their similarity scores.
4. Class: QuantizedLLM
Purpose: A memory-efficient wrapper for the Large Language Model. It enables running a powerful model on limited hardware (e.g., Colab free tier).
    • __init__(self, model_name="unsloth/Llama-3.2-1B-Instruct")
        ◦ Defines the model architecture to be used.
    • load_model(self)
        ◦ Configures 4-bit quantization using BitsAndBytesConfig (NF4 format).
        ◦ Loads the model and tokenizer onto the GPU.
    • generate(self, prompt: str, max_length: int = 512)
        ◦ Tokenizes the input prompt.
        ◦ Runs inference to generate text with specific sampling parameters (temperature=0.7, top_p=0.9).
        ◦ Decodes and returns the answer string.
5. Class: CancerMutationRAG
Purpose: The main controller class that integrates all previous components into a single pipeline.
    • __init__(self)
        ◦ Instantiates the Filter, UniProt, Retriever, and LLM objects.
    • initialize(self)
        ◦ Orchestrates the startup sequence: Download data -> Filter -> Preload Proteins -> Build Vector Index.
        ◦ Note: Delays LLM loading until the first query to save memory.
    • query(self, question: str)
        ◦ The Main Execution Pipeline:
            1. Retrieves relevant docs via CancerRAGRetriever.
            2. Identifies proteins in the query and fetches facts via UniProtCache.
            3. Constructs a context-rich prompt.
            4. Generates the final answer using QuantizedLLM.
    • _extract_proteins(self, text: str)
        ◦ Scans the user's input string to detect gene names (e.g., "Tell me about BRAF") to trigger UniProt lookups.
    • _build_context(self, docs, proteins)
        ◦ Formats the retrieved text and protein metadata into a structured string for the LLM.
    • _build_prompt(self, question, context)
        ◦ Wraps the context and question into a strict system prompt (e.g., "You are an expert in cancer biology...").
