#rag_bot #RAG #information_retrieval #embedding_models #sentence_transformers #multilingual 
## Code change for hindi:
RAG_BOT is  an  agentic RAG system using langgraph and run as a telegram bot with chromadb as vector store.  I am storing the pdf documents in chromadb with date metadata and then trying to query the vectordb using natural language. The murli pdf documents are in hindi and I am using google gemini 2.5 flash as the LLM. I want the LLM to respond to my questions about the murlis in hindi. Will the existing code base work for hindi murli documents and something needs to change in the code and architecture. 

## Gemini code assist response:
Okay, let's break down how your current RAG system might handle Hindi documents and what changes would be needed for it to work effectively and respond in Hindi.

The core architecture using LangGraph, ChromaDB, and Telegram is generally language-agnostic. However, the effectiveness hinges on specific components, especially the models and prompts used.

Here's an analysis of the relevant parts of your codebase and the necessary adjustments:

1. **PDF Processing (`pdf_processor.py`)**:
    
    - **Text Extraction**: `PyPDFLoader` usually works for text-based PDFs regardless of language, assuming standard encoding. **Check**: Verify that it correctly extracts the Devanagari script from your Hindi PDFs. If your PDFs are image scans, you'll need to integrate an OCR (Optical Character Recognition) step, possibly using a tool like Tesseract with Hindi language packs, before `PyPDFLoader`.
    - **Date Extraction (`extract_date_from_text`)**: Your current regex patterns look for `YYYY-MM-DD`, `DD/MM/YYYY`, `DD/MM/YY`. These will only work if dates in the PDFs use Arabic numerals. **Change Needed**: If dates use Devanagari numerals (e.g., २०/०१/१९६९) or Hindi month names, you'll need to add new regex patterns or use a more sophisticated date parsing library that understands Hindi locales to update this function.
    - **Murli Type (`get_murli_type`)**: This function checks for the lowercase Roman script word "avyakt". **Change Needed**: If the PDFs use the Devanagari script "अव्यक्त", you need to modify this function to search for that specific string.
    - **Chunking**:
        - `RecursiveCharacterTextSplitter`: This is language-agnostic but might not be optimal for Hindi sentence structure. It should still work reasonably well.
        - `SentenceTransformersTokenTextSplitter` (Semantic Chunking): This relies heavily on the embedding model (`all-MiniLM-L6-v2`). Since this model is English-focused, semantic chunking performance on Hindi will likely be poor. **Recommendation**: Either switch `SEMANTIC_CHUNKING` to `False` in `config.py` to use `RecursiveCharacterTextSplitter`, or find a multilingual sentence transformer model specifically for this splitter (though changing the main embedding model is more critical).
2. **Embedding Model (`config.py`, `vector_store.py`)**:
    
    - **CRITICAL ISSUE**: You're using `all-MiniLM-L6-v2` (`EMBEDDING_MODEL_NAME`). This model is primarily trained on English and will generate very poor vector representations for Hindi text. This will lead to inaccurate document retrieval (the RAG part won't find relevant Hindi context).
    - **CRITICAL CHANGE**: You **must** switch to a multilingual embedding model. Good options available via HuggingFace include:
        - `paraphrase-multilingual-mpnet-base-v2`
        - `LaBSE`
        - Or potentially newer models specifically designed for multilingual tasks.
    - **Action**: Update `EMBEDDING_MODEL_NAME` in `config.py` and **re-index all your Hindi PDFs** using the new model by deleting the old `VECTOR_STORE_PATH` directory and letting the application rebuild it on startup.
3. **Reranker Model (`config.py`, `retrieval_nodes.py`)**:
    
    - **CRITICAL ISSUE**: You're using `cross-encoder/ms-marco-MiniLM-L-6-v2` (`RERANKER_MODEL_NAME`). This is also an English-focused model and won't effectively rerank Hindi documents based on a Hindi query.
    - **CRITICAL CHANGE**: Switch to a multilingual cross-encoder model. A common choice is:
        - `cross-encoder/mmarco-mMiniLMv2-L12-H384-v1`
    - **Action**: Update `RERANKER_MODEL_NAME` in `config.py`.
4. **LLM and Prompts (`config.py`, `agent/prompts.py`, `agent/agent_node.py`)**:
    
    - **LLM**: `gemini-2.5-flash` is a strong multilingual model and should understand/generate Hindi well.
    - **Prompts**: Your current prompts (`system_prompt`, `evaluate_context_prompt`, `reframe_question_prompt`, `final_answer_prompt_system`, etc. in `prompts.yaml`) are in English.
    - **CHANGE NEEDED**: To get responses _in Hindi_, you need to instruct the LLM. The easiest way is to modify the system prompt part of your `FINAL_ANSWER_PROMPT`.
        - In `prompts.yaml`, find `final_answer_prompt_system`. Add an instruction like: "कृपया केवल हिंदी में उत्तर दें।" (Please respond only in Hindi.) or "आपका उत्तर हमेशा हिंदी भाषा में होना चाहिए।" (Your response should always be in the Hindi language.)
        - Consider translating other prompts (like evaluation prompts) if you find the LLM performs better when prompted entirely in Hindi, although Gemini might handle mixed-language prompts reasonably well.
    - **Hardcoded Responses**: The "cannot find" message in `agent_node.py` is currently hardcoded English JSON. **Change Needed**: Modify this to provide the message in Hindi, still within the JSON structure.
        
        ```python
        # In agent_node.py, inside the 'else' block for insufficient context: 
        cannot_find_content = {     
        # "answer": "Relevant information cannot be found in the database..." #Old 
            "answer": "प्रश्न का उत्तर देने के लिए डेटाबेस में प्रासंगिक जानकारी नहीं मिल पा रही है। कृपया अपना प्रश्न दोबारा पूछने का प्रयास करें।" # New Hindi version 
            } 
            
            cannot_find_json_string = f"```json\n{json.dumps(cannot_find_content, indent=2, ensure_ascii=False)}\n```" # Add ensure_ascii=False 
            cannot_find_message = AIMessage(content=cannot_find_json_string) return {"messages": [cannot_find_message]}`
        
```
        
        _Note the `ensure_ascii=False` in `json.dumps` to preserve Hindi characters._
5. **Bot Interface (`bot.py`, `message_handler.py`)**:
    
    - **CHANGE NEEDED**: The welcome message (`/start`), help message (`/help`), and any error messages sent directly by the bot (e.g., "Please upload a PDF document.") are in English. Translate these strings into Hindi for a consistent user experience.

**In Summary:**

Yes, the existing codebase _can_ be adapted, but it **will not work correctly** for Hindi queries and responses without significant changes:

1. **Must Change**: Embedding model (`EMBEDDING_MODEL_NAME`) to a multilingual one.
2. **Must Change**: Reranker model (`RERANKER_MODEL_NAME`) to a multilingual one.
3. **Must Change**: Update LLM prompts (especially `final_answer_prompt_system`) to request responses in Hindi.
4. **Must Change**: Translate hardcoded English bot responses/messages (like the "cannot find" JSON and UI messages) to Hindi.
5. **Likely Change**: Update date/metadata extraction in `pdf_processor.py` if Hindi numerals/names are used.
6. **Verify**: Ensure PDF text extraction works correctly for your Hindi PDFs.
7. **Consider**: Switching off semantic chunking if its performance is poor with the English-centric model.

After making these changes (especially re-indexing with a multilingual embedding model), the system should be able to retrieve relevant Hindi document chunks based on Hindi queries and generate answers in Hindi using the Gemini LLM.


## Date extraction logic:
So I am indexing murli pdf documents to a vector database. Each murli pdf may contain multiple murlis. Murli is a spiritual discourse. And each murli has an assoiated date, the date on which that murli was spoken. Sometimes you may have two dates on a murli, one the original date on which the murli was first spoken and second date when the murli was redistributed some years later for spiritual study. The dates can be in various formats but the most common format is dd.mm.yy. 
I need to extract metadata from both the english and hindi murli pdf documents. The dates in both hindi and english murli pdfs uses arabic numerals like 21.01.69. In some of the pdfs the date is mentioned at the start of a particular murli along with the title and then you have several pages of murli with no title and date at the top of the page. And after this murli, another murli starts from a new page with title and date and several pages of murli text after that with no title and date and so on. 
How do I implement date extraction logic to reliably extract the dates from the pdf documents corresponding to a murli and associate that date as metadata to all the pages of that murli extracted from the pdf.

### Prompt 2:
In addition to the existing date patterns we can also have dates like 1.1.1980 (d.m.yyyy) or 1.03.80 or 10.1.90. Basically we can have a single digit day if day < 10 or we can have a double digit day with 0 prepended if day < 10, a single digit month if month < 10  or we can have a double digit month with 0 prepended if month < 10 and year in two digits or four digits. Also we can have a ".", a "-" or a "/" as the separator. Can you add these additional date formats to the date extraction logic.


## Embedding models for hindi:
How good is 'paraphrase-multilingual-mpnet-base-v2' for hindi embeddings. Is there a better model than this for hindi embeddings.

## GPT-4o response:
Great question — and it shows you’re digging into **semantic embeddings for multilingual or low-resource languages** like Hindi, which is an important (and often under-researched) space.

Let’s evaluate how good `paraphrase-multilingual-mpnet-base-v2` is for **Hindi**, and whether there are **better models** depending on your use case.

---

### 🔍 About `paraphrase-multilingual-mpnet-base-v2`

This is a **Sentence Transformer model** trained by the SBERT team. Key features:

- ✅ Trained on **50+ languages**, including Hindi
- ✅ Based on **MPNet**, which is more expressive than BERT
- ✅ Works well for **semantic similarity, search, clustering**
- ❗Trained primarily on **parallel (translated) sentence pairs**, which may not capture **deep Hindi semantics**

**Verdict for Hindi:**  
👍 **Good general-purpose multilingual model** for Hindi, especially for tasks like:
- Paraphrase detection
- Semantic search
- Embedding sentences in a multilingual context

But it's not **Hindi-specialized** — it treats Hindi as one of many.

---

### 🧠 So... is there a **better model for Hindi embeddings**?

Yes, potentially — **depending on your exact use case**:

---

### ✅ 1. **IndicSBERT / IndicSentenceBERT**
- Built by AI4Bharat (IIT Madras)
- Specifically trained on **Indic languages** including Hindi, Marathi, Tamil, Bengali, etc.
- Based on **MuRIL** or **XLM-R** backbones

```bash
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('ai4bharat/IndicSBERT')
```

🧠 **Much better cultural + linguistic grounding** than generic multilingual models.

---

### ✅ 2. **LaBSE (Language-Agnostic BERT Sentence Embedding)**

- From Google Research
- Trained on **125+ languages**
- Works well for **cross-lingual semantic search**
- Still widely used in many multilingual RAG setups

```python
SentenceTransformer('sentence-transformers/LaBSE')
```

👍 Performs *very well* on Hindi for cross-lingual tasks, but **not as sharp** as IndicSBERT for pure Hindi-Hindi use.

---

### ✅ 3. **IndicTrans 2 + LLM-based Embeddings (advanced)**
- Newer AI4Bharat models like IndicTrans 2 can be used in a pipeline
- If you have access to **Gemini 1.5** or **Mistral/Claude** with multilingual embeddings, you can also embed Hindi text directly via APIs

---

### 🧪 Performance Comparison (Unofficial)

| Model | Strengths | Hindi Quality | Use Case Fit |
|-------|-----------|----------------|---------------|
| `paraphrase-multilingual-mpnet-base-v2` | General-purpose multilingual | ✅ Good | Cross-lingual, zero-shot tasks |
| `ai4bharat/IndicSBERT` | Hindi + Indic-focused | ✅✅ Better | Hindi-native tasks |
| `LaBSE` | Robust multilingual alignment | ✅ Good | Semantic search, translation-aligned |
| `XLM-R` fine-tunes | Can be excellent with domain data | ✅✅ Best (with fine-tuning) | Domain-specific Hindi embeddings |

---

### 🎯 Final Verdict

If your task is **Hindi → Hindi**, or Indic-centric, then:
> **✅ Use `ai4bharat/IndicSBERT`** — it’s *better trained and aligned* for Indian languages.

If you're doing **cross-lingual** search (e.g., English ↔ Hindi):
> **✅ Use `LaBSE` or `paraphrase-multilingual-mpnet-base-v2`**

---

### Bonus 💡
If you're working on RAG or agentic tasks in Hindi:
- You can **translate Hindi to English**, embed, retrieve, and **respond in Hindi** using Gemini or GPT with translation prompts.
- Or, use **IndicTrans2** for smarter translation before/after embedding.

---

Want me to demo a script comparing these models on sample Hindi sentences using cosine similarity or clustering? 😎


## Making RAG_BOT multilingual:
So the logic to load hindi murli from htm documents and then index them in chromadb is working, reranking using 'cross-encoder/ms-marco-MiniLM-L-6-v2' is also working. Also I have tested bot.py by asking questions in hindi about murli and it can retrieve the relevant murli context and then answer in hindi from that context. What I want next is to enable the capability to answer both in English or Hindi. For that I am thinking of indexing both english (from pdfs) and hindi murlis (from htm) in the same chromadb instance using "paraphrase-multilingual-mpnet-base-v2". When bot.py is run, we will run the indexing of both hindi and english murlis from respective data folders (within data directory we will have two subdirectories hindi and english). We will have a language setting in Config.py. Default can be english. Based on what the language is, we will set the EMBEDDING_MODEL_NAME, RERANKER_MODEL_NAME in Config.py. 

Secondly in bot.py in the help section there will be an option to set the bot language using /language hindi or english. WHen the user sends the /language command the language setting in config instance should be overwritten. Also along with that the relevant EMBEDDING_MODEL_NAME and RERANKER_MODEL_NAME should get loaded in config.py. 

Thirdly the prompts for hindi and english languages will need to handled. I mean for hindi language we have the additional instruction to respond in hindi both in bk_persona and final_answer_prompt_system. 

What are your thoughts on the above approach. Could you review and suggest any improvements to this approach. And if it looks ok, can you prepare a detailed plan on how to implement the suggested changes to make the RAG_BOT app multilingual.

## GPT 4.1 response:

```markdown
Review & Suggestions
1. Indexing Both Languages
Good: Using a multilingual embedding model like "paraphrase-multilingual-mpnet-base-v2" is the right choice for cross-lingual retrieval.
Suggestion:
Store a language field in the metadata for each document (e.g., "en" or "hi"). This allows for language-specific filtering during retrieval.
Index both Hindi (from HTM) and English (from PDF) murlis into the same ChromaDB instance, as planned.
2. Dynamic Language Setting
Good: Allowing users to set the bot language via /language command is user-friendly.
Suggestion:
Instead of changing global config, maintain a per-user language setting (e.g., in USER_SESSIONS in Config). This allows different users to interact in their preferred language simultaneously.
When the language changes, reload or re-instantiate the embedding and reranker models as needed.
3. Prompt Management
Good: Having separate prompt templates for Hindi and English is necessary.
Suggestion:
Store both Hindi and English versions of each prompt in prompts.yaml (e.g., bk_persona_en, bk_persona_hi).
In Config, add logic to select the correct prompt based on the current language.
4. Bot Command Handling
Good: /language command to switch language is intuitive.
Suggestion:
After changing the language, confirm to the user in the selected language.
Optionally, provide /language without arguments to show the current language and usage instructions.
Implementation Plan
A. Data & Indexing
Add Language Metadata

When processing PDFs (English) and HTMs (Hindi), add a "language": "en" or "language": "hi" field to each document’s metadata.
Update PdfProcessor and HtmProcessor to include this.
Index Both Languages

In your indexing script (or startup logic in bot.py), index both data/hindi and data/english folders.
Use "paraphrase-multilingual-mpnet-base-v2" as the embedding model for both.
B. Config & Model Management
Config Language Setting

Add a LANGUAGE attribute to Config (default "en").
Add a LANGUAGE_MODEL_MAP dict to map language to embedding and reranker model names.
Add methods to update language and reload model names.
Per-User Language Setting

Store user language in USER_SESSIONS (e.g., USER_SESSIONS[user_id]['language']).
Default to Config.LANGUAGE if not set.
Dynamic Model Selection

When handling a message, use the user’s language to select the embedding and reranker models.
If models need to be re-instantiated, do so per request or cache per language.
C. Prompt Management
Prompts for Both Languages
In prompts.yaml, add both Hindi and English versions for each prompt.
In Config, add logic to fetch the correct prompt based on language.
D. Bot Command Handling
/language Command

In bot.py, add a handler for /language <lang>.
Update the user’s session language and reload models if necessary.
Respond with a confirmation in the selected language.
Help Message

Update /help to mention the /language command and show usage.
E. Retrieval Logic
Language-Aware Retrieval
When retrieving documents, filter by the user’s language (e.g., filter={"language": "hi"}).
Update the context retriever tool to accept and use the language filter.
F. Testing
Test Cases
Add integration tests for both Hindi and English queries.
Test language switching and retrieval in both languages.

Example: Config Changes
# ...existing code...
class Config:
    # ...existing code...
    LANGUAGE = os.environ.get('LANGUAGE', 'en')
    LANGUAGE_MODEL_MAP = {
        "en": {
            "EMBEDDING_MODEL_NAME": "paraphrase-multilingual-mpnet-base-v2",
            "RERANKER_MODEL_NAME": "cross-encoder/ms-marco-MiniLM-L-6-v2",
        },
        "hi": {
            "EMBEDDING_MODEL_NAME": "paraphrase-multilingual-mpnet-base-v2",
            "RERANKER_MODEL_NAME": "cross-encoder/ms-marco-MiniLM-L-6-v2",
        }
    }
    # ...existing code...
    @classmethod
    def set_language(cls, lang):
        cls.LANGUAGE = lang
        cls.EMBEDDING_MODEL_NAME = cls.LANGUAGE_MODEL_MAP[lang]["EMBEDDING_MODEL_NAME"]
        cls.RERANKER_MODEL_NAME = cls.LANGUAGE_MODEL_MAP[lang]["RERANKER_MODEL_NAME"]
		

Example: Prompt Selection
# ...existing code...
@classmethod
def get_bk_persona_prompt(cls, lang=None):
    lang = lang or cls.LANGUAGE
    return cls.PROMPTS.get('system_prompt', {}).get(f'bk_persona_{lang}', '')
# ...repeat for other prompts...		

Example: Bot Command Handler
# ...existing code...
def handle_language_command(self, message):
    # Parse /language <lang>
    lang = message.text.split(' ', 1)[-1].strip().lower()
    if lang not in ['en', 'hi']:
        self.bot.reply_to(message, "Supported languages: english, hindi. Use /language english or /language hindi.")
        return
    lang_code = 'en' if lang == 'english' else 'hi'
    Config.set_language(lang_code)
    # Update user session
    user_id = message.from_user.id
    Config.USER_SESSIONS[user_id]['language'] = lang_code
    reply = "Language set to English." if lang_code == 'en' else "भाषा हिंदी में सेट कर दी गई है।"
    self.bot.reply_to(message, reply)
```

## Need for language specific prompts in multilingual RAG_BOT: 