# Аудит поддержки pricing в auto_ai_router (AIR)

> Проверка: какие token pricing фичи заложены в код, а какие нет.
> Репозиторий: [MiXaiLL76/auto_ai_router](https://github.com/MiXaiLL76/auto_ai_router) (commit на Jul 2026)

---

## 1. Audio Input — ✅ Полностью реализовано

| Этап | Статус |
|------|--------|
| Парсинг из ответа провайдера | ✅ `prompt_tokens_details.audio_tokens` (OpenAI Chat), `input_tokens_details.audio_tokens` (Responses API), Vertex AI |
| Поле в структуре | ✅ `TokenUsage.AudioInputTokens`, `StreamUsageInfo.AudioInputTokens` |
| Цена в модели | ✅ `ModelPrice.InputCostPerAudioToken` |
| Fallback цены | ✅ Если не задана — падает на `InputCostPerToken` |
| Расчёт стоимости | ✅ Вычитается из `PromptTokens` (чтобы не было двойного счёта), считается отдельно |
| Проброс в ответ | ✅ В `prompt_tokens_details.audio_tokens` |
| Spend-логи | ✅ Логируется в метаданные и БД |
| Тесты | ✅ `price_calculator_test.go`, `stream_test.go`, `converter_test.go`, `vertex/response_test.go` |

---

## 2. Audio Cache Read — ❌ НЕ реализовано

| Этап | Статус |
|------|--------|
| Парсинг из LiteLLM DB | ⚠️ Поле `cache_read_input_audio_token_cost` читается из модели, но... |
| Маппинг в ModelPrice | ❌ **НЕ маппится** — нет поля `InputCostPerCachedAudioToken` или аналогичного |
| Fallback | ❌ Аудио-кэш считается как обычный `cached_input_tokens` по `CacheReadInputTokenCost` → `InputCostPerToken` |
| Итог | Поле из DB прочитано и выброшено. Цена аудио-кэша = цена обычного кэша (или обычного инпута) |

**Почему:** Исторически не было массовых моделей с аудио-cache pricing. Поле в LiteLLM схеме есть, но код его не подхватил.

---

## 3. Image Cache Read — ❌ Отсутствует

| Этап | Статус |
|------|--------|
| Поле в LiteLLM DB | ❌ Даже в схеме `CustomPricingLiteLLMParams` нет `cache_read_input_image_token_cost` |
| Парсинг | ❌ Нет |
| ModelPrice | ❌ Нет |
| Расчёт | ❌ Нет |
| Изображения учитываются только как `InputCostPerImage` / `OutputCostPerImage` (per-image) или `OutputCostPerImageToken` (per-token), без cached-ставок |

---

## 4. Cache Read Long Context (above 200k) — ⚠️ Частично

| Этап | Статус |
|------|--------|
| Поле в LiteLLM DB | ✅ `cache_read_input_token_cost_above_200k_tokens` |
| Маппинг в ModelPrice | ❌ **НЕ маппится** — `convertPricingToModelPrice` игнорирует это поле |
| ModelPrice | ❌ Нет поля `InputCostPerCachedTokenAbove200k` |
| Расчёт | ❌ Кэш выше 200k считается по той же цене, что и обычный кэш |
| Регулярные токены above 200k | ✅ **Работает** — `InputCostPerTokenAbove200k` / `OutputCostPerTokenAbove200k` с пропорциональным распределением |

**Нюанс:** Только regular-токены имеют tiered pricing. Кэш-токены (и аудио, и текст) не имеют отдельной ставки для превышения 200k.

---

## 5. Cache Write (Cache Creation) — ✅ Единая ставка

| Этап | Статус |
|------|--------|
| Парсинг из ответа Anthropic | ✅ `cache_creation_input_tokens` |
| Парсинг из OpenAI формата | ✅ `prompt_tokens_details.cache_creation_tokens` |
| Поле в структуре | ✅ `TokenUsage.CacheCreationTokens` |
| Цена в модели | ✅ `ModelPrice.CacheCreationInputTokenCost` |
| Fallback | ✅ На `InputCostPerToken` |
| Расчёт | ✅ Вычитается из `PromptTokens`, считается по своей цене |
| Проброс в ответ | ✅ |
| Spend-логи | ✅ Агрегируется в `LiteLLM_DailyUserSpend` и другие таблицы |
| **Per-TTL разбивка (5m / 1h)** | ❌ **НЕ поддерживается** |

### Per-TTL детали
- Anthropic возвращает `CacheCreationDetails` с `ephemeral_5m_input_tokens` и `ephemeral_1h_input_tokens`
- LiteLLM также имеет отдельные поля: `cache_creation_input_token_cost_above_128k_tokens` и `cache_creation_input_token_cost_above_200k_tokens`
- В AIR эта структура **определена** (`types.go:119-122`), но **нигде не читается и не используется**
- Ни `ephemeral_5m`, ни `ephemeral_1h`, ни tiered cache write pricing не реализованы — все cache write токены тарифицируются по единой ставке `CacheCreationInputTokenCost`

---

## Сводная таблица

| Фича | Статус | Причина |
|------|--------|---------|
| Audio Input | ✅ Полностью | OpenAI / Vertex стандарт |
| Audio Output | ✅ Полностью | Аналогично input |
| Audio Cache Read | ❌ Не реализовано | Поле в DB есть, маппинг отсутствует |
| Image Cache Read | ❌ Отсутствует | Нет даже в LiteLLM схеме |
| Cache Read above 200k | ❌ Не реализовано | Поле в DB есть, не маппится |
| Cache Write | ✅ Единая ставка | Per-TTL разбивка (5m/1h) не используется |
| Regular above 200k | ✅ Работает | Только для regular токенов |
| Video/Image per second | ❌ Не проверялось | — |

