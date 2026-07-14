# Как снять HAR и диагностировать проблему с CDN

HAR (HTTP Archive) - это запись всех сетевых запросов браузера при загрузке страницы. Это главный инструмент для диагностики проблем с CDN InSales.

---

## Как снять HAR

### Chrome / Edge (рекомендуется)

1. Откройте страницу магазина которую хотите проверить
2. Нажмите `F12` → вкладка **Network**
3. Поставьте галочку **Disable cache**
4. Нажмите `Ctrl+Shift+R` (жёсткая перезагрузка без кеша)
5. Дождитесь полной загрузки страницы
6. Правый клик на любом запросе в списке → **Save all as HAR with content**

> Снимайте HAR именно с того устройства и провайдера где наблюдаете проблему. HAR с хорошего соединения не покажет блокировку.

### Firefox

1. `F12` → вкладка **Network**
2. Нажмите иконку шестерёнки → **Persist logs**
3. `Ctrl+Shift+R`
4. После загрузки: правый клик → **Save All As HAR**

### Мобильный браузер

Снять HAR с телефона напрямую сложно. Варианты:

* Подключите телефон через USB → включите USB отладку → откройте `chrome://inspect` на компьютере
* Либо используйте Charles Proxy или аналог на компьютере как прокси для телефона
* Либо попросите пользователя у которого проблема сохранить страницу через десктоп Chrome подключившись к его WiFi

---

## Как читать HAR - на что смотреть

### 1. Найдите медленные и упавшие запросы

В DevTools Network отсортируйте по колонке **Time** (по убыванию). Проблемные запросы сразу видны:

| Что видите                             | Что это значит                                  |
| -------------------------------------- | ----------------------------------------------- |
| Status **0**                           | Запрос упал - соединение оборвалось, ответа нет |
| Status **200**, время **30–70 секунд** | CDN работает но очень медленно                  |
| Status **200**, время **< 1 секунды**  | Всё хорошо                                      |

### 2. Проверьте какие домены проблемные

Отфильтруйте по `insales-cdn` в строке поиска Network. Если видите статус 0 или время >5 секунд - CDN заблокирован.

```
static.insales-cdn.com  ← основной CDN платформы (JS, CSS, шрифты, иконки)
c6.cdn.insales-shop.ru  ← CDN для картинок товаров (другой домен)
```

### 3. Трассировка - убедитесь что это РКН

Запустите трассировку из командной строки (Windows):

```cmd
tracert static.insales-cdn.com
```

Признак блокировки РКН - обрыв на 6–7 хопе внутри сети провайдера:

```
6     6 ms    as208677.asbr.router [212.188.45.6]
7     *   *   *   Request timed out.
8–30  *   *   *   Request timed out.
```

Если трассировка доходит до сервера InSales - проблема не в блокировке, а в чём-то другом.

### 4. Проверьте Timing конкретного запроса

Кликните на любой запрос → вкладка **Timing**. Ключевые поля:

| Поле                     | Норма                    | Проблема                                              |
| ------------------------ | ------------------------ | ----------------------------------------------------- |
| DNS Lookup               | < 100мс                  | > 300мс - нет preconnect                              |
| Initial connection (TCP) | < 100мс                  | > 300мс - медленный сервер                            |
| SSL                      | < 200мс                  | > 500мс - нет preconnect                              |
| Waiting (TTFB)           | < 500мс                  | > 1с - медленный origin                               |
| Content Download         | зависит от размера файла | -                                                     |
| **Queued at X.Xs**       | < 1с                     | > 5с - файл загрузился поздно, что-то его блокировало |

"Queued at" - особенно важный показатель. Если hero-картинка "Queued at 7с" - значит браузер добрался до неё только через 7 секунд после начала загрузки. Причина обычно в блокирующих скриптах в `<head>`.

---

## Как определить нужные версии библиотек из HAR

После снятия HAR откройте его в браузере или через инструмент (например [har.tech](https://har.tech)). Найдите все запросы к `static.insales-cdn.com` и скопируйте пути.

Вам нужны пути вида:

```
/assets/static-versioned/4.81/static/libs/jquery/3.5.1/jquery-3.5.1.min.js
/assets/static-versioned/5.13/static/libs/vanilla-lazyload/17.9.0/lazyload.min.js
/assets/static-versioned/5.83/static/libs/my-layout/1.0.0/my-layout.js
/assets/common-js/common.v2.27.6.js
/assets/static-versioned/5.31/static/fonts/Montserrat/stylesheet.css
/assets/static-versioned/6.81/static/icons/icons-insales-default/style.css
```

Номера версий (`4.81`, `5.13`, `2.27.6` и т.д.) используйте в вашем `layout-head.liquid` при подключении через прокси.

> ⚠️ Не копируйте версии из этого репозитория вслепую. Ваша тема может использовать другие версии платформы.

---

## Быстрый Python скрипт для анализа HAR

Если хотите быстро получить сводку по HAR без DevTools - сохраните скрипт ниже и запустите:

```bash
python analyze_har.py путь/к/файлу.har
```

```python
import json
import sys
from urllib.parse import urlparse

def analyze(filepath):
    with open(filepath, 'r', encoding='utf-8') as f:
        har = json.load(f)

    entries = har['log']['entries']
    pt = har['log']['pages'][0]['pageTimings']

    print(f"Страница: {har['log']['pages'][0]['title']}")
    print(f"onLoad:            {pt['onLoad']/1000:.1f} с")
    print(f"DOMContentLoaded:  {pt['onContentLoad']/1000:.1f} с")
    print(f"Всего запросов:    {len(entries)}")

    failed  = [e for e in entries if e['response']['status'] == 0]
    slow_3  = [e for e in entries if e['time'] > 3000]
    slow_10 = [e for e in entries if e['time'] > 10000]
    cdn     = [e for e in entries if 'insales-cdn' in e['request']['url']]
    cdn_fail = [e for e in cdn if e['response']['status'] == 0]

    print(f"Failed (status 0): {len(failed)}")
    print(f"Медленных >3с:     {len(slow_3)}")
    print(f"Медленных >10с:    {len(slow_10)}")
    print(f"CDN запросов:      {len(cdn)}  (failed: {len(cdn_fail)})")

    if slow_3:
        print(f"\nТоп медленных запросов:")
        for e in sorted(slow_3, key=lambda x: -x['time'])[:10]:
            u = e['request']['url']
            domain = urlparse(u).netloc
            path = urlparse(u).path[-60:]
            print(f"  {e['time']/1000:.1f}с  [{e['response']['status']}]  {domain}{path}")

    if cdn_fail:
        print(f"\nCDN запросы которые упали:")
        for e in cdn_fail:
            u = e['request']['url']
            print(f"  {e['time']/1000:.1f}с  {urlparse(u).path[-70:]}")

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Использование: python analyze_har.py файл.har")
        sys.exit(1)
    analyze(sys.argv[1])
```

---

## Как сравнить свой сайт с конкурентами

Снимите HAR конкурентов на том же провайдере и устройстве что и свой. Ключевые метрики для сравнения:

* **onLoad** - общее время загрузки страницы
* **Медленных >3с** - сколько запросов висело дольше 3 секунд
* **CDN failed** - сколько файлов платформы не загрузилось вообще
* **jQuery / common.js время** - когда стала доступна интерактивность

Подробная таблица сравнения с реальными цифрами - в [docs/comparison.md](./comparison.md).

---

## Частые вопросы

**CDN работает нормально - зачем вообще что-то делать?**

Блокировка нестабильна - сегодня работает, завтра нет. Лучше настроить прокси заранее пока сайт работает и ничего не горит, чем в панике разбираться когда корзина перестала работать у половины пользователей.

**Как понять что проблема именно у пользователей а не только у меня?**

Попросите 2–3 человек с разных провайдеров (МТС, Мегафон, Билайн) открыть ваш сайт и сказать работает ли корзина. Либо используйте [ecom-tech-speed-checker](https://github.com/DmitriiDmallSick/ecom-tech-speed-checker) для автоматического ежедневного мониторинга.

**Можно ли снять HAR автоматически без DevTools?**

Да - через Playwright. Именно так работает [ecom-tech-speed-checker](https://github.com/DmitriiDmallSick/ecom-tech-speed-checker): запускает headless браузер, записывает HAR, анализирует и присылает отчёт в Telegram каждое утро.
