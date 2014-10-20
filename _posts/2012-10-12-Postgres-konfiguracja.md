---
layout: post
title: (PL) Postgres - konfiguracja na linuksie
---

Problem
-------

Jeśli chcesz skonfigurować pod swój użytek postgresql, ale występuje u Ciebie błąd:

{% highlight bash %}
FATAL: role "xxx" does not exist
{% endhighlight %}

lub
{% highlight bash %}
FATAL: password authentication failed for user "postgres"
{% endhighlight %}

to przeczytaj dalej - jest wyjaśnienie, jak skonfigurować użytkownika postgres.

Zakładam, że Twój system to linuks - rozwiązanie testowane na Ubuntu 12.04 i Debianie : )

Rozwiązanie
-----------

Po pierwsze musimy się zalogować na użytkownika postgres:
{% highlight bash %}
$ su - postgres@
{% endhighlight %}
Hasło powinno być domyślne: **postgres** (lub ew. powinno być puste).

Potem włączamy konsolę psql:
{% highlight bash %}
$ psql@
{% endhighlight %}

Następnie zmieniamy hasło na "postgres" - oczywiście może być też jakieś inne, dowolnie przez siebie wybrane.
{% highlight bash %}
postgres=# ALTER USER postgres WITH PASSWORD 'postgres';
{% endhighlight %}

Ważne, żeby był średnik na końcu, bo polecenie nie zwróci żadnego błędu i łatwo to przeoczyć.*

Aby wyjść z psql, wpisujemy **\q**

I to właściwie wystarczy. Rozwiązanie jest dedykowane railsom, więc jeśli mamy już nasz nowy projekt oparty na bazie postgresql, to musimy w `config/database.yml` zmienić wartości pól `username` i `password` na `postgresql` oraz dopisać do wszystkich trybów (`production`, `development`, `test`) dwie wartości:
{% highlight bash %}
hostname: localhost
port: 5432
{% endhighlight %}

Potem można jeszcze profilaktycznie (nie wiem, czy jest to wymagane) zrestartować serwer postgresa:
{% highlight bash %}
sudo /etc/init.d/postgresql restart
{% endhighlight %}



