# Модуль з архітектури №1
**Студент:** Саакян Давид ІПЗ-32

## Аналіз та план еволюції архітектури OrderFlow

### 1. Аналіз поточного стану (Current Architectural Boundaries & Dependencies)

Поточна система OrderFlow є модульним монолітом, де всі 6 модулів (Checkout, Orders, Inventory, Payments, Notifications, Reporting) виконуються в одному процесі та використовують спільну in-memory базу даних[cite: 8].

<img width="3952" height="5064" alt="OrderFlow Monolith Architecture-2026-10-22-115646" src="https://github.com/user-attachments/assets/3fc3a0a7-6467-4119-b5c9-8604e0eb74d4" />


#### Виявлені проблеми та критичні залежності:

1. **Жорсткі зв'язки (Coupling) та витік предметних областей:**
   - Payments звертається безпосередньо до `Orders.OrderRepository.findById()`, порушуючи інкапсуляцію[cite: 8].
   - 48% feature PR зачіпають Orders та Payments одночасно[cite: 9].
   - Середня кількість модулів, що змінюються за одну фічу, зросла з 1.7 до 4.3[cite: 9].

2. **Блокуючий синхронний каскад у Checkout (Synchronous Chain):**
   - Ланцюжок `Checkout -> Orders -> Inventory -> Payments -> Notifications` виконується синхронно у межах одного HTTP-запиту[cite: 8].
   - Затримка або збій у зовнішніх сервісах (FakePay до 3000ms/5% помилок, FakeMail) прямим чином затримує або повністю ламає користувацький Checkout flow[cite: 8].
   - Два останні інциденти сталися саме через затримки payment provider[cite: 9].

3. **Неефективність деплойменту та тестування:**
   - 37% змін шаблонів сповіщень вимагали повного регресійного тестування всього Checkout[cite: 9].
   - Частота релізів впала з 5 до 2 на тиждень, а Rollback rate виріс з 2% до 9% (при цьому rollback повертає весь моноліт)[cite: 4, 9].


---

### 2. Пріоритетні Architectural Characteristics & Design Principles

#### Ключові архітектурні характеристики (в порядку пріоритету):
1. **Availability (Доступність) Checkout:** Найвищий пріоритет, оскільки відмова Checkout прямо веде до втрати виручки[cite: 4].
2. **Deployability (Розгортаємість) & Modifiability (Змінюваність):** Зниження вартості та ризику внесення змін для часто оновлюваних модулів (Notifications, Payments)[cite: 3, 4].
3. **Data Consistency (Правильність даних):** Запобігання overselling в Inventory та гарантія точного списання коштів[cite: 3, 4].
4. **Performance / Scalability (Продуктивність та масштабованість):** Здатність Reporting витримувати зростання навантаження в 8-10 разів без впливу на Checkout[cite: 3].

#### Релевантні принципи проєктування:
- **Single Responsibility & Bounded Context (DDD):** Чітке відокремлення меж контекстів та закриття прямого доступу до чужих БД/репозиторіїв[cite: 5].
- **Loose Coupling & Asynchronous Messaging:** Перехід від синхронного виклику непокритичних операцій (наприклад, надсилання листівок) до подійно-орієнтованої моделі (Event-Driven)[cite: 5].
- **Smart Endpoints, Dumb Pipes / Modular Monolith First:** Відмова від передчасного розпилу на мікросервіси для малої команди[cite: 4, 5].


---

### 3. Аналіз сценаріїв змін (Change Scenarios Analysis)

| Scenario | Stimulus | Response | Affected Components | Measurable Response Measure |
| :--- | :--- | :--- | :--- | :--- |
| **A: Notifications** | Зміна шаблонів/правил 2-3 рази/тижд[cite: 3]. | Зміна деплоїться без регресійного тестування Checkout[cite: 3]. | Notifications, Email Provider[cite: 8] | Time to Deploy < 30 хв; 0 впливу на Checkout uptime |
| **B: 2nd Payment Provider** | Додавання fallback provider / async confirmation[cite: 3]. | Інтеграція нового провайдера через єдину абстракцію[cite: 3]. | Payments, Orders, Checkout[cite: 8] | Час реалізації нового провайдера < 3 днів |
| **C: Reporting Growth** | Зростання трафіку в 8-10 разів[cite: 3]. | Запити звітів не навантажують OLTP базу даних[cite: 3]. | Reporting, DB[cite: 8] | p99 latency Checkout < 500ms під час генерації важких звітів |
| **D: Inventory Idempotency** | Повторний запит reserve/release[cite: 3]. | Повторна операція повертає той самий стан без дублювання[cite: 3]. | Inventory[cite: 8] | 0 випадків overselling при повторних мережевих запитах |
| **E: Checkout Availability** | Таймаут/помилка Payment Provider (2-3с)[cite: 3]. | Checkout завершується або пропонує fallback; лист надсилається згодом[cite: 3]. | Checkout, Payments, Notifications[cite: 8] | Доступність Checkout >= 99.9%; відсутність втрачених замовлень |
| **F: Discount Rule** | Щомісячні нові правила знижок[cite: 3]. | Розрахунок знижки відображається клієнту і йде в Reporting[cite: 3]. | Checkout, Orders, Reporting[cite: 8] | Зміна правил знижок займає < 4 годин розробки |


---

### 4. Оцінка Cost of Change, Risks & Architecture Tax

Використовуючи базову тарифну ставку $50/годину[cite: 6]:

1. **Change Cost (Вартість змін):**
   - **Поточний стан (Часті зміни Notifications):** Розробка + Регрес всього Checkout + Спільний деплой = 30 годин ($1,500)[cite: 6, 9].
   - **Цільовий стан (Після виділення Boundary / Event-Driven):** Розробка + Ізольований тест = 6 годин ($300).

2. **Architecture Tax (Архітектурний податок):**
   $$\text{Architecture Tax} = \text{Current Change Cost} - \text{Baseline Change Cost}$$
   $$\text{Architecture Tax} = \$1,500 - \$300 = \$1,200 \text{ на кожний реліз сповіщень} \cite{6}$$
   *При 2-3 релізах на тиждень переплата через погані кордони становить ~$10,000–$14,000 на місяць.*

3. **Expected Risk Cost (Очікувана вартість ризику):**
   - **Ризик:** Помилка під час деплою шаблону сповіщень провалює Checkout.
   - **Ймовірність (Probability):** Rollback rate зараз становить 9%[cite: 9]. Припустимо, 5% деплоїв сповіщень ламають Checkout.
   - **Business Impact:** 2 години падіння Checkout під час пікового навантаження = $20,000 втраченої виручки.
   - **Expected Risk Cost:** $0.05 \times \$20,000 = \$1,000$ на один реліз[cite: 6].


---

### 5. Обґрунтований план еволюції: Прагматична цільова архітектура

#### Відхилення радикального розпилу (Rejecting Proposal)
Пропозиція перетворити одразу всі 5 модулів у мікросервіси є **НЕРИЗИКОВАНОЮ та НЕПРИЙНЯТНОЮ** для команди з 8 інженерів[cite: 4]. Це створить додаткове операційне навантаження[cite: 5].

#### Покроковий план покращень (Target Architecture):

##### Етап 1: Строгі межі всередині Моноліту (Refactoring Boundaries)
1. Прибрати прямий доступ Payments до Orders, впровадити DTO/Contracts[cite: 5, 8].
2. Додати `idempotency_key` в Inventory[cite: 3].

##### Етап 2: Асинхронність та Read Model Separation
1. Перевести Notifications на обробку подій `OrderCompletedEvent` через Event Bus / Queue[cite: 5, 8].
2. Виділити Read Replica / CQRS Read Model для Reporting, щоб звіти не навантажували транзакційну БД[cite: 3, 5, 8].


<img width="8192" height="3777" alt="OrderFlow Monolith Architecture-2026-10-22-115737" src="https://github.com/user-attachments/assets/27930730-c04a-4acb-96c6-a767b349936b" />



### 6. Метрики та Fitness Functions для перевірки результату

1. **Автоматизовані тестування архітектурних меж (Architecture Unit Tests):**
   - *ArchUnit / NetArchTest / ESLint Rule:* Модуль Payments або Notifications не має права імпортувати файли з внутрішніх папок/репозиторіїв Orders[cite: 5, 8].

2. **Ключові цільові engineering-метрики:**
   - **Rollback Rate:** Зменшення з 9% до < 2%[cite: 9].
   - **Modules Changed per Feature:** Зменшення з 4.3 до < 2.0[cite: 9].
   - **Checkout Latency (p99):** < 800ms навіть при відмовах/затримках FakePay або зростанні трафіку Reporting[cite: 3, 8].


---

### 7. Глибокий аналіз AI-assisted delivery та Time to Market (Рівень 90+)

#### 1. Розрахунок прогнозованого Time to Market (TTM)

$$\text{TTM} = \text{Analysis} + \text{Implementation} + \text{Integration} + \text{Validation} + \text{Release}$$
[cite: 7]

| Stage | Поточне значення | Новий прогноз (Target) | Обґрунтування / Допущення (Assumptions) |
| :--- | :---: | :---: | :--- |
| **Analysis** | 20 днів[cite: 7] | **15 днів** | Скорочення на 5 днів за рахунок AI-prototyping, швидкого дослідження доменної моделі[cite: 7]. |
| **Implementation** | 40 днів[cite: 7] | **20 днів** | Скорочення вдвічі (а не в 4 рази) через вимогу людського Code Review для критичних модулів (Payments/Security)[cite: 7]. |
| **Integration** | 25 днів[cite: 7] | **12 днів** | Скорочення завдяки чітким контрактам модулів, Contract-тестам та асинхронним подіям[cite: 7]. |
| **Validation** | 25 днів[cite: 7] | **12 днів** | Автоматизація тестових середовищ та розпаралелювання CI/CD[cite: 7]. |
| **Release** | 10 днів[cite: 7] | **4 дні** | Зменшення blast radius, частіші релізні вікна для ізольованих модулів[cite: 5, 7]. |
| **Всього TTM** | **120 днів (4 міс.)**[cite: 7] | **63 дні (~2 міс.)** | **Реалістичний цільовий показник замість 30 днів (1 міс.)**[cite: 7]. |

#### 2. Відповіді на критичні запитання аналізу (11 питань)

1. **Прогнозований Time to Market:** 63 робочих дні (~2 місяці)[cite: 7].
2. **Assumptions (Основні припущення):** AI прискорює лише написання шаблонів коду/тестів, але не час на прийняття рішень бізнесом, рев'ю коду та безпеку[cite: 7].
3. **Що AI реально прискорює:** Генерацію boilerplate-коду, написання Unit-тестів, рефакторинг та документацію (Implementation stage)[cite: 7].
4. **Залишкові вузькі місця (Bottlenecks):** Доставка коду зміститься в Code Review (нестача Senior інженерів) та Integration/Validation (черги на тестування)[cite: 7].
5. **Архітектурні обмеження throughput:** Єдиний деплоймент-календар, відсутність авто-тестів на межах модулів, спільне тестове середовище (queue 3-5 днів)[cite: 4, 7].
6. **Процесні обмеження:** Необхідність суворого людського контролю фінансової логіки, узгодження вимог між командами[cite: 4, 7].
7. **Чи реалістична ціль в 1 місяць (30 днів)?** НІ, нереалістична[cite: 7]. Навіть якщо скоротити Implementation з 40 днів до 0, сума решти стадій (20+25+25+10 = 80 днів) все одно значно перевищує 30 днів[cite: 7].
8. **Що потрібно для досягнення цілі в 1 місяць?**
   - Паралельні тестові оточення (Ephemeral Environments).
   - Повна автоматизація регресії та CI/CD без ручних approve-кроків[cite: 7, 9].
   - Виділені незалежні сервіси зі своїми релізами (що дорого для команди з 8 людей)[cite: 4, 5].
9. **Нові ризики при зростанні AI throughput:**
   - Збільшення розміру PR -> роздутий контекст для Code Review -> пропущені баги у проді[cite: 7, 9].
   - "Hallucinated code" та прогалини в безпеці[cite: 7].
10. **Як вимірювати вплив AI (без LOC/PRs):**
    - Lead Time for Changes (час від першого коміту до продакшену).
    - Defect Escape Rate (кількість багів, що пройшли на прод).
    - Cycle Time of Implementation Stage.
11. **Evidence для перегляду прогнозу:**
    - Фактичний час перебування PR у статусі "Awaiting Review".
    - Динаміка кількості знайдених дефектів під час Integration testing.
