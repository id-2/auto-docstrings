# ✨ LangChain.js Auto TSDoc Comment Creator

This repo uses [LangChain.js](https://js.langchain.com/) to automatically generate annotated TSDoc
comments for declared methods, classes, interfaces, types, and functions in the [LangChain.js repo](https://github.com/hwchase17/langchainjs).

![Sample TSDoc comment shown in Intellisense in VSCode](/public/images/intellisense_tooltip.png)

TSDoc comments are invaluable in helping developers understand code at the interface level, giving insight into
intricacies, parameters, and return types without needing to check the actual implementation. The comments show
up in LangChain's autogenerated API reference pages as well as inline Intellisense tooltips, and the provided context
helps developers save time and avoid frustrating bugs.

## 🔨 Usage

Install packages, then create a `.env` file with the keys specified in `.env.example`.

Then run `yarn start path/to/folder` to recursively run the script on all non-test `.ts` files in the folder.

## 🔎 How it works

As a popular open-source project that includes many hyper-specific technical concepts and integrations more recent than the knowledge
cutoff dates for popular models, auto-generating these comments for LangChain.js presented some unique issues. After some
experimentation, I settled on the following high-level flow for each file:

1. Pass the file contents into [retrieval-focused agent](https://js.langchain.com/docs/use_cases/question_answering/conversational_retrieval_agents) prompted specifically to define the LangChain-specific terms mentioned in the code for a technical writer. The agent used an existing [Weaviate](https://js.langchain.com/docs/modules/data_connection/vectorstores/integrations/weaviate) index we previously created over our Python documentation, which should mostly be a superset of the JS integrations and abstractions.
2. Use an OpenAI Functions model to take the generated context, as well as the original code, to output suggested comments, the name of the declaration they apply to, and metadata in a structured format.
3. For each generated comment, use TypeScript's abstract syntax tree (AST) parsing APIs to identify the position of the declaration in the code.
4. Create the actual comment string using the metadata from step 2.
5. Splice it into the file and write it back to the filesystem.

As far as prompting, giving the agent an example workflow and generally few-shotting the prompts with examples of good outputs seemed to help get
results in a good state.

All steps use OpenAI's GPT-4.

Here's a link to an example LangSmith trace demonstrating this sequence:

https://smith.langchain.com/public/24456a63-4af0-48b6-9e1a-edd2dd0087f4/r

## 📋 Results

[The resulting PR](https://github.com/hwchase17/langchainjs/pull/2341) from running this script on the repo encompassed 280 files!

Some changes were required after the initial pass as roughly 10 generated `@returns` annotations resulted in TSDoc comments incompatible with
our automatically generated Docusaurus API reference docs, but overall quality seemed high. Here are some of the few I was impressed by:

A good explanation with parameters of the convenience method to create an OpenAI Functions-powered extraction chain:

```typescript
/**
 * Function that creates an extraction chain using the provided JSON schema.
 * It sets up the necessary components, such as the prompt, output parser, and tags.
 * @param schema JSON schema of the function parameters.
 * @param llm Must be a ChatOpenAI model that supports function calling.
 * @returns A LLMChain instance configured to return data matching the schema.
 */
export function createExtractionChain(
  schema: FunctionParameters,
  llm: ChatOpenAI
) {
  ...
}
```

A nice explanation of a highly-specific and technical retriever:

```typescript
/**
 * A class for retrieving relevant documents based on a given query. It
 * extends the VectorStoreRetriever class and uses a BaseLanguageModel to
 * generate a hypothetical answer to the query, which is then used to
 * retrieve relevant documents.
 */
export class HydeRetriever<
  V extends VectorStore = VectorStore
> extends VectorStoreRetriever<V> {
  ...
}
```

Adding caveats to the popular `BufferMemory` class:

```typescript
/**
 * The `BufferMemory` class is a type of memory component used for storing
 * and managing previous chat messages. It is a wrapper around
 * `ChatMessageHistory` that extracts the messages into an input variable.
 * This class is particularly useful in applications like chatbots where
 * it is essential to remember previous interactions. Note: The memory
 * instance represents the history of a single conversation. Therefore, it
 * is not recommended to share the same history or memory instance between
 * two different chains. If you deploy your LangChain app on a serverless
 * environment, do not store memory instances in a variable, as your
 * hosting provider may reset it by the next time the function is called.
 */
export class BufferMemory extends BaseChatMemory implements BufferMemoryInput {
  ...
}
```

A nice warning on an unused Vectara vector store method (they provide their own embeddings, and don't support adding vectors directly):

```typescript
  /**
   * Throws an error, as this method is not implemented. Use addDocuments
   * instead.
   * @param _vectors Not used.
   * @param _documents Not used.
   * @returns Does not return a value.
   */
  async addVectors(
    _vectors: number[][],
    _documents: Document[]
  ): Promise<void> {
    throw new Error(
      "Method not implemented. Please call addDocuments instead."
    );
  }
```

## 🧪 Considerations and Tradeoffs

### Retrieval Agent vs. Retrieval Chain

I considered using a [simpler retrieval chain](https://js.langchain.com/docs/modules/chains/popular/vector_db_qa) rather than a more
advanced retrieval agent as I generally knew that the vector store should retrieve information on the main class mentioned in the
code.

However, because many classes in LangChain.js extend base classes and import other concepts, I chose to try to prompt
the agent to look up information on the main class from the retriever, then have the flexibility to decide if it needed
more information and make further queries.

### Structured Output vs. Unstructured

The first naive approach I took was to try to just have the model take input code and rewrite it with TSDoc comments.
However, I ran into issues where the model would overwrite handwritten existing TSDoc comments.

I then thought to structure output and splice the comments in as a final step. I initially tried to simply ask the model to output a line
number where the comment should be spliced, but again ran into hallucination issues.

I then decided if I made each item in the returned array also include the name of the function and type of declaration, and then make
a final pass that finds the correct place for the comment in the code using the AST approach described above.

### Call Per Desired TSDoc Comment vs. One Pass

For large files, first extracting applicable declarations for TSDoc comments, then writing the comments may improve quality approach.

However, it would be more token intensive and slower, and structuring my OpenAI schema to be an array gave satisfactory results.

### Tokens

Running this repo over all the code in LangChain.js took millions of tokens.

During development, I added some filtering steps to try to only perform the LLM-powered steps on non-test TypeScript files that
have at least one uncommented declaration that would be a good candidate for a TSDoc comment.

# 🙏 Thank you!

Thanks for reading! I hope you can extract (haha) some of the techniques mentioned here and apply them to your own projects.

For more, follow me on X (formerly Twitter) [@Hacubu](https://x.com/hacubu) as well as the official [LangChain account](https://x.com/langchainai).
