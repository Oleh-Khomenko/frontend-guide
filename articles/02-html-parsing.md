# Оптимізація фронтенду від першого кліку – Частина 2

Перші байти HTML прилетіли з сервера, ми зупинились на цьому в кінці минулої частини. На екрані досі нічого. Далі – що браузер робить з цим потоком байтів і де саме все ламається.

Далі по порядку: парсинг HTML, CSSOM, скрипти, preload scanner, пріоритети, шрифти, картинки і події життєвого циклу – усе, що відбувається між першим байтом і готовим до рендерингу DOM.

---

## 1. Парсинг HTML

### Де живе парсинг: майже все на main thread

Парсер виглядає просто: байти → DOM. Під капотом кроків більше, але майже всі на main thread:

```
   TCP ──▶ Network I/O (окремий процес, поза main thread)
                  │ bytes
                  ▼
        ┌───────────────────────────────────────┐
        │              MAIN THREAD               │
        │                                        │
        │   tokenizer ──▶ tree construction ──▶ DOM
        │                                        │
        │   + JS + CSSOM + layout + paint        │
        │   (парситься чанками, з yield-паузами) │
        └───────────────────────────────────────┘
                  ▲
                  │ поки парсер стоїть на скрипті
        ┌────────────────────────┐
        │     Preload scanner     │  спекулятивно сканує сирий
        │                         │  HTML попереду, качає ресурси
        └────────────────────────┘
```

**Network I/O** живе поза main thread (у Chrome – окремий мережевий процес): читає байти з сокета й передає їх рушію. **Main thread** робить решту: токенізує байти, будує з токенів DOM (tree construction) і тут же виконує JS, парсить CSSOM, рахує layout, малює. Окремо працює **preload scanner** – легкий спекулятивний прохід, що біжить попереду основного парсера по сирому HTML і завчасно качає ресурси.

Чому токенізація і tree construction на одному thread? Бо tree construction виконує зустрінуті скрипти, а скрипт може викликати `document.createElement`, прочитати computed styles, тригернути layout – усе це підв'язане до однопоточного JS. Сама токенізація формально окрема (байти → токени), але `document.write` зі скрипта вміє переписати вхідний потік, тож і її тримають поряд зі скриптами на main thread.

Виняток – Firefox: він виносить декодування, токенізацію і навіть спекулятивний tree construction на окремий thread, а на main thread лише «програє» готові операції. Chrome і Safari парсять на main thread, а паралелізм дає мережа й preload scanner.

### Як це працює в часі

Байти надходять по TCP сегментами (~1460 байт – це MSS), а парсер споживає їх із read-буфера порціями (часто описують близько 16 КБ – це розмір TLS-record, не «TCP-чанк»). Парсер не тримає main thread весь час: він працює чанками і періодично віддає main thread іншим задачам – подіям, compositing – через планувальник. Інакше довгий HTML підвісив би сторінку.

```
час →

Network:        [█chunk1█]   [█chunk2█]   [█chunk3█]   [█chunk4█]
                     │            │            │            │
Main thread:   [tok+tree──tok+tree──SCRIPT░░░░░░░░──tok+tree──tok+tree]
                                      ▲
                                      блок на 200мс
Preload                               └──▶ сканує HTML попереду й
scanner:                                   качає main.css, app.js, hero.jpg
```

Коли main thread стоїть на sync-скрипті, токенізація і tree construction стоять разом з ним – вони на тому ж thread. Працює лише preload scanner: він уже просканував HTML нижче й качає знайдені там ресурси, поки парсер чекає.

Звідси неінтуїтивна річ: поки DOM застряг на скрипті, мережа не простоює. Ресурси під скриптом знаходить не tree construction (вона стоїть), а preload scanner – окремим проходом по сирих байтах.

### Speculative parsing і document.write

Поки main thread тримає виконання скрипта, preload scanner біжить попереду по сирому HTML і, коли бачить `<link rel="stylesheet">`, `<script src>`, `<img>`, каже мережі качати ці ресурси наперед. Це і є спекулятивне завантаження: окремий легкий прохід, не основний парсер – той стоїть разом зі скриптом (детальніше – у розділі 4).

І тут є болюча точка – виклик `document.write` зі sync-скрипта (той самий легасі-API синхронної ін'єкції HTML, що згадується у розділі 3):

1. Скрипт вставляє новий HTML-текст у поточну позицію парсера.
2. Уся спекулятивна робота, яку scanner зробив нижче цієї точки, стає **невалідною**.
3. Парсер мусить перечитати потік від місця інжекції заново.
4. Усі preload-сигнали, які вже були для зачищеного куска, – даремна робота. Шкода CPU, шкода network.

Сценарій-приклад: sync-скрипт у `<head>` через цей виклик додає ще один `<link rel="stylesheet">` перед уже задекларованими `main.css` і `app.js`. Preload scanner уже знайшов ці двоє, надіслав preload-команди. Інжекція інвалідує цей хвіст – спекулятивна робота викидається, парсер перечитує. На повільному з'єднанні це 100–300 мс втрат на рівному місці.

Саме тому Chrome з 2016 року (M54) блокує такі parser-blocking виклики cross-site (різний eTLD+1) на 2G, для некешованих скриптів у top-frame (інтервенція «Slow document write»).

### Що блокує парсер і чому

Зіставимо все докупи. Точок, де конвеєр може зупинитись, насправді небагато:

**1. Sync `<script>` без `defer`/`async`.** Tree construction зупиняється на тегу `<script>` до завантаження і виконання файла, бо скрипт може виконати синхронну ін'єкцію HTML і змінити структуру нижче. Preload scanner при цьому біжить попереду й качає ресурси нижче – мережа не простоює. Деталі – розділ 3.

**2. Sync-скрипт після незавантаженого `<link rel="stylesheet">`.** Скрипт може запитати computed styles, а CSSOM ще не готовий. Main thread чекає одночасно і на CSS, і на скрипт. Preload scanner все одно йде попереду. Деталі – розділ 2.

**3. Синхронна ін'єкція HTML зі sync-скрипта (`document.write`).** Інвалідує speculation, як описано вище. На low-end чи slow network – головний убивця перформансу з тих, що залишилися у вільному обігу.

**4. Пустий receive buffer.** Природний backpressure: парсер чекає, поки прилетять байти. Обмежує швидкість мережі, а не парсер.

**Чого парсер НЕ блокує** (типові помилки в інтуїції):

- `<link rel="stylesheet">` блокує **render**, але **не парсер**. DOM-дерево росте далі.
- `<script defer>` не блокує нічого. Виконається після завершення парсингу.
- `<script async>` не блокує парсер до моменту прибуття файла; коли прийшов – блокує до завершення виконання (деталі – розділ 3).
- Картинки, шрифти, відео – нічого не блокують. Їх завантаженням займається preload scanner, але парсер їх не чекає.

### Tree construction і стек відкритих елементів

Tree construction – алгоритм поверх **стека відкритих елементів**:

- `StartTag` → push на стек, новий вузол стає дочірнім до вершини.
- `EndTag` для звичайного (блокового) елемента → парсер закриває все між ним і вершиною. Тут і ховається error recovery.
- `EndTag` для formatting-тега не на вершині → не просте pop, а *adoption agency algorithm* (див. нижче), який переносить вміст, а не викидає його.

Звідси та сама «несподівана структура в DevTools». Приклад – забутий закриваючий тег:

```html
<p>Перший <b>жирний</b>
<p>Другий
```

Парсер при зустрічі другого `<p>` дивиться на стек: бачить, що перший `<p>` уже там – і **неявно** його закриває. Підсумок:

```
<p>
  Перший
  <b>жирний</b>
</p>
<p>
  Другий
</p>
```

Жодного warning у консолі. Просто DOM, який ви не писали.

Для misnested formatting tags типу `<b><i></b></i>` працює окремий *adoption agency algorithm* – у специфікації він розписаний на кілька сторінок псевдокоду. Для повсякденного коду досить знати: `<b>` у такій ситуації може створитись у DOM двічі, з усіма наслідками для CSS-селекторів.

### Дрібниці

- Без `<!DOCTYPE html>` вмикається [quirks mode](https://developer.mozilla.org/en-US/docs/Glossary/Quirks_Mode) – набір legacy-сумісностей у CSS-моделі й layout. Швидкий тест: `document.compatMode === "BackCompat"` означає, що ви туди потрапили (`CSS1Compat` – ні; щоправда, no-quirks і limited-quirks він не розрізняє).
- `<html>`, `<head>`, `<body>` браузер додасть сам, навіть якщо ви їх не писали.
- `<template>` – інертний контейнер: вміст парситься, але не рендериться, ресурси не завантажуються, і у звичайний DOM-обхід не потрапляє. Доступ – через `template.content` (`DocumentFragment`).
- На `<svg>` і `<math>` парсер перемикається у [foreign content режим](https://html.spec.whatwg.org/multipage/parsing.html#parsing-main-inforeign): case-sensitive теги, namespaces, самозакриття.
- `<meta charset>` має стояти в перших ~1024 байтах `<head>`. Якщо браузер уже вгадав кодування, а пізній `<meta charset>` його спростував – парсер перечитує документ заново з правильним кодуванням. Зайва робота на рівному місці.
- `<base href>` змінює резолвинг усіх відносних URL на сторінці, зокрема й тих, що знаходить preload scanner. Один `<base>`, якомога раніше.

> **Метрики:**
> - `domInteractive` у Navigation Timing – момент, коли DOM готовий і всі sync-скрипти виконались. Різниця `domInteractive - responseEnd` ≈ «час парсингу плюс блокуючих скриптів».
> - `document.compatMode === "BackCompat"` – швидкий тест на quirks mode.
> - `PerformanceObserver({type: 'longtask'})` ловить chunks парсингу і handlers довші за 50 мс – головний сигнал, що щось не так на low-end пристроях.

---

## 2. CSSOM

CSSOM – це дерево всіх стилів сторінки (як DOM, тільки для CSS). Render tree, який браузер врешті малює, = DOM + CSSOM, тому без готового CSSOM немає й paint (сам render tree – у наступній частині). CSS завантажується по мережі паралельно з HTML. А ось парсинг отриманого CSS у CSSOM іде на main thread – там же, де токенізація і tree construction (див. розділ 1). Тобто CSSOM-парсинг, побудова DOM і виконання скриптів змагаються за один thread; паралельно з ними працює лише мережа (качає файли) і preload scanner (знаходить, що качати).

Найголовніше правило про CSS: він блокує перший paint, але не блокує парсинг DOM. Малювати без фінальних стилів браузер не буде, а от DOM-дерево росте далі, поки CSS летить.

Декілька болючих моментів.

Якщо `<script>` йде після `<link rel="stylesheet">`, який ще не завантажений, скрипт чекає. Логіка проста: скрипт може запитати computed styles, а їх ще нема. На практиці це часта причина waterfall в `<head>`.

У термінах розділу 1: tree construction на main thread зупиняється на тегу `<script>`, sync-скрипт чекає на CSSOM, CSSOM-парсинг чекає на network. Preload scanner усе одно йде попереду – це частково гасить waterfall, але не повністю.

`@import` всередині CSS – серійний waterfall. Перший файл качається, парситься, в ньому `@import`, качаємо другий, в ньому ще один. У продакшені їх не повинно бути.

`media="print"` – старий фокус, який досі працює: CSS завантажиться з низьким пріоритетом, але не блокує screen-рендер. Preload scanner його зазвичай не підхоплює, тож fetch пізній (сучасна альтернатива – `fetchpriority`). Корисно для друку чи стилів, які вмикаєте через JS пізніше.

Critical CSS inline в `<head>` прибирає round-trip для above-the-fold стилів, решту виносимо окремим файлом і завантажуємо неблокуюче. Працює, але руками виходить код, який потім важко підтримувати. На щастя, сучасні фреймворки беруть це на себе: Nuxt інлайнить CSS відрендерених на сервері компонентів через [`features.inlineStyles`](https://nuxt.com/docs/4.x/guide/going-further/features#inlinestyles) (дефолт – функція `(id) => id.includes('.vue')`, лише Vite). Astro за замовчуванням інлайнить стилі менші за ~4 КБ ([`build.inlineStylesheets: 'auto'`](https://docs.astro.build/en/reference/configuration-reference/#buildinlinestylesheets)), більші лишає окремим файлом. SvelteKit – [`kit.inlineStyleThreshold`](https://svelte.dev/docs/kit/configuration#inlineStyleThreshold) (дефолт `0`, тобто інлайнинг вимкнено).

> **Метрики:** [FCP](https://web.dev/articles/fcp) (First Contentful Paint) залежить від render-blocking CSS напряму. У [`PerformanceResourceTiming`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceResourceTiming) поле [`renderBlockingStatus`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceResourceTiming/renderBlockingStatus) показує `"blocking"` для кожного блокуючого ресурсу. У DevTools колонка «Render Blocking» – те саме візуально.

---

## 3. Скрипти

Скрипти – головний спосіб зупинити tree construction. Preload scanner це не блокує – він іде попереду (див. розділ 1), і завдяки цьому критичні ресурси нижче скрипта вже починають завантажуватись. Але DOM далі тегу не росте до завершення виконання.

`<script>` без атрибутів – класика і найдорожчий варіант. Tree construction зупиняється на тегу до завантаження і виконання файла. Все нижче чекає. Inline-скрипт без атрибутів зупиняє так само, тільки без мережі – і без переваги preload scanner. Не використовуйте без причини.

[`async`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script#async) – fetch паралельно, виконання щойно файл прийде. Гарантій порядку немає. Підходить error-tracking (Sentry, Bugsnag), де треба стартувати рано. Для звичайної аналітики краще `defer` або load-on-idle – gtag.js важить ~110+ КБ, і його парсинг та виконання на mobile б'ють по TBT/LCP, а тих, хто одразу закриває сторінку, ви все одно майже не зловите.

[`defer`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script#defer) – fetch паралельно, виконання після парсингу всього HTML, у порядку появи в документі. Це той дефолт, який повинен стояти на майже всіх скриптах сторінки. Якщо у вашому проєкті більшість скриптів без `defer`, є над чим попрацювати в перший же спринт.

Усе це наочніше на таймлайні:

```
        парсинг HTML ─────────────────────────────▶
sync:   ──парс──[fetch+exec]──парс────────────────   блокує парсинг
async:  ──парс───────────────[exec]──парс─────────   exec будь-коли, рве парсинг
            └─fetch───────────┘
defer:  ──парс─────────────────────────────[exec]─   після парсингу, у порядку
            └─fetch───────────┘
```

[`type="module"`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) – defer за замовчуванням, плюс граф імпортів, плюс кожен модуль завантажується один раз. Поряд з ним [`type="importmap"`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script/type/importmap), який має йти **до** першого module-скрипта, інакше карту імпортів просто проігнорують. (Історично карта одна на документ; нові Chromium уже підтримують кілька, що зливаються.)

І окремо – [`document.write`](https://developer.mozilla.org/en-US/docs/Web/API/Document/write), старий синхронний API ін'єкції HTML. Чому це окремий клас зла на рівні tokenizer, описано в розділі 1 (інвалідація speculation). Тут одна додаткова дрібниця: з async-скрипта (або після завершення парсингу) виклик стирає документ. Chrome блокує parser-blocking варіант cross-site на 2G (некешований, top-frame). Якщо побачите такий код, виносьте без жалю.

> **Метрики:** sync-скрипти ловить [Long Tasks API](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceLongTaskTiming) (`PerformanceObserver({type: 'longtask'})`, поріг 50 мс); для атрибуції, який саме скрипт винен, точніший [Long Animation Frames](https://developer.chrome.com/docs/web-platform/long-animation-frames) (LoAF). Це основний внесок у [TBT](https://web.dev/articles/tbt) і пізніше у [INP](https://web.dev/articles/inp). Defer затримує `domContentLoadedEventStart`. Якщо DCL стрибнув, копайте в defer-чанках.

---

## 4. Preload scanner

Як описано в розділі 1, поки основний парсер заблокований sync-скриптом, паралельно працює preload scanner – окремий легкий спекулятивний прохід по сирому HTML. Він шукає теги з ресурсами (`<script>`, `<link>`, `<img>`, preload-и) і одразу запускає завантаження по знайдених URL.

Без нього кожен sync-скрипт серіалізував би все. З ним ресурси летять паралельно, поки парсер чекає.

Важливо: сканер бачить тільки HTML-теги. Він не побачить ресурси, додані через JS (`createElement` і `appendChild`), CSS `background-image` та шрифти з `@font-face` (їхні URL у CSS, не в HTML).

Звідси рекомендація, яку команди регулярно ігнорують: критичні ресурси (LCP-картинка, основний шрифт, ключовий CSS) пишіть тегами в HTML. Динамічна ін'єкція з JS – гарантований програш у waterfall на 100–300 мс. Це класичне місце, де SPA-команди втрачають LCP «ні з того, ні з сього».

Альтернативний механізм, який працює ще до того, як парсер взагалі побачить HTML – `103 Early Hints`. Сервер шле preload/preconnect-натяки разом з interim-відповіддю, поки ще генерує фінальний HTML. Браузер починає завантажувати критичні ресурси паралельно з рендерингом сервера. Детальніше – у частині 1.

> **Метрики:** [`initiatorType`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceResourceTiming/initiatorType) у Resource Timing – ідеальний детектор. `link`, `img`, `script` для статики означає, що тег був у HTML і сканер знайшов одразу. `script` чи `fetch` для статики означає, що ресурс додав JS. У DevTools колонка Initiator показує те саме.

---

## 5. Пріоритети

Браузер сам розставляє пріоритети завантаження, і робить це більш-менш притомно. Шкала з п'яти рівнів, видно у DevTools → Network → Priority:

- Highest: HTML, render-blocking CSS у `<head>`, preload `as="style"`
- High: sync-скрипти у `<head>`, шрифти / preload `as="font"`, картинки у viewport (після layout), async fetch/XHR
- Medium: перші 5 великих картинок (>10000px², до layout), preload `as="script"`
- Low: картинки за замовчуванням (до бусту), async/defer-скрипти, пізні блокуючі скрипти у `<body>`, media (відео/аудіо)
- Lowest: картинки поза viewport, prefetch, favicon, CSS з media-mismatch

Часто цього вистачає. Але є випадки, де браузер не може здогадатись.

[`<link rel="preload" as="...">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preload) для явного завантаження. Тут дві пастки. Перша: помилковий `as`. Preload не співпаде з реальним типом, і файл завантажиться двічі. Друга: забутий `crossorigin` для шрифтів. Реальний запит з CORS, preload без, дві гілки, файл вдруге.

[`<link rel="modulepreload">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/modulepreload) – preload модуля; у браузерах, що це реалізують (напр. Chrome), – і його статичних залежностей (специфікація цього не гарантує). Корисно для критичного entry point.

[`fetchpriority="high"`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/fetchpriority) на LCP-картинці. Це найрозповсюдженіше застосування і часто єдине, що реально треба зробити вручну. За замовчуванням картинки нижче fold отримують Low. LCP-картинка туди потрапляє часто, бо браузер не знає, що вона головна, поки не зробить layout. `fetchpriority="high"` вирішує це за один рядок.

`fetchpriority="low"` на below-fold ресурсах, щоб не конкурували з критичним.

Поряд із `preload` – ще три resource hints. [`preconnect`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preconnect) заздалегідь відкриває з'єднання (DNS + TCP + TLS) до критичного origin, щоб перший запит до нього не чекав хендшейк; ставте на 2–3 найважливіші домени, не більше – кожне відкрите з'єднання коштує. [`dns-prefetch`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/dns-prefetch) – дешевший варіант, лише DNS-резолв. [`prefetch`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/prefetch) – навпаки, найнижчий пріоритет: качає ресурс для *наступної* навігації наперед, а не для поточної сторінки.

Поряд – два механізми, які варто пам'ятати. [SRI](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity) (`integrity="sha384-..."`) перевіряє хеш, для third-party CDN базова гігієна. [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy) (`script-src`, `style-src`, `img-src`) налаштовується на сервері, inline без nonce/hash блокується.

> **Метрики:** [`PerformanceObserver({type: 'largest-contentful-paint'})`](https://developer.mozilla.org/en-US/docs/Web/API/LargestContentfulPaint). Entry містить `element`, `url`, `loadTime`, `renderTime`. Великий `loadTime` означає, що ресурс отримав низький пріоритет і прийшов пізно. І не забуваємо: [TTFB](https://web.dev/articles/ttfb) з частини 1 залишається базою. Жодний `fetchpriority` не врятує, якщо TTFB вже 800 мс.

---

## 6. Шрифти

Шрифт важить 50–100 КБ і здається дрібницею. Дарма: якщо LCP-елемент текстовий, у статистиці LCP/CLS найчастіше винні саме шрифти.

Деталь, яку легко пропустити: [`@font-face`](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face) сам по собі нічого не завантажує. Браузер тригерить fetch шрифта тільки коли в layout з'являється DOM-вузол, що використовує цей font + weight + style. Тобто шрифт, описаний у CSS, завантажиться лише після обчислення стилів, не раніше.

З одного боку, добре. Не завантажуємо те, що не використовуємо. З іншого, preload scanner шрифтів у CSS не бачить, fetch стартує пізно. Тому для критичних шрифтів `<link rel="preload" as="font" crossorigin>` без варіантів.

[`font-display`](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display) керує тим, що бачить користувач, поки шрифт летить (числа нижче – типові дефолти Chrome; специфікація задає лише відносні періоди block/swap/failure):

- `swap` – fallback показуємо одразу, потім підмінюємо на основний. Видимо, але layout стрибає (CLS).
- `block` – ~3 с невидимий текст; якщо шрифт не встиг, показуємо fallback, але справжній шрифт усе одно підміниться, коли прийде (swap-період нескінченний).
- `optional` – якщо шрифт уже в кеші, беремо. Якщо ні, fallback назавжди для цієї сесії. Без CLS, але і без шрифта на першому візиті.
- `fallback` – ~100 мс невидимого тексту, потім fallback; якщо шрифт прийде у коротке вікно (~3 с), підміниться, інакше fallback лишається назавжди.

Робоче правило: `swap` для основного шрифта, `optional` для декоративних. Решту по ситуації.

Ще [`unicode-range`](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/unicode-range). Розбиваєте шрифт на підмножини (кирилиця, латиниця, додаткові символи), і браузер не завантажує ті, які на сторінці не використовуються. Google Fonts робить це за вас, для self-hosted – бібліотека `fonttools` у CI.

> **Метрики:** [CLS](https://web.dev/articles/cls) PerformanceObserver, поле `sources` показує елементи, що зсунулись. Класична картина – fallback з іншою шириною, потім основний шрифт, layout стрибає. Якщо LCP-елемент текстовий, `font-display` напряму впливає на [LCP](https://web.dev/articles/lcp) timing.

---

## 7. Зображення

Зазвичай найважчий актив на сторінці – і оптимізується кількома атрибутами.

[`loading="lazy"`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#loading) – браузер не завантажує, поки картинка далеко від viewport; нативний lazy реалізований у самому движку через viewport-distance пороги (концептуально схоже на IntersectionObserver). Обов'язково для всього below-fold. [`decoding="async"`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#decoding) – не блокує main thread синхронним commit при появі картинки (саме декодування й так іде поза main thread у сучасних браузерах). [`srcset` плюс `sizes`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#srcset) – браузер вибирає розмір під viewport і DPR; без `sizes` він припускає `100vw` і може взяти завеликий варіант (стосується `w`-дескрипторів). [`<picture>` плюс `<source>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/picture) – вибір формату: avif → webp → jpeg як fallback, економія 30–50% на сучасних форматах. `fetchpriority="high"` на LCP-картинці.

Класична помилка: `loading="lazy"` на LCP-картинці. Виглядає логічно («ну ж бо, не блокувати»), а LCP стрибає на 200–400 мс, бо картинка завантажується з низьким пріоритетом і пізніше за решту критичних ресурсів. LCP-картинка має бути або взагалі без `loading`, або з `loading="eager"` плюс `fetchpriority="high"`.

Друга класична помилка – `<img>` без `width` і `height`. Поки картинка не прийшла, браузер не знає її розмірів і не резервує місце; коли вона завантажується, контент під нею стрибає (CLS). Задавайте `width`/`height` (або `aspect-ratio` у CSS) – браузер порахує співвідношення і притримає місце ще до завантаження. З `srcset` це теж працює: атрибути задають пропорцію, а не фіксований розмір.

> **Метрики:** LCP entry містить `url`, `element`, `loadTime`, `renderTime`. Великий `loadTime` – картинка пізно прилетіла. Розрив `renderTime - loadTime` – картинка чекає, поки браузер її намалює (шрифт не готовий, main thread зайнятий, що завгодно).

---

## 8. DOMContentLoaded і load

Дві події, які регулярно плутають.

[`DOMContentLoaded`](https://developer.mozilla.org/en-US/docs/Web/API/Document/DOMContentLoaded_event) – tree construction завершено і всі `defer`/module-скрипти виконались. Картинок, фреймів, async-скриптів не чекає. Це момент, коли DOM «готовий» з точки зору структури.

[`load`](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event) – все інше теж завантажилось: картинки, iframe-и, stylesheets, async- і defer-скрипти. Шрифти `load` НЕ чекає – вони вантажаться незалежно (керуються `font-display`); для них є `document.fonts.ready`. Може бути значно пізніше за DCL.

Дві дрібниці, які стріляють у ногу. Defer-скрипт, що чекає stylesheet (бо CSS блокує його виконання), затримує `DOMContentLoaded`. Якщо DCL виглядає аномально пізно, шукайте цю залежність. І [`document.readyState`](https://developer.mozilla.org/en-US/docs/Web/API/Document/readyState) має три стани: `loading` → `interactive` (DOM розпарсено; одразу по тому виконуються defer/module-скрипти і настає DCL) → `complete` (перед `load`).

### Куди їх вішають у житті

DCL – улюблений тригер для гідратації React/Vue/Svelte (хоча сучасні фреймворки дрейфують у бік `requestIdleCallback`/scheduler) і для запуску аналітики: GA4, Mixpanel, Sentry – їхні snippets чекають саме на цю подію. У legacy на jQuery весь init-код висів на `$(document).ready()`, що під капотом – той самий DCL (jQuery додатково запускає й пізно навішані хендлери, чого raw-слухач DCL не робить).

`load` – рідше: lazy-завантаження below-fold секцій, ініціалізація віджетів, які залежать від картинок/iframe-ів.

Класична пастка – запхати в DCL-handler важку синхронну роботу:

```js
document.addEventListener('DOMContentLoaded', () => {
  const state = JSON.parse(localStorage.getItem('app-state'));  // 200KB парсу
  document.querySelectorAll('button').forEach(initButton);      // 500 нод × handler
  analytics.init({ trackAll: true });                           // ще 50мс
});
```

Результат: `domContentLoadedEventEnd - domContentLoadedEventStart` стрибає з 1–2 мс до 200+ мс. Main thread заблокована весь цей час. INP першої взаємодії – двозначний, бо click потрапляє у середину цього блоку. Винесіть важке у `requestIdleCallback`, окремий `defer`-скрипт, або взагалі за межі DCL.

> **Метрики:** `domContentLoadedEventStart`/`domContentLoadedEventEnd`. Різниця показує час DCL-handlers. Туди часто пхають аналітику й важку ініціалізацію, що б'є по INP.

---

## Чеклист

**Парсинг (розділ 1)**
- [ ] `<!DOCTYPE html>` присутній (інакше – quirks mode)
- [ ] `<meta charset>` у перших ~1KB `<head>` (інакше – перечитування документа)
- [ ] Жодного `document.write` у sync-скриптах (інвалідує speculation)
- [ ] У моніторингу аномалія: `document.compatMode === "BackCompat"`

**CSS (розділ 2)**
- [ ] Critical CSS inline у `<head>`, решта неблокуюче
- [ ] Без `@import`-ланцюжків
- [ ] Sync `<script>` не йде після незавантаженого `<link rel="stylesheet">`
- [ ] `media="print"` для деферного CSS, що використовується пізніше

**Скрипти (розділ 3)**
- [ ] `defer` за замовчуванням; sync – тільки з причиною
- [ ] `<script type="importmap">` перед першим module-скриптом
- [ ] `async` – лише для error-tracking (Sentry/Bugsnag); аналітику – `defer` або load-on-idle
- [ ] Жодного `document.write`

**Preload scanner і Early Hints (розділ 4)**
- [ ] LCP-картинка, основний шрифт, ключовий CSS – теги в HTML, не JS-ін'єкція
- [ ] `103 Early Hints` на сервері/CDN, якщо TTFB > 200 мс

**Пріоритети (розділ 5)**
- [ ] `fetchpriority="high"` на LCP-картинці
- [ ] `fetchpriority="low"` на below-fold ресурсах
- [ ] `preconnect` для 2–3 критичних origin (не більше)
- [ ] У `preload` перевірено `as` (інакше подвійне завантаження)
- [ ] SRI для third-party CDN-скриптів

**Шрифти (розділ 6)**
- [ ] `<link rel="preload" as="font" crossorigin>` для критичних
- [ ] `font-display: swap` або `optional`
- [ ] `unicode-range` (Google Fonts автоматично, self-hosted – `fonttools` у CI)

**Зображення (розділ 7)**
- [ ] `loading="lazy"` на below-fold, **не** на LCP
- [ ] `decoding="async"` всюди, де є `<img>`
- [ ] `width`/`height` (або `aspect-ratio`) на кожному `<img>` – проти CLS
- [ ] `srcset` + `sizes`, або `<picture>` для format negotiation (avif/webp)
- [ ] LCP-картинка з `loading="eager"` + `fetchpriority="high"`

**Події життєвого циклу (розділ 8)**
- [ ] DCL-handlers легкі (< 50 мс); важку ініціалізацію – у `requestIdleCallback`
- [ ] Defer-скрипти не залежать від stylesheet (інакше DCL стрибає)

**Моніторинг (RUM)**
- [ ] `domInteractive`, `domContentLoadedEventEnd` логуються в аналітику
- [ ] `PerformanceObserver({type: 'longtask'})` (точніше – Long Animation Frames) ловить sync-скрипти й важкі handlers
- [ ] LCP, FCP, CLS, INP – основа Core Web Vitals для RUM (good p75: LCP ≤ 2.5 с, INP ≤ 200 мс, CLS ≤ 0.1)
- [ ] `compatMode === "BackCompat"` – алерт на quirks mode

---

## Шпаргалка: resource hints

| Hint | Що робить | Коли застосовувати |
|------|-----------|--------------------|
| `preload` | качає ресурс для *поточної* сторінки з високим пріоритетом | критичний шрифт, LCP-картинка, ключовий CSS/JS |
| `modulepreload` | те саме для ES-модуля (+ залежності в Chrome) | критичний module entry point |
| `preconnect` | відкриває з'єднання наперед (DNS + TCP + TLS) | 2–3 критичні сторонні origin |
| `dns-prefetch` | лише DNS-резолв, дешевше за `preconnect` | домени, потрібні трохи згодом |
| `prefetch` | качає ресурс для *наступної* навігації, найнижчий пріоритет | імовірна наступна сторінка |
| `103 Early Hints` | preload/preconnect ще до фінального HTML | критичні ресурси, поки сервер думає (TTFB > 200 мс) |

---

## Що далі?

DOM є. CSSOM є. Скрипти виконались. Але пікселів ще немає. Браузер має побудувати render tree, порахувати layout, розкласти на шари і нарешті намалювати. Про це в наступній частині.
