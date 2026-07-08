# Аудит поддержки pricing в auto_ai_router (AIR)

> Проверка: какие token pricing фичи заложены в код, а какие нет.
> Репозиторий: [MiXaiLL76/auto_ai_router](https://github.com/MiXaiLL76/auto_ai_router) (commit на Jul 2026)

---

## 1. Audio Input — ✅ Полностью реализовано

| Этап | Статус |
|------|--------|
| Парсинг из ответа провайдера | ✅ `prompt_tokens_details.audio_tokens` (OpenAI Chat), `input_tokens_details.audio_tokens` (Responses API), Vertex AI |
| Поле в структуре | ✅ `TokenUsage.AudioInputTokens` |
| Цена в модели | ✅ `ModelPrice.InputCostPerAudioToken`, fallback на `InputCostPerToken` |
| Расчёт стоимости | ✅ Вычитается из `PromptTokens`, считается отдельно |
| Проброс в ответ | ✅ В `prompt_tokens_details.audio_tokens` |

---

## 2. Audio Cache Read — ❌ НЕ реализовано

| Этап | Статус |
|------|--------|
| Парсинг из LiteLLM DB | ⚠️ Поле `cache_read_input_audio_token_cost` читается, но не маппится |
| Расчёт | ❌ Аудио-кэш считается как обычный кэш по `CacheReadInputTokenCost` → `InputCostPerToken` |

---

## 3. Image Cache Read — ❌ Отсутствует

| Этап | Статус |
|------|--------|
| Поле в LiteLLM DB | ❌ Нет `cache_read_input_image_token_cost` |
| Расчёт | ❌ Нет. Изображения — только `InputCostPerImage` / `OutputCostPerImage` |

---

## 4. Cache Read Long Context (above 200k) — ❌ НЕ реализовано

| Этап | Статус |
|------|--------|
| Поле в LiteLLM DB | ✅ `cache_read_input_token_cost_above_200k_tokens` |
| Маппинг в ModelPrice | ❌ **НЕ маппится**, нет поля `InputCostPerCachedTokenAbove200k` |
| Расчёт | ❌ Кэш выше 200k считается по обычной ставке кэша |

**Regular-токены** above 200k — ✅ работают (`InputCostPerTokenAbove200k` / `OutputCostPerTokenAbove200k`).

---

## 5. 5m / 1h Cache Write — ❌ НЕ поддерживается

| Этап | Статус |
|------|--------|
| Парсинг из ответа Anthropic | ⚠️ `CacheCreationDetails` с полями `ephemeral_5m_input_tokens` / `ephemeral_1h_input_tokens` определена в `types.go:119-122`, но **не используется** |
| Парсинг из LiteLLM DB | ❌ В схеме `CustomPricingLiteLLMParams` нет полей для per-TTL cache write pricing (`cache_creation_input_token_cost_above_128k_tokens`, `cache_creation_input_token_cost_above_200k_tokens`) |
| Цена в модели | ❌ Единая `CacheCreationInputTokenCost` без разбивки по TTL |
| Расчёт | ❌ Все cache write токены тарифицируются одинаково, per-TTL разбивка игнорируется |

---

## Сводная таблица

| Фича | Статус |
|------|--------|
| Audio Input | ✅ Полностью |
| Audio Cache Read | ❌ Не реализовано (поле есть, не маппится) |
| Image Cache Read | ❌ Отсутствует |
| Cache Read above 200k | ❌ Не реализовано (поле есть, не маппится) |
| 5m Cache Write | ❌ Не поддерживается |
| 1h Cache Write | ❌ Не поддерживается |
