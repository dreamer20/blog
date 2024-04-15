Title: Использование ssh-agent для одноразового ввода парольной фразы для ключа
Date: 21.03.2024 15:30
Category: Заметки
Tags: ssh, ssh-agent


ssh-agent позволяет хранить ключи для SSH в памяти, готовые к использования ssh. Это позволяет не вводить пароль от ключа каждый раз, когда вы подключаетесь к серверу.

Для начала ssh-agent должен быть запущен. Для этого необходимо выполнить команду:
```
eval `ssh-agent`
```
Команда запускает ssh-agent и устанавливает определенные переменные окружения для оболочки.

Далее необходимо добавить ключи в ssh-agent. Это можно сделать просто введя команду `ssh-add`. Без аргументов в агент будет добавлены стандартные приватные ключи ~/.ssh/id_rsa, .ssh/id_dsa, ~/.ssh/id_ecdsa, ~/.ssh/id_ed25519, и ~/.ssh/identity. Если необходимо добавить конкретный ключ необходимо просто указать его в качестве аргумента. Например:
```
ssh-add ~/my_private_key
```
При этом будет запрошена парольная фраза от ключа.
Чтобы убедиться что ключи добавлены выполните команду:
```
ssh-add -l
```

Теперь когда ssh-agent запомнил ключ можно подключаться к удаленному хосту без ввода парольной фразы. ssh-agent работает до закрытия текущей командной оболочки. Если открыть новый терминал, то запуск `ssh-agent` и добавления ключа с помощью `ssh-add` необходимо повторять снова (несмотря на то что процесс ssh-agent от предыдущей оболочки до сих пор может существовать). Для дополнительной безопасности можно ограничить использование ключа в оболочке временными рамками при помощи опции `-t` с командой `ssh-add`:
```
ssh-add -t 1h
```
