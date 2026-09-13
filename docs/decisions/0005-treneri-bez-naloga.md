# 0005 — Treneri ne koriste aplikaciju

**Status:** usvojeno

## Kontekst
Plan je predviđao ulogu `trainer` sa ekranom na telefonu za označavanje termina. Vlasnik želi da aplikaciju koriste samo zaposleni na recepciji, menadžer i vlasnik.

## Odluka
- Uloge u `gym_users`: samo `owner`, `manager`, `reception`.
- Treneri su evidencija u tabeli `trainers` (ime, telefon, aktivan), bez naloga i bez pristupa.
- Personalne termine zakazuje i označava (održano / `no_show` / `excused` / otkazano) recepcija, menadžer ili vlasnik.
- Nema rute `(trainer)`.
- Izveštaj učinka trenera (8.4) ostaje nepromenjen.

## Posledice
- `pt_packages`, `pt_credits`, `pt_sessions`, `class_schedule`, `class_sessions` — `trainer_id` pokazuje na `trainers`.
- Pravilo čuvanja za bivše zaposlene važi i za neaktivne trenere (telefon posle 24 meseca).
