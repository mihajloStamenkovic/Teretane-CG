# 0003 — Izveštaj učinka trenera

Datum: 2026-09-13 · Status: prihvaćeno

## Kontekst
Vlasnik želi da vidi koliko termina ima svaki trener i koliko novca donosi teretani.

## Odluka
- Svaka sesija pri prelasku u `completed` ili `no_show` zamrzava `revenue_cents` (vrednost sesije) i `commission_cents`.
- Vrednost sesije iz paketa = cena paketa / broj sesija; ostatak od zaokruživanja ide na poslednju sesiju.
- Prihod i provizija pripadaju treneru koji je održao sesiju, ne treneru na paketu.
- Izveštaj prikazuje i „naplaćeno" (novac pri prodaji) i „realizovano" (odrađene sesije), plus neto za teretanu.
- Procenat provizije se čuva u baznim poenima (`commission_percent_bp`), fiksna provizija u `commission_fixed_cents`.

## Posledice
- Izveštaj 8.4 u planu, samo za vlasnika.
- Kriterijum Faze 6: realizovano + preostali krediti + istekli krediti = naplaćeni PT paketi.
