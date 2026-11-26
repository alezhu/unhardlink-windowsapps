# unhardlink-windowsapps

PowerShell-скрипт для поиска жёстких ссылок NTFS (`HardLink`), ведущих в `C:\Program Files\WindowsApps`, и замены их внешних копий на независимые файлы с правами, унаследованными от их собственных каталогов.

## Зачем это нужно

После работы некоторых инструментов дедупликации диска одинаковые файлы в `Program Files`, `AppData`, `.nuget` и других папках могут оказаться связаны жёсткой ссылкой с файлами внутри `C:\Program Files\WindowsApps`.

Поскольку в NTFS все жёсткие ссылки разделяют один и тот же дескриптор безопасности (ACL), внешние программы и утилиты внезапно получают строгие права `WindowsApps` / `TrustedInstaller`. Это приводит к ошибкам `Отказано в доступе` (`Access Denied`) при загрузке DLL, запуске или обновлении программ.

## Как работает скрипт

1. Рекурсивно сканирует `C:\Program Files\WindowsApps` и отбирает файлы с `LinkType -eq 'HardLink'`.
2. Через `fsutil.exe hardlink list` получает все пути, указывающие на тот же физический файл.
3. Пропускает пути внутри `C:\Program Files\WindowsApps` и `%SystemRoot%` (`C:\Windows`), чтобы не затронуть Store-приложения и системные файлы.
4. Для каждого внешнего файла:
   - создаёт новую копию рядом (она автоматически наследует права целевого каталога);
   - удаляет исходную жёсткую ссылку;
   - переименовывает копию на место исходного файла.

## Состав

- `unhardlink-windowsapps.ps1` — основной скрипт (поддерживает параметр `-DryRun`).
- `run.cmd` — обёртка для быстрого запуска `unhardlink-windowsapps.ps1` с политикой `-ExecutionPolicy Bypass`.

## От имени кого и как запускать

Каталог `C:\Program Files\WindowsApps` недоступен на чтение даже обычному администратору (`Access is denied`), поэтому скрипт нужно запускать **от имени `TrustedInstaller`** (с включёнными привилегиями, например `Enable All Privileges` в NSudo).

> **Важно:** если вы используете NSudo, запускайте `powershell` или `cmd` **напрямую из NSudo**, а не через `explorer.exe` и пункт контекстного меню «Открыть в Терминале» — иначе Windows откроет терминал в обычной пользовательской сессии без прав `TrustedInstaller`.

Перед запуском можно убедиться, что токен корректный:

```powershell
whoami
whoami /groups | Select-String "TrustedInstaller"
```

Ожидается `nt authority\system` и наличие группы `NT SERVICE\TrustedInstaller`.

### Примеры запуска

Предварительный просмотр без изменения файлов (`-DryRun`):

```powershell
powershell -ExecutionPolicy Bypass -File .\unhardlink-windowsapps.ps1 -DryRun
```

Запуск на выполнение:

```powershell
powershell -ExecutionPolicy Bypass -File .\unhardlink-windowsapps.ps1
```

Либо через `run.cmd` из консоли с правами `TrustedInstaller`:

```cmd
run.cmd
```

Либо через `gsudo`:

```powershell
gsudo --ti powershell -ExecutionPolicy Bypass -File .\unhardlink-windowsapps.ps1
```

## Примечание

В работающей Windows удаление некоторых жёстких ссылок, связанных с `WindowsApps`, может блокироваться даже под `TrustedInstaller` (например, из-за фильтров файловой системы или блокировки файла запущенным процессом). Такие файлы можно отвязать либо из среды восстановления Windows (WinRE), либо переустановкой конкретного приложения.
