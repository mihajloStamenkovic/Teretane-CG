# CLAUDE.md

Kontekst i pravila za rad na ovom repozitorijumu. Pročitaj u celini pre prve izmene u sesiji.

---

## Projekat

Interna aplikacija za vođenje teretane. Koriste je **isključivo zaposleni** — vlasnik, menadžer, recepcija. **Treneri ne koriste aplikaciju** — oni su samo evidencija (`trainers`), bez naloga. Evidencija članova, dolazaka, članarina, uplata, personalnih treninga, prodaje, ormarića i leadova, sa izveštajima.

Tržište: Crna Gora. Valuta: EUR. Interfejs: crnogorski/srpski, latinica.

Multi-tenant od početka — jedna instalacija opslužuje više teretana. Start je sa jednom teretanom, ali šema i RLS se nikad ne pojednostavljuju zbog toga.

**Detaljna specifikacija je u `docs/plan.md`. Ovaj fajl su pravila; plan je sadržaj.**

---

## Van obima — nikad ne implementiraj

Ako zadatak ili tvoja procena vode ka nečemu sa ove liste, **stani i prijavi konflikt**. Ne implementiraj i ne predlaži zaobilazno rešenje.

- **Fiskalizacija** — nema IKOF, JIKR, XML potpisivanja, sertifikata, komunikacije sa Poreskom upravom. Teretana fiskalizuje na svojoj kasi, van ovog sistema.
- **Online naplata** — nema Stripe-a ni bilo kog payment gateway-a. Sve uplate se unose ručno.
- **Naplata pretplate teretanama** — licenca se fakturiše van sistema.
- **Portal za članove** — član nema nalog, login, mobilnu aplikaciju, niti rezerviše sam. Ne postoji nijedan ekran namenjen članu.
- **Turnikéti i kontrola pristupa** — nije u obimu.
- **Knjigovodstvo** — nema glavne knjige, PDV prijava, knjigovodstvenih izvoza.

---

## Tvrda pravila

Prekršaj bilo kog od ovih znači da zadatak nije završen.

1. **RLS uz migraciju.** Svaka nova tabela dobija `gym_id`, uključen RLS i politike **u istoj migraciji** u kojoj je kreirana. Nema „dodaću politike kasnije".
2. **Novac je `integer` u centima.** Nikad `float`, `real`, `double precision` ni `money`. Kolone nose sufiks `_cents`.
3. **Uplate se ne brišu i ne menjaju.** `payments` ima trigger koji blokira UPDATE i DELETE. Ispravka je isključivo novi red sa `is_reversal = true` i negativnim iznosom.
4. **`service_role` ključ samo na serveru.** Nikad u klijentskoj komponenti, nikad u `NEXT_PUBLIC_*` promenljivoj.
5. **Offline mutacije nose `client_op_id`.** Svaka operacija koja može nastati bez mreže (check-in, uplata, POS prodaja) ima `client_op_id uuid UNIQUE`. Idempotencija se garantuje unique indeksom u bazi, ne proverom u kodu.
6. **Primenjene migracije se ne menjaju.** Nova promena = nova migracija. Nikad izmena fajla koji je već pušten.
7. **Ne izmišljaj funkcionalnosti.** Ako nešto deluje da nedostaje, napiši to u odgovoru i nastavi sa onim što je traženo.
8. **Bez `localStorage` za poslovne podatke.** Isključivo IndexedDB kroz `/lib/offline`.
9. **Korisnički tekst ide u `/lib/i18n/sr.ts`**, ne u JSX. Nijedan string na ekranu ne sme biti hardkodovan u komponenti.
10. **Član ima najviše jedan aktivan paket.** Ovo čuva partial unique index na `memberships(member_id) WHERE status IN ('active','frozen')` — zamrznut paket se računa kao aktivan. Ne zaobilazi ga i ne uklanjaj ga.
11. **Ručni popust sme samo `owner`.** Svi ostali popust ostvaruju isključivo kroz `discount_code_id`. Provera je u bazi, ne samo u UI.
12. **Član se ne briše, već anonimizuje.** Finansijska istorija ostaje; lični podaci nestaju iz `members`, `audit_log`, `sms_messages` i Storage-a. Rokovi su u `docs/plan.md`, poglavlje 12.

---

## Model podataka — pravila koja se stalno krše

- Svaka tabela: `id uuid PK`, `gym_id uuid NOT NULL`, `created_at`, `updated_at`.
- Poslovne tabele imaju `deleted_at` (soft delete). **Finansijske nemaju** — nema brisanja.
- Enumi su Postgres `enum` tipovi, ne `text` sa proverom u kodu.
- Vremena su `timestamptz` u UTC. Prikaz u `Europe/Podgorica`. Čisti datumi (`start_date`, `end_date`) su tip `date`.
- Nazivi tabela i kolona: engleski, `snake_case`, tabele u množini.
- Kod kartice (`cards.code`) je **nasumičan token**. Nikad `member_no`, nikad sekvenca, nikad nešto što se može pogoditi.
- Ključ ormarića se vezuje za **`check_in_id`**, ne za `member_id`.
- Vrednost sesije se **zamrzava** u `pt_sessions.revenue_cents` pri prelasku u `completed` ili `no_show`. Kasnija promena cene ne dira istoriju.
- **Treneri nemaju proviziju.** Ne dodaji kolone, obračun ni izveštaj provizije.
- Kod za popust važi za članarine, PT pakete, pojedinačne PT i POS — sve četiri tabele nose `discount_cents`, `discount_code_id`, `discount_reason`.
- `no_show` (neopravdano) troši kredit; `excused` (opravdano, razlog obavezan) ne troši.
- Procenti se čuvaju kao `integer` u baznim poenima (`_bp`, 2500 = 25%), nikad u `_cents` koloni.
- Kodovi za popust (`discount_codes.code`) su nasumični, ne brišu se — samo deaktiviraju.

---

## Struktura

```
/app              rute, grupisane po ulozi: (reception), (management)
/components       deljene komponente
/lib/db           upiti i tipovi generisani iz Supabase šeme
/lib/offline      IndexedDB sloj i outbox
/lib/sms          apstrakcija SMS provajdera
/lib/i18n         korisnički tekst
/supabase/migrations   numerisane migracije
/supabase/seed.sql     test podaci
/docs/plan.md     puna specifikacija
/docs/decisions/  zapisi donetih odluka
/.claude/agents/  definicije Claude Code agenata
```

---

## Komande

```bash
npm run dev            # razvojni server
npm run build          # produkcijski build — mora proći pre kraja faze
npm run lint
npm run typecheck
npm run test           # uključuje test izolacije tenanta

supabase migration new <naziv>
supabase db reset      # ponovo primeni sve migracije + seed
supabase gen types typescript --local > lib/db/types.ts
```

Posle svake izmene šeme **obavezno** regeneriši tipove.

---

## Tok rada u sesiji

1. Pročitaj `docs/plan.md` — deo koji se odnosi na fazu na kojoj radimo.
2. Pre pisanja koda opiši šta ćeš uraditi i sačekaj potvrdu ako zadatak dodiruje šemu baze, RLS politike ili finansijske tabele.
3. Radi u malim koracima. Migracija → tipovi → upiti → UI, tim redom.
4. Posle izmene šeme pokreni `supabase db reset` i `npm run typecheck`.
5. Pre kraja faze pokreni test izolacije tenanta. Ako padne, faza nije gotova.
6. Kriterijum „Gotovo je kada" za svaku fazu stoji u `docs/plan.md`. Ne prelazi na sledeću dok nije ispunjen.
7. Agenti: posle migracije `migration-guard` i `access-tester`, pre commita `rules-reviewer`, na kraju faze `phase-verifier`. Ostali po tabeli u `docs/plan.md` §13. PALO bilo kog agenta blokira sledeći korak. Agenti ne pišu kod aplikacije — to radi glavna sesija.

---

## Test izolacije tenanta

Postoji od Faze 0 i pokreće se na kraju svake faze. Proverava da korisnik iz teretane A ne može pročitati ni jedan red iz teretane B, ni preko jednog query-ja, ni kroz jednu novu tabelu.

**Svaka nova tabela mora biti dodata u ovaj test.** Ako si dodao tabelu a nisi dodao test, faza nije gotova.

---

## Kada stati i pitati

Stani i pitaj umesto da pretpostaviš:

- Zadatak traži nešto sa liste „van obima"
- Zadatak zahteva promenu već primenjene migracije
- Zadatak bi prekršio pravilo „jedan aktivan paket po članu"
- Treba doneti poslovnu odluku koja nije u planu (prag, limit, podrazumevana vrednost)
- Postoje dva razumna pristupa sa različitim posledicama po model podataka

Ne pitaj za: imenovanje komponenti, raspored fajlova, izbor biblioteke za nešto sitno, formatiranje. To odluči sam i idi dalje.

---

## Specifičnosti ovog proizvoda

**Recepcija je primarni korisnik.** Ekrani za recepciju su gusti, tabelarni, optimizovani na broj klikova i rad sa tastature. Ne na lepotu. Check-in ekran mora dati vizuelni odgovor ispod 300 ms, čitljiv sa dva metra.

**Vlasnik gleda na telefonu i samo čita.** Njegov dashboard nema nijedno polje za unos.

**Skener kartica se ponaša kao tastatura.** Nema drajvera ni integracije — kod stiže u fokusirano polje praćen Enterom.

**Offline nije opcija.** Check-in, uplata i POS moraju raditi bez mreže. Izveštaji, podešavanja i SMS ne moraju.

**Sezonalnost je velika.** Poređenja u izveštajima idu sa istim mesecom prethodne godine, ne sa prethodnim mesecom.

---

## Stil odgovora

Kratko i direktno, na srpskom. Bez preambula i bez rezimiranja onoga što je već rečeno. Kad završiš zadatak, reci šta si promenio i šta treba proveriti — ne prepričavaj kod.

Ako nešto ne radi ili si nesiguran, reci to odmah umesto da nastaviš.
