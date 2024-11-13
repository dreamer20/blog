Title: Передача строки на удаленный хост используя ssh
Date: 11.04.2024 18:48
Category: Заметки
Tags: ssh

Допустим ситуацию, необходимо передать на удаленный хост отпечаток конкретного сервера в known_host чтобы выполнять соединение без первичного запроса на доверие этому серверу.
Найдем интересующую нас строку при помощи команды:
```text
$ cat -n .ssh/known_hosts

1  192.168.108.127 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN+pE0oXKTHwpEMFqaUx16umGQrm6HUZTn+xYC+TEV
2  site.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJs
3  192.168.108.128 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN+pE0oXKTHwpEMFqaUx16umGQrm6HUZTn+xYU+TEV
```

Теперь мы знаем что необходимый нам ключ находится на второй строке файла. Передать его на удаленный хость мы может таким образом:

```bash
$ var=$(sed '2q;d' .ssh/known_hosts)
$ ssh user@server "echo $var >> .ssh/known_hosts"
```
