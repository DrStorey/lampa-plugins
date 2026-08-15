# Lampa serials-skip (Vimu/DDD research fork)

Производная от файла [`hauryllin03/Plugins/serials-skip`](https://github.com/hauryllin03/Plugins/blob/main/serials-skip). Существующая цепочка источников таймингов и механизм определения сезона/серии сохранены:

1. The IntroDB v2 (`api.theintrodb.org`);
2. AniSkip v2 для аниме (`api.aniskip.com`);
3. `ipavlin98/lmp-series-skip-db` для Кинопоиска;
4. IntroDB (`api.introdb.app`) для IMDb.

Плагин по-прежнему подставляет найденные сегменты в `params.segments.skip`, сохраняет их в элементах playlist и показывает кнопку пропуска во встроенном плеере Lampa. Исправлено получение media-element через актуальный публичный объект `Lampa.PlayerVideo.video()` с fallback на старый `Lampa.Player.Video.video()`.

## Установка

После публикации репозитория добавьте в Lampa URL файла:

```text
https://cdn.jsdelivr.net/gh/DrStorey/lampa-plugins@main/skip_opening/plugin.js
```

или через GitHub Pages:

```text
https://drstorey.github.io/lampa-plugins/skip_opening/plugin.js
```

`plugin.js` — standalone-копия локального `serials-skip`. Расширение исходного файла не обязательно: загрузчик Lampa исполняет полученный JavaScript.

## Совместимость плееров

| Player/backend | Определение и запуск | Текущая позиция во время просмотра | Runtime seek | Сегменты OP/ED | Callback без закрытия | Уровень |
|---|---|---:|---:|---:|---:|---|
| Встроенный Lampa Player | `Lampa.Player` + `Lampa.PlayerVideo.video()` | Да | Да, `HTMLMediaElement.currentTime` | Да, `params.segments.skip` | Да | `FULL` |
| Vimu | Android `ACTION_VIEW`; packages `net.gtvbox.videoplayer`, `net.gtvbox.vimuhd`, legacy `net.gtvbox.vimu` | Нет | Нет | Нет | Нет | `UNSUPPORTED_EXTERNAL` |
| DDD Video Player | Android `ACTION_VIEW`; package `top.rootu.dddplayer` | Нет | Нет | Нет | Нет | `UNSUPPORTED_EXTERNAL` |

`startfrom`/`position` — только стартовая позиция, поэтому они намеренно не считаются реализацией автопропуска.

## Результаты исследования Android-клиента

Проверены исходники [`lampa-app/LAMPA`](https://github.com/lampa-app/LAMPA) и diff `v1.12.3..v1.12.4`, а также исходники [`usmanec/dddplayer`](https://github.com/usmanec/dddplayer) и официальный [Vimu Player API](https://vimu.tv/player-api/).

### Vimu

- Intent action при запуске: `android.intent.action.VIEW`.
- Packages LAMPA: `net.gtvbox.videoplayer`, `net.gtvbox.vimuhd`, `net.gtvbox.vimu`.
- Старт: extras `position` и `startfrom`, миллисекунды. Официально документирован `startfrom`.
- Playlist: MIME `application/vnd.gtvbox.filelist`, extras `asusfilelist`, `asusnamelist`, для Vimu 7.99+ — `startindex`.
- Result actions: `net.gtvbox.videoplayer.result` и `net.gtvbox.vimuhd.result`.
- Result после возврата: `position`, `duration`; коды `0` (прервано), `1` (завершено), `4` (ошибка), LAMPA также обрабатывает `2`/`3`.
- Документированного intent extra для skip-сегментов, live callback или команды seek нет.

### DDD Video Player

- Поддержка добавлена в LAMPA v1.12.4.
- Intent action при запуске: `android.intent.action.VIEW`.
- Package: `top.rootu.dddplayer`.
- Extras: `title`, `return_result=true`, `position`, `headers`; playlist использует `video_list`, `video_list.name`, `video_list.filename`, `video_list.thumbnail`, `video_list.subtitles`.
- Result action: `top.rootu.dddplayer.intent.result.VIEW`.
- В `PlayerActivity.finish()` возвращаются `position`, `duration`, `end_by`; то есть callback появляется только при закрытии Activity.
- Публичного broadcast/service/binder API для чтения позиции или runtime seek в исходниках нет. Media3 `seekTo()` существует лишь внутри процесса DDD и не экспортирован вызывающему приложению.

### Android bridge LAMPA

JS-интерфейс предоставляет `AndroidJS.openPlayer(link, json)`. Нативная сторона запускает player через `ActivityResultContracts.StartActivityForResult`. В bridge отсутствуют экспортированные методы чтения позиции, отправки seek и подписки на состояние внешнего плеера. Результат преобразуется в `Lampa.Timeline.update(...)` уже после возврата из внешнего Activity.

Опция LAMPA «сохранять соединение с плеером» удерживает WebView/foreground service активными, но сама по себе не создаёт канал управления Vimu или DDD.

## Поведение во внешнем плеере

Найденные сегменты остаются в playback data, но текущий Android-клиент их игнорирует при построении Vimu/DDD Intent. Плагин прекращает DOM-tracking и показывает безопасное уведомление «Автопропуск недоступен во внешнем плеере». Воспроизведение не перезапускается и стартовая позиция не подменяется концом OP/ED.

Для настоящей поддержки необходимы изменения с обеих сторон:

1. экспортированный runtime API в Vimu/DDD (позиция + seek либо нативное принятие списка skip-сегментов);
2. методы и callbacks в `AndroidJS` LAMPA;
3. адаптер в этом плагине, использующий только фактически добавленные методы bridge.

Таблица возможностей доступна также во время выполнения как `window.LampaSerialSkipExternalPlayers`.

## Публикация

1. Создайте fork `hauryllin03/Plugins` на GitHub или новый репозиторий с разрешения автора.
2. Поместите обновлённые `plugin.js` и `serials-skip` в нужную ветку.
3. Включите GitHub Pages (Deploy from branch) либо используйте jsDelivr URL выше.
4. Добавьте HTTPS URL в «Настройки → Расширения → Добавить плагин» Lampa.

## Лицензирование

В upstream-репозитории на момент исследования отсутствует файл лицензии. Поэтому этот форк не добавляет от себя лицензию на исходный код. Перед публичным распространением следует получить разрешение автора либо попросить автора явно лицензировать репозиторий.


