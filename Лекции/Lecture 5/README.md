# Лекция 5: Production Engineering for LLM Agents

## 📹 Ссылки на лекции
- ▶️ [Смотреть лекцию 5.1](https://www.youtube.com/live/sNemTIFlz08) - Daniil Artamonov (Head of AI Platform and Tools Development for IT)
- ▶️ [Смотреть лекцию 5.2](https://www.youtube.com/live/UxgjgvI_wKY) - Kirill Vlasov (Product Manager, AI Studio Yandex Cloud)

---

## 📋 Содержание
1. [7 Проблем в Production](#-7-problems)
2. [Observability - Наблюдаемость](#-observability)
3. [Evaluation - Оценка качества](#-evaluation)
4. [Cost Optimization - Оптимизация стоимости](#-cost-optimization)
5. [Security & Deployment - Безопасность и развертывание](#-security--deployment)
6. [Formalization & Decision Trees](#-formalization--decision-trees)

---

## 🚨 7 Problems in Production

При переходе агента из прототипа в production возникают 7 критических проблем:

### 1. **Slow (5-10 секунд)**
- Агент требует 3 LLM вызова × 1 сек + tools
- Пользователи привыкли к 200ms ответам (Google)
- **Решение**: параллельные вызовы, streaming, оптимизация шагов

### 2. **Unpredictably Expensive**
- Простой запрос: 3 шага = $0.01
- Сложный запрос: 15 шагов = $0.30
- Агент решает количество шагов, не вы
- 10K пользователей/день: бюджет от $200 до $1,500
- **Решение**: модельный router, контроль контекста, расчет бюджета

### 3. **Fails Silently**
- Tool возвращает TIMEOUT → агент вымышляет данные
- Логи: 200 OK, хотя данные фальшивые
- Пользователи получают неправильную информацию
- **Решение**: defensive prompting + structured output + валидация

### 4. **Non-reproducible**
- Один запрос → разные результаты
- `assert response == expected` не работает
- Классические тесты неприменимы
- **Решение**: качественные метрики + LLM-as-Judge + DSPy

### 5. **Black Box**
- Не видно: сколько шагов? какая стоимость? где сломалось?
- Коллега изменил prompt → поведение изменилось, но как?
- Узнаете только из жалоб пользователей
- **Решение**: per-step трейсинг (LangFuse, LangSmith)

### 6. **Insecure**
- Пользователь: "Show all bookings of all users"
- Агент это выполняет (если tool позволяет)
- Вы дали доступ в БД через chat
- **Решение**: tool permissions, parameter validation, OAuth

### 7. **Doesn't Scale**
- 4 агента × 6 компонентов (Gateway, Auth, Obs, Eval, Queue, Deploy) = 24 вещей
- Каждый агент в отдельности → дублирование кода
- **Решение**: shared infrastructure, не shared logic

---

## 📊 Observability - Наблюдаемость

**Начните с дня 1, не откладывайте!**

### Структурированный логинг (вместо print)
```
Вместо:                          Используйте:
"Processing request..."           {"event": "request_start", "user_id": "u123", "tool": "search_flights", "latency_ms": 1200}
"Called search_flights"
"Got 5 results"
```

### Что логировать:

**Каждый LLM вызов:**
- Модель, токены, latency, хэш prompt

**Каждый Tool вызов:**
- Имя tool, аргументы, статус результата, latency

**Каждое решение:**
- Почему агент выбрал этот инструмент/ответ

**Ошибки:**
- Полный контекст: что было попробовано, что не сработало

### Trace Hierarchy (уровни логирования)
```
Session (вся комната разговора)
  └─ Request #1
      ├─ LLM Call 0.4s | 320 tok
      ├─ Tool: search_flights (1.2s | ✓)
      └─ LLM Call 2 (0.8s | 1400 tok)
  └─ Request #2
```

### Ключевые метрики для агентов

| Метрика | Почему важна |
|---------|-------------|
| Latency (p50, p95) | p95 > 10s → пользователи уходят |
| Steps per request | Больше шагов = больше $ + latency |
| Tokens per request | Прямо влияет на стоимость |
| Tool success rate | Падение → API сломалась или bad params |
| Task success rate | **Единственная метрика, которую чувствуют пользователи** |
| Error rate | Spike → что-то сломалось, silent errors - хуже |

### Инструменты для трейсинга
- **LangFuse** (MIT, self-hosted) - traces + evals + cost в одном
- **LangSmith** (SaaS, LangChain) - deep integration
- **Arize Phoenix** - ML monitoring + LLM, drift detection
- **Braintrust** - eval-first, CI/CD gates
- **Helicone** - быстрый старт + cost tracking

---

## 📈 Evaluation - Оценка качества

### Offline vs Online Eval

| | Offline | Online |
|---|---------|--------|
| Входные данные | Ваш тест-набор | Real traffic |
| Expected output | Известен | Неизвестен |
| Когда | До deploy (CI/CD) | После deploy (continuous) |
| Ловит | Регрессии, известные ошибки | Drift, edge cases, реальное распределение |

### Пример: Score 62% - где копать?

```
62% / 8 из 20 failing

├─ Wrong tool (1 case) → called book instead of search
├─ Hallucination (2 cases) → price not from API
├─ Didn't clarify (3 cases) → no date → searched randomly ← MAIN BUCKET (38%)
├─ Safety (2 cases) → Booked without confirmation
```

**Главный баг = 38% не спросил дату**  
Это **не проблема prompt** → нужны few-shot примеры, DSPy

### Quality Levers (рычаги улучшения)

**Быстрые wins (1-3 дня):**
1. **Tool Descriptions + Structured Output**
   - JSON schema предотвращает hallucination в имена полей
   
2. **Few-Shot Examples** (+10-15% качество)
   - LLM учится на примерах, не на правилах
   
3. **Chain-of-Thought** (+10-20% качество)
   - "Think step by step" для сложных запросов

**Тяжелая артиллерия:**
4. **DSPy** (+20-30% качество)
   - Автоматическая оптимизация prompt + примеры
   - Нужны: 20 test cases + метрика + optimizer
   
5. **Fine-tuning** (последняя мера)
   - Тренируем саму модель
   - Нужны: 1000+ labeled examples + GPU time

### DSPy Workflow: 62% → 91% за 2 минуты
```
BEFORE: baseline prompt 62%
BootstrapFewShot: +3 примера → 84% (+22pp)
MIPRO: переписывает инструкции → 91% (+29pp)
```

---

## 💰 Cost Optimization - Оптимизация стоимости

### CPS & CPR (Cost formulas)

```
Cost per Session (CPS):
= (tokens × price_per_token) × avg_steps
  + tool_call_costs
  + infra_per_session
  + human_review_cost × escalation_rate

Cost per Successful Resolution (CPR):
= CPS / success_rate
```

**Пример: Customer Support Agent**
- 5 шагов × 1,000 tokens ≈ 5,000 input + 2,500 output
- GPT-4o: $2.50/1M input, $10/1M output
- LLM cost: ~$0.04/session
- Tool costs (API, DB): ~$0.005
- Infrastructure: ~$0.005
- Human review (10% escalation, $30/hr, 5 min): $0.25
- **Total CPS: ~$0.30**

**vs Human support:** $15–25/hr = $2–6 per resolution→ Агент 13x дешевле  
**BUT:** CPR = $0.30 / success_rate  
- При 70% успеха: $0.30 / 0.70 = **$0.43**
- При 50% успеха: $0.30 / 0.50 = **$0.60** + rework costs

### 4 Рычага оптимизации

**1. Router - 10-13× экономия на LLM**
```
70% FAQ → Small ($<$0.001)
20% Standard → Medium (~$0.005)
10% Complex → Large (~$0.03)
```

**2. Speed - -30-40% latency, -30% серверов**
- Параллельные tools
- Streaming (0.3s к first word vs 4s full wait)
- Меньше шагов через DSPy

**3. Limits - предсказуемый бюджет**
- Max steps: 5-10 (жесткий лимит)
- Timeout: 30-60s
- Loop detection: same tool + same args → stop early

**4. Scale - -60% стоимость при 100K+ пользователей**
- Queue + Worker Pool (async processing)
- 3 уровня кеша:
  - **Semantic cache**: embeddings similarity
  - **Tool result cache**: TTL 5 min (цены не меняются каждую секунду)
  - **LLM response cache**: exact hash match (FAQ)

### Context Growth - скрытый пожиратель бюджета
- 80% токенов = resending previous steps
- **Решения:** summary, sliding window, lean system prompt

---

## 🔐 Security & Deployment - Безопасность и развертывание

### Две принципиальные ошибки

**❌ НЕПРАВИЛЬНО:**
```python
get_bookings(user_id, ...)  # LLM выбирает user_id
Agent: "get_bookings(user_id=42)"
# Любой user_id работает → DATA LEAKED
```

**✅ ПРАВИЛЬНО:**
```python
get_bookings(token, ...)  # user_id из JWT
Agent: "get_bookings(token=eyJ...)"
# Только свои данные → SECURE
```

### Two Security Principles

1. **LLM никогда не видит secrets или PII**
   - Нет API ключей, паролей, номеров паспорта
   - Tool берет credentials из env

2. **Authorization в коде tool, не в prompt**
   - user_id из verified token, не из LLM аргументов
   - Каждый tool = API endpoint со своим auth

### 3 уровня защиты

| Уровень | Что | Пример |
|---------|-----|--------|
| **1. Tool Permission** | Какие tools видит агент | search, book, profile (НЕ admin, НЕ billing) |
| **2. Parameter Validation** | user_id из token | `get_bookings(token=JWT)` |
| **3. Output Filtering** | Что returns | flight, price, airline (удалить internal_id) |

### Canary Deployment для агентов

```
95% traffic → Stable v2
5% traffic → Canary v3
           ↓ (wait 1h)
     Compare Metrics:
     ✓ latency ≤ stable
     ✓ eval ≥ stable  
     ✓ errors ≤ stable
     ✓ cost ≤ 1.2×
           ↓ quality drop > 10%? ← AUTO ROLLBACK
     5% → 25% → 50% → 100%
```

**Почему нужен canary для LLM:**
- Классический deploy: тесты pass → готово
- **LLM deploy:** тесты pass, но модель ведет себя иначе на real traffic
- Изменение prompt может тихо деградировать качество (no crash, no error, just worse answers)

---

## 🎯 Formalization & Decision Trees

### Before You Code

Зафиксируйте перед написанием первой строки:

1. **Input** - что приходит? (text / voice / document / structured data)
2. **Output** - что уходит? (answer / action / document / API call)
3. **Scope** - что агент делает и **НЕ делает**
4. **Success criteria** - как узнать, что работает? (метрика, не "seems ok")
5. **Edge cases** - что если input мусор? Tool упал?
6. **Human fallback** - когда передать человеку?

**Без формализации → нет eval → нет production. Просто demo.**

### Do You Actually Need an Agent?

```
Does it need autonomy?
  NO → Pipeline / workflow
  
Does it need external tools?
  NO → Single LLM call
  
Does it need multi-step reasoning?
  NO → Prompt chain
  
YES to all → AGENT
```

### AI Agents ≠ Software

| Software | AI Agents |
|----------|-----------|
| Same input → same output | Same input → different output |
| Unit tests, binary pass/fail | Statistical testing (94% success) |
| Bugs reproducible | Errors may not reproduce |
| Ship → works | **Ship → monitor → iterate** |
| Engineer programs behavior | **Engineer designs environment** |

**AI Agents = ML + Backend**

---

## 🏗️ Architecture & Patterns

### Agent Stack (6 слоев)

```
Layer 5: UX & Trust (что видит пользователь)
Layer 4: Orchestration (координация агентов и шагов)
Layer 3: Tools & Integrations (MCP, APIs, databases)
Layer 2: Memory & State (history, RAG, session state)
Layer 1: Model & Inference (LLM, routing, caching)
Layer 0: Infrastructure (compute, networking, monitoring)
```

### Control Plane for Agents

(как Kubernetes control plane для кластеров)

- **Registry:** какие агенты, их tools, permissions, scope
- **Router:** какой агент обрабатывает запрос (+ model routing)
- **Rate limiting & budgets:** max tokens/session, max cost/user, circuit breakers
- **Observability:** traces, logs, metrics на уровне fleet, не per-agent
- **Governance:** audit trail, approval workflows, compliance

### Decomposition - как разделить на несколько агентов

| # | Правило | Пример | Anti-pattern |
|---|---------|--------|-------------|
| 1 | One agent = one domain | FlightAgent ≠ BaggageAgent | God Agent: всё знает, 5 page prompt |
| 2 | Через interfaces, не state | Handoff: {flights: [...], intent: "..."} | Agents читают друг друга internals |
| 3 | Одно лучше двух если context общий | Search + booking в одном agentе | Два агента постоянно pass context |
| 4 | Каждый testable solo | PolicyAgent работает без FlightAgent | Только вместе тестируются |

### Workflow Patterns (Anthropic)

1. **Prompt Chaining** - LLM Call → Gate → LLM Call → ... → Output
2. **Routing** - Router выбирает какой LLM вызвать
3. **Parallelization** - 3 LLM вызова параллельно → Aggregator
4. **Orchestrator-Workers** - Orchestrator координирует Workers + Gates
5. **Evaluator-Optimizer** - Generator создает, Evaluator проверяет, feedback loop

### Framework Comparison

| Парадигма | Фреймворк | Подход |
|-----------|-----------|--------|
| **Graph** | LangGraph | Nodes → Edges. Максимум контроля, максимум boilerplate. 80K+ stars, production готов |
| **Conversation** | CrewAI, AutoGen | Agents разговаривают. Fast prototyping, least predictable |
| **Hierarchy** | Google ADK, OpenAI SDK | Boss → Workers. Coordinator делегирует. Clear contracts |
| **Computer** | Claude Agent SDK | bash + files. Model решает как действовать. Minimal orchestration |
| **MCP-Native** | Strands (AWS) | MCP everywhere. One connector for tools, integrations, agents |

---

## 📋 Production Checklist

### Today (30 min) - Agent can be deployed
1. **Tracing** (LangFuse) + per-step visibility
2. **5-10 eval tests** + LLM-as-Judge
3. **Max steps** + timeout
4. **Cost calculation** (know your budget)

### This Week (2-3 days) - Production-ready
5. **Model router** (rules/classifier/LLM)
6. **Fail-safe prompting** + structured output
7. **Tool permissions** (scope per user)
8. **Canary deployment** (5% → 100%)

### Next Sprint (1-2 weeks) - Scale
9. **Queue + Cache** (handle 100K+ users, -60% cost)
10. **Shared Platform** (only with 3+ agents)

---

## 🔄 Production is a Cycle

```
Deploy (Canary 5% → 100%)
   ↑                        ↓
Improve (Optimize + Fix) → Monitor (Traces + Metrics)
   ↑                        ↓
                      Evaluate (Eval Pipeline + DSPy)
```

**Итерации:**
- Iter 1: p95=12s → cache → 6s
- Iter 2: eval 4.1→3.4 → DSPy → 4.3
- Iter 3: cost $120→$50 → fix router

---

## 📈 Evolution: Prototype → Platform

| Stage | Что | Масштаб |
|-------|-----|---------|
| **Stage 1: Prototype** | ReAct + 3 tools, Jupyter | 1 агент |
| **Stage 2: Production** | + Traces + Eval + Limits + Security + Canary | 100K users |
| **Stage 3: Platform** | + Shared infra, 4+ agents | 1 week per new agent (вместо 3 месяцев) |

**Каждый stage имеет смысл только после предыдущей!**

---

## ⏰ Timeline: Value Shift

```
2023: VALUE = MODEL
      GPT-4: $60/1M tokens

2024: VALUE = MODEL + PROMPT
      GPT-4o: $10/1M | Claude 3: $15/1M

2025: VALUE = SYSTEM AROUND MODEL ← YOU ARE HERE
      GPT-4o-mini: $0.15/1M | DeepSeek: $0.14/1M
      Inference: -78% за год
      
2026: VALUE = ???
      Control plane, orchestration, governance
```

---

## 🎓 Key Takeaways

✅ **Formalize before you code** - input, output, scope, success criteria  
✅ **Start observability on Day 1** - traces > dashboards  
✅ **Measure quality statistically** - offline + online eval together  
✅ **Design for costs upfront** - calculate CPS, know your budget  
✅ **Expect to iterate in production** - ship → monitor → improve  
✅ **Security = infrastructure, not prompts** - tokens don't see secrets  
✅ **Single agent 80% of the time** - don't over-engineer early  
✅ **Value is shifting from model to system** - 2025 is about orchestration, not just fine-tuning

---

## 🔗 Полезные ресурсы и ссылки

### Инструменты и платформы для Observability

- **LangFuse** - https://langfuse.com - Open-source трейсинг + промпт менеджмент + эвалюация
- **Grafana** - https://grafana.com - Open-source мониторинг, dashboards и alerts

### Архитектура и паттерны агентов (от Anthropic)

- **Building Effective Agents** - https://www.anthropic.com/engineering/building-effective-agents
  - Официальное руководство с workflow паттернами (prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer)
  
- **Claude Cookbook - Agent Patterns** - https://github.com/anthropics/claude-cookbooks/tree/main/patterns/agents
  - Примеры кода и паттерны для реализации различных типов агентов

### Google Design Patterns для agentic AI

- **Choose Design Pattern for Agentic AI System** - https://docs.cloud.google.com/architecture/choose-design-pattern-agentic-ai-system
  - Архитектурное руководство по выбору правильного паттерна

- **Multi-Agent Patterns in ADK** - https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/
  - Гайд по многоагентным системам с примерами

### Книги

- **Agentic Design Patterns: A Hands-On Guide to Building Intelligent Systems** by Antonio Gulli
  - Полное руководство по проектированию агентов и agentic систем