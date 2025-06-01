Um wirklich alle Dokumente ohne Builder‐Klasse zu löschen, kannst du statt FilterExpressionBuilder einfach die String‐Variante von delete(...) verwenden. Wenn in jedem Dokument ein bestimmtes Metadatenfeld (z. B. foo) existiert und du dieses Feld in deinen application.properties unter filter-metadata-fields registriert hast, löscht der Ausdruck "foo != ''" alle Einträge. Das ist noch kompakter:

⸻

1. application.properties

Stelle sicher, dass du mindestens folgendes gesetzt hast:

spring.ai.vectorstore.weaviate.endpoint-url=https://my-weaviate.example.com/v1
spring.ai.vectorstore.weaviate.api-key=${WEAVIATE_API_KEY}
spring.ai.vectorstore.weaviate.object-class=MeineDokumente

# <– Hier das Metadatenfeld „foo“ so angeben, dass es in jedem Dokument vorhanden ist.
spring.ai.vectorstore.weaviate.filter-metadata-fields=foo

Tipp: Wenn jedes Dokument unter "foo" einen (nicht‐leeren) Wert hat, trifft der Filter "foo != ''" auf alle Dokumente zu.

⸻

2. Service‐Klasse (ultrakompakt)

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
     * Löscht alle Dokumente, indem wir per String‐Filter auf das Metadatenfeld "foo"
     * prüfen, das in jedem Dokument existiert. "foo != ''" trifft daher auf alle Einträge zu.
     */
    @Transactional
    public void deleteAll() {
        vectorStore.delete("foo != ''");
    }
}

Warum das funktioniert:
	•	vectorStore.delete(String) wandelt den String intern in eine Filter.Expression um.
	•	Mit "foo != ''" forderst du: Lösche alle Dokumente, bei denen das Feld foo nicht leer ist.
	•	Da du in application.properties angegeben hast, dass foo in jedem Dokument vorhanden ist, löscht dieser Filter wirklich alles.

⸻

3. Controller‐Klasse (ebenfalls kompakt)

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
     * Einfacher Endpunkt, der alle Dokumente löscht.
     * Liefert JSON { "message": "Alle Einträge wurden gelöscht." } zurück.
     */
    @PostMapping("/delete-all")
    public ResponseEntity<Map<String, String>> deleteAll() {
        deletionService.deleteAll();
        return ResponseEntity.ok(Map.of("message", "Alle Einträge wurden gelöscht."));
    }
}

Erklärung:
	•	Die React‐Komponente macht einen POST auf /embedding/delete-all.
	•	Der Service führt direkt vectorStore.delete("foo != ''") aus, was alle Dokumente löscht.
	•	Anschließend liefert der Controller nur noch { "message": "Alle Einträge wurden gelöscht." } mit HTTP 200.

⸻

4. React‐Komponente (bleibt unverändert)

Deine Deletion‐Komponente kann wie bisher bleiben, weil wir nur das JSON‐Feld message zurückliefern:

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
                // Falls Fehler kommen, erwartet dein Code JSON mit { error, trace }
                const err = await response.json();
                throw Object.assign(
                    new Error(err.error || `HTTP ${response.status}`),
                    { status: response.status, trace: err.trace }
                );
            }
            const data = await response.json();
            // Nur 'message' wird erwartet
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
	•	In deiner Übersetzungsdatei (de.json) brauchst du mindestens einen Fallback‐Key wie

"DELETE_EMBEDDINGS": {
  "MSG_DELETE_ALL_SUCCESS": "Alle Einträge wurden gelöscht."
}

falls data.message mal fehlen sollte.

⸻

5. Fazit
	•	Keine Builder-Klasse mehr nötig: Statt FilterExpressionBuilder nutzt du einfach vectorStore.delete("foo != ''").
	•	Maximal kurz: Service und Controller bestehen nur noch aus je einer Methode.
	•	Sicherstellen: In application.properties “filter-metadata-fields=foo” setzen, und in jedem Dokument unter foo einen nicht-leeren Wert speichern.
	•	Voll funktional: Auf Klick löscht das Backend alles aus deinem Weaviate-Index, und die React-Komponente zeigt die Erfolgsmeldung an.

So bleibt dein Lösch-Flow wirklich einfach und kompakt, ohne weitere Builder- oder Aggregate-Schritte.