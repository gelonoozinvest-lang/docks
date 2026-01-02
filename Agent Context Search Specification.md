Agent Context Search Specification
1. Purpose
This document defines the specification for how AI agents in the Business Factory search for and use context to complete their tasks. This includes querying the RAG service, using dialogue history, and accessing data from previous steps in a workflow.

2. Context Sources
AI agents can use the following sources to build context:

2.1. RAG Service (Knowledge Base)
The primary source of context is the RAG service, which provides access to a vast knowledge base of documents, data, and other information. Agents can query the RAG service to get relevant information for a given task.

2.2. Dialogue History
For conversational agents, the history of the conversation with the user is a crucial source of context. The dialogue history helps the agent to understand the user's intent and to provide more relevant responses.

2.3. Data from Previous Steps (n8n)
In n8n workflows, agents can access the output of previous steps. This allows agents to chain together multiple operations and to build complex workflows.

3. Context Search Process
The process of searching for and using context involves the following steps:

3.1. Forming a Query to the RAG Service
The agent forms a query to the RAG service based on the current task and the available context. The query should be as specific as possible to get the most relevant results.

3.2. Filtering the Results
The agent can filter the results from the RAG service based on various criteria, such as the source of the information, the date of creation, and the relevance to the current task.

3.3. Using the Context in the Prompt for the LLM
The agent uses the retrieved context to construct a prompt for the Large Language Model (LLM). The prompt should include the original query, the retrieved context, and any other relevant information.

4. Examples
4.1. Example Query to the RAG Service
To get information about a specific customer, an agent might send the following query to the RAG service:
```json
{
  "query": "Get all information about customer with ID 12345",
  "filters": {
    "collection": "customers"
  }
}
```

4.2. Example of Using Context in a Prompt
An agent might use the following prompt to generate a response to a customer query:
```
The customer is asking the following question: "{customer_query}"

Here is some information about the customer from our knowledge base:
{rag_context}

Based on this information, please generate a response to the customer's question.
```
