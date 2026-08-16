# AzureChatWithDocs

A dev-only, cost-conscious RAG (retrieval-augmented generation) chatbot, built
incrementally as a hands-on study project for the **AI-200: Develop AI Solutions
in Microsoft Azure** exam.

The idea: take a small mock corpus of HTML documents, chunk and embed them, store the
vectors in Azure Cosmos DB, and query them from a simple chat UI — while covering the
same ground as these Microsoft Learn paths:

- [Develop AI Solutions with Azure Cosmos DB](https://learn.microsoft.com/en-us/training/paths/develop-ai-solutions-azure-cosmos-db/)
- [Develop AI Solutions with Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/training/paths/develop-ai-solutions-azure-database-postgresql/)
- [Implement Container App Hosting on Azure](https://learn.microsoft.com/en-us/training/paths/implement-container-app-hosting-azure/)

## How this project is built

This is being built one phase at a time as a teaching exercise, not generated in one
shot — see [`docs/roadmap.md`](docs/roadmap.md) for the phase-by-phase plan and current
status, and [`docs/plans/semantic-search-learning-project.md`](docs/plans/semantic-search-learning-project.md)
for the original plan and key architecture decisions.

## Repo layout

```
content/        Mock HTML corpus (the "documents" we chunk and index)
src/            .NET solution
  Shared/       Chunking models/logic shared by Ingestion and Api
  Ingestion/    Console app: chunk content, generate embeddings, upsert to Cosmos DB
  Api/          ASP.NET Core minimal API: chat endpoint + static chat UI (wwwroot)
infra/          Bicep infrastructure-as-code (added in later phases)
.github/        GitHub Actions CI/CD (added in later phases)
docs/           Roadmap and plan docs
```

## Running it

Filled in as each phase lands. For now:

```
dotnet build src/AzureChatWithDocs.sln
```

## Cost posture

This is a dev-only learning project. Wherever Azure gives us a choice, we pick the
cheapest/serverless/burstable tier, and we get in the habit of tearing resources down
(`azd down` or `az group delete`) when we're not actively using them.
