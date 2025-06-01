import React, { useState, useContext } from 'react';
import { Headline, Paragraph, Button } from '@lsg/components';
import { routeStrategy } from '../../shared/app-routing';
import { ErrorContext } from '../../context/error.context.provider';
import ErrorMessage from '../../service/error.message';
import { useTranslation } from 'react-i18next';

export const Embeddings : React.FC = () => {
    const { t } = useTranslation();
    const errorContext = useContext(ErrorContext);
    const [loading, setLoading] = useState<boolean>(false);
    const [result, setResult] = useState<string>('');

    const handleInit = async () => {
        setLoading(true);
        try {
            const route = await routeStrategy();
            const response = await fetch(route.api('/embedding/init'), {
                method: 'GET',
                credentials: 'include'
            });

            if (!response.ok) {
                throw Object.assign(
                    new Error(`HTTP Error: ${response.status}`),
                    { status: response.status, response }
                );
            }
            const text = await response.text();
            setResult(text || t('EMBEDDINGS.MSG.SUCCESS'));
        } catch (error: any) {
            errorContext.setErrorInfo(new ErrorMessage(error.status, error.message, error.trace));
        } finally {
            setLoading(false);
        }
    };

    return (
        <div className="embedded-model-init">
            <Headline size="h4" as="h3">
                {t('EMBEDDINGS.TITLE')}
            </Headline>
            <Paragraph>{t('EMBEDDINGS.DESCRIPTION')}</Paragraph>
            <div className="button-container">
                <Button
                    label={loading ? t('EMBEDDINGS.BUTTON_LOADING') : t('EMBEDDINGS.BUTTON_INIT')}
                    onClick={handleInit}
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
