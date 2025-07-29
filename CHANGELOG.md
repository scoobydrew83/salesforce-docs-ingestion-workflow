# Changelog

All notable changes to this Salesforce documentation auto-ingestion workflow.

## [2.0.0] - 2025-01-28

### 🎉 Initial Fixed Release

#### Fixed Major Issues from Original Workflow

- **❌ Broken LangChain Connections**: Fixed incorrect connection types between document processing nodes
- **❌ Invalid Vector Store Usage**: Converted Supabase Vector Store Insert to proper root node architecture
- **❌ Missing Embedding Pipeline**: Integrated OpenAI embeddings directly into vector store node
- **❌ Incorrect Node Types**: Updated to use proper `@n8n/n8n-nodes-langchain` node types

#### ✅ New Features

- **Proper LangChain Architecture**: 
  - Root node: `vectorStoreSupabase` with embedded OpenAI embeddings
  - Sub-nodes: Document loader → Text splitter → Vector store
  - Correct `ai_document` connection types throughout pipeline

- **Enhanced Error Handling**:
  - Error filtering and logging for failed URLs
  - Optional email notifications for processing failures
  - Graceful handling of document processing errors

- **Better Metadata Enhancement**:
  - Salesforce-specific categorization: `apex`, `lightning`, `lwc`, `soql`, `api`, `help`
  - Platform detection: `developer`, `lwc`, `design`, `help`
  - URL path and domain extraction for better searchability
  - Timestamp tracking for document ingestion

- **Optimized Configuration**:
  - Chunk size: 1500 characters with 200 character overlap
  - Embedding model: `text-embedding-3-small` (1536 dimensions)
  - Batch processing: 50 documents per embedding batch
  - Upsert mode for handling document updates

#### Technical Improvements

- **Connection Flow**:
  ```
  Schedule → URLs → Split → Document Loader 
                                    ↓ ai_document
                              Text Splitter
                                    ↓ ai_document  
                              Supabase Vector Store (Root)
                                    ↓ main
                              Metadata Enhancement → Progress Tracking
  ```

- **Error Branch**:
  ```
  Document Loader → Filter Errors → Log Errors → Email Notification
       ↓ main            ↑ main        ↑ main         ↑ main
  (on error)     (error items)  (processed)    (optional)
  ```

#### Dependencies

- **Required n8n Nodes**:
  - `@n8n/n8n-nodes-langchain.vectorStoreSupabase`
  - `@n8n/n8n-nodes-langchain.documentCheerioWebScraper`
  - `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`

- **Required Credentials**:
  - Supabase API (service role key)
  - OpenAI API (for embeddings)

- **Database Requirements**:
  - PostgreSQL with pgvector extension
  - `salesforce_docs` table with `match_documents` function

#### Breaking Changes

- **Node Type Changes**: All LangChain nodes now use proper `@n8n/n8n-nodes-langchain` types
- **Connection Types**: Document processing uses `ai_document` connections instead of `main`
- **Vector Store Architecture**: Now uses root node pattern instead of insert node
- **Credential Requirements**: Requires both Supabase and OpenAI credentials

#### Migration from Original

If upgrading from the original broken workflow:

1. **Delete old workflow** - cannot be directly upgraded due to architectural changes
2. **Import new workflow** from `workflows/salesforce-docs-auto-ingestion-fixed.json`
3. **Set up credentials** for Supabase and OpenAI
4. **Create database table** using provided SQL schema
5. **Test workflow** with manual execution before enabling schedule

#### Performance Improvements

- **Faster Processing**: Proper LangChain connections eliminate processing bottlenecks
- **Better Error Recovery**: Continue-on-fail prevents single URL failures from stopping entire workflow
- **Optimized Chunking**: Tiktoken-based length function for better token management
- **Batch Processing**: Configurable batch sizes for embedding generation

#### Documentation URLs Covered

- Salesforce Apex Documentation
- Lightning Platform Documentation  
- Lightning Web Components (LWC) Guide
- SOQL/SOSL Reference
- REST API Documentation
- LWC.dev Guides
- Lightning Design System Components
- Salesforce Help Articles

---

## How to Read This Changelog

- **Major Version** (X.0.0): Breaking changes, architecture overhauls
- **Minor Version** (0.X.0): New features, enhanced functionality
- **Patch Version** (0.0.X): Bug fixes, small improvements

## Contributing

When contributing changes, please:
1. Update this changelog with your modifications
2. Follow semantic versioning principles
3. Include both technical details and user-facing impact
4. Test workflow thoroughly before releasing
