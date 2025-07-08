# Agent_AI
version: 1.0
name: get_answer_graph

description: |
  This LangGraph handles the /getAnswer POST endpoint. It routes the request to a query-based or document-based skill
  based on whether files are uploaded. It includes memory integration based on conversationId and updates memory after each interaction.

inputs:
  - conversationId
  - questionUid
  - skillId
  - input:
      userQuery: string         # The main query from the user
      uploadedFiles: list       # List of uploaded documents (optional)

start: memory_node

end:
  - memory_update

nodes:
  memory_node:
    type: memory
    id: memory_node
    scope: conversation
    input:
      - conversationId

  router:
    type: router
    input:
      - input
    routes:
      - condition: "len(input.uploadedFiles) > 0"
        next: document_agent
      - condition: "len(input.uploadedFiles) == 0"
        next: query_agent

  query_agent:
    type: llm
    input:
      - input.userQuery
      - memory_node.history
    output: answer
    prompt_template: |
      You are a helpful assistant having a conversation with a user.

      Conversation so far:
      {memory_node.history}

      Current question:
      {input.userQuery}

      Answer:

  document_agent:
    type: llm
    input:
      - input.userQuery
      - input.uploadedFiles
      - memory_node.history
    output: answer
    prompt_template: |
      You are an assistant answering questions using only the uploaded documents.
      Use only the content from the files and provide references in your answer.

      Conversation so far:
      {memory_node.history}

      Uploaded Files:
      {input.uploadedFiles}

      Current question:
      {input.userQuery}

      Answer (with references):

  memory_update:
    type: memory.update
    input:
      - conversationId
      - input.userQuery
      - answer
    data_template: |
      User: {input.userQuery}
      Assistant: {answer}

edges:
  - from: memory_node
    to: router
  - from: router
    to: query_agent
    condition: "len(input.uploadedFiles) == 0"
  - from: router
    to: document_agent
    condition: "len(input.uploadedFiles) > 0"
  - from: query_agent
    to: memory_update
  - from: document_agent
    to: memory_update

output:
  - conversationId
  - questionUid
  - skillId
  - answer
