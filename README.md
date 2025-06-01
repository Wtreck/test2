Unten findest du ein vollständiges Beispielprojekt mit allen relevanten Dateien und ihren Pfaden, die du in Deinem Fall benötigst. Die wichtigsten Bestandteile sind:
	1.	Spring Boot Backend
	•	application.yml (Konfiguration für Weaviate und Spring AI)
	•	CobaAzureOpenAiRagConfiguration.java (manuelle Konfiguration des VectorStore/Weaviate-Connectors)
	•	EmbeddingsService.java (Erzeugung und Einfügen der Dokumente ins VectorStore)
	•	DeletionService.java (Lösch-Service, der alle Dokumente entfernt)
	•	EmbeddingController.java (Controller-Endpunkt für „DELETE ALL“)
	•	EmbeddingApplication.java (Hauptklasse mit @SpringBootApplication)
	2.	React Frontend
	•	Deletion.jsx bzw. Deletion.tsx (React-Komponente, die auf /embedding/delete-all zugreift)
	•	ggf. übersetzungsrelevante Keys in der i18n-Datei (z. B. public/locales/de/translation.json)

Projektstruktur

my-weaviate-app/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/embedding/
│   │   │       ├── EmbeddingApplication.java
│   │   │       ├── configuration/
│   │   │       │   └── CobaAzureOpenAiRagConfiguration.java
│   │   │       ├── controller/
│   │   │       │   └── EmbeddingController.java
│   │   │       ├── service/
│   │   │       │   ├── EmbeddingsService.java
│   │   │       │   └── DeletionService.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/ … (optional)
└── frontend/
    ├── package.json
    └── src/
        ├── components/
        │   └── Deletion.jsx
        ├── i18n/
        │   └── de.json
        └── shared/
            └── app-routing.js

1. pom.xml

<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>my-weaviate-app</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>my-weaviate-app</name>
    <properties>
        <java.version>17</java.version>
        <spring.boot.version>3.4.2</spring.boot.version>
        <spring.ai.version>1.0.0</spring.ai.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring.boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Spring Boot Web (REST-Endpunkte) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring AI Core -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-core</artifactId>
            <version>${spring.ai.version}</version>
        </dependency>

        <!-- Spring AI Auto-Configure für Weaviate -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-autoconfigure-vector-store-weaviate</artifactId>
            <version>${spring.ai.version}</version>
        </dependency>

        <!-- Weaviate Java Client (wird transitiv vom Weaviate-Starter gezogen) -->
        <!-- (Optional: Du kannst hier explizit definieren, wenn nötig) -->
        <!--
        <dependency>
            <groupId>io.weaviate</groupId>
            <artifactId>weaviate-java-client</artifactId>
            <version>3.0.0</version>
        </dependency>
        -->

        <!-- SLF4J Logging (Beispiel binding) -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>2.0.9</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Compiler-Plugin -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>


⸻

2. src/main/resources/application.yml

spring:
  ai:
    vectorstore:
      weaviate:
        # 1. Vollständige Weaviate-URL (inkl. /v1)
        endpoint-url: "https://frame-docs-store.entw.apps.cloud.internal/v1"

        # 2. (Optional) API-Key, falls Weaviate abgesichert ist
        # api-key: ${WEAVIATE_API_KEY}

        # 3. Name der Weaviate-Klasse, in der du deine Embeddings speicherst
        object-class: MeineDokumente

# Hier folgen deine anderen spring.ai.azure.openai-Parameter, wie im ursprünglichen YAML
spring:
  ai:
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

# FRAME-spezifische Azure-OpenAI-Konfiguration (bleibt unverändert)
frame:
  ai:
    azure:
      openai:
        service-version: V2024_06_01
        max-retries: 5
        base-delay: 100
        max-delay: 1000
        http-client:
          connect-timeout: 250
          call-timeout: 10

Wichtig:
	•	Hier befinden sich nur die Properties, die Spring AI für den Weaviate-Connector braucht:
	•	endpoint-url (inkl. /v1)
	•	(Optional) api-key
	•	object-class
	•	Kein filter-metadata-fields, weil wir das in der Java-Konfiguration per Builder erledigen.
	•	Achte darauf, dass endpoint-url tatsächlich mit "https://..." inklusive /v1 angegeben ist und object-class exakt dem Namen der Weaviate-Klasse entspricht.

⸻

3. src/main/java/com/example/embedding/EmbeddingApplication.java

package com.example.embedding;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class EmbeddingApplication {
    public static void main(String[] args) {
        SpringApplication.run(EmbeddingApplication.class, args);
    }
}


⸻

4. src/main/java/com/example/embedding/configuration/CobaAzureOpenAiRagConfiguration.java

In dieser Klasse baust du manuell den WeaviateVectorStore auf und teilst Spring AI mit, welche Metadaten-Felder gefiltert werden dürfen.

package com.example.embedding.configuration;

import io.weaviate.client.WeaviateClient;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.weaviate.WeaviateVectorStore;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

/**
 * Konfiguriert den Weaviate VectorStore manuell, damit wir die Metadatenfelder
 * (charset, source, path, creation_timestamp) setzen können.
 */
@Configuration
public class CobaAzureOpenAiRagConfiguration {

    /**
     * Erzeugt den WeaviateVectorStore mit eingebautem WeaviateClient und dem EmbeddingModel.
     * Wir geben manuell an, welche Metadaten-Felder filterbar sind.
     */
    @Bean
    public VectorStore vectorStore(WeaviateClient weaviateClient,
                                   EmbeddingModel embeddingModel) {
        return WeaviateVectorStore.builder(weaviateClient, embeddingModel)
                // Konsistenzlevel (optional nach Bedarf)
                .consistencyLevel(WeaviateVectorStore.ConsistencyLevel.ONE)
                // Filterbare Metadatenfelder: charset, source, path, creation_timestamp
                .filterMetadataFields(List.of(
                    WeaviateVectorStore.MetadataField.text("charset"),
                    WeaviateVectorStore.MetadataField.text("source"),
                    WeaviateVectorStore.MetadataField.text("path"),
                    WeaviateVectorStore.MetadataField.number("creation_timestamp")
                ))
                .build();
    }
}

Erläuterung:
	•	WeaviateVectorStore.builder(...) erhält den automatisch von Spring AI erzeugten WeaviateClient und dein EmbeddingModel.
	•	Mit .filterMetadataFields(List.of(...)) meldest du Spring AI, dass jedes Dokument/Chunk genau diese vier Metadaten-Felder besitzt und künftig filterbar sind:
	1.	"charset" (Typ Text)
	2.	"source" (Typ Text)
	3.	"path" (Typ Text)
	4.	"creation_timestamp" (Typ Number)
	•	Diese Felder werden später beim vectorStore.delete("path != ''") oder ähnlichem verwendet.

⸻

5. src/main/java/com/example/embedding/service/EmbeddingsService.java

Diese Klasse erstellt deine Embeddings, indem sie die Markdown-Dateien klont, ausliest, aufteilt und anschließend in Weaviate einfügt.

package com.example.embedding.service;

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
 * Service, der die FRAME-Markdown-Dokumentation klont, in Chunks splittet,
 * Metadaten hinzufügt und in den Weaviate VectorStore einfügt.
 */
@Service
public class EmbeddingsService {
    private static final Logger LOG = LoggerFactory.getLogger(EmbeddingsService.class);

    private final GitRepoAccessor gitRepoAccessor;
    private final VectorStore store;

    // TextSplitter, der lange Text in kleinere Chunks unterteilt
    private final TextSplitter splitter = new TokenTextSplitter(6000, 50, 50, 10000, true);

    public EmbeddingsService(VectorStore store, GitRepoAccessor gitRepoAccessor) {
        this.store = store;
        this.gitRepoAccessor = gitRepoAccessor;
    }

    /**
     * Klont das Git-Repo, liest Markdown-Dateien, splittet sie und lädt die Chunks
     * mit ihren Metadaten in Weaviate.
     */
    public void embed() {
        // 1. Vorherige Clone-Daten löschen (lokal)
        gitRepoAccessor.deleteCloneDir();
        // 2. Git-Repo klonen
        gitRepoAccessor.cloneRepository();

        // 3. Alle Markdown-Dateien durchgehen, extrahieren, splitten und in Weaviate einfügen
        List<Document> allChunks = new ArrayList<>();
        for (Path markdown : gitRepoAccessor.loadFiles()) {
            Document extracted = extract(markdown);
            List<Document> chunks = transform(extracted);
            allChunks.addAll(chunks);
        }

        // 4. Alle Chunks im VectorStore speichern
        store.accept(allChunks);
        LOG.info("Loaded in total {} chunked documents into vector store", allChunks.size());
    }

    /**
     * Liest eine Markdown-Datei, setzt den relativen Pfad als Metadaten-Feld "path" und
     * gibt ein einzelnes Document-Objekt zurück.
     */
    private Document extract(Path markdown) {
        TextReader reader = new TextReader(new PathResource(markdown));
        reader.getCustomMetadata().put("path",
            gitRepoAccessor.getRootDir().relativize(markdown).toString());
        // Der TextReader füllt automatisch "charset" und "source" in die Metadaten
        Document doc = reader.read().get(0);
        LOG.info("Extracted document with meta data '{}' from file '{}'",
                 doc.getMetadata(), markdown);
        return doc;
    }

    /**
     * Splittet das gegebene Document in Chunks und fügt jedem Chunk das Metadaten-Feld
     * "creation_timestamp" hinzu.
     */
    private List<Document> transform(Document document) {
        List<Document> chunks = new ArrayList<>();
        int chunkCount = 0;
        for (Document chunk : splitter.split(document)) {
            chunk.getMetadata().put("creation_timestamp", Instant.now().toEpochMilli());
            LOG.debug("Created chunk with meta data '{}'", chunk.getMetadata());
            chunks.add(chunk);
            chunkCount++;
        }
        LOG.info("Transformed document with meta data '{}' into {} chunks",
                 document.getMetadata(), chunkCount);
        return chunks;
    }
}

Dateiname & Pfad:
src/main/java/com/example/embedding/service/EmbeddingsService.java

Erklärung Metadaten:
	•	Beim Extract-Schritt (reader.getCustomMetadata().put("path", …)) in jedes Dokument (Document) wird automatisch auch das Feld "charset" (TextReader übernimmt das) und "source" (TextReader) gefüllt.
	•	Im Transform-Schritt fügst du per chunk.getMetadata().put("creation_timestamp", …) das vierte Metadaten-Feld hinzu.

⸻

6. src/main/java/com/example/embedding/service/DeletionService.java

Dieser Service löscht alle Dokumente, indem er den String-Filter "path != ''" nutzt. Da jedes Chunk genau das Metadaten-Feld "path" gesetzt hat, gilt dieser Ausdruck für alle Dokumente.

package com.example.embedding.service;

import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class DeletionService {

    private final VectorStore vectorStore;

    public DeletionService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    /**
     * Löscht alle Dokumente in der Weaviate-Klasse, indem der String-Filter
     * "path != ''" verwendet wird. Da jedes Chunk ein nicht-leeres "path" enthält,
     * trifft dieser Filter auf alle Dokumente zu und löscht sie.
     */
    @Transactional
    public void deleteAll() {
        vectorStore.delete("path != ''");
    }
}

Dateiname & Pfad:
src/main/java/com/example/embedding/service/DeletionService.java

⸻

7. src/main/java/com/example/embedding/controller/EmbeddingController.java

Der Controller bietet den Endpunkt POST /embedding/delete-all an, der den Lösch-Service aufruft und eine minimale JSON-Antwort zurückgibt. Deine React-Komponente erwartet genau { message: string }.

package com.example.embedding.controller;

import com.example.embedding.service.DeletionService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/embedding")
public class EmbeddingController {

    private final DeletionService deletionService;

    public EmbeddingController(DeletionService deletionService) {
        this.deletionService = deletionService;
    }

    /**
     * POST /embedding/delete-all
     * Löscht alle Dokumente per Filter "path != ''".
     * Gibt JSON { "message": "Alle Einträge wurden gelöscht." } zurück.
     */
    @PostMapping("/delete-all")
    public ResponseEntity<Map<String, String>> deleteAll() {
        deletionService.deleteAll();
        return ResponseEntity.ok(Map.of("message", "Alle Einträge wurden gelöscht."));
    }
}

Dateiname & Pfad:
src/main/java/com/example/embedding/controller/EmbeddingController.java

⸻

8. React-Frontend

Lege folgenden Komponenten-Code in deinem Frontend-Ordner an, z. B. frontend/src/components/Deletion.jsx. Er ruft beim Klick POST /embedding/delete-all auf und zeigt die Erfolgsmeldung an.

// Dateiname: frontend/src/components/Deletion.jsx

import React, { useState, useContext } from 'react';
import { Headline, Paragraph, Button } from '@lsg/components';
import { routeStrategy } from '../../shared/app-routing';
import { ErrorContext } from '../../context/error.context.provider';
import ErrorMessage from '../../service/error.message';
import { useTranslation } from 'react-i18next';

export const Deletion = () => {
    const { t } = useTranslation();
    const errorContext = useContext(ErrorContext);
    const [loading, setLoading] = useState(false);
    const [result, setResult] = useState('');

    const handleDeleteAll = async () => {
        setLoading(true);
        setResult('');
        try {
            const route = await routeStrategy();
            const response = await fetch(route.api('/embedding/delete-all'), {
                method: 'POST',
                credentials: 'include',
            });

            if (!response.ok) {
                const err = await response.json();
                throw Object.assign(
                    new Error(err.error || `HTTP ${response.status}`),
                    { status: response.status, trace: err.trace }
                );
            }
            const data = await response.json();
            // Liest nur das Feld “message”
            setResult(data.message || t('DELETE_EMBEDDINGS.MSG_DELETE_ALL_SUCCESS'));
        } catch (error) {
            errorContext.setErrorInfo(
                new ErrorMessage(
                    error.status?.toString() || '500',
                    error.message,
                    error.trace || ''
                )
            );
        } finally {
            setLoading(false);
        }
    };

    return (
        <div className="embedded-model-delete">
            <Headline size="h4" as="h3">
                {t('DELETE_EMBEDDINGS.TITLE')}
            </Headline>
            <Paragraph>{t('DELETE_EMBEDDINGS.DESCRIPTION')}</Paragraph>
            <div className="button-container">
                <Button
                    label={loading ? t('DELETE_EMBEDDINGS.BUTTON_LOADING') : t('DELETE_EMBEDDINGS.BUTTON_DELETE_ALL')}
                    onClick={handleDeleteAll}
                    disabled={loading}
                />
            </div>
            {result && (
                <div className="result-container">
                    <Paragraph>{result}</Paragraph>
                </div>
            )}
        </div>
    );
};

Dateiname & Pfad:
frontend/src/components/Deletion.jsx

⸻

9. i18n‐Datei (Beispiel)

Wenn du react-i18next verwendest, leg in frontend/src/i18n/de.json (oder locales/de/translation.json) folgende Keys an, damit deine UI-Texte übersetzt werden:

{
  "DELETE_EMBEDDINGS": {
    "TITLE": "Embedding‐Einträge löschen",
    "DESCRIPTION": "Hier kannst du alle existierenden Embedding‐Dokumente in Weaviate entfernen.",
    "BUTTON_DELETE_ALL": "Alle Einträge löschen",
    "BUTTON_LOADING": "Lösche Einträge…",
    "MSG_DELETE_ALL_SUCCESS": "Alle Einträge wurden gelöscht."
  }
}

Pfadbeispiel:
frontend/src/i18n/de.json

Dein i18n-Initialisierungscode (i18n.js) sollte dieses JSON laden und registrieren.

⸻

10. frontend/src/shared/app-routing.js

Damit route.api('/embedding/delete-all') funktioniert, solltest du eine Helferfunktion haben, die URLs korrekt voranstellt. Ein Minimal­beispiel:

// Dateiname: frontend/src/shared/app-routing.js

export const routeStrategy = async () => {
  // Beispiel: Basis-URL aus Umgebungsvariable oder relativen Pfad
  // Falls dein Backend auf demselben Host/Port läuft, genügt:
  return {
    api: (path) => path, 
  };
};

Erklärung:
	•	Falls dein Frontend per Proxy (z.B. package.json Proxy) auf Spring Boot weiterleitet, genügt api: path => path.
	•	Wenn du das Backend auf einem anderen Host/Port laufen hast, könntest du hier z.B. api: path => "http://localhost:8080" + path.

⸻

Zusammenfassung aller Dateinamen & Pfade

my-weaviate-app/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/
    │   │   └── com/example/embedding/
    │   │       ├── EmbeddingApplication.java
    │   │       ├── configuration/
    │   │       │   └── CobaAzureOpenAiRagConfiguration.java
    │   │       ├── controller/
    │   │       │   └── EmbeddingController.java
    │   │       └── service/
    │   │           ├── EmbeddingsService.java
    │   │           └── DeletionService.java
    │   └── resources/
    │       └── application.yml
    └── test/
        └── java/… (optional)
frontend/
├── package.json
└── src/
    ├── components/
    │   └── Deletion.jsx
    ├── i18n/
    │   └── de.json
    └── shared/
        └── app-routing.js

Mit dieser Komplett-Auflistung hast du alle Dateien, ihre Pfadnamen und den exakt notwendigen Inhalt, um:
	1.	Deine Embeddings (mit den Metadaten path, charset, source, creation_timestamp) in Weaviate zu speichern (via EmbeddingsService).
	2.	Alle diese Dokumente per String‐Filter "path != ''" zu löschen (via DeletionService).
	3.	Den Controller‐Endpunkt POST /embedding/delete-all anzubieten, den dein React‐Frontend aufruft.
	4.	Das React‐Frontend die Erfolgsmeldung korrekt anzuzeigen.

Damit ist dein Anwendungsfall vollständig und doku-konform umgesetzt.