# insales-yc-cdn-proxy

Обход блокировки CDN InSales через Яндекс CDN прокси + переписывание URL в DOM. Только шаблоны темы и YC - без доступа к API платформы.

> ⚠️ **Это костыль, а не production-ready решение.** В процессе внедрения у нас ломалась корзина (двойное добавление товара из-за двойной инициализации InSales API), возникали конфликты между кастомными виджетами и переписанными URL, некоторые виджеты пришлось переделывать под новую логику загрузки. У вас будет что-то своё - тема, виджеты и кастомный JS у всех разные. Подходите к внедрению итерационно, тестируйте на копии темы и обязательно мониторьте работоспособность корзины и ключевых сценариев покупки после каждого изменения. Для ежедневного автоматического мониторинга смотрите [ecom-tech-speed-checker](https://github.com/DmitriiDmallSick/ecom-tech-speed-checker).

---

## Проблема

Начиная с середины 2026 года `static.insales-cdn.com` (IP `87.242.124.98`) стал недоступен для значительной части пользователей российских провайдеров - МТС, Мегафон, Теле2 и других. Трафик обрывался на 7-м хопе внутри сети провайдера и до серверов InSales не доходил. Ни поддержка провайдера, ни поддержка InSales не могут это починить: пакеты не доходят до серверов платформы до того как InSales вообще их видит.

```text
Трассировка с МТС домашний → static.insales-cdn.com:
  6     6 ms    as208677.asbr.router [212.188.45.6]
  7     *   *   *   Request timed out.  ← обрыв здесь
  8-30  *   *   *   Request timed out.
```

### Что переставало работать

* **JS-библиотеки не загружались совсем** - jQuery, Splide, common.js, lazyload, bodyScrollLock, microAlert, micromodal, fslightbox и другие. Статус 0 (failed), таймаут 40-70 секунд на каждый файл
* **Корзина не работала** - добавить товар было невозможно, кнопки не реагировали
* **Слайдеры не инициализировались** - галерея товара, главный баннер, виджеты коллекций
* **Шрифты не грузились** - сайт отображался системным шрифтом вместо фирменного
* **Иконки пропадали** - кнопки корзины, поиска, меню без иконок, только квадраты
* **Картинки товаров не загружались или грузились минутами** - средняя скорость загрузки одной картинки у конкурентов на той же платформе составляла 16-44 секунды
* **Страница оставалась нерабочей 1-2 минуты** вместо нормальных 3-7 секунд
* **Тема выглядела сломанной визуально** - без стилей, без шрифтов, без иконок

### Масштаб

Проблема воспроизводилась на нескольких провайдерах одновременно - это не локальный сбой одного клиента, а блокировка конкретного IP на уровне РКН. Обращение к провайдеру бессмысленно: они не будут разблокировать IP ради одного клиента. InSales подтвердила проблему, но решить её со своей стороны не смогла.

---

## Решение

Яндекс CDN как прозрачный прокси перед `static.insales-cdn.com` + перехват и переписывание всех CDN URL на уровне DOM через MutationObserver. Между дата-центрами Яндекса и серверами InSales блокировки РКН нет - Яндекс CDN забирает ресурсы и отдаёт их пользователю с российских edge-серверов.

### Архитектура

```text
Браузер пользователя (МТС/Мегафон, CDN заблокирован)
        │
        ▼
assets.ваш-домен.ru  ←── Яндекс CDN edge (РФ, не блокируется)
        │
        ▼
static.insales-cdn.com  ←── Origin (недоступен напрямую,
                             но доступен из инфраструктуры Яндекса)
```

---

## Структура репозитория

```text
insales-yc-cdn-proxy/
├── README.md
├── snippets/
│   ├── layout-head.liquid          # Critical JS/CSS в <head> через прокси
│   ├── styles.liquid               # Шрифты и тема с YC Storage
│   └── cdn-rewrite-script.html     # MutationObserver скрипт (вставляется в layout)
├── css/
│   ├── insales-icons-style.css     # Иконки с YC URL (генерируется из HAR)
│   └── montserrat-stylesheet.css   # Шрифты с YC URL (генерируется из HAR)
└── docs/
    ├── har-analysis.md             # Как снять и прочитать HAR для диагностики
    └── comparison.md               # Таблица сравнения с конкурентами
```

---

## Установка

### Шаг 1 - Создать YC CDN ресурс

1. Яндекс Облако → CDN → Создать ресурс
2. Origin: `static.insales-cdn.com`, протокол HTTP
3. Подключить свой домен через CNAME: `assets.ваш-домен.ru` → YC CDN endpoint
4. Включить HTTPS на CDN ресурсе (Let's Encrypt через YC)

> ⏳ **Выпуск SSL сертификата может занять до 12-24 часов.** Это нормально - YC запрашивает сертификат через Let's Encrypt, который должен пройти DNS-валидацию. Пока сертификат не выпущен, HTTPS на вашем CDN-домене не работает. Не паникуйте и не пересоздавайте ресурс - просто подождите. Лучше запускать этот шаг с вечера, к утру всё будет готово.

5. Настроить CORS в Object Storage бакете если используете его для шрифтов:

```xml
<CORSConfiguration>
  <CORSRule>
    <AllowedOrigin>https://ваш-домен.ru</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedHeader>*</AllowedHeader>
  </CORSRule>
</CORSConfiguration>
```

### Шаг 2 - Снять HAR и определить версии библиотек

Откройте DevTools → Network → Disable cache → обновите страницу. Сохраните HAR. В нём найдите все запросы к `static.insales-cdn.com` - это ваши версии библиотек. Они будут в путях вида:

```text
/assets/static-versioned/4.81/static/libs/jquery/3.5.1/jquery-3.5.1.min.js
/assets/static-versioned/5.13/static/libs/vanilla-lazyload/17.9.0/lazyload.min.js
/assets/common-js/common.v2.27.6.js
```

> ⚠️ Версии у вас могут отличаться от примеров в этом репозитории. Всегда берите актуальные пути из своего HAR, не копируйте вслепую.

### Шаг 3 - Перенести шрифты и иконки на YC Storage

Из HAR найдите запросы к `static-versioned/.../fonts/Montserrat/stylesheet.css` и `.../icons-insales-default/style.css`. Скачайте эти файлы когда CDN работает (через VPN или с другого провайдера). Замените в них относительные пути шрифтов на абсолютные URL вашего YC Storage бакета. Залейте файлы и сами woff2 файлы в публичный бакет.

Готовые примеры с заменёнными путями - в папке `css/` этого репозитория (под версию `6.81` иконок и `5.31` Montserrat).

### Шаг 4 - Добавить Critical JS/CSS в `<head>`

В `layout.html` темы, внутри `<head>`, **до** `{% widgets_assets %}`, добавьте явное подключение критичных библиотек через ваш прокси-домен:

```html
<link rel="preconnect" href="https://assets.ваш-домен.ru" crossorigin>

<!-- Critical CSS -->
<link rel="stylesheet" href="https://assets.ваш-домен.ru/assets/static-versioned/6.80/static/libs/my-layout/1.0.0/core-css.css">
<link rel="stylesheet" href="https://assets.ваш-домен.ru/assets/static-versioned/4.81/static/libs/splide/2.4.21/css/splide.min.css">
<link rel="stylesheet" href="https://assets.ваш-домен.ru/assets/static-versioned/4.81/static/libs/microalert/0.1.0/microAlert.css">

<!-- Critical JS - jQuery первым, он нужен остальным -->
<script src="https://assets.ваш-домен.ru/assets/static-versioned/4.81/static/libs/jquery/3.5.1/jquery-3.5.1.min.js"></script>
<script src="https://assets.ваш-домен.ru/assets/static-versioned/5.83/static/libs/my-layout/1.0.0/my-layout.js"></script>
<script src="https://assets.ваш-домен.ru/assets/static-versioned/4.81/static/libs/splide/2.4.21/js/splide.min.js"></script>
<script src="https://assets.ваш-домен.ru/assets/static-versioned/4.81/static/libs/microalert/0.1.0/microAlert.js"></script>
<script src="https://assets.ваш-домен.ru/assets/static-versioned/4.81/static/libs/js-cookie/3.0.0/js.cookie.min.js"></script>
<script src="https://assets.ваш-домен.ru/assets/static-versioned/4.81/static/libs/body-scroll-lock/v3.1.3/bodyScrollLock.min.js"></script>
<script src="https://assets.ваш-домен.ru/assets/static-versioned/5.13/static/libs/vanilla-lazyload/17.9.0/lazyload.min.js"></script>
<script src="https://assets.ваш-домен.ru/assets/static-versioned/5.4/static/libs/cut-list/1.0.0/jquery.cut-list.min.js"></script>
<script src="https://assets.ваш-домен.ru/assets/static-versioned/4.81/static/libs/micromodal/0.4.6/micromodal.min.js"></script>
<script src="https://assets.ваш-домен.ru/assets/static-versioned/4.81/static/libs/fslightbox/3.4.1/fslightbox.js"></script>
```

### Шаг 5 - Добавить MutationObserver для перехвата остальных URL

`widgets_assets` и платформенные теги подключают ресурсы с `static.insales-cdn.com` минуя шаблон - их нельзя убрать. MutationObserver перехватывает любой элемент в DOM с CDN URL и переписывает его. Добавьте в `layout.html` сразу после `<body>`:

```javascript
<script>
(function () {
  var CDN_FROM = 'static.insales-cdn.com';
  var CDN_TO = 'assets.ваш-домен.ru';

  function rewriteUrl(value) {
    if (!value || typeof value !== 'string') return value;
    return value.replaceAll(CDN_FROM, CDN_TO);
  }

  function rewriteElement(el) {
    if (!el || el.nodeType !== 1) return;
    ['src', 'data-src', 'data-original', 'data-lazy', 'data-bg', 'href'].forEach(function(attr) {
      if (el.hasAttribute(attr)) {
        var old = el.getAttribute(attr);
        var next = rewriteUrl(old);
        if (next !== old) el.setAttribute(attr, next);
      }
    });
    if (el.hasAttribute('srcset')) {
      var old = el.getAttribute('srcset');
      var next = rewriteUrl(old);
      if (next !== old) el.setAttribute('srcset', next);
    }
    if (el.style && el.style.backgroundImage) {
      el.style.backgroundImage = rewriteUrl(el.style.backgroundImage);
    }
  }

  function rewriteAll(root) {
    var nodes = (root || document).querySelectorAll(
      'img, source, script, link, [data-src], [data-original], [data-lazy], [data-bg], [style]'
    );
    nodes.forEach(rewriteElement);
  }

  rewriteAll(document);
  document.addEventListener('DOMContentLoaded', function() { rewriteAll(document); });

  new MutationObserver(function(mutations) {
    mutations.forEach(function(m) {
      m.addedNodes.forEach(function(node) {
        rewriteElement(node);
        if (node.querySelectorAll) rewriteAll(node);
      });
      if (m.type === 'attributes') rewriteElement(m.target);
    });
  }).observe(document.documentElement, {
    childList: true,
    subtree: true,
    attributes: true,
    attributeFilter: ['src', 'srcset', 'data-src', 'data-original',
                      'data-lazy', 'data-bg', 'href', 'style']
  });
})();
</script>
```

### Шаг 6 - Обновить сниппет styles

В сниппете `styles` закомментируйте `{% include "system_v4_fonts" %}` и замените подключение `theme.css` и шрифтов на YC URLs. Пример в `snippets/styles.liquid`.

### Шаг 7 - Проверить корзину

После каждого изменения обязательно проверяйте:

* добавление товара в корзину (один раз, не дважды)
* открытие мини-корзины
* переход в корзину
* слайдеры и галереи

Если товар добавляется дважды - скорее всего `common.js` загружается дважды (наш + платформенный). Смотрите `snippets/layout-head.liquid` - там есть вариант с защитой от двойной инициализации.

---

## Применимость к другим платформам

Решение описано на примере InSales, но сам подход - YC CDN как прокси перед заблокированным CDN + MutationObserver для переписывания URL - применим к любой платформе где:

* статика раздаётся с внешнего CDN который может попасть под блокировку
* есть доступ к шаблонам (возможность вставить JS в `<head>` или `<body>`)

Потенциально подходит для **Shopify**, **Tilda**, **Битрикс**, **WordPress** с внешними CDN, самописных магазинов и любых других систем. Конкретные домены CDN, версии библиотек и механизм подключения будут другими, но архитектура та же.

Основная работа при адаптации - снять HAR, найти какие домены блокируются, создать YC CDN ресурс с нужным origin и подставить правильный домен в `CDN_FROM` в скрипте MutationObserver.

---

## Кэширование

Важный нюанс который влияет на работоспособность: **не всё стоит кэшировать на CDN**.

**CSS файлы - кэшируем.** Стили меняются редко, версионируются через хэш в URL (`theme.css?v=123`). Кэширование безопасно и значительно ускоряет повторные загрузки.

**JS файлы - не кэшируем (или кэшируем осторожно).** Особенно это касается `common.js` и платформенных библиотек. Причина: InSales может обновить версию common.js на своей стороне, и если у вас в CDN-кэше будет старая версия - сайт может сломаться. Платформа уже версионирует JS через пути (`common.v2.27.6.js` → `common.v2.27.8.js`), но лучше не рисковать.

**Практическая настройка в YC CDN:**

Для CSS ресурсов установите TTL кэша на 7-30 дней. Для JS оставьте кэш выключенным или TTL не более 1 часа - CDN будет проксировать запросы к origin без кэширования, что немного увеличивает нагрузку на origin но исключает проблемы со стале скриптами.

В YC CDN это настраивается через правила кэширования в настройках CDN ресурса (раздел "Кэширование" → "Переопределить заголовки cache-control") или через заголовки которые отдаёт origin.

---

## Известные проблемы и ограничения

**Двойная загрузка JS** - платформа грузит свои версии библиотек через `widgets_assets` параллельно с нашими. Это создаёт дубли в Network. Для большинства библиотек двойная загрузка безопасна. Для `common.js` может быть проблема с двойной инициализацией.

**Двойная инициализация `common.js`** - симптом: товар добавляется в корзину дважды. Решение - условная загрузка с проверкой `typeof InSales === 'undefined'` с таймаутом, либо не подключать `common.js` явно.

**Обновления платформы** - InSales периодически обновляет версии библиотек. MutationObserver автоматически переписывает новые URL на прокси - основная защита продолжит работать. Явные теги в `<head>` нужно обновить вручную.

**Кастомные виджеты** - если у вас есть виджеты с жёстко прописанными URL на `static.insales-cdn.com` внутри JS (не в атрибутах DOM), MutationObserver их не поймает. Такие места нужно найти и исправить вручную.

**`c6.cdn.insales-shop.ru`** - домен для изображений товаров отличается от `static.insales-cdn.com`. Для него нужен отдельный CDN ресурс или хотя бы `<link rel="preconnect">` в `<head>`.

**Шаблоны у всех разные** - в старых темах InSales могут быть другие наборы библиотек, другие версии, другие механизмы подключения. Список библиотек из этого репозитория - только пример. Свой список всегда берите из HAR.

---

## Стоимость

С 1 июля 2026 года тарифы Яндекс CDN следующие:

| Позиция                                         | Цена                     |
| ----------------------------------------------- | ------------------------ |
| Включённый пакет исходящего трафика             | **150 ₽/ресурс в месяц** |
| Запросы к CDN ресурсу (первые 100 млн операций) | Не тарифицируется        |
| Запросы сверх предоплаченного объёма            | 1 ₽ / 100 тыс. операций  |
| Исходящий трафик сверх пакета                   | 1.054 ₽/ГБ               |
| Экранирование источников (опционально)          | 3 843 ₽/ресурс в месяц   |
| Выгрузка логов (опционально)                    | 5 490 ₽/ресурс в месяц   |

Экранирование источников и выгрузка логов - опциональные платные функции, для базового прокси они не нужны.

**Примерный расчёт для разных размеров магазина:**

| Магазин   | Посетителей/день | Трафик через CDN | Стоимость/месяц            |
| --------- | ---------------- | ---------------- | -------------------------- |
| Маленький | до 200           | ~150-200 ГБ      | **150 ₽** (входит в пакет) |
| Средний   | 200-1 000        | ~200-800 ГБ      | **150 + ~50-650 ₽**        |
| Крупный   | 1 000-5 000      | ~800-3 000 ГБ    | **150 + ~650-3 000 ₽**     |

Для магазина с 200-250 посетителями в день весь трафик укладывается в включённый пакет - итого **150 ₽/месяц** и ни рублём больше.

Актуальные тарифы - на [официальной странице Яндекс CDN](https://yandex.cloud/ru/docs/cdn/pricing).

---

## Результаты

Сравнение с двумя магазинами на InSales без оптимизации, холодная загрузка, МТС домашний, CDN заблокирован:

| Метрика          | С YC прокси   | Конкурент 1   | Конкурент 2   |
| ---------------- | ------------- | ------------- | ------------- |
| **onLoad**       | **4.9 с**     | 47.6 с        | 21.3 с        |
| DOMContentLoaded | **3.9 с**     | 33.9 с        | 14.1 с        |
| Медленных >10 с  | **0**         | 27            | 14            |
| jQuery           | **250 мс YC** | 32 700 мс CDN | не загрузился |
| common.js        | **960 мс YC** | 32 900 мс CDN | 40 000 мс CDN |
| Splide           | **252 мс YC** | 24 500 мс CDN | не загрузился |
| Картинки avg     | **1.2 с**     | 1.3 с         | 16.0 с        |

jQuery: 32 секунды → 250 миллисекунд. Страница товара у конкурентов при падении CDN - корзина не работает, галерея не крутится. С прокси - всё функционирует.

---

## Мониторинг

После внедрения важно отслеживать что ничего не сломалось. Для ежедневного автоматического мониторинга корзины, галереи, скорости и ключевых сценариев покупки:

**[ecom-tech-speed-checker](https://github.com/DmitriiDmallSick/ecom-tech-speed-checker)** - Playwright-based n8n шаблон, который проверяет добавление товара в корзину, работу галереи, скорость загрузки и отправляет отчёт в Telegram каждое утро.

---

## Стек

* Яндекс Облако CDN + Object Storage
* InSales шаблонизатор (Liquid)
* Vanilla JS (MutationObserver, IntersectionObserver)

---

## Лицензия

MIT
