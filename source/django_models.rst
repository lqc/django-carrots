
Modele
======

Następnym krokiem będzie zdefiniowanie modeli naszej aplikacji.
Model opisuje, co i w jaki sposób może być przechowywane w bazie danych.
Z punktu widzenia języka Python, jest to zwykła klasa, dziedzicząca po ``models.Model``.

Nasza aplikacja będzie zawierać pytania oraz odpowiedzi do nich, dlatego utworzymy dwa modele: ``Poll`` oraz ``Choice``.
Model ``Poll`` zawiera treść pytania i datę publikacji. Model ``Choice`` zawiera odwołanie do odpowiedniego pytania,
treść odpowiedzi oraz liczbę głosów.

W pliku ``polls/models.py`` wpisujemy::

    from django.db import models

    class Poll(models.Model):
        question = models.CharField(max_length=200)
        pub_date = models.DateTimeField('date published')

    class Choice(models.Model):
        poll = models.ForeignKey(Poll)
        choice_text = models.CharField(max_length=200)
        votes = models.IntegerField(default=0)

Dodając nowe modele, zmieniliśmy schemat bazy danych.
Musimy ponownie wykonać ``syncdb``, aby nowe modele pojawiły się w bazie danych.

.. warning::
    Po wykonaniu ``syncdb`` nie można już dodać nowych pól do modelu. Można dodawać tylko nowe modele.
    Są sposoby, żeby to obejść, ale o tym w innej bajce.

.. code-block:: sh

   (warsztaty) ~/carrots$ python manage.py syncdb
   Creating tables ...
   Creating table polls_poll
   Creating table polls_choice
   Installing custom SQL ...
   Installing indexes ...
   Installed 0 object(s) from 0 fixture(s)

I tyle! Pewnie chcielibyśmy jednak mieć też możliwość edytowania obiektów. Najłatwiej to zrobić w interfejsie
administracyjnym.

W pliku ``polls/admin.py`` wpisujemy::

    from django.contrib import admin
    from polls.models import Poll, Choice

    admin.site.register(Poll)
    admin.site.register(Choice)

W ten sposób modele ``Poll`` oraz ``Choice`` będą dostępne w panelu administracyjnym.

.. note::

    Niektóre zmiany wymagają ponownego uruchomienia serwera.  W konsoli, gdzie jest uruchomiony
    serwer, wciskamy ``Ctrl+C`` i wykonujemy ``python manage.py runserver`` raz jeszcze.

Gdy ponownie wejdziemy na http://localhost:8000/admin/, zobaczymy, że pojawiła się tam nowa zakładka `Polls`.


Zabawa w konsoli
----------------

Django udostępnia swoją konsolę. Jest to zwykła konsola Pythona (tzn. możemy
robić dokładnie te same rzeczy, co po uruchomieniu polecenia ``python``), ale
dodatkowo możemy korzystać z narzędzi i modeli Django.

.. code-block:: sh

   (warsztaty) ~/carrots$ python manage.py shell

Gdy już jesteś w shellu::

    >>> from polls.models import Poll, Choice

Wszystkie ankiety w bazie; teraz nie ma tam nic, dlatego dostajemy pustą listę::

    >>> Poll.objects.all()
    []

Tworzymy pierwszą ankietę::

    >>> import datetime
    >>> p = Poll(question="What's new?", pub_date=datetime.datetime.now())

Zapisujemy ankietę w bazie danych. W tym celu zawsze trzeba wywołać metodę ``save()``::

    >>> p.save()

Każdy obiekt w bazie danych ma przypisane unikalne dla siebie ID::

    >>> p.id
    1

``p`` jest zwykłym obiektem. Możemy czytać jego atrybuty::

    >>> p.question
    "What's new?"
    >>> p.pub_date
    datetime.datetime(2012, 2, 26, 13, 0, 0, 775217)

Po zmianie atrybutów ponownie wywołujemy ``save()``, aby zapisać zmiany do bazy::

    >>> p.question = "What's up?"
    >>> p.save()

``objects.all()`` zwraca listę wszystkich obiektów w bazie danych::

    >>> Poll.objects.all()
    [<Poll: Poll object>]

Modele w Django są klasami, a w klasach możemy definiować metody. Metoda to taka funkcja, która dodatkowo dostaje
parametr ``self``, będący aktualnym obiektem (np. aktualną ankietą). Metody w klasach (modelach) pozwalają dodawać
dodatkowe zachowania lub zmieniać istniejące.

Jedną z takich metod jest ``__str__``, która pozwala zmienić sposób wyświetlania modelu (ankiety lub pytania).
``<Poll: Poll object>`` niewiele nam mówi. Naprawmy to, dodając metodę ``__str__`` do ``Poll`` i ``Choice``::

    class Poll(models.Model):
        # ...
        def __str__(self):
            return self.question

    class Choice(models.Model):
        # ...
        def __str__(self):
            return self.choice_text

Django będzie używało tych metod przy wyświetlaniu obiektów, nie tylko w konsoli, ale również we wspomnianym wcześniej
interfejsie administracyjnym.

Możemy też dodawać inne metody. W pliku ``carrots/polls/models.py`` dopisz
(koemntarze ``# ...`` tutaj oznaczają kod znajdujący się już w pliku)::

    import datetime
    from django.utils import timezone
    # ...
    class Poll(models.Model):
        # ...
        def was_published_recently(self):
            return self.pub_date >= datetime.datetime.now() - datetime.timedelta(days=1)

Zauważcie, że musieliśmy dodać ``import datetime``, aby móc korzystać z obiektów reprezentujących czas w Pythonie.

Zapiszmy te zmiany i uruchommy intepreter za pomocą polecenia ``python manage.py shell`` raz jeszcze::

    >>> from polls.models import Poll, Choice

    # Sprawdzmy czy dziala nasza nowa metoda __str__()
    >>> Poll.objects.all()
    [<Poll: What's up?>]

Do tej pory używaliśmy metody ``all``, która pozwala wyciągnąć listę wszystkich obiektów danego typu (np. wszystkich
pytań). Istnieją też inne metody pozwalające wyciągnąć obiekty spełniające określone warunki:

.. code-block:: python

    # Django pozwala na bardzo łatwe wyszukiwanie obiektów w bazie danych. Spójrzmy na kilka przykładów.
    >>> Poll.objects.filter(id=1)
    [<Poll: What's up?>]
    >>> Poll.objects.filter(question__startswith='What')
    [<Poll: What's up?>]
    >>> Poll.objects.get(pub_date__year=2012)
    <Poll: What's up?>

    # Próba pobrania nieistniejącego obiektu spowoduje głośne protesty Pythona.
    # Ale do tego jesteśmy już przyzwyczajeni.
    >>> Poll.objects.get(id=2)
    Traceback (most recent call last):
        ...
    DoesNotExist: Poll matching query does not exist. Lookup parameters were {'id': 2}

    # Wypróbujmy teraz naszą własną metodę.
    >>> p = Poll.objects.get(pk=1)
    >>> p.was_published_recently()
    True

Możemy też uzyskać dostęp do odpowiedzi (``Choice``) na pytania:

.. code-block:: python

    # Na razie nasza ankieta nie zawiera żadnych odpowiedzi. Dodajmy jakieś!
    >>> p.choice_set.all()
    []

    # ... na przykład trzy. Użyjemy do tego metody "create". W wyniku dostaniemy obiekt "Choice".
    >>> p.choice_set.create(choice_text='Not much', votes=0)
    <Choice: Not much>
    >>> p.choice_set.create(choice_text='The sky', votes=0)
    <Choice: The sky>
    >>> c = p.choice_set.create(choice_text='Just hacking again', votes=0)

    # Mając obiekt "Choice", możemy też znaleźć ankietę, do której on należy.
    >>> c.poll
    <Poll: What's up?>

    # ...I na odwrót, wszystkie odpowiedzi dla danej ankiety.
    >>> p.choice_set.all()
    [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]
    >>> p.choice_set.count()
    3

    # A teraz coś trudniejszego. Co robi to zapytanie?
    >>> Choice.objects.filter(poll__pub_date__year=2012)
    [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]

    # Na koniec usuńmy jedną z odpowiedzi. Służy do tego metoda "delete".
    >>> c = p.choice_set.filter(choice_text__startswith='Just hacking')
    >>> c.delete()

Podsumowanie
------------

* Modele tworzymy definiując klasy dziedziczące po ``models.Model`` w pliku ``polls/models.py``.
* Po utworzeniu nowego modelu pamiętaj, aby uruchomić ``python manage.py syncdb``.
* Pobranie wszystkich obiektów danego modelu::

    Poll.objects.all()

* Pobranie obiektów spełniających określony warunek::

    Poll.objects.filter(question__startswith='What')

* Pobranie pojedynczego obiektu::

    Poll.objects.get(id=2)

