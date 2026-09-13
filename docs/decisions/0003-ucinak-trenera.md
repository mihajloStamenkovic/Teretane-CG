# 0003 — Izveštaj učinka trenera, bez provizije

Datum: 2026-09-13 · Status: prihvaćeno

## Kontekst
Vlasnik želi da vidi koliko termina ima svaki trener i koliko novca donosi teretani.

## Odluka
- **Treneri ne dobijaju proviziju, nikada.** Sistem je ne računa i ne čuva.
- Svaka sesija pri prelasku u `completed` ili `no_show` zamrzava `revenue_cents` (vrednost sesije).
- Vrednost sesije iz paketa = plaćena cena paketa / broj sesija; ostatak od zaokruživanja ide na poslednju sesiju.
- Prihod se pripisuje treneru koji je održao sesiju, ne treneru na paketu.
- Izveštaj prikazuje i „naplaćeno" (novac pri prodaji) i „realizovano" (odrađene sesije).

## Posledice
- Nema `commission_*` kolona na `gym_users` ni `pt_sessions`.
- Izveštaj 8.4 u planu, samo za vlasnika.
- Kriterijum Faze 6: realizovano + preostali krediti + istekli krediti = naplaćeni PT paketi.
