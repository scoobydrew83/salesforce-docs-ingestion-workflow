# Setup Guide: Salesforce Documentation Auto-Ingestion

This guide walks you through setting up the corrected Salesforce documentation auto-ingestion workflow.

## Prerequisites

### Required Services
- **n8n instance** (v1.0+ with LangChain nodes)
- **Supabase account** with PostgreSQL database
- **OpenAI account** with API access

### Required n8n Nodes
Ensure these LangChain nodes are available in your n8n instance:
- `@n8n/n8n-nodes-langchain.vectorStoreSupabase`
- `@n8n/n8n-nodes-langchain.documentCheerioWebScraper`
- `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`

## Step 1: Database Setup

### 1.1 Enable pgvector Extension

In your Supabase SQL editor, run:

```sql
-- Enable the pgvector extension to work with embedding vectors
create extension if not exists vector;
```

### 1.2 Create Documents Table

```sql
-- Create a table to store your documents
create table salesforce_docs (
  id bigserial primary key,
  content text, -- corresponds to Document.pageContent
  metadata jsonb, -- corresponds to Document.metadata
  embedding vector(1536) -- 1536 works for OpenAI embeddings, change if needed
);

-- Create an index for better performance
create index on salesforce_docs using ivfflat (embedding vector_cosine_ops)
with (lists = 100);
```

### 1.3 Create Search Function

```sql
-- Create a function to search for documents
create or replace function match_documents (
  query_embedding vector(1536),
  match_count int default null,
  filter jsonb DEFAULT '{}'
) returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
#variable_conflict use_column
begin
  return query
  select
    salesforce_docs.id,
    salesforce_docs.content,
    salesforce_docs.metadata,
    1 - (salesforce_docs.embedding <=> query_embedding) as similarity
  from salesforce_docs
  where metadata @> filter
  order by salesforce_docs.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

### 1.4 Verify Setup

```sql
-- Test the function
select match_documents(
  '[0,0,0]'::vector, -- dummy vector for testing
  1,
  '{}'
);
```

## Step 2: Credential Configuration

### 2.1 Supabase API Credentials

1. Go to your Supabase project dashboard
2. Navigate to **Settings** → **API**
3. Copy the following:
   - **Project URL** (e.g., `https://your-project.supabase.co`)
   - **Service Role Key** (NOT the anon key - service role has full access)

4. In n8n, create new credentials:
   - **Credential Type**: Supabase API
   - **Name**: `supabase_credentials`
   - **Host**: Your project URL
   - **Service Role Key**: Your service role key

### 2.2 OpenAI API Credentials

1. Go to [OpenAI API Keys](https://platform.openai.com/api-keys)
2. Create a new API key
3. In n8n, create new credentials:
   - **Credential Type**: OpenAI API
   - **Name**: `openai_credentials`
   - **API Key**: Your OpenAI API key

## Step 3: Workflow Import

### 3.1 Download Workflow

1. Download the workflow file:
   ```bash
   curl -O https://raw.githubusercontent.com/scoobydrew83/salesforce-docs-ingestion-workflow/main/workflows/salesforce-docs-auto-ingestion-fixed.json
   ```

### 3.2 Import to n8n

1. In n8n, click the **menu (☰)** → **Import workflow**
2. Select **"From file"**
3. Choose the downloaded JSON file
4. Click **"Import"**

### 3.3 Link Credentials

1. Open the **"Supabase Vector Store (Root Node)"**
2. In the **Supabase API** field, select `supabase_credentials`
3. In the **OpenAI API** field, select `openai_credentials`
4. Click **"Save"**

## Step 4: Configuration

### 4.1 Customize Documentation URLs (Optional)

1. Open the **"Documentation URLs"** node
2. Modify the `urls` array to add/remove Salesforce documentation URLs:
   ```json
   [
     "https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/",
     "https://developer.salesforce.com/docs/atlas.en-us.lightning.meta/lightning/",
     "https://your-custom-docs-url.com"
   ]
   ```

### 4.2 Configure Error Notifications (Optional)

1. Open the **"Send Error Notification"** node
2. Enable the node (currently disabled)
3. Configure your email settings:
   - **To Email**: Your admin email
   - **SMTP Settings**: Configure in n8n credentials

### 4.3 Adjust Processing Settings (Optional)

**Text Splitter Settings:**
- **Chunk Size**: 1500 characters (default)
- **Chunk Overlap**: 200 characters (default)
- **Length Function**: tiktoken (recommended for OpenAI)

**Embedding Settings:**
- **Model**: `text-embedding-3-small`
- **Dimensions**: 1536
- **Batch Size**: 50 documents per batch

**Vector Store Settings:**
- **Table Name**: `salesforce_docs`
- **Query Name**: `match_documents`
- **Upsert**: `true` (allows updates)

## Step 5: Testing

### 5.1 Manual Test Run

1. Click **"Execute Workflow"** to run manually
2. Monitor the execution in real-time
3. Check for any errors in the error handling branch

### 5.2 Verify Data Ingestion

In Supabase SQL editor:

```sql
-- Check if documents were inserted
select 
  count(*) as total_documents,
  count(distinct metadata->>'doc_type') as doc_types,
  count(distinct metadata->>'platform') as platforms
from salesforce_docs;

-- View sample documents
select 
  id,
  substring(content, 1, 100) as content_preview,
  metadata->>'doc_type' as doc_type,
  metadata->>'platform' as platform,
  metadata->>'url_path' as url_path
from salesforce_docs 
limit 5;
```

### 5.3 Test Search Function

```sql
-- Test similarity search
select 
  id,
  substring(content, 1, 200) as content_preview,
  metadata->>'doc_type' as doc_type,
  similarity
from match_documents(
  -- You'll need an actual embedding vector here
  (select embedding from salesforce_docs limit 1),
  3,
  '{}'
);
```

## Step 6: Schedule Activation

### 6.1 Enable Scheduled Execution

1. In the workflow, click **"Activate"** (toggle in top-right)
2. The workflow will now run daily at midnight UTC

### 6.2 Modify Schedule (Optional)

1. Open the **"Daily Schedule Trigger"** node
2. Modify the interval:
   - **Field**: days, hours, minutes
   - **Interval**: frequency

## Step 7: Monitoring & Maintenance

### 7.1 Monitor Executions

1. Go to **Executions** tab in n8n
2. Check recent runs for success/failure status
3. Review error logs if needed

### 7.2 Database Maintenance

```sql
-- Clean up old documents (optional)
delete from salesforce_docs 
where (metadata->>'scraped_at')::timestamp < now() - interval '30 days';

-- Reindex for better performance (occasionally)
reindex index salesforce_docs_embedding_idx;
```

### 7.3 Performance Tuning

**If processing is slow:**
- Reduce batch size in embeddings
- Increase chunk size in text splitter
- Add more specific CSS selectors in document loader

**If vector search is slow:**
- Adjust ivfflat index parameters
- Consider using HNSW index for larger datasets
- Optimize match_documents function

## Troubleshooting

See the main [README.md](../README.md#troubleshooting) for common issues and solutions.

## Next Steps

Once your workflow is running successfully:

1. **Build Query Interface**: Create a frontend to search your documentation
2. **Add More Sources**: Extend URLs to include more Salesforce resources  
3. **Implement RAG**: Use the vector store for Retrieval-Augmented Generation
4. **Set up Monitoring**: Add webhooks or notifications for execution status

Your Salesforce documentation knowledge base is now ready! 🚀
