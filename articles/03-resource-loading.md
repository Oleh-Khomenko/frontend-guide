# Оптимізація фронтенду від першого кліку – Частина 3

*Пріоритети, шрифти і зображення: хто виграє боротьбу за мережу і коли сторінка нарешті щось покаже*

У [першій частині](https://medium.com/@olehhomenko12/frontend-optimization-from-first-click-part-1-4be1ae9a6637) ми пройшли шлях від кліку до першого байта HTML. У другій – браузер розпарсив цей HTML: токенізація, tree construction, CSSOM, скрипти і preload scanner. <!-- TODO: лінк на Частину 2 після публікації -->

DOM будується, але сторінка все ще порожня: CSS, шрифти й картинки змагаються за мережу, і від того, хто виграє цю боротьбу, залежить, коли користувач побачить контент.

Далі по порядку: пріоритети завантаження і resource hints, шрифти, зображення і події життєвого циклу – `DOMContentLoaded` і `load`.

---

## 1. Пріоритети

Браузер сам розставляє пріоритети завантаження, і робить це більш-менш притомно. Шкала з п'яти рівнів, видно у DevTools → Network → Priority:

- Highest: HTML, render-blocking CSS у `<head>`, preload `as="style"`
- High: sync-скрипти у `<head>`, шрифти / preload `as="font"`, картинки у viewport (після layout), async fetch/XHR
- Medium: перші 5 великих картинок (>10000px², до layout), preload `as="script"`
- Low: картинки за замовчуванням (до бусту), async/defer-скрипти, пізні блокуючі скрипти у `<body>`, media (відео/аудіо)
- Lowest: картинки поза viewport, prefetch, favicon, CSS з media-mismatch

Пріоритет – це не просто порядок у списку. Запити з Low браузер до старту рендера взагалі тримає в черзі (білий сегмент Queueing у waterfall), поки летять рендер-блокуючі ресурси. На скріншоті Network у частині 2 це видно наживо: defer-скрипт запитано разом з усіма, а смугу він отримав останнім.

Часто цього вистачає. Але є випадки, де браузер не може здогадатись.

[`<link rel="preload" as="...">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preload) для явного завантаження. Тут дві пастки. Перша: помилковий `as`. Preload не співпаде з реальним типом, і файл завантажиться двічі. Друга: забутий `crossorigin` для шрифтів. Реальний запит з CORS, preload без, дві гілки, файл вдруге.

[`<link rel="modulepreload">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/modulepreload) – preload модуля; у браузерах, що це реалізують (напр. Chrome), – і його статичних залежностей (специфікація цього не гарантує). Корисно для критичного entry point.

[`fetchpriority="high"`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/fetchpriority) на LCP-картинці. Це найрозповсюдженіше застосування і часто єдине, що реально треба зробити вручну. За замовчуванням картинки нижче fold отримують Low. LCP-картинка туди потрапляє часто, бо браузер не знає, що вона головна, поки не зробить layout. `fetchpriority="high"` вирішує це за один рядок.

`fetchpriority="low"` на below-fold ресурсах, щоб не конкурували з критичним.

Поряд із `preload` – ще три resource hints. [`preconnect`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preconnect) заздалегідь відкриває з'єднання (DNS + TCP + TLS) до критичного origin, щоб перший запит до нього не чекав хендшейк; ставте на 2–3 найважливіші домени, не більше – кожне відкрите з'єднання коштує. [`dns-prefetch`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/dns-prefetch) – дешевший варіант, лише DNS-резолв. [`prefetch`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/prefetch) – навпаки, найнижчий пріоритет: качає ресурс для *наступної* навігації наперед, а не для поточної сторінки.

Поряд – два механізми, які варто пам'ятати. [SRI](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity) (`integrity="sha384-..."`) перевіряє хеш, для third-party CDN базова гігієна. [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy) (`script-src`, `style-src`, `img-src`) налаштовується на сервері, inline без nonce/hash блокується.

> **Метрики:** [`PerformanceObserver({type: 'largest-contentful-paint'})`](https://developer.mozilla.org/en-US/docs/Web/API/LargestContentfulPaint). Entry містить `element`, `url`, `loadTime`, `renderTime`. Великий `loadTime` означає, що ресурс отримав низький пріоритет і прийшов пізно. І не забуваємо: [TTFB](https://web.dev/articles/ttfb) з [частини 1](https://medium.com/@olehhomenko12/frontend-optimization-from-first-click-part-1-4be1ae9a6637) залишається базою. Жодний `fetchpriority` не врятує, якщо TTFB вже 800 мс.

---

## 2. Шрифти

Шрифт важить 50–100 КБ і здається дрібницею. Дарма: якщо LCP-елемент текстовий, у статистиці LCP/CLS найчастіше винні саме шрифти.

Деталь, яку легко пропустити: [`@font-face`](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face) сам по собі нічого не завантажує. Браузер тригерить fetch шрифта тільки коли в layout з'являється DOM-вузол, що використовує цей font + weight + style. Тобто шрифт, описаний у CSS, завантажиться лише після обчислення стилів, не раніше.

З одного боку, добре. Не завантажуємо те, що не використовуємо. З іншого, preload scanner шрифтів у CSS не бачить (його URL живуть у CSS, не в HTML – чому це важливо, ми розбирали в частині 2), fetch стартує пізно. Тому для критичних шрифтів `<link rel="preload" as="font" crossorigin>` без варіантів.

[`font-display`](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display) керує тим, що бачить користувач, поки шрифт летить (числа нижче – типові дефолти Chrome; специфікація задає лише відносні періоди block/swap/failure):

- `swap` – fallback показуємо одразу, потім підмінюємо на основний. Видимо, але layout стрибає (CLS).
- `block` – ~3 с невидимий текст; якщо шрифт не встиг, показуємо fallback, але справжній шрифт усе одно підміниться, коли прийде (swap-період нескінченний).
- `optional` – якщо шрифт уже в кеші, беремо. Якщо ні, fallback назавжди для цієї сесії. Без CLS, але і без шрифта на першому візиті.
- `fallback` – ~100 мс невидимого тексту, потім fallback; якщо шрифт прийде у коротке вікно (~3 с), підміниться, інакше fallback лишається назавжди.

Робоче правило: `swap` для основного шрифта, `optional` для декоративних. Решту по ситуації.

У `swap` лишається вроджена проблема: fallback і основний шрифт мають різні метрики, тож у момент підміни текст «передзбирається» – і це чистий CLS. Сучасний фікс – **метрично підігнаний fallback**: дескриптори [`size-adjust`](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/size-adjust), [`ascent-override`](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/ascent-override), `descent-override` і `line-gap-override` підганяють локальний шрифт під метрики основного, і підміна проходить без зсуву. Цифри не підбирають руками – їх рахують інструменти: [fontaine](https://github.com/unjs/fontaine) (Nuxt-екосистема), [Capsize](https://seek-oss.github.io/capsize/), а `next/font` робить це автоматично.

Ще [`unicode-range`](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/unicode-range). Розбиваєте шрифт на підмножини (кирилиця, латиниця, додаткові символи), і браузер не завантажує ті, які на сторінці не використовуються. Google Fonts робить це за вас, для self-hosted – бібліотека `fonttools` у CI.

Усе разом – робочий мінімум для критичного self-hosted шрифта:

```html
<!-- у <head>: preload, бо preload scanner шрифтів у CSS не бачить -->
<link rel="preload" as="font" type="font/woff2"
      href="/fonts/inter-latin.woff2" crossorigin>
```

```css
/* основний шрифт: одна підмножина = один @font-face */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-latin.woff2') format('woff2');
  font-weight: 400 700;            /* variable-діапазон або конкретна вага */
  font-display: swap;
  unicode-range: U+0000-00FF;      /* латиниця; кирилиця – окремим файлом */
}

/* fallback, метрично підігнаний під Inter: swap без CLS */
@font-face {
  font-family: 'Inter Fallback';
  src: local('Arial');
  size-adjust: 107.4%;             /* цифри генерує fontaine/Capsize */
  ascent-override: 90.2%;
  descent-override: 22.48%;
  line-gap-override: 0%;
}

body {
  font-family: 'Inter', 'Inter Fallback', sans-serif;
}
```

> **Метрики:** [CLS](https://web.dev/articles/cls) PerformanceObserver, поле `sources` показує елементи, що зсунулись. Класична картина – fallback з іншою шириною, потім основний шрифт, layout стрибає. Якщо LCP-елемент текстовий, `font-display` напряму впливає на [LCP](https://web.dev/articles/lcp) timing.

---

## 3. Зображення

Зазвичай найважчий актив на сторінці – і оптимізується кількома атрибутами:

- [`loading="lazy"`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#loading) – браузер не завантажує, поки картинка далеко від viewport; нативний lazy реалізований у самому рушії через viewport-distance пороги (концептуально схоже на IntersectionObserver). Обов'язково для всього below-fold.
- [`decoding="async"`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#decoding) – не блокує main thread синхронним commit при появі картинки (саме декодування й так іде поза main thread у сучасних браузерах).
- [`srcset` плюс `sizes`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#srcset) – браузер вибирає розмір під viewport і DPR; без `sizes` він припускає `100vw` і може взяти завеликий варіант (стосується `w`-дескрипторів).
- [`<picture>` плюс `<source>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/picture) – вибір формату: avif → webp → jpeg як fallback, економія 30–50% на сучасних форматах.
- `fetchpriority="high"` – на LCP-картинці (розділ 1).

Класична помилка: `loading="lazy"` на LCP-картинці. Виглядає логічно («ну ж бо, не блокувати»), а LCP стрибає на 200–400 мс, бо картинка завантажується з низьким пріоритетом і пізніше за решту критичних ресурсів. LCP-картинка має бути або взагалі без `loading`, або з `loading="eager"` плюс `fetchpriority="high"`.

Друга класична помилка – `<img>` без `width` і `height`. Поки картинка не прийшла, браузер не знає її розмірів і не резервує місце; коли вона завантажується, контент під нею стрибає (CLS). Задавайте `width`/`height` (або `aspect-ratio` у CSS) – браузер порахує співвідношення і притримає місце ще до завантаження. З `srcset` це теж працює: атрибути задають пропорцію, а не фіксований розмір.

Звідси два готові патерни, які закривають 90% випадків.

**LCP-картинка** – качаємо одразу, з максимальним пріоритетом, місце зарезервовано:

```html
<img src="/hero-1600.avif"
     srcset="/hero-800.avif 800w, /hero-1600.avif 1600w"
     sizes="100vw"
     width="1600" height="900"
     loading="eager"
     fetchpriority="high"
     alt="…">
```

**Below-fold картинка** – лінива, responsive, із fallback формату:

```html
<picture>
  <source type="image/avif"
          srcset="/photo-400.avif 400w, /photo-800.avif 800w"
          sizes="(max-width: 600px) 100vw, 50vw">
  <source type="image/webp"
          srcset="/photo-400.webp 400w, /photo-800.webp 800w"
          sizes="(max-width: 600px) 100vw, 50vw">
  <img src="/photo-800.jpg"
       width="800" height="600"
       loading="lazy"
       decoding="async"
       alt="…">
</picture>
```

Різниця між ними – і є вся секція в чотирьох атрибутах: `eager`+`fetchpriority` проти `lazy`+`decoding`.

> **Метрики:** LCP entry містить `url`, `element`, `loadTime`, `renderTime`. Великий `loadTime` – картинка пізно прилетіла. Розрив `renderTime - loadTime` – картинка чекає, поки браузер її намалює (шрифт не готовий, main thread зайнятий, що завгодно).

---

## 4. DOMContentLoaded і load

Дві події, які регулярно плутають.

[`DOMContentLoaded`](https://developer.mozilla.org/en-US/docs/Web/API/Document/DOMContentLoaded_event) – tree construction (частина 2) завершено і всі `defer`/module-скрипти виконались. Картинок, фреймів, async-скриптів не чекає. Це момент, коли DOM «готовий» з точки зору структури.

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

## Все разом: правильний `<head>`

Кожен рядок нижче – висновок одного з розділів цієї або попередніх частин. Це не догма, а скелет, з якого варто стартувати:

```html
<!DOCTYPE html>                                       <!-- без нього – quirks mode (ч. 2) -->
<html lang="uk">
<head>
  <meta charset="utf-8">                              <!-- у перших 1024 байтах (ч. 2) -->
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>…</title>

  <!-- з'єднання до критичного origin наперед: DNS + TCP + TLS (розділ 1) -->
  <link rel="preconnect" href="https://cdn.example.com" crossorigin>

  <!-- критичний шрифт: preload, бо preload scanner його в CSS не побачить (розділ 2) -->
  <link rel="preload" as="font" type="font/woff2"
        href="/fonts/inter-latin.woff2" crossorigin>

  <!-- critical CSS inline: перший paint без round-trip (ч. 2) -->
  <style>/* above-the-fold стилі */</style>

  <!-- решта CSS: блокує render, але не парсер (ч. 2) -->
  <link rel="stylesheet" href="/css/main.css">

  <!-- скрипти: defer за замовчуванням, ПІСЛЯ стилів, щоб не зловити CSSOM-чекання (ч. 2) -->
  <script src="/js/app.js" defer></script>
</head>
<body>
  <!-- LCP-картинка: тег у HTML, не JS-ін'єкція + явний пріоритет (розділи 1, 3) -->
  <img src="/hero-1600.avif" width="1600" height="900"
       fetchpriority="high" alt="…">
  …
</body>
</html>
```

Чого тут *немає* – так само важливо: жодного sync-скрипта, жодного `@import`, жодного `document.write`, аналітика не стоїть `async` у `<head>` (їй місце після `load` або on-idle).

---

## Чеклист

**Пріоритети (розділ 1)**
- ✅ `fetchpriority="high"` на LCP-картинці
- ✅ `fetchpriority="low"` на below-fold ресурсах
- ✅ `preconnect` для 2–3 критичних origin (не більше)
- ✅ У `preload` перевірено `as` (інакше подвійне завантаження)
- ✅ SRI для third-party CDN-скриптів

**Шрифти (розділ 2)**
- ✅ `<link rel="preload" as="font" crossorigin>` для критичних
- ✅ `font-display: swap` або `optional`
- ✅ Метрично підігнаний fallback (`size-adjust`/`ascent-override`) – fontaine, Capsize або `next/font`
- ✅ `unicode-range` (Google Fonts автоматично, self-hosted – `fonttools` у CI)

**Зображення (розділ 3)**
- ✅ `loading="lazy"` на below-fold, **не** на LCP
- ✅ `decoding="async"` всюди, де є `<img>`
- ✅ `width`/`height` (або `aspect-ratio`) на кожному `<img>` – проти CLS
- ✅ `srcset` + `sizes`, або `<picture>` для format negotiation (avif/webp)
- ✅ LCP-картинка з `loading="eager"` + `fetchpriority="high"`

**Події життєвого циклу (розділ 4)**
- ✅ DCL-handlers легкі (< 50 мс); важку ініціалізацію – у `requestIdleCallback`
- ✅ Defer-скрипти не залежать від stylesheet (інакше DCL стрибає)

**Моніторинг (RUM)**
- ✅ LCP, FCP, CLS, INP – основа Core Web Vitals для RUM (good p75: LCP ≤ 2.5 с, INP ≤ 200 мс, CLS ≤ 0.1)
- ✅ `domContentLoadedEventEnd - domContentLoadedEventStart` логується (час DCL-handlers)
- ✅ `PerformanceObserver({type: 'longtask'})` (точніше – Long Animation Frames) ловить важкі handlers

---

## Шпаргалка: resource hints

**`preload`** – качає ресурс для *поточної* сторінки з високим пріоритетом.
→ критичний шрифт, LCP-картинка, ключовий CSS/JS

**`modulepreload`** – те саме для ES-модуля (+ залежності в Chrome).
→ критичний module entry point

**`preconnect`** – відкриває з'єднання наперед (DNS + TCP + TLS).
→ 2–3 критичні сторонні origin

**`dns-prefetch`** – лише DNS-резолв, дешевше за `preconnect`.
→ домени, потрібні трохи згодом

**`prefetch`** – качає ресурс для *наступної* навігації, найнижчий пріоритет.
→ імовірна наступна сторінка

**`103 Early Hints`** – preload/preconnect ще до фінального HTML (частина 2).
→ критичні ресурси, поки сервер думає (TTFB > 200 мс)

---

## Що далі?

DOM є. CSSOM є. Скрипти виконались, ресурси прилетіли. Але пікселів ще немає. Браузер має побудувати render tree, порахувати layout, розкласти на шари і нарешті намалювати. Про це в наступній частині.
