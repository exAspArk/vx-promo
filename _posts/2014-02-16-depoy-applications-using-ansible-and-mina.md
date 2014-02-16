---
layout: post
title: How To Deploy Rails Applications Using Ansible And Mina
---

С проблемами, которые возникают, когда нужно с нуля поставить на сервер Rails application,
сталкиваются многие. Не смотря на то, что правильные способы решения задачи уже
давно существуют, например [heroku][heroku] или [engine yard][engine yard], к
сожалению, по различным причинам не всегда получается ими воспользоваться.

Если отбросить вариант "все сделать руками", то самое очевидное решение:

* Подготовить сервер, воспользовавшись Puppet, Chef, etc. Установить нужные пакеты,
  поставить и сконфигурировать сервисы, необходимые для приложения, завести юзеров
  в системе и прочее.
* Средствами Capistrano или Mina описать задачи выполняемые при деплое, и запустить
  деплой.

К сожалению, это решение обладает обладает серьезными минусами:

* По сути, конфигурация находится в 2-х разных местах, что серьезно затрудняет ее
  поддержку и тестирование
* provision tools и deploy tools должны знать друг про друга, под каким пользователем
  будет выполняться деплой, какие пароли и сертификаты используются, на каких адресах
  и портах висят сервисы

Поэтому придумана масса способов, как можно совместить вместе 2 задачи
- provision и deploy, здесь provision и deploy только с помощью Puppet/Chef, или
интеграция Chef/Puppet в deploy tool, и много чего еще.

Дальше об еще одном способе, который, по моему мнению, обладает большим плюсом - простота.

Для решения задачи понадобятся 2 вещи [ansible][ansible] и [mina][mina], подробнее про них

__Mina__

Deploy tool, которая, в отличие от всем знакомой Capistrano, для деплоя использует
немного другой подход, она сразу из config/deploy.rb генерит один большой bash скрипт,
который заливается на сервер и выполяется одной командой. Помимо этого, она умеет
просто сгенерить скрипт, не выполняя его, именно это и нужно для того, что бы совместить
ее с ansible.

__Ansible__

Provision tool. Вообще, на данный момент, у нас в компании используется Puppet, к
внедрению puppet я имел непосредсвенное отношение, и в целом как я, так и коллеги
результатом довольны. Но когда столкнулся с ansible, она меня очень заинтересовала,
потому как:

* Не нужен центральный сервер, запуск задач идет с локальной машины. Центральный
  сервер может быть вещь и полезная, но на нашей текущей конфигурации когда
  машин несколько десятков, он явно избыточен
* На сервере, на котором запускаетcя provision, не нужен никакой специальный софт,
  только python, который входит в base system практически всех дистрибутивов
* ansible нереально простая, причем не в ущерб функциональности, начинать работать
  можно стразу после _brew install ansible_ и беглого просмотра
  [примеров][ansible examples]. Вообще ansible по своей идеологии напоминает vim,
  пара простых решений внутри которое позволяют делать сложные вещи.
* Вся конфигурация описывается в YAML файлах, она декларативная и сразу же
  интегрирована с документацией.
* Позволяет выполнять разовые задачи на серверах, в Puppet для этого нужно использовать
  отдельный сервис [mcollective][mcollective]

После пары тестовых внедрений ansible, остались сугубо положительные впечатления,
и, скорее всего, в нашей компании вместе с плановым обновлением серверов
Ubuntu 12.04 -> 14.04, будет и одновременная миграция с puppet на ansible.

__Provision & Deploy__

С помощью ansible на сервере можно развернуть нужное окружение и из шаблона
сгенерить конфиг деплоя для mina, после чего запустить ее. Подробно шаги выглядят
так:

* Готовим сервер для приложения, устанавливаем нужные пакеты, настраиваем
* Ставим mina
* Из шаблона генерим файл _deploy.rb_
* Получившися _deploy.rb_ передаем в _mina_, и она нам из него создает файл _up_
  с командами для деплоя
* Запускаем _up_, который разворачивает приложение

В результате, все настройки только в одном месте в ansible, все нужные шаги выполяет
только ansible, задача mina просто создать один скрипт для деплоя. Описание
конфигурации благодаря ansible простое и понятное.

Cкачав и запустив [demo приложение][demo], можно увидеть, как это работает в живую

    $ git clone https://github.com/dima-exe/demo-deploy-using-ansible-and-mina.git
    $ brew install ansible
    $ vagrant up

В примере ставится postgres, nginx, и деплоится rails application. Само приложение
доступно по адресу [http://localhost:8080](http://localhost:8080).

Файлы которые используются:

* [ansible/site.yml][site.yml] переменные
* [ansible/provision.yml][provision.yml] настройка сервера
* [ansible/deploy.yml][deploy.yml] деплой приложения
* [ansible/templates][templates] шаблоны файлов


[ansible]: http://www.ansible.com/home
[mina]: http://nadarei.co/mina/
[heroku]: https://www.heroku.com/
[engine yard]: https://www.engineyard.com/
[ansible examples]: https://github.com/ansible/ansible-examples
[mcollective]: http://puppetlabs.com/mcollective
[demo]: https://github.com/dima-exe/demo-deploy-using-ansible-and-mina
[ci installer]: https://github.com/vexor/vx-install
[site.yml]: https://github.com/dima-exe/demo-deploy-using-ansible-and-mina/blob/master/ansible/site.yml
[provision.yml]: https://github.com/dima-exe/demo-deploy-using-ansible-and-mina/blob/master/ansible/provision.yml
[deploy.yml]: https://github.com/dima-exe/demo-deploy-using-ansible-and-mina/blob/master/ansible/deploy.yml
[templates]: https://github.com/dima-exe/demo-deploy-using-ansible-and-mina/tree/master/ansible/templates