# Прототип Hub Wallet -
(https://github.com/sonm-io/Smart-dummy/tree/master/contracts/Hubs)
(По ссылке можно почитать ридми, так что желательно посмотреть одним глазком).

## Суть условий контракта:

Контракт работает в 4х состояниях - Created, Registred, Idle, Suspected (+Punished)

При создании контракта функция конструктора задает адреса ДАО, фабрики, вайтлиста, владельца кошелька, а так же некоторые другие переменные, например длительность рассчетного периода (сейчас стоит 30 дней). Рассчетный период - это период в течении которого хаб может проводить выплаты майнерам, но не может забрать весь баланс себе ( и смыться на карибы, почему - опишу дальше).

В состоянии Created контракт может зарегистрировать себя в вайт-листе, при этом замораживая строго определенную часть собственного баланса (1 токен SONM). Это сделано для того, что бы хаб не мог например сначала положить себе 0.00000001 SNM, зарегатся, а после докинуть основные (допустим) 100SNM - первоначальная сумма всегда фиксированная. Кроме того, в процессе регистрации контракта в вайт-листе, он запоминает время регистрации.

После того, как контракт зарегистрировался в вайт-листе он переходит в состояние Registred в котором ему доступны функции
 ``` transfer , payday, suspect ```
Рассмотрим их по порядку.

## Функиция ```transfer```
позволяет контракту проводить выплаты майнерам хаба. Она работает следующим образом. Вначале определяется
```lockFee```-  % с выплаты, который будет заблокирован на время рассчетного периода. По умолчанию равен 30%. Затем определяется лимит (общее количество заблокированных средств + замороженные средства(1 SNM) + процент с конкретно этой транзакции) и проверяется условие баланса - если баланс меньше лимита, то соответственно транзакция не проходит, если норм - то заблокированный процент прибавляется к общему количеству залоченых средств, а контракт вызывает функцию ```Approve```(см.ниже) по отношению к майнеру. Почему это реализованно именно так - см. опять же ниже в разделе функции
PayDay


## Функция ```Approve```
не переводит токены на счет майнера, но позволяет майнеру самому перевести себе средства со счета кошелька. Это не позволяет хабу регистрировать в системе один кошелек, а проводить выплаты через другой - т.к. майнер будет ожидать подтверждения именно с того кошелька, который зарегистрирован в системе и никак иначе. Approve является стандартной функцией (стандарт ERC20)

## Функция ```PayDay```
переводит состояние контракта из состояния
```Registred```
в состояние простоя
```Idle```
.  Эта функция сверяет время регистрации с текущей датой и поэтому может быть вызвана только в конце рассчетного периода. Если это условие выполняется, то она переводит 0.5% из залоченых (30% с каждой операции) средств на счет ДАО, после чего разблокирует залоченные средства и переводит контракт в состояние простоя. В этом состоянии (простоя) контракт далее может вывести весь остаток средств на счет владельца и/или зарегистрироваться в вайт-листе заново. В состоянии простоя хаб не может проводить выплаты майнерам или быть раскулачен.

Таким образом, если владелец хаба захочет вывести средства с кошелька себе на счет, то у него есть два варианта - пойти честным путем, дождаться конца рассчетного периода и  заплатить DAO 0,5% от заблокированных средств, после чего получить весь остаток себе на счет - либо сжульничать и через функцию transfer вывести все средства себе под видом выплат майнерам, но в  таком случае на контракте останутся заблокированы 30% всех средств + 1 SNM, что стимулирует хаб действовать честным путем.

Так же у контракта еще есть состояния
```Suspected``` + ```Punished``` , которые мы так же рассмотрим.
В состоянии
```Registred``` -  т.е. в состоянии когда контракт  зарегистрирован в вайт-листе - DAO ( и только DAO!) может вызвать функцию
```suspect``` ( в последствии будет переименована на blame) таким образом переводя контракт в состояние
```suspected``` - т.е. подозреваемый в мошенничестве. Данная функция блокирует ВСЕ средства на счету контракта на 120 дней.

Из состояния
```suspected```
могут быть вызваны (так же только DAO) следующие функции:
## Функция ```rehub```
 реабилитация хаба, которая снимает все блокировки и переводит контракт в состояние ```idle```. Может быть вызвана в любое время.
## Функция ```gulag```
Перефразируя известную фразу из "магазинчика Бо", эта функция выполняет "макание головой в говно особей выбираемых общим собранием".

Функция может быть вызвана только комитетом ДАО, и только по истечению 120 дней с момента перевода контракта в состояние
```suspected```, при этом все заблокированные средства контракта пересылаются на счет ДАО, состояние контракта окончательно и безповоротно переводится в состояние
```punished```
,а сам владелец контракта кошелька раскулачивается и ссылается в Сибирь.