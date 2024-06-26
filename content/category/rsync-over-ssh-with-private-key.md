Title: Использование rsync для передачи файлов при помощи оболочки ssh и ключей
Date: 21.03.2024 15:26
Category: Заметки
Tags: rsync, ssh

rsync позволяет выполнять синхронизацию файлов и каталогов в двух местах. При синхронизации rsync не копирует файлы которые не изменялись с прошлой синхронизации.

Для начала сгенерируем ключ для безпарольного доступа к удаленному хосту (passphrase оставим пустым):
```
ssh-keygen
```

Сгенерированный публичный ключ необходимо добавить на удаленный хост. Для этого лучше всего подходит команда `ssh-copy-id`. Предварительно удаленный хост должен быть доступен по ssh с использованием пароля:
```
ssh-copy-id
```

Подключимся к удаленному хосту по ssh, убедившись что доступ по ключу работает:
```
ssh <username>@<hostname>
```

> Если при генерации ключа указывалось не стандартное имя и расположение то необходимо указать путь к ключу используя опцию `-i` (identity file)

Теперь можно приступить к использованию rsync. Просто выполняем команду rsync для синхронизации файлов с удаленным хостом:
```
rsync -Pav /path/to/source <username>@<hostname>:/path/to/destication
```

> Если при генерации ключа указывалось не стандартное имя и расположение то необходимо к команде rsync добавить опцию указания оболочки для транспорта `-e` и указать ключ явно:
```
rsync -Pav -e "ssh -i <path/to/private/key>" /path/to/source <username>@<hostname>:/path/to/destication
```

Таким образом можно к примеру автоматизировать синхронизацию каталогов на удаленном хосте используя задачу `cron` или таймеры `systemd`.
