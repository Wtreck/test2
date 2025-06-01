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
        try {
            const route = await routeStrategy();
            const response = await fetch(route.api('/embedding/delete-all'), {
                method: 'POST', credentials: 'include'
            });
            if (!response.ok) {
                const err = await response.json();
                throw Object.assign(new Error(err.error || `HTTP ${response.status}`), {
                    status: response.status, trace: err.trace
                });
            }
            const data = await response.json();
            setResult(data.message || t('DELETE_EMBEDDINGS.MSG_DELETE_ALL_SUCCESS', { count: data.deletedCount }));
        } catch (error: any) {
            errorContext.setErrorInfo(new ErrorMessage(error.status?.toString() || '500', error.message, error.trace || ''));
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
