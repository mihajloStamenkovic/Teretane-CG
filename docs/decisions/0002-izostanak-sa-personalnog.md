# 0002 — Opravdan i neopravdan izostanak sa personalnog

Datum: 2026-09-13 · Status: prihvaćeno

## Kontekst
Plan je predviđao da `no_show` uvek troši kredit, uz podešavanje po teretani.

## Odluka
- Neopravdan izostanak (`no_show`) troši kredit.
- Opravdan izostanak (`excused`) ne troši kredit i traži razlog.
- Pravilo je fiksno, nije podešavanje po teretani.
- `no_show` se naknadno može prebaciti u `excused`; kredit se vraća, promena ide u audit log.

## Posledice
- Novi status `excused` u enumu `pt_sessions.status`, kolone `excuse_reason`, `marked_by`, `marked_at`.
- Otvoreno: da li trener dobija proviziju za `no_show`.
