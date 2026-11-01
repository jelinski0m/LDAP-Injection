# LDAP injection lab

> Krótka ściąga po prezentacji: **co to był za atak**, **jak odpalić lab** i **jak zdobyć kolejne flagi**.

---

## LDAP injection w pigułce

LDAP injection to ten sam problem co SQL injection, tylko w zapytaniu do serwera katalogowego (LDAP, czyli książka adresowa firmy: użytkownicy, hasła, działy). Aplikacja **skleja tekst wpisany przez użytkownika z filtrem zapytania** — zamiast potraktować go jako zwykłe dane. Jeśli wpiszemy znaki specjalne filtra (`* ( ) & | !`), zmieniamy **sens** zapytania.

**Przykład.** Portal loguje filtrem:

```
(&(uid=NICK)(userPassword=HASŁO))
```

Wpiszmy w pole hasła samą gwiazdkę `*`. Filtr zaczyna znaczyć „użytkownik ma *jakiekolwiek* hasło" — czyli zawsze prawda. Hasło przestaje mieć znaczenie i logujemy się bez niego.

Dwie rzeczy warto zapamiętać:

- **`*` znaczy „cokolwiek"** (wildcard / „pole istnieje") — stąd logowanie bez hasła i wyciąganie listy kont.
- **Triki z SQL nie przenoszą się 1:1** — w LDAP nie ma komentarzy ani `UNION`. Filtr to drzewo warunków: `(& ...)` = „oraz", `(| ...)` = „lub", `(! ...)` = „nie".

**Obrona w jednym zdaniu:** nie wklejaj surowego tekstu do filtra — zawsze go „escapuj" (w PHP `ldap_escape`, reguły RFC 4515), ogranicz dozwolone znaki (allowlist) i sprawdzaj hasło przez operację `bind`, a nie przez wpisanie go do filtra.

---

## Jak wejść do aplikacji

Aplikacja jest **już postawiona i działa** — nie musisz nic uruchamiać. Po prostu otwórz ją w przeglądarce:

**Link do aplikacji:** https://atomwoz.pl/app/

> Aplikacja **pokazuje wygenerowany filtr LDAP** pod formularzem — podglądaj go, to najlepsza pomoc: od razu widać, co zrobił Twój payload.

---

## Flagi — wskazówki

Są **4 flagi**. Każda ma podpowiedź — spróbuj najpierw sam, dopiero potem czytaj dalej.
Flagi 1–3 to rozgrzewka; **flaga 4 jest wyraźnie trudniejsza** (wymaga skryptu).

### Flaga 1 — logowanie bez hasła → `FLAG{ldap_auth_byp4ss}`
Zaloguj się jako `administrator`, nie znając hasła.
- Filtr logowania: `(&(uid=administrator)(userPassword=...))`.
- W pole **hasła** wpisz `*`. Warunek hasła spełnia wtedy „jakiekolwiek hasło".
- Flaga czeka na panelu admina.

### Flaga 2 — enumeracja przez wyszukiwarkę → `FLAG{ldap_enum_w1ldcard}`
Znajdź ukryte konto i jego sekret przez wyszukiwarkę.
- Twój tekst trafia do `(cn=...)`.
- Wpisz samą `*` — zobaczysz **wszystkie** konta.
- Przejrzyj kolumnę `title` — jedno z kont trzyma tam sekret.

### Flaga 3 — blind (ślepe wyciąganie znak po znaku) → `FLAG{bl1nd_ldap_1nj3ction}`
Wydobądź atrybut `description` użytkownika `administrator` — choć aplikacja go **nie wyświetla** (mówi tylko „znaleziono / nie znaleziono").
- Dołóż własny warunek, np. `)(description=F*)`, i zgaduj znak po znaku: `Fa*`, `Fb*`, `Fc*`… patrząc, czy coś się znalazło.
- Ręcznie to działa, ale jest mozolne - zautomatyzować można tak jak w zadaniu 4.

### 🔴 Flaga 4 (trudniejsza) — blind z bisekcją i skryptem → `FLAG{ldap_0r_b1s3ct}`
To samo co flaga 3, ale **efektywnie** i z użyciem skryptu — zamiast setek żądań wyklikiwanych ręcznie.
- Cel: konto `svc-vault` (cn „Service Vault" — znajdź je przez `q=Service*`).
- Zamiast pytać znak po znaku, pytaj o **pół alfabetu naraz** operatorem `(|...)` i połowkuj grupę wg odpowiedzi (bisekcja, ~log₂ zamiast skanu liniowego).


