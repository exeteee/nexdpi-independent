# Сборка NexDPI в GitHub Actions

Этот проект независим от ByeDPI. Для него не нужен VPS или внешний VPN-шлюз.

## Важно

Загрузи исходники в **новый пустой репозиторий**, например `nexdpi-independent`. Не помещай их поверх старого репозитория с workflow, который клонирует ByeDPIAndroid.

## Шаги

1. Создай пустой GitHub-репозиторий без README и без шаблонных файлов.
2. Распакуй архив NexDPI на компьютере.
3. Загрузи в новый репозиторий всё содержимое папки `nexdpi-v2`, включая скрытую папку `.github`.
4. Открой вкладку **Actions**. Разреши запуск workflows, если GitHub попросит.
5. Выбери **Build independent NexDPI** и нажми **Run workflow**.
6. В успешном запуске открой раздел **Artifacts** и скачай `nexdpi-independent-local-engine-debug-apk`.
7. Распакуй artifact ZIP и установи `app-debug.apk` на Android.

Workflow сначала запускает `:app:testDebugUnitTest`, затем `:app:assembleDebug`. В нём используется JDK 17, Android SDK Platform 35 и Gradle 8.10.2.

## Что важно знать перед использованием

- При первом включении Android покажет системное окно разрешения VPN.
- SOCKS5 с `t.me/socks` / `tg://socks` или `socks5://` используется для TCP и UDP, если сервер поддерживает UDP ASSOCIATE.
- MTProxy с `t.me/proxy` / `tg://proxy` открывается в Telegram и не проксирует остальные приложения.
- HTTP CONNECT поддерживает TCP. UDP при таком выборе блокируется, а не отправляется напрямую.
- DoH-запросы самого NexDPI пока идут напрямую к выбранному DNS-провайдеру, не через системный proxy profile.
- Сборка workflow создаёт debug APK, а не подписанный релизный APK.
- Перед повседневным использованием проверь запуск, остановку, переключение Wi-Fi/мобильной сети и работу прокси на своём устройстве.
