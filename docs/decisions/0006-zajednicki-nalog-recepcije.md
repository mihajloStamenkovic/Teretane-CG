# 0006 — Zajednički nalog recepcije

**Status:** usvojeno

## Kontekst
Svi recepcioneri koriste jedan nalog „recepcija" koji ne ističe. Bez dodatnog koraka sistem ne bi znao ko je naplatio, pustio člana bez članarine ili napravio razliku u kasi.

## Odluka
- Po teretani jedan `gym_users` nalog sa `role = reception`, sesija se ne odjavljuje sama.
- Tabela `receptionists` (ime, telefon, aktivan) — bez naloga.
- Pri otvaranju smene recepcioner bira svoje ime → `shifts.receptionist_id`.
- Nalog recepcije ne upisuje uplate, check-in ni POS bez otvorene smene; sve se pripisuje recepcioneru te smene.
- Izveštaji „po zaposlenom" za recepciju idu po recepcioneru smene.

## Posledice
- `check_ins` dobija `shift_id`.
- Ko god sedne za računar recepcije ima prava recepcije — zato storno, ručni popust, izveštaji i podešavanja ostaju samo za vlasnika i menadžera.

## Uz ovu odluku usvojeno
- Kompozitni FK `(gym_id, id)` između tabela.
- RLS proverava `gym_users` (aktivan, uloga) pri svakom upitu, ne samo JWT.
- `location_id` na `payments`, `memberships`, `pt_sessions`.
- Postojeći Supabase projekat je razvojni; produkcijski se pravi pri predaji vlasniku.
