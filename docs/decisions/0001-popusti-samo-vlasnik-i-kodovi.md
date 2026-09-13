# 0001 — Popuste daje samo vlasnik; ostali koriste kodove

Datum: 2026-09-13 · Status: prihvaćeno

## Kontekst
Plan je predviđao da recepcija daje popust do nekog limita. Vlasnik ne želi da iko osim njega odlučuje o popustu.

## Odluka
- Ručni popust (izmena cene uz razlog) sme isključivo `owner`.
- Vlasnik generiše kodove za popust: procenat ili fiksan iznos, sa rokom važenja i opcionim brojem korišćenja.
- Recepcija i menadžer popust ostvaruju samo unosom važećeg koda.
- Limit popusta za recepciju se ukida.

## Posledice
- Nova tabela `discount_codes`, kolona `memberships.discount_code_id`.
- Provera uloge i koda u bazi, ne samo u UI.
- Kod se ne može iskoristiti offline (`max_uses` zahteva server).
- Otvoreno: da li kodovi važe i za personalne pakete i POS.
