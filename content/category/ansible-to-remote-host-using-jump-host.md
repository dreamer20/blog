Title: Получение доступа к удаленным хостам через бастион
Date: 10.04.2024 15:16
Category: Заметки
Tags: ansible, ssh

Ansible позволяет получать доступ к удаленным хостам через хост посредник (бастион). При этом нет необходимости проводить дополнительную настройку для хоста, помимо как наличия доступа при помощи ssh.

Для удобства сгенерируем ключ который будет использоваться для доступа на бастион и удаленные хосты (парольную фразу устанавливать не будем):
```
ssh-keygen
```

Далее скопируем публичный ключ на хосты (предварительно должна быть разрешена аутентификация ssh по паролю):
```
ssh-copy-id <user>@<bastion_host_address>
```

По задумке мы не имеем прямого дуступа к необходимым удаленным хостам, поэтому чтобы добавить на эти хосты публичный ключ воспользуеся командой:
```
cat ~/.ssh/id_rsa.pub | ssh -J <bastion_user>@<bastion_host_address> <remote_user>@<remote_host_address> -T "cat >> ~/.ssh/authorized_keys"
```

id_rsa.pub - имя публичного ключа сгенерированного по умолчанию

На удаленных хостах используется публичный ключ который мы сгенерировали непосредственно на нашем пк.

Теперь можно приступить непосредственно к настройке `ansible`. Для этого нам понадобится файл inventory.ini с списком хостов с такой информацией:
```
[remote_hosts]
<remote_host_address>
<another_remote_host_address>

[remote_hosts:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -q <user>@<bastion_host_address>"'
```

Все что находится в кавычках является аргументами команды ssh. Аргумент `-W` говорит ssh выполнить проброс ввода и вывода через хост `<bastion_host_address>`.

Теперь можно выполнять команды `ansible` как обычно.