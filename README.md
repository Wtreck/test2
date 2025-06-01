Hier eine ultrakompakte Anpassung, bei der wir statt Filter.builder() den richtigen DSL‐Builder FilterExpressionBuilder verwenden. Damit funktioniert in Spring AI 1.0.0 Final das Löschen aller Dokumente ohne IDE-Fehler:

⸻

1. Service-Klasse: DeletionService.java

package com.example.embedding.service;

import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.filter.FilterExpressionBuilder;
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
     * indem eine leere Filter.Expression erzeugt wird.
     * Spring AI übersetzt das zu einer GraphQL-Mutation ohne WHERE-Klausel.
     */
    @Transactional
    public void deleteAll() {
        // 1. FilterExpressionBuilder erzeugen (ohne Bedingungen)
        FilterExpressionBuilder builder = new FilterExpressionBuilder();
        Filter.Expression emptyFilter = builder.build();

        // 2. Löschen aller Objekte in der Weaviate-Klasse
        vectorStore.delete(emptyFilter);
    }
}

Wichtig:
	•	Wir erstellen eine leere Filter.Expression via new FilterExpressionBuilder().build().
	•	Spring AI übersetzt das intern in eine Weaviate-Mutation ohne WHERE, was wirklich alle Objekte der in application.properties konfigurierten object-class entfernt.
	•	@Transactional sorgt nur dafür, dass bei einem Fehler eine Exception nach oben geht (Weaviate selbst kennt keine DB-Transaktion).

⸻

2. Controller-Klasse: EmbeddingController.java

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
     * Löscht alle Einträge im Weaviate-VectorStore und gibt
     * einfach { "message": "Deleted" } zurück.
     */
    @PostMapping("/delete-all")
    public ResponseEntity<Map<String, String>> deleteAll() {
        deletionService.deleteAll();
        return ResponseEntity.ok(Map.of("message", "Alle Einträge wurden gelöscht."));
    }
}

Hinweis:
	•	Deine React-Komponente macht einen POST auf /embedding/delete-all.
	•	Wir geben nur das Feld "message" als JSON zurück, damit im Frontend data.message gelesen werden kann.

⸻

3. application.properties (zur Erinnerung)

spring.ai.vectorstore.weaviate.endpoint-url=https://my-weaviate.example.com/v1
spring.ai.vectorstore.weaviate.api-key=${WEAVIATE_API_KEY}
spring.ai.vectorstore.weaviate.object-class=MeineDokumente
spring.ai.vectorstore.weaviate.filter-metadata-fields=foo

	•	object-class: Der Name deiner Weaviate-Klasse.
	•	filter-metadata-fields: Mindestens ein (Dummy-)Feld, damit ein leerer Filter überhaupt validiert wird.

⸻

4. Anpassung in der React-Komponente

Da wir jetzt nur noch { message: "Alle Einträge wurden gelöscht." } zurückgeben, genügt dieser Code (deine Komponente bleibt nahezu unverändert):

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
            // Nur "message" wird erwartet
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

Achte darauf, dass in deinen i18n-Dateien nur noch DELETE_EMBEDDINGS.MSG_DELETE_ALL_SUCCESS (als Fallback) existiert, falls data.message leer sein sollte.

⸻

Zusammenfassung
	1.	Service: Nutzt new FilterExpressionBuilder().build(), um eine leere Filter.Expression zu erzeugen, und ruft vectorStore.delete(emptyFilter) auf.
	2.	Controller: Bietet POST /embedding/delete-all an, ruft deletionService.deleteAll() und gibt { message: "Alle Einträge wurden gelöscht." } zurück.
	3.	Front-end: fetch('/embedding/delete-all', { method: 'POST' }) liest nur noch data.message.

Damit ist der gesamte Lösch-Flow maximal schlank und du vermeidest den Fehler „Can’t resolve method builder in Filter“.