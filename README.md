Hier eine maximal kompakte Lösung, die wirklich nur „alle Dokumente löschen“ macht. Wir zählen o. Ä. nicht, sondern rufen direkt vectorStore.delete(...) auf und liefern nur eine einfache Bestätigung zurück.

⸻

1. Service-Klasse (sehr kurz)

package com.example.embedding.service;

import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.filter.Filter;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class DeletionService {

    private final VectorStore vectorStore;

    public DeletionService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    /**
     * Löscht alle Dokumente in der konfigurierten Weaviate-Klasse,
     * indem einfach eine leere Filter-Expression übergeben wird.
     */
    @Transactional
    public void deleteAll() {
        vectorStore.delete(Filter.builder().build());
    }
}

Erläuterung:
	•	Filter.builder().build() erzeugt exakt eine leere Filter-Expression.
	•	Spring AI übersetzt das in eine Weaviate-Mutation ohne WHERE-Klausel, wodurch wirklich alle Objekte in der angegebenen Klasse gelöscht werden.

⸻

2. Controller-Klasse (ultrakurz)

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
     * Löscht einfach alle Einträge. Liefert { "message": "Deleted" } zurück.
     */
    @PostMapping("/delete-all")
    public ResponseEntity<Map<String, String>> deleteAll() {
        deletionService.deleteAll();
        return ResponseEntity.ok(Map.of("message", "Alle Einträge wurden gelöscht."));
    }
}

Erläuterung:
	•	POST /embedding/delete-all ruft nur deletionService.deleteAll() auf.
	•	Anschließend geben wir ein minimales JSON mit message zurück und Status 200.

⸻

3. Anwendungskonfiguration (unverändert, nur zur Erinnerung)

In src/main/resources/application.properties braucht es weiterhin:

spring.ai.vectorstore.weaviate.endpoint-url=https://my-weaviate.example.com/v1
spring.ai.vectorstore.weaviate.api-key=${WEAVIATE_API_KEY}
spring.ai.vectorstore.weaviate.object-class=MeineDokumente
spring.ai.vectorstore.weaviate.filter-metadata-fields=foo

	•	object-class muss der Name deiner Weaviate-Klasse sein.
	•	filter-metadata-fields darf ruhig nur ein einziges Dummy-Feld (z. B. foo) enthalten, damit ein leerer Filter intern funktioniert.

⸻

4. Anpassung in der React-Komponente

Da Du jetzt vom Backend einfach nur { message: "…" } zurücklieferst, genügt ein Minimum an Änderungen in deiner Component. So kann sie bleiben:

import React, { useState, useContext } from 'react';
import { Headline, Paragraph, Button } from '@lsg/components';
import { routeStrategy } from '../../shared/app-routing';
import { ErrorContext } from '../../context/error.context.provider';
import ErrorMessage from '../../service/error.message';
import { useTranslation } from 'react-i18next';

export const Deletion: React.FC = () => {
    const { t } = useTranslation();
    const errorContext = useContext(ErrorContext);
    const [loading, setLoading] = useState<boolean>(false);
    const [result, setResult] = useState<string>('');

    const handleDeleteAll = async () => {
        setLoading(true);
        setResult('');
        try {
            const route = await routeStrategy();
            const response = await fetch(route.api('/embedding/delete-all'), {
                method: 'POST',
                credentials: 'include'
            });

            if (!response.ok) {
                const err = await response.json();
                throw Object.assign(
                    new Error(err.error || `HTTP ${response.status}`),
                    { status: response.status, trace: err.trace }
                );
            }
            const data = await response.json();
            // Wir lesen einfach nur "message" aus dem JSON
            setResult(data.message || t('DELETE_EMBEDDINGS.MSG_DELETE_ALL_SUCCESS'));
        } catch (error: any) {
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

Wichtig:
	•	Wir erwarten nur noch eine Zeichenkette unter data.message.
	•	Ein deletedCount wird nicht mehr benötigt, daher entfällt die Logik dazu.
	•	In Deinen i18n-Dateien solltest du nur DELETE_EMBEDDINGS.MSG_DELETE_ALL_SUCCESS als Fallback definieren, falls data.message leer ist.

⸻

5. Zusammenfassung
	•	Service: Eine Einzeiler-Methode vectorStore.delete(Filter.builder().build()), um alles zu löschen.
	•	Controller: Ein Einzeiler, der deleteAll() aufruft und {"message":"…"} zurückgibt.
	•	React: Liest nur noch data.message und zeigt es an.

So bleibt der gesamte Löschen-Flow maximal schlank und erfüllt genau deine Anforderung: alle Dokumente löschen – mehr ist nicht nötig.