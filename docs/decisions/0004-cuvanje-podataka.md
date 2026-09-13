# 0004 — Politika čuvanja podataka

Datum: 2026-09-13 · Status: prihvaćeno, potvrditi sa pravnikom pre prvog kupca van pilota

## Odluka
- Član se ne briše fizički, već anonimizuje; finansijska istorija ostaje.
- Bivši član: automatska anonimizacija posle 24 meseca bez aktivnosti.
- Na zahtev člana: vlasnik anonimizuje odmah.
- Leadovi koji nisu postali članovi: lični podaci se brišu posle 12 meseci.
- SMS: telefon i tekst se brišu posle 12 meseci.
- Bivši zaposleni: telefon se briše posle 24 meseca, ime ostaje.
- Teretana koja prekine saradnju: izvoz, pa brisanje posle 90 dana.

## Posledice
- Kolone `members.last_activity_at` i `members.anonymized_at`.
- Anonimizacija čisti i `audit_log.before/after` za tog člana — jedini izuzetak od pravila da se audit log ne dira.
- Detalji u `docs/plan.md`, poglavlje 12.
