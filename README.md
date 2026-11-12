# Лабораторная: Ansible role для nginx и GitHub Actions на macOS

Mac используется как управляющая машина. Роль устанавливает nginx на **Ubuntu VM в UTM** по SSH: `apt`, `/etc/nginx` и `systemd` относятся к Linux. В предыдущей лабораторной уже настроены Ubuntu 26.04.1, пользователь `devops` и ключ `~/.ssh/id_ed25519_devops_lab`. Сейчас VM доступна напрямую по адресу `192.168.64.8`.

## 1. Подготовить Mac и Ubuntu

На Mac нужны Ansible, Git и SSH. Ansible уже установлен. Если потребуется переустановить: `brew install ansible` (или см. [официальную инструкцию](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html)). Проверьте `ansible --version`.

Запустите VM в UTM. На Mac проверьте доступ: `ssh -i ~/.ssh/id_ed25519_devops_lab devops@192.168.64.8`. Внутри Ubuntu можно проверить `python3 --version` и `sudo -n true`.

Ранее для VM использовался проброс `127.0.0.1:2222 → 22`, но сейчас этот порт не принимает соединение. Прямой IP работает. Если после перезапуска VM адрес изменится, узнайте новый адрес внутри Ubuntu командой `hostname -I` и обновите `ansible_host` в `inventory.ini`.

## 2. Заполнить inventory

```sh
cd /путь/к/ansible-nginx-lab
cp inventory.ini.example inventory.ini
nano inventory.ini
```

Пример inventory уже содержит рабочие параметры VM. `inventory.ini` исключён из Git. Если адрес, имя пользователя или путь к ключу изменились, поправьте их в `inventory.ini`.

```sh
ansible webservers -m ansible.builtin.ping
ansible-playbook --syntax-check playbook.yml
ansible-playbook playbook.yml --diff
```

Для `devops` в первой лабораторной уже настроен `sudo` без пароля. Повторите последнюю команду: при втором запуске задачи должны дать `changed=0`.

Проверено 12 ноября 2026 года на этой VM: `ansible.builtin.ping` вернул `pong`; первый запуск playbook завершился без ошибок, повторный — с `changed=0`. `ansible-lint` прошёл без ошибок и предупреждений.

## 3. Проверить сайты с Mac

Можно проверить сайты напрямую по адресу VM:

```sh
curl -H 'Host: etis.com' http://192.168.64.8/
curl -H 'Host: fizfak.ru' http://192.168.64.8/
curl -H 'Host: mehmat.ru' http://192.168.64.8/
```

Проверено: каждый ответ содержит `Hello! Site: <имя сайта>`. На VM `nginx -t` проходит, сервис `active` и `enabled`.

Заголовок `Host` выбирает виртуальный сайт без правки `/etc/hosts` на Mac. Если HTTP-порт VM не открывается напрямую, создайте SSH-туннель в отдельном окне Terminal:

```sh
ssh -i ~/.ssh/id_ed25519_devops_lab -N -L 8080:127.0.0.1:80 devops@192.168.64.8
```

Затем в другом окне Terminal:

```sh
curl -H 'Host: etis.com' http://127.0.0.1:8080/
curl -H 'Host: fizfak.ru' http://127.0.0.1:8080/
curl -H 'Host: mehmat.ru' http://127.0.0.1:8080/
```

Ещё один способ проверки из Terminal: `curl --resolve etis.com:8080:127.0.0.1 http://etis.com:8080/`. Чтобы открыть сайт в браузере по имени, временно добавьте в `/etc/hosts` Mac строку `127.0.0.1 etis.com fizfak.ru mehmat.ru`, откройте `http://etis.com:8080/`, а после лабораторной удалите строку. SSH-туннель остановите через `Ctrl+C`.

## 4. Репозиторий и CI

Проект опубликован в [GitHub-репозитории lab6_devops](https://github.com/shanerone/lab6_devops). В его [Actions](https://github.com/shanerone/lab6_devops/actions) находятся два workflow: `first-run.yml` показывает работу runner, `lint.yml` запускает ansible-lint на push и pull request. Оба сценария проверены на GitHub.

Для работы на другом Mac клонируйте репозиторий и создайте локальный inventory:

```sh
git clone https://github.com/shanerone/lab6_devops.git
cd lab6_devops
cp inventory.ini.example inventory.ini
```

Для демонстрации проверки pull request можно создать отдельную ветку, изменить файл, отправить ветку и открыть pull request в `main`.

В методичке встречается «GitLab», но примеры `github.com` и `.github/workflows` относятся к GitHub Actions. Для этой части используйте GitHub.

## Структура

`playbook.yml` задаёт сайты и вызывает роль. `roles/nginx_vhosts/tasks/main.yml` устанавливает nginx, создаёт каталоги и страницы, формирует конфиги. Handler перезапускает nginx только при изменении конфигурации.
