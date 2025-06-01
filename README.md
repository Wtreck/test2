spring:
  ai:
    # (CoBa) Azure OpenAI configuration
    # ===============================
    azure:
      openai:
        endpoint: "https://api-int-dev.intranet.commerzbank.com:8065/utilities-api/azure-openai/v1"
        chat:
          options:
            deployment-name: "gpt4o20240806"
            max-tokens: 2000
            temperature: 0.7
            top-p: 0.95
        embedding:
          options:
            deployment-name: "teada0022"
    vectorstore:
      weaviate:
        host: frame-docs-store.entw.apps.cloud.internal
  security:
    oauth2:
      # oauth2 client configuration to access CoBa Azure OpenAI API (internal LLM Services API)

frame:
  ai:
    # FRAME (CoBa) Azure OpenAI configuration
    # ===============================
    azure:
      openai:
        service-version: V2024_06_01
        max-retries: 5
        base-delay: 100
        max-delay: 1000
        http-client:
          connect-timeout: 250
          call-timeout: 10


   package com.commerzbank.frame.ai.etl.business.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.ai.document.Document;
import org.springframework.ai.reader.TextReader;
import org.springframework.ai.transformer.splitter.TextSplitter;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.core.io.PathResource;
import org.springframework.stereotype.Service;

import java.nio.file.Path;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

/**
 * Service creating and maintaining the embeddings for the Markdown documents comprising FRAME's
 * documentation, excluding images, PDFs, and other media.
 * <p>
 * As the Markdown files' source is the FRAME documentation Git repository, this service uses a
 * {@link GitRepoAccessor} to clone the repository and read the Markdown files, before Extracting,
 * Transforming, and Loading the documents into the vector store using an ETL pipeline.
 *
 * @author cb2ro7y - Leon Rosche
 * @author cb2strc - Silvana Sadewasser
 */
@Service
public class EmbeddingsService {
	/** Logger. */
	private static final Logger LOG = LoggerFactory.getLogger(EmbeddingsService.class);

	/** The accessor to the Git repository containing the FRAME documentation. */
	private final GitRepoAccessor gitRepoAccessor;

	/** The vector store where the embedded documents are stored. */
	private final VectorStore store;

	/**
	 * The document transformer for splitting the {@link Document}s into appropriate chunks for
	 * embedding.
	 * <p>
	 * The splitter is parameterized especially for this sample use case, i.e. for the embedding of
	 * FRAME's Markdown documents. In other use cases, the parameters are likely to be different
	 * and should be adjusted accordingly.
	 */
	private final TextSplitter splitter = new TokenTextSplitter(6000, 50, 50, 10000, true);

	public EmbeddingsService(VectorStore store, GitRepoAccessor gitRepoAccessor) {
		this.store = store;
		this.gitRepoAccessor = gitRepoAccessor;
	}

	/**
	 * Embeds the FRAME documentation's Markdown documents into the associated vector store.
	 * <p>
	 * The Markdown files are cloned from the FRAME documentation repository to a directory. Then,
	 * they are read from this clone directory, parsed into {@link Document}s, and split into
	 * smaller chunks. Each chunk resp. new document is enriched with metadata, i.e. the path to
	 * the Markdown file and the creation timestamp of the document, before being loaded into the
	 * vector store.
	 */
	public void embed() {
		gitRepoAccessor.deleteCloneDir();
		gitRepoAccessor.cloneRepository();

		final var documents = new ArrayList<Document>();
		for (var markdown : gitRepoAccessor.loadFiles()) {
			final var chunks = transform(extract(markdown));
			documents.addAll(chunks);
		}
		store.accept(documents);
		LOG.info("Loaded in total {} chunked documents into vector store", documents.size());
	}

	/**
	 * Extracts a {@link Document} from the given Markdown file of the FRAME documentation using an
	 * instance of {@link TextReader} instead of Spring AI's 'MarkdownDocumentReader', to ensure
	 * that the document is split into appropriate chunks for embedding. It adds the relative path
	 * to the Markdown file as metadata to the document and thus to all chunks created from it.
	 *
	 * @param markdown The path to the Markdown source file to be processed.
	 * @return The extracted {@link Document} with metadata.
	 */
	private Document extract(Path markdown) {
		final var reader = new TextReader(new PathResource(markdown));
		reader.getCustomMetadata().put("path",
			gitRepoAccessor.getRootDir().relativize(markdown).toString());
		final var document = reader.read().get(0);
		LOG.info("Extracted document with meta data '{}' from file '{}'", document.getMetadata(),
			markdown);
		return document;
	}

	/**
	 * Transforms the given {@link Document} into smaller chunks using the {@link TokenTextSplitter}
	 * and adds the chunk's creation timestamp as metadata to each chunk.
	 *
	 * @param document The original {@link Document} to be transformed into suitable chunks for
	 *                 embedding.
	 * @return A list of transformed {@link Document}s, each representing a chunk of the original
	 * document.
	 */
	private List<Document> transform(Document document) {
		var chunks = new ArrayList<Document>();
		var chunkCount = 0;
		for (var chunk : splitter.split(document)) {
			chunk.getMetadata().put("creation_timestamp", Instant.now().toEpochMilli());
			LOG.debug("Created chunk with meta data '{}'", chunk.getMetadata());
			chunks.add(chunk);
			chunkCount++;
		}
		LOG.info("Transformed document with meta data '{}' into {} chunks", document.getMetadata(),
			chunkCount);
		return chunks;
	}
}
