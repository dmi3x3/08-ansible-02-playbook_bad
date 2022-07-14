# Домашнее задание к занятию "08.02 Работа с Playbook"

## Подготовка к выполнению

1. Создайте свой собственный (или используйте старый) публичный репозиторий на github с произвольным именем.
2. Скачайте [playbook](./playbook/) из репозитория с домашним заданием и перенесите его в свой репозиторий.
3. Подготовьте хосты в соответствии с группами из предподготовленного playbook.

## Основная часть

1. Приготовьте свой собственный inventory файл `prod.yml`.
```yaml
---
clickhouse:
  hosts:
    clickhouse-01:
      ansible_host: root@centos7-n
vector:
  hosts:
    vector-01:
      ansible_host: root@centos7-n

```
2. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает [vector](https://vector.dev).
3. При создании tasks рекомендую использовать модули: `get_url`, `template`, `unarchive`, `file`.
4. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, установить vector.
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
6. Попробуйте запустить playbook на этом окружении с флагом `--check`.
7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
9. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.


### Playbook содержит два play

- Install Clickhouse - содержит handler и tasks, необходимые для установки Clickhouse
- Install Vector - содержит tasks, необходимые для установки Vector

Краткая структура файла с пояснениями:
- name: Install Clickhouse <span style="color:green"> -- начало play с установкой Clickhouse</span>

  handlers: <span style="color:green"> -- В этом разделе записываются задачи, выполнять которые требуется, если какая либо задача произвела изменение и сообщила об этом</span>
    - name: Start clickhouse service <span style="color:green"> -- Здесь выполняется перезагрузка установленного Clickhouse </span>
  tasks: <span style="color:green"> -- В этом разделе записываются основные задачи для play "Install Clickhouse"</span>
      tags: clickhouse <span style="color:green"> -- тег, позволяющий ansible выполнять только помеченные тегом задачи </span>
    - block: <span style="color:green"> -- Этот модуль объединяет две задачи, если первая завершится с ошибкой, запустится вторая - rescue</span>
        - name: Get clickhouse distrib <span style="color:green"> -- Получение дистрибутива</span>

          register: res_sc <span style="color:green"> -- регистрация в переменную фактов работы модуля</span>
        - name: Get clickhouse distrib <span style="color:green"> -- Получение дистрибутива, если в предыдущей задаче возникла ошибка</span>
      
      tags: clickhouse <span style="color:green"> -- тег, позволяющий ansible выполнять только помеченные тегом задачи</span>
    - name: Install clickhouse packages <span style="color:green"> -- Установка Clickhouse с помощью модуля yum</span>
  
      notify: Start clickhouse service <span style="color:green"> -- Сообщение модуля, с именем handler'а, который надо запустить, если были произведены изменения</span>

      tags: clickhouse <span style="color:green"> -- тег, позволяющий ansible выполнять только помеченные тегом задачи</span>

    - name: Remove clickhouse packages distribs <span style="color:green"> -- Удаление файлов с дистрибутивом</span>

      tags: [for_delete, never] <span style="color:green"> -- тег, позволяющий ansible выполнять только помеченные тегом задачи</span>

  post_tasks: <span style="color:green"> -- В этом разделе записываются дополнительные задачи, выполнять которые требубуется после работы tasks и срабатывания ее handlers</span>
    - name: Create database <span style="color:green"> -- Создание БД Clickhouse</span>
      
      tags: clickhouse <span style="color:green"> -- тег, позволяющий ansible выполнять только помеченные тегом задачи</span>
  
  tasks: <span style="color:green"> -- В этом разделе записываются основные задачи для play "Install Vector"</span>
    - name: Install Vector<span style="color:green"> -- начало play с установкой Vector</span>
    - name: Create directrory for vector "{{ vector_dir }}" <span style="color:green"> -- Создание каталога для установки в него Vector</span>
      
      tags: vector <span style="color:green"> -- тег, позволяющий ansible выполнять только помеченные тегом задачи</span>
    - name: Get vector distrib <span style="color:green"> -- </span>
    - name: Extract vector in the installation directory <span style="color:green"> -- Разархивирование дистрибутива Vector в каталог установки</span>
      
      tags: vector <span style="color:green"> -- тег, позволяющий ansible выполнять только помеченные тегом задачи</span>
    
    - name: Remove vector packages distribs <span style="color:green"> -- Удаление архива с дистрибутивом</span>
      tags: [for_delete, never] <span style="color:green"> -- тег, позволяющий ansible выполнять только помеченные тегом задачи</span>


Для работы playbook требуется выполнить команду:
```shell
ansible-playbook -i inventory/prod.yml site.yml
```
если добавить к команде --tags clickhouse - будут выполнены все задачи, относящиеся к Clichouse,
если добавить к команде --tags vector - будут выполнены все задачи, относящиеся к Vector.
опция --tags 'never, for_delete' выполнит задачи, производящие удаление файлов с дистрибутивами.
В случае с clickhouse можно это сделать двумя способами: 
- собрать имя из переменных так же, как при скачивании.
```yaml
    - name: Remove clickhouse packages distribs variant 2
      ansible.builtin.file:
        path: './{{ item.0 }}-{{ item.1 }}.rpm'
        state: absent
      with_nested:
        - "{{ clickhouse_packages }}"
        - "{{ clickhouse_version }}"
      tags:
        - for_delete
        - never
```
- зарегистрировать факты работы модуля скачивания ansible.builtin.get_url
и получить требуемые имена файлов оттуда. Этот вариант предсталяется мне более логичным, что скачали, то и удалили.
Правда в реализации есть нюанс: факты регистрируются только в первой задаче, несмотря на то, что в ней не все файлы могут быть скачены, имена соответствующих файлов одинаковые в основной и rescue тасках. 
```yaml
    - name: Remove clickhouse packages distribs
      ansible.builtin.file:
        path: "{{ item.dest }}"
        state: absent
      with_items:
        - "{{ res_sc.results }}"
      tags:
        - for_delete
        - never
```


10. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-02-playbook` на фиксирующий коммит, в ответ предоставьте ссылку на него.

[репозиторий c домашней работой `08-ansible-02-playbook`](https://github.com/dmi3x3/08-ansible-02-playbook/tree/08-ansible-02-playbook)

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---