Testování
=========

V tomto cvičení se budeme zabývat automatickým testováním kódu.
Modul unittest ze standardní knihovny už byste měli znát,
co to jsou jednotkové testy a k čemu slouží tedy rovnou přeskočím.

Pokud si chcete přečíst krátký text o tom, jak testovat, zkuste [blogový
zápisek Michala Hořejška](http://blog.horejsek.com/matka-moudrosti-jak-testovat).

pytest
------

Rovnou se podíváme na velmi oblíbený balíček [pytest], který oproti standardnímu
unittestu přináší mnoho výhod. Začneme jednoduchou ukázkou z modulu `isholiday`
z [předchozího cvičení](03_moduly.md).

```python
import isholiday

def test_xmas_2016():
    """Test whether there is Christmas in 2016"""
    holidays = isholiday.getholidays(2016)
    assert (24, 12) in holidays
```

Test uložíme někam do projektu, třeba do souboru `test/test_holidays.py` a
nainstalujeme a spustíme `pytest`:

```bash
(env)$ python -m pip install pytest
(env)$ PYTHONPATH=. pytest
=============================== test session starts ================================
platform linux -- Python 3.5.2, pytest-3.0.3, py-1.4.31, pluggy-0.4.0
rootdir: ..isholiday, inifile: 
collected 1 items 

test/test_holidays.py .

============================= 1 passed in 0.24 seconds =============================
```

Všimněte si několika věcí:

 * Stačí funkci pojmenovat `test_*` a `pytest` pozná, že se jedná o test.
 * Pokud balíček nemáme nainstalovaný, je třeba nastavit `PYTHONPATH`. Vždy je ale lepší testovat nainstalovaný balíček.
 * V ukázce je použit obyčejný `assert` a žádná metoda z `unittest`.

Pytest upravuje chování assertu, což oceníte především, pokud test selže:

```python
    ...
    assert (23, 12) in holidays
```

```
===================================== FAILURES =====================================
__________________________________ test_xmas_2016 __________________________________

    def test_xmas_2016():
        """Test whether there is Christmas in 2016"""
        holidays = isholiday.getholidays(2016)
>       assert (23, 12) in holidays
E       assert (23, 12) in {(1, 1), (1, 5), (5, 7), (6, 7), (8, 5), (17, 11), ...}

test/test_holidays.py:6: AssertionError
============================= 1 failed in 0.24 seconds =============================
```

S obyčejným assertem si vystačíte pro většinu testovaných případů kromě
ověření vyhození výjimky. To se dělá takto:

```python
import pytest

def f():
    raise SystemExit(1)

def test_mytest():
    with pytest.raises(SystemExit):
        f()
```

Více o základním použití pytestu najdete v [dokumentaci].

[pytest]: http://pytest.org/
[dokumentaci]: http://docs.pytest.org/en/latest/getting-started.html

### Parametrické testy

Jednou z vlastností pytestu, která často přichází vhod jsou [parametrické testy].
Pokud bychom například chtěli otestovat, jestli je štědrý den svátkem nejen
v roce 2016, ale v jiných letech, nemusíme spát testů více ani použít forcyklus.

Nevýhoda více téměř stejných testů je patrná sama o sobě, nevýhoda forcyklu je 
v tom, že celý test selže, i pokud selže jen jeden průběh cyklem, zároveň se
průběh testu při selhání ukončí.

Místo toho tedy použijeme parametrický test:

```python
import pytest
import isholiday

@pytest.mark.parametrize('year', (2015, 2016, 2017, 2033, 2048))
def test_xmas(year):
    """Test whether there is Christmas"""
    holidays = isholiday.getholidays(year)
    assert (24, 12) in holidays

```

(Místo výpisu hodnot v `tuple` lze použít jakýkoliv objekt, přes který jde
iterovat, tedy i např. volání `range()`.)

Pro více podrobný výpis výsledku testů můžete použít přepínač `-v`:

```bash
(env)$ PYTHONPATH=. pytest -v
...
test/test_holidays.py::test_xmas[2015] PASSED
test/test_holidays.py::test_xmas[2016] PASSED
test/test_holidays.py::test_xmas[2017] PASSED
test/test_holidays.py::test_xmas[2033] PASSED
test/test_holidays.py::test_xmas[2048] PASSED
...
```

Vždy je dobré pokusit se nějaký test rozbít v samotném kódu, který testujeme,
abychom se ujistili, že testujeme správně.
Přidám tedy na konec funkce `getholidays()` tento pesimistický kus kódu:

```python
    if year > 2020:
        # After the Zygon war, the puppet government canceled all holidays
        holidays = set()
```

```bash
(env)$ PYTHONPATH=. pytest -v
...
test/test_holidays.py::test_xmas[2015] PASSED
test/test_holidays.py::test_xmas[2016] PASSED
test/test_holidays.py::test_xmas[2017] PASSED
test/test_holidays.py::test_xmas[2033] FAILED
test/test_holidays.py::test_xmas[2048] FAILED
...
```

[parametrické testy]: http://doc.pytest.org/en/latest/parametrize.html

### Fixtures

Často se stává, že před samotným testem potřebujte spustit nějaký kus kódu,
abyste získali to, co teprve chcete testovat. Příkladem může být například
inicializace objektu pro komunikaci s nějakým API.

V pytestu k tomuto účelu nejlépe slouží tz. [fixtures], které se v samotných
testech používají jako argumenty funkcí.

[fixtures]: http://doc.pytest.org/en/latest/fixture.html

```python
import pytest

@pytest.fixture
def client():
    import twitter
    return twitter.Client(...)

def test_seacrh_python(client):
    tweets = client.search('python', size=1)
    assert len(tweets) == 1
    assert 'python' in tweets[0].text.lower()
```

Pokud potřebujete po použití s fixturou ještě něco udělat, můžete místo `return`
použít `yield`. Zde příklad, který si můžete rovnou vyzkoušet:

```python
import pytest


class Dummy:
    def __init__(self):
        print('Creating Dummy instance')
        ...

    def forward(self, arg):
        return arg

    def cleanup(self):
        print('Cleaning up Dummy instance')
        ...


@pytest.fixture
def dummy():
    d = Dummy()
    yield d
    d.cleanup()


@pytest.mark.parametrize('arg', (1, float, None))
def test_with_fixture(dummy, arg):
    assert arg == dummy.forward(arg)
```

Pro zobrazení věcí, co se v testech něco vypisují na standardní výstup, je třeba
použít `pytest -s`.

Hromadu dalších příkladů použití pytestu najdete dokumentaci, v
[sekci s příklady](http://doc.pytest.org/en/latest/example/index.html).

flexmock
--------

Při psaní testů se občas hodí trochu podvádět. Například když nechceme,
aby testy měli nějaký vedlejší účinek, když chceme testovat něco, co závisí na
náhodě a podobně. Obecně se tomuto říká *mocking*, a existuje více různých
knihoven, které to umožňují, jednou z nich je [flexmock].

[flexmock]: https://flexmock.readthedocs.io/

### Falešné objekty

Při testování často potřebujeme nějaký objekt, který má určité atributy a
metody. Vytvářet si pro každý takový objekt třídu (jako v příkladě výše)
může být ubíjející.

```python
class FakePlane:
    operational = True
    model = 'MIG-21'
    def fly(self): pass

plane = FakePlane()  # this is tedious!
```
Flexmock umožňuje vytvoření objektu rychle a jednoduše:

```python
plane = flexmock(operational=True,
                 model='MIG-21',
                 fly=lambda: None)
```

betamax
-------

obecně o problému testování živých API

jak to betamax řeší

jak ho použít (dependency injection se session)

Poznámky
--------

kam patří testy v balíčku? kam patří případní potřebné soubory?

Tox a Travis?

Úkol
----

Napsat testy k dosavadním úlohám
