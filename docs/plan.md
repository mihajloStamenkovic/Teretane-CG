# PLAN — Aplikacija za vođenje teretane

Interni sistem za zaposlene u teretani. Evidencija članova, dolazaka, članarina, uplata, personalnih treninga, prodaje, ormarića i leadova, sa izveštajima za vlasnika.

Tržište: Crna Gora. Valuta: EUR. Jezik interfejsa: crnogorski/srpski (latinica).

---

## 1. Granice proizvoda

### 1.1 Šta aplikacija JESTE

Interna evidencija koju koriste isključivo zaposleni u teretani. Zamenjuje svesku i Excel tabelu na recepciji.

### 1.2 Šta aplikacija NIJE — izričito van obima

Ove stavke se NE implementiraju ni u jednoj fazi. Ako se pojavi predlog da se dodaju, odbaciti bez diskusije:

- **Nema fiskalizacije.** Ni IKOF, ni JIKR, ni komunikacija sa Poreskom upravom, ni digitalni sertifikati. Teretana fiskalizuje na svojoj postojećoj kasi, potpuno izvan ovog sistema.
- **Nema online naplate.** Ni Stripe, ni bilo koji payment gateway, ni kartično plaćanje kroz aplikaciju. Sve uplate se unose ručno.
- **Nema naplate pretplate teretanama.** Licenca se fakturiše van sistema. U aplikaciji postoji samo status pretplate (aktivna/suspendovana) koji podešava super-admin.
- **Nema portala za članove.** Član nema nalog, nema login, nema mobilnu aplikaciju, ne rezerviše sam termine. Član fizički dolazi i komunicira sa recepcijom.
- **Nema integracije sa turniketima** u prvoj godini.
- **Nema knjigovodstva.** Nema glavne knjige, nema PDV prijava, nema izvoza za knjigovođu osim običnog CSV-a.

### 1.3 Poznato ograničenje koje se prihvata

Iznosi u ovoj aplikaciji neće se automatski poklapati sa prometom na fiskalnoj kasi. Sistem ne pokušava da ih uskladi. Kontrola se ostvaruje kroz zatvaranje smene sa brojanjem keša i kroz audit log, ne kroz poređenje sa kasom.

---

## 2. Ključne poslovne odluke

Ove odluke su donete i ne preispituju se tokom implementacije:

| # | Odluka | Posledica po model |
|---|--------|--------------------|
| 1 | Član ima **tačno jedan aktivan paket (članarinu)**; zamrznuta se računa kao aktivna | Partial unique index na `memberships(member_id) WHERE status IN ('active','frozen')` |
| 2 | Personalni treninzi su **odvojen entitet**, ne članarina | Ne krše pravilo iz reda 1; član može imati mesečnu + kredit za personalce |
| 3 | Personalci se plaćaju **i unapred u paketu i pojedinačno po treningu** | Dva toka: `pt_credits` (paket) i sesija sa `credit_id = NULL` (pojedinačno) |
| 4 | Kartice **izdaje teretana**, barkod Code128 | Zaseban entitet `cards`, član može imati istoriju više kartica |
| 5 | Uplata se **nikad ne briše ni menja** | Ispravka isključivo storno zapisom |
| 6 | **Multi-tenant od nulte faze** | `gym_id` na svakoj tabeli, RLS na svakoj tabeli |
| 7 | **SMS obaveštenja postoje** | Provajder, šablon, kvota, saglasnost člana |
| 8 | **Grupni treninzi nisu u prvoj verziji**, ali model ih predviđa | Tabele se kreiraju u Fazi 0, UI tek u Fazi 13 |
| 9 | Offline režim je **obavezan** za check-in i uplate | Outbox obrazac, idempotency ključevi |
| 10 | **Ručni popust daje isključivo vlasnik.** Svi ostali popust ostvaruju samo unosom koda koji je vlasnik generisao | Tabela `discount_codes`; kod važi za članarine, personalne (paket i pojedinačno) i POS; provera uloge u bazi |
| 11 | Personalni: **neopravdan izostanak troši kredit, opravdan ne troši** | Statusi `no_show` i `excused` na `pt_sessions`, razlog obavezan za `excused` |
| 12 | Vlasnik vidi **učinak svakog trenera**: termine i novac koji donosi | `pt_sessions.revenue_cents` zamrznut pri završetku, izveštaj 8.4 |
| 14 | **Treneri ne dobijaju proviziju — nikada** | Nema kolona za proviziju, nema obračuna provizije, nema izveštaja o proviziji |
| 13 | Bivši član se **anonimizuje, ne briše** | Finansijska istorija ostaje, lični podaci nestaju (poglavlje 12) |
| 15 | **Treneri ne koriste aplikaciju.** Koriste je samo vlasnik, menadžer i recepcija | Treneri su evidencija u tabeli `trainers`, bez naloga; uloge u `gym_users` su samo owner, manager, reception; nema rute `(trainer)` |
| 16 | **Start sa jednom teretanom**, model spreman za više | `gym_id` i RLS svuda od početka; druga teretana se dodaje bez izmene šeme |
| 17 | **Recepcija radi na jednom zajedničkom nalogu** „recepcija", bez isteka sesije; recepcioner bira svoje ime pri otvaranju smene | Tabela `receptionists` (bez naloga); `shifts.receptionist_id`; sve što nalog recepcije uradi pripisuje se recepcioneru otvorene smene |
| 18 | **Veze između tabela idu preko `(gym_id, id)`** | Kompozitni FK — baza sama sprečava da red iz teretane A pokazuje na red iz teretane B |
| 19 | **Prava se proveravaju u bazi pri svakom upitu**, ne samo iz JWT-a | RLS pomoćna funkcija čita `gym_users` (aktivan, uloga) jednom po upitu; deaktivacija i promena uloge važe odmah |

---

## 3. Tehnički stack

| Sloj | Izbor | Napomena |
|------|-------|----------|
| Framework | Next.js (App Router) + TypeScript | Server Actions za mutacije |
| Stilizacija | Tailwind + shadcn/ui | Gusti, tabelarni UI za recepciju |
| Baza | Supabase (PostgreSQL) | RLS kao primarni sigurnosni sloj |
| Auth | Supabase Auth | `gym_id` i `role` u JWT claim-u |
| Hosting | Vercel, EU region | Baza takođe EU region. Postojeći Supabase projekat je **razvojni** (bez pravih podataka); produkcijski se pravi posebno pri predaji vlasniku |
| Cron | pg_cron | Isticanje članarina, SMS red, noćni obračuni |
| Offline | PWA + IndexedDB | Service worker, outbox sinhronizacija |
| SMS | Infobip ili lokalni agregator | Apstrahovan iza internog interfejsa |
| Barkod | USB skener kao HID uređaj | Nula integracije, ponaša se kao tastatura |

### 3.1 Zabrane na nivou stacka

- **Bez `float` za novac.** Svi iznosi su `integer` u centima. Kolone se imenuju sa sufiksom `_cents`.
- **Bez `localStorage` za poslovne podatke.** Isključivo IndexedDB kroz definisani sloj.
- **Bez filtriranja po `gym_id` samo u aplikaciji.** RLS politika je obavezna za svaku tabelu.
- **Bez `service_role` ključa u klijentskom kodu.** Isključivo u cron poslovima i serverskim rutama.

---

## 4. Model podataka

Sve tabele imaju: `id uuid PK`, `gym_id uuid NOT NULL`, `created_at`, `updated_at`. Tabele sa poslovnim podacima dodatno imaju `deleted_at` (soft delete). Finansijske tabele NEMAJU `deleted_at` — nema brisanja.

### 4.1 Tenant i pristup

**`gyms`**
`name`, `slug` (subdomen), `address`, `phone`, `email`, `logo_url`, `timezone` (default `Europe/Podgorica`), `subscription_status` (active | suspended | trial), `settings jsonb`

**`locations`**
`gym_id`, `name`, `address`, `phone`, `is_default`, `active`
U prvoj verziji svaka teretana ima jednu lokaciju, ali sve transakcione tabele nose `location_id` od početka.

**`gym_users`**
`gym_id`, `user_id` (FK na `auth.users`), `role` (owner | manager | reception), `location_id` (nullable), `full_name`, `phone`, `active`
Jedan korisnik može biti u više teretana — zato je ovo zasebna tabela, a ne kolona na useru. Ovde su samo nalozi koji se prijavljuju u aplikaciju: vlasnik i menadžer lično, a recepcija kao **jedan zajednički nalog** po teretani (`role = reception`), bez isteka sesije.

**`receptionists`**
`gym_id`, `full_name`, `phone`, `active`
Ljudi koji rade na recepciji. Nemaju nalog. Pri otvaranju smene na nalogu recepcije bira se ime sa ovog spiska; sve akcije naloga recepcije dok je smena otvorena pripisuju se tom recepcioneru (pazar, override, storno zahtevi, audit). Nalog recepcije ne može da upisuje uplate, check-in ni POS bez otvorene smene.

**`trainers`**
`gym_id`, `full_name`, `phone`, `active`
Treneri nemaju nalog i ne koriste aplikaciju. Svi `trainer_id` u šemi pokazuju na ovu tabelu. Termine im zakazuje i označava recepcija, menadžer ili vlasnik.

**`audit_log`**
`gym_id`, `user_id`, `action`, `entity_type`, `entity_id`, `before jsonb`, `after jsonb`, `ip`, `created_at`
Piše se kroz Postgres trigger, ne iz aplikacije. Nikad se ne briše. Jedini izuzetak: anonimizacija člana briše lične podatke iz `before`/`after` za tog člana (poglavlje 12).

### 4.2 Članovi

**`members`**
`member_no` (sekvencijalan po teretani), `first_name`, `last_name`, `phone`, `email`, `birth_date`, `gender`, `photo_url`, `address`, `joined_at`, `status` (active | inactive | blocked), `notes`, `source`, `emergency_contact_name`, `emergency_contact_phone`, `data_consent_at`, `sms_consent` (bool), `sms_consent_at`, `last_activity_at`, `anonymized_at`, `created_by`, `deleted_at`

`last_activity_at` se ažurira triggerom na check-in, uplatu i prodaju članarine — koristi ga pravilo čuvanja podataka.

Indeksi: trigram index na `first_name`, `last_name`, `phone` — pretraga na recepciji mora biti trenutna.

**`cards`**
`member_id`, `code` (unique po `gym_id`, nasumičan token — **nikad** `member_no` ni sekvenca), `issued_at`, `issued_by`, `status` (active | lost | replaced | revoked), `replaced_by_card_id`, `deposit_cents`

Član može imati više redova; samo jedan sa `status='active'`.

### 4.3 Članarine

**`packages`** — katalog
`name`, `kind` (time_based | visit_based | daily), `duration_days`, `visit_count`, `price_cents`, `allow_freeze`, `max_freeze_days`, `color`, `sort_order`, `active`

**`memberships`** — kupljena članarina
`location_id`, `member_id`, `package_id`, `price_paid_cents`, `discount_cents`, `discount_code_id` (nullable), `discount_reason`, `start_date`, `end_date`, `visits_total`, `visits_used`, `status` (active | expired | frozen | cancelled), `sold_by`, `previous_membership_id`

> **Ograničenje:** `CREATE UNIQUE INDEX ... ON memberships(member_id) WHERE status IN ('active','frozen')`

> **Popust:** ako je `discount_cents > 0`, mora biti popunjeno tačno jedno od `discount_code_id` ili `discount_reason` (CHECK). Ručni popust (`discount_reason` bez koda) sme da upiše samo `owner` — proverava se u bazi (trigger/funkcija prodaje), ne samo u UI.

**`discount_codes`** — kodovi za popust, generiše ih isključivo vlasnik
`code` (unique po `gym_id`, nasumičan, 8 znakova bez dvosmislenih slova 0/O/1/I, lako se kuca), `kind` (percent | fixed), `percent_bp` (bazni poeni, 2500 = 25%, samo za `percent`), `amount_cents` (samo za `fixed`), `valid_from`, `valid_until` (nullable), `max_uses` (nullable = neograničeno), `uses_count`, `note`, `active`, `created_by`

Kod važi svuda gde se nešto prodaje: `memberships`, `pt_credits`, pojedinačne `pt_sessions` i `pos_sales`. Svaka od tih tabela nosi isti trojac kolona `discount_cents`, `discount_code_id`, `discount_reason` i isti CHECK i proveru uloge kao `memberships`.

- Popust se primenjuje na cenu paketa, sesije ili ukupan iznos POS računa; fiksni popust ne može oboriti cenu ispod nule.
- Percent se ne čuva u `_cents` koloni — to nije novac.
- `uses_count` se povećava u istoj transakciji kao prodaja, uz zaključavanje reda — `max_uses` se ne može prekoračiti ni istovremenim prodajama.
- Kod se ne briše (vezan je za finansijsku istoriju), samo se deaktivira.
- Unos koda zahteva vezu sa serverom; offline se kod ne može iskoristiti. POS prodaja bez koda i dalje radi offline.

**`membership_freezes`**
`membership_id`, `start_date`, `end_date`, `days`, `reason`, `created_by`
Odmrzavanje produžava `memberships.end_date` za broj iskorišćenih dana.

### 4.4 Dolasci

**`check_ins`**
`location_id`, `shift_id` (nullable — vlasnik/menadžer mogu bez smene), `member_id`, `membership_id` (nullable), `card_id` (nullable), `checked_in_at`, `checked_out_at`, `method` (card | search | manual), `created_by`, `device_id`, `is_override` (bool), `override_reason`, `client_op_id` (uuid, unique — idempotency za offline)

`is_override` znači da je recepcija pustila člana bez aktivne članarine. Ovo je izveštaj koji vlasnik gleda.

### 4.5 Novac

**`shifts`**
`location_id`, `receptionist_id` (obavezan kad smenu otvara nalog recepcije), `opened_by`, `opened_at`, `closed_by`, `closed_at`, `opening_cash_cents`, `counted_cash_cents`, `expected_cash_cents`, `difference_cents`, `difference_reason`, `status` (open | closed | locked)

**`payments`**
`location_id`, `member_id` (nullable — anonimna POS prodaja), `amount_cents`, `method` (cash | card | transfer | other), `purpose` (membership | personal_training | pos | locker | card_replacement | other), `ref_type`, `ref_id`, `paid_at`, `shift_id`, `created_by`, `note`, `is_reversal` (bool), `reverses_payment_id`, `client_op_id`

> **Pravilo:** `payments` nema UPDATE ni DELETE. Postgres trigger to blokira na nivou baze, ne na nivou aplikacije. Ispravka = novi red sa `is_reversal=true` i negativnim `amount_cents`.

### 4.6 Personalni treninzi

**`pt_packages`** — katalog
`name`, `sessions_count`, `price_cents`, `validity_days` (nullable), `trainer_id` (nullable — cena može biti opšta ili po treneru), `active`

**`pt_credits`** — kupljen paket
`member_id`, `pt_package_id`, `trainer_id`, `sessions_total`, `sessions_used`, `price_paid_cents`, `discount_cents`, `discount_code_id`, `discount_reason`, `sold_by`, `purchased_at`, `expires_at`, `status` (active | used_up | expired)

**`pt_sessions`**
`location_id`, `member_id`, `trainer_id`, `credit_id` (nullable — NULL znači plaćanje po treningu), `scheduled_at`, `duration_min`, `status` (scheduled | completed | no_show | excused | cancelled), `excuse_reason`, `marked_by`, `marked_at`, `price_cents` (samo kad `credit_id` je NULL), `discount_cents`, `discount_code_id`, `discount_reason` (samo kad `credit_id` je NULL), `payment_id`, `revenue_cents`, `notes`

Statusi:
- `completed` — održano, troši kredit
- `no_show` — **neopravdan** izostanak, troši kredit
- `excused` — **opravdan** izostanak, ne troši kredit; `excuse_reason` obavezan (CHECK), promena ide u audit log
- `cancelled` — termin otkazan unapred, ne troši kredit

`trainer_id` na sesiji je trener koji je stvarno održao termin (zamena je moguća); prihod se pripisuje njemu, ne treneru sa `pt_credits`.

**Zamrzavanje pri prelasku u `completed` ili `no_show`:** `revenue_cents` — vrednost sesije za teretanu. Za kredit: `price_paid_cents / sessions_total` zaokruženo naniže, a poslednja sesija kredita dobija ostatak, tako da zbir sesija tačno odgovara plaćenoj ceni paketa (posle popusta). Za pojedinačnu: `price_cents − discount_cents`. Upisuje se jednom i ne menja se. Za `excused` i `cancelled` je 0.

Treneri ne dobijaju proviziju — sistem je ne računa i ne čuva.

### 4.7 Prodaja i zalihe

**`products`**
`name`, `category`, `barcode`, `price_cents`, `cost_cents`, `stock_qty`, `min_stock`, `track_stock` (bool), `active`

**`pos_sales`**
`location_id`, `member_id` (nullable), `shift_id`, `sold_by`, `sold_at`, `total_cents`, `discount_cents`, `discount_code_id`, `discount_reason`, `payment_id`, `client_op_id`

**`pos_sale_items`**
`sale_id`, `product_id`, `qty`, `unit_price_cents`, `line_total_cents`

**`stock_movements`**
`product_id`, `type` (in | sale | writeoff | correction), `qty`, `reason`, `ref_id`, `created_by`

### 4.8 Ormarići

**`lockers`**
`location_id`, `number`, `zone`, `type` (daily | rented), `status` (free | occupied | out_of_service)

**`locker_assignments`** — dnevni ključ
`locker_id`, `member_id`, `check_in_id`, `issued_at`, `issued_by`, `returned_at`, `returned_by`

Ključ se vezuje za **check-in sesiju**, ne za člana. Bez toga se ne zna ko ga trenutno drži.

**`locker_rentals`** — mesečni najam
`locker_id`, `member_id`, `start_date`, `end_date`, `price_cents`, `payment_id`, `status`

### 4.9 Leadovi

**`leads`**
`first_name`, `last_name`, `phone`, `email`, `source` (instagram | facebook | google | preporuka | prolaznik | ostalo), `source_detail`, `status` (new | visited | trial | offer | won | lost), `lost_reason`, `assigned_to`, `trial_date`, `converted_member_id`, `next_followup_at`, `created_by`

**`lead_activities`**
`lead_id`, `type` (call | visit | sms | note), `note`, `created_by`, `created_at`

### 4.10 Oprema

**`equipment`**
`location_id`, `name`, `serial`, `purchased_at`, `warranty_until`, `status` (ok | needs_service | broken | retired), `last_service_at`, `next_service_at`, `notes`

**`equipment_service_logs`**
`equipment_id`, `serviced_at`, `performed_by`, `cost_cents`, `description`

### 4.11 SMS

**`sms_templates`**
`name`, `trigger` (expiry_soon | expired | birthday | inactive | welcome | manual), `days_offset`, `body`, `active`

**`sms_messages`**
`member_id` (nullable), `phone`, `body`, `template_id`, `status` (queued | sent | delivered | failed), `provider_message_id`, `cost_cents`, `sent_at`, `error`, `created_by`

### 4.12 Grupni treninzi — model se kreira u Fazi 0, UI tek u Fazi 13

**`class_types`** — `name`, `duration_min`, `capacity`, `color`, `active`
**`class_schedule`** — `class_type_id`, `trainer_id`, `weekday`, `start_time`, `capacity`, `active`
**`class_sessions`** — `schedule_id`, `date`, `trainer_id`, `capacity`, `status`
**`class_bookings`** — `session_id`, `member_id`, `status` (booked | attended | no_show | cancelled | waitlist)

Razlog za kreiranje sada: dodavanje ovih tabela kasnije je jeftino, ali prepravljanje `check_ins` i `memberships` da bi ih podržali nije.

### 4.13 Troškovi

**`expenses`**
`location_id`, `category` (kirija | struja | plate | oprema | marketing | ostalo), `amount_cents`, `expense_date`, `description`, `is_recurring`, `created_by`

Bez ovoga dashboard prikazuje prihod, a vlasnika zanima profit.

---

## 5. Uloge i prava

| Modul | owner | manager | reception |
|-------|:-----:|:-------:|:---------:|
| Članovi — pregled i unos | ✓ | ✓ | ✓ |
| Članovi — anonimizacija (umesto brisanja) | ✓ | — | — |
| Check-in | ✓ | ✓ | ✓ |
| Prodaja članarine | ✓ | ✓ | ✓ |
| Ručni popust / izmena cene | ✓ | — | — |
| Popust unosom koda | ✓ | ✓ | ✓ |
| Generisanje i gašenje kodova za popust | ✓ | — | — |
| Storno uplate | ✓ | ✓ | — |
| Zatvaranje smene | ✓ | ✓ | ✓ (svoje) |
| Izmena zatvorene smene | ✓ | ✓ | — |
| POS prodaja | ✓ | ✓ | ✓ |
| Katalog paketa i cene | ✓ | ✓ | — |
| Treneri — evidencija | ✓ | ✓ | — |
| Personalni — raspored | ✓ | ✓ | ✓ |
| Personalni — označavanje održano / izostanak | ✓ | ✓ | ✓ |
| Učinak trenera (izveštaj 8.4) | ✓ | — | — |
| Leadovi | ✓ | ✓ | ✓ |
| Finansijski izveštaji | ✓ | ✓ | — |
| Troškovi | ✓ | — | — |
| Podešavanja teretane | ✓ | — | — |
| Audit log | ✓ | ✓ (čita) | — |

Treneri nisu uloga — nemaju nalog i ne pristupaju aplikaciji.

Ova matrica se implementira **dvaput**: kao RLS politika u bazi i kao provera u UI sloju. RLS je izvor istine; UI samo sakriva dugmad.

---

## 6. Offline strategija

### 6.1 Šta mora da radi bez interneta

- Check-in skeniranjem kartice i pretragom po imenu
- Pregled kartona člana i statusa članarine
- Unos uplate
- POS prodaja
- Izdavanje ormarića

### 6.2 Šta NE mora

- Izveštaji, dashboard, podešavanja, SMS, izmena kataloga, brisanje bilo čega

### 6.3 Mehanizam

1. **Lokalna kopija** u IndexedDB: aktivni članovi, aktivne kartice, aktivne članarine, katalog paketa i proizvoda. Osvežava se pri svakom uspešnom sinhronizovanju.
2. **Outbox**: svaka mutacija izvršena offline dobija `client_op_id` (uuid v4) i upisuje se u lokalni red.
3. **Idempotency**: server odbacuje duplikate po `client_op_id`. Kolona je unique u bazi, ne provera u kodu.
4. **Sinhronizacija**: pri povratku veze red se šalje redom, append-only, bez brisanja neuspelih. Neuspeli ostaju sa `error` poljem.
5. **Vidljiv indikator** u zaglavlju: online / offline / N nesinhronizovanih.
6. **Zabrana zatvaranja smene** dok postoje nesinhronizovane stavke.

### 6.4 Konflikti

Jedini realan konflikt je da član obnovi članarinu na jednoj lokaciji dok je druga offline. Rešenje: `memberships` se ne kreira offline ako član već ima aktivnu — offline se dozvoljava samo produžetak postojeće, a nova prodaja čeka vezu. U praksi se ovo dešava retko i bolje je blokirati nego dobiti dva aktivna paketa.

---

## 7. Ključni tokovi

### 7.1 Check-in (najvažniji ekran u aplikaciji)

Ekran je stalno otvoren na recepciji, fokus uvek u polju za unos.

1. Skener unosi kod kartice + Enter (ili recepcioner kuca ime/telefon).
2. Sistem pronalazi člana i njegov aktivan paket.
3. Prikaz u jednoj velikoj kartici: ime, slika, status, datum isteka ili preostali dolasci.
4. Vizuelni odgovor u **manje od 300 ms**, čitljiv sa dva metra:
   - **Zeleno** — ulaz dozvoljen
   - **Žuto** — članarina ističe za ≤ 5 dana ili je ostalo ≤ 2 dolaska
   - **Crveno** — nema aktivnog paketa, članarina istekla, ili kartica prijavljena kao izgubljena
5. Kod crvenog: dugme „Obnovi" vodi pravo u prodaju paketa, ili „Pusti svejedno" koje traži razlog i upisuje `is_override`.
6. Za `visit_based` paket inkrementira se `visits_used`; kad dođe do nule status prelazi u `expired`.
7. Sporedno: brojač trenutno prisutnih i lista poslednjih 10 ulazaka.

**Ivični slučajevi:** kartica skenirana dvaput u roku od 2 minuta ignoriše se bez greške; kartica koja ne postoji prikazuje jasnu poruku i dugme za vezivanje za člana; zamrznuta članarina daje crveno sa posebnom porukom.

### 7.2 Obnova članarine (dugme)

1. Na kartonu člana dugme „Obnovi" predlaže **poslednji paket** koji je član imao, ne najskuplji i ne default.
2. Datum početka = **kasniji od (danas, `end_date` tekuće članarine)**. Član koji obnovi tri dana ranije ne sme da izgubi ta tri dana. Ako je članarina istekla pre više od 7 dana, počinje od danas.
3. Cena je popunjena iz paketa. Recepcija i menadžer je **ne mogu menjati** — popust ostvaruju isključivo unosom koda; sistem proverava kod (aktivan, u roku važenja, ispod `max_uses`) i upisuje `discount_code_id` i `discount_cents`. Samo vlasnik može ručno smanjiti cenu, uz obavezan `discount_reason`. Popust je uvek vidljiv u `discount_cents`, nikad kao tiša niža cena.
4. Jedan klik kreira: novu `memberships`, prebacuje staru u `expired`, kreira `payments` vezanu za tekuću smenu.
5. Sve u jednoj transakciji. Ako bilo šta padne, ništa se ne upisuje.

### 7.3 Zatvaranje smene

1. Recepcioner na početku smene bira svoje ime sa spiska `receptionists`, unosi početni keš i otvara smenu. Jedna otvorena smena po lokaciji.
2. Sve uplate u toku smene automatski dobijaju `shift_id`.
3. Na kraju unosi prebrojan keš.
4. Sistem prikazuje očekivano, prebrojano i razliku. Razlika se **ne sakriva**.
5. Ako je |razlika| veća od praga iz podešavanja, unos razloga je obavezan.
6. Smena prelazi u `closed`. Izmena uplata iz zatvorene smene moguća je samo za manager/owner i uvek ide u audit log.

### 7.4 Personalni trening

**Tok A — paket unapred:** prodaja `pt_credits` → uplata → svaka održana sesija troši jedan kredit, bez nove uplate.

**Tok B — plaćanje po treningu:** sesija se kreira sa `credit_id = NULL` i `price_cents` → po završetku se evidentira uplata.

Recepcija (ili menadžer/vlasnik) u rasporedu po treneru označava „održano" / „nije se pojavio — neopravdano" / „opravdano odsutan". Neopravdan izostanak (`no_show`) **troši kredit**; opravdan (`excused`) **ne troši** i traži razlog. Pravilo je fiksno, nije podešavanje po teretani. Recepcija može naknadno prebaciti `no_show` u `excused` (npr. član donese potvrdu) — kredit se vraća, promena ide u audit log.

### 7.5 Lead

1. Recepcija unosi lead kad neko svrati da pogleda ili pozove.
2. Obavezno: izvor i ko ga je primio.
3. Automatski se postavlja `next_followup_at` na +3 dana.
4. Na dashboardu recepcije stoji lista leadova za kontakt danas.
5. Konverzija kreira člana i puni `converted_member_id`.
6. Gubitak traži `lost_reason` iz fiksne liste (cena, lokacija, radno vreme, otišao kod konkurencije, bez odgovora, ostalo).

---

## 8. Izveštaji

### 8.1 Dashboard vlasnika (telefon, samo čitanje)

Jedan ekran, bez unosa: pazar juče i ovaj mesec, broj aktivnih članova, koliko ističe ove nedelje, razlika u kasi juče, broj `is_override` ulazaka ove nedelje, prihod od personalnih ovaj mesec po treneru (link na 8.4).

### 8.2 Operativni

- Dnevni pazar po načinu plaćanja i po zaposlenom (za nalog recepcije — po recepcioneru smene)
- Lista članova sa isteklom članarinom (sa telefonima, za zvanje)
- Dolasci po satu i danu u nedelji — određuje raspored osoblja
- Override ulasci — ko je i koliko puta puštao bez članarine
- Nevraćeni ključevi
- Zalihe ispod minimuma

### 8.3 Vlasnički

- **Očekivani prihod sledećeg meseca** — koliko članarina ističe i koliko to vredi
- **Churn** — ko nije obnovio prošlog meseca, poimenično
- **Poređenje sa istim mesecom prošle godine** — sezonalnost u Crnoj Gori je prevelika da bi poređenje sa prethodnim mesecom išta značilo
- Prosečan vek člana i ukupan prihod po članu
- Konverzija leadova po izvoru i po zaposlenom
- Prihod minus troškovi = profit po mesecu
- Iskorišćenost kodova za popust — koliko puta, ukupan iznos popusta, prodaja po kodu

### 8.4 Učinak trenera (vlasnik, telefon, samo čitanje)

Lista trenera za izabrani mesec, sortirana po realizovanom prihodu. Za svakog trenera:

| Metrika | Izvor |
|---------|-------|
| Zakazani termini | `pt_sessions` u periodu, svi statusi |
| Održani | `status = completed` |
| Neopravdani izostanci | `status = no_show` |
| Opravdani izostanci | `status = excused` |
| Otkazani | `status = cancelled` |
| Broj aktivnih klijenata | različiti `member_id` sa bar jednom održanom sesijom u periodu |
| Prodati paketi (broj i naplaćeno) | `pt_credits` sa tim `trainer_id`, `purchased_at` u periodu |
| Pojedinačni treninzi (naplaćeno) | `payments` vezane za `pt_sessions` sa `credit_id IS NULL` |
| **Realizovan prihod** | zbir `pt_sessions.revenue_cents` (`completed` + `no_show`) |
| Dati popusti | zbir `discount_cents` na PT paketima i pojedinačnim treninzima tog trenera |
| Preostali krediti (obaveza) | neiskorišćene sesije aktivnih kredita × vrednost sesije |
| Isto, isti mesec prošle godine | poređenje zbog sezonalnosti |

Ispod liste: **istekli neiskorišćeni krediti** u periodu — prihod teretane koji nije pripisan nijednom treneru.

Razlika između „naplaćeno" i „realizovano": paket plaćen u januaru a odrađen do marta donosi novac u januaru, a treneru se pripisuje kroz mesece u kojima je stvarno radio. Vlasnik vidi oba broja.

Klik na trenera otvara listu njegovih sesija u periodu (datum, član, status, vrednost).

---

## 9. Faze izgradnje

Svaka faza je jedna ili više sesija sa Claude Code. Faza se ne napušta dok kriterijum „gotovo je kada" nije ispunjen.

### Faza 0 — Temelji
Repo, Next.js, Supabase projekat, migracije, **sve tabele iz poglavlja 4** (uključujući grupne treninge), RLS politike, auth, `gym_users`, audit trigger, osnovni layout i navigacija po ulozi, seed skripta sa test korisnicima. Seed ima dve teretane samo lokalno, da test izolacije ima šta da dokaže; u Supabase projekat u oblaku idu samo migracije, nikad seed.

> **Gotovo je kada:** korisnik sa `role=reception` iz teretane A ne može pročitati nijedan red iz teretane B ni preko jednog query-ja, i to je dokazano testom.

### Faza 1 — Članovi i kartice
CRUD članova, trenutna pretraga, slika, saglasnosti, izdavanje i zamena kartice, import iz CSV/Excel, anonimizacija na zahtev i automatska anonimizacija kroz pg_cron (poglavlje 12).

> **Gotovo je kada:** 200 članova uvezeno iz stvarne tabele pilot teretane, pretraga po imenu vraća rezultat ispod 200 ms, a anonimizovan član nema nijedan lični podatak ni u `members`, ni u `audit_log`, ni u Storage-u.

### Faza 2 — Paketi i članarine
Katalog paketa, prodaja, obnova dugmetom, ručni popust (samo vlasnik), generisanje kodova za popust (vlasnik) i unos koda pri prodaji, zamrzavanje, automatsko isticanje kroz pg_cron.

> **Gotovo je kada:** član ne može imati dva aktivna paketa ni preko UI-ja ni direktnim insertom u bazu; korisnik koji nije `owner` ne može upisati popust bez važećeg koda ni direktnim insertom; kod sa `max_uses` ne može biti iskorišćen više puta ni istovremenim prodajama.

### Faza 3 — Check-in
Ekran sa skenerom, pretraga, semafor, override sa razlogom, brojač prisutnih, istorija dolazaka.

> **Gotovo je kada:** skeniranje kartice do vizuelnog odgovora traje ispod 300 ms na stvarnom hardveru recepcije.

### Faza 4 — Uplate i smene
Unos uplate, storno, otvaranje i zatvaranje smene, dnevni pazar.

> **Gotovo je kada:** UPDATE i DELETE nad `payments` su odbijeni na nivou baze, dokazano testom.

### Faza 5 — Offline
Service worker, IndexedDB kopija, outbox, idempotency, indikator statusa, blokada zatvaranja smene sa nesinhronizovanim stavkama.

> **Gotovo je kada:** sa isključenom mrežom obavljeno 20 check-inova i 5 uplata, a po povratku veze u bazi je tačno 20 i 5 — ni jedan više ni manje, uz dvostruko slanje reda.

### Faza 6 — Personalni treninzi
Katalog, kupovina kredita, pojedinačne sesije, evidencija trenera, raspored po treneru, opravdan/neopravdan izostanak, kodovi za popust na PT paketima i pojedinačnim treninzima, obračun prihoda po sesiji, izveštaj učinka trenera (8.4).

> **Gotovo je kada:** promena cene PT paketa ne menja `revenue_cents` na već završenim sesijama; `excused` ne troši kredit a `no_show` troši; na test podacima zbir realizovanog prihoda svih trenera + preostali krediti + istekli krediti tačno odgovara ukupno naplaćenim PT paketima.

### Faza 7 — POS i zalihe
Katalog proizvoda, brza prodaja, veza sa članom ili anonimno, kod za popust na račun (samo online), kretanje zaliha, alarm ispod minimuma.

> **Gotovo je kada:** stanje zaliha posle 50 prodaja i jednog prijema robe odgovara ručnom prebrojavanju.

### Faza 8 — Ormarići i oprema
Mapa ormarića, izdavanje vezano za check-in, alarm za nevraćene, mesečni najam, evidencija opreme i servisa.

> **Gotovo je kada:** na kraju dana lista nevraćenih ključeva tačno odgovara ormarićima u statusu `occupied`.

### Faza 9 — Leadovi
Unos, statusi, follow-up lista, konverzija u člana, izveštaj po izvoru i zaposlenom.

> **Gotovo je kada:** konverzija leada kreira člana i zadržava vezu, a izveštaj po izvoru daje tačne brojeve na test podacima.

### Faza 10 — SMS
Apstrakcija provajdera, šabloni, automatski okidači (isticanje, rođendan, neaktivnost), ručno slanje, kvota po teretani, poštovanje `sms_consent`.

> **Gotovo je kada:** član bez saglasnosti ne dobija nijednu poruku ni kroz jedan okidač, i kvota se ne može prekoračiti.

### Faza 11 — Izveštaji i dashboard
Svi izveštaji iz poglavlja 8, izvoz u CSV.

> **Gotovo je kada:** vlasnik pilot teretane potvrdi da se brojke poklapaju sa njegovom evidencijom za prethodni mesec.

### Faza 12 — Multi-lokacija i super-admin
Prebacivanje između lokacija, super-admin panel (lista teretana, suspendovanje, impersonacija sa logovanjem), onboarding nove teretane, izvoz svih podataka tenanta.

> **Gotovo je kada:** nova teretana je kreirana, popunjena i operativna za manje od 30 minuta bez pisanja SQL-a.

### Faza 13 — Grupni treninzi
Kalendar, raspored, rezervacije, lista čekanja, otkazivanje sa rokom, evidencija dolaska na čas.

> **Gotovo je kada:** pilot teretana ili prvi kupac to zatraži. Ne ranije.

---

## 10. Konvencije

### 10.1 Kod i baza

- Nazivi tabela i kolona: engleski, `snake_case`, tabele u množini
- Interfejs: crnogorski/srpski latinica; sav korisnički tekst u jednom fajlu, ne razbacan po komponentama
- Novac: `integer`, centi, sufiks `_cents`
- Vreme: čuva se u UTC (`timestamptz`), prikazuje u `Europe/Podgorica`
- Datumi bez vremena (`start_date`, `end_date`): tip `date`
- Enumi: Postgres `enum` tipovi, ne slobodan tekst
- Migracije: numerisane, jedna po logičkoj promeni, **nikad izmena već primenjene migracije**
- Svaka nova tabela dobija RLS politiku **u istoj migraciji** u kojoj je kreirana

### 10.2 Struktura

```
/app             rute (grupisane po ulozi: (reception), (management))
/components      deljene komponente
/lib/db          upiti, tipovi generisani iz Supabase šeme
/lib/offline     IndexedDB sloj i outbox
/lib/sms         apstrakcija provajdera
/lib/i18n        korisnički tekst
/supabase        migracije, RLS politike, seed
/docs/plan.md    ovaj plan
/docs/decisions  zapisi donetih odluka
/.claude/agents  definicije Claude Code agenata
CLAUDE.md        pravila za agenta (mora ostati u korenu)
```

### 10.3 Pravila za agenta

Ova pravila idu u `CLAUDE.md` u korenu repozitorijuma:

1. **Ne dodaj funkcionalnost koja nije u planu.** Ako deluje da nešto nedostaje, prijavi to, nemoj implementirati.
2. **Nikad ne piši migraciju bez RLS politike** za tabelu koju kreira.
3. **Nikad ne koristi `float` za novac** i nikad `service_role` ključ van serverskog koda.
4. **Nikad ne brišeš finansijske zapise.** Ako zadatak traži brisanje uplate, prijavi konflikt sa planom.
5. **Svaka mutacija koja može nastati offline mora imati `client_op_id`.**
6. Pre kraja faze pokreni test izolacije tenanta iz Faze 0. Ako padne, faza nije gotova.
7. Ne menjaj već primenjene migracije. Nova promena = nova migracija.
8. Poslovni tekst ide u fajl sa prevodima, ne u JSX.

---

## 11. Otvorena pitanja

Za odluku pre odgovarajuće faze, ne blokiraju start:

1. **Prag razlike u kasi** iznad kojeg je razlog obavezan — konkretan iznos, čuva se u `gyms.settings` (Faza 4)
2. **SMS provajder i cena po poruci** u Crnoj Gori, registracija sender ID-ja (Faza 10)

Odloženo, nije tema do daljeg: model licenciranja prema drugim teretanama.

---

## 12. Politika čuvanja podataka

Osnovno pravilo: **lični podaci se ne čuvaju duže nego što treba, a finansijska istorija ostaje cela.** Zato se član nikad ne briše fizički — anonimizuje se. Uplate, članarine, sesije i dolasci ostaju, ali pokazuju na člana bez ličnih podataka, pa izveštaji i prošle godine ostaju tačni.

### 12.1 Anonimizacija člana

Postgres funkcija (SECURITY DEFINER), jedna transakcija:
- `first_name` → „Bivši član", `last_name` → `member_no`
- `phone`, `email`, `birth_date`, `address`, `photo_url`, `notes`, `emergency_contact_*` → NULL; slika se briše iz Storage-a
- `sms_consent` → false; `anonymized_at` → now()
- aktivne kartice → `revoked`
- lični podaci tog člana brišu se iz `audit_log.before/after` i iz `sms_messages.phone/body`
- u `audit_log` se upisuje samo da je anonimizacija izvršena, ko i kada — bez ličnih podataka

Anonimizacija je nepovratna.

### 12.2 Rokovi

| Podatak | Rok | Šta se dešava |
|---------|-----|---------------|
| Aktivan član | dok je aktivan | ništa |
| Bivši član | **24 meseca** bez aktivnosti (`last_activity_at`) i bez aktivne članarine ili kredita | automatska anonimizacija, pg_cron jednom nedeljno |
| Član traži brisanje | odmah | vlasnik pokreće anonimizaciju ručno |
| Lead koji nije postao član | **12 meseci** od poslednje aktivnosti | ime, telefon, email i beleške se brišu; izvor i status ostaju za statistiku |
| SMS poruke | **12 meseci** | telefon i tekst se brišu; status i cena ostaju za kvotu i troškove |
| Finansijski zapisi (`payments`, `memberships`, `pt_*`, `pos_*`, `shifts`, `expenses`) | trajno | ne brišu se; nemaju lične podatke osim veze na člana |
| Dolasci (`check_ins`) | trajno | ostaju vezani za anonimizovanog člana |
| Bivši zaposleni, recepcioneri i treneri (`active = false` u `gym_users`, `receptionists`, `trainers`) | ime trajno (potrebno za izveštaje), telefon **24 meseca** | telefon → NULL |
| Teretana koja prekine saradnju | **90 dana** od suspenzije | ponuđen izvoz svih podataka, zatim trajno brisanje tenanta |

### 12.3 Napomena

Rokovi su postavljeni po principima GDPR-a, na kojima se zasniva i crnogorski zakon o zaštiti podataka o ličnosti. Pre prvog kupca van pilota potvrditi sa pravnikom; rokovi su u jednom mestu i lako se menjaju.

---

## 13. Agenti

Definicije su u `.claude/agents/`. Princip: **kod piše glavna sesija** (ima kontekst plana i odluka); agenti proveravaju, testiraju i rade zaokružene jednokratne poslove. Nijedan agent ne menja kod aplikacije ni migracije.

### 13.1 Spisak

| Agent | Grupa | Model | Sme da piše | Zadatak |
|-------|-------|-------|-------------|---------|
| `migration-guard` | čuvar | Sonnet | ništa | Pregled migracija: `gym_id`, RLS u istoj migraciji, `_cents`, enumi, `client_op_id`, nepromenjene stare migracije |
| `rules-reviewer` | čuvar | Sonnet | ništa | Izmene naspram `CLAUDE.md`: van obima, izmišljene funkcije, `service_role`, `localStorage`, hardkodovan tekst, vremenska zona |
| `finance-guard` | čuvar | **Opus** | `tests/finance/` | Uplate, storno, smene, popusti i kodovi, zaokruživanje, `revenue_cents`, zalihe |
| `reception-ux-reviewer` | čuvar | Sonnet | ništa | Tastatura, fokus skenera, semafor, broj klikova, vlasnik bez unosa |
| `privacy-guard` | čuvar | Sonnet | ništa | Anonimizacija, rokovi iz poglavlja 12, SMS saglasnost, curenje ličnih podataka |
| `access-tester` | tester | **Opus** | `tests/access/` | Izolacija tenanta za svaku tabelu u bazi + matrica uloga iz poglavlja 5 |
| `offline-chaos-tester` | tester | **Opus** | `tests/offline/` | Prekidi mreže, dvostruko slanje, više uređaja, blokada zatvaranja smene |
| `perf-verifier` | tester | Sonnet | `tests/perf/` | Check-in < 300 ms, pretraga < 200 ms, izveštaji, `EXPLAIN ANALYZE` |
| `reports-verifier` | tester | **Opus** | `tests/reports/` | Izveštaji naspram nezavisno izračunatih vrednosti; zone, DST, naplaćeno vs realizovano |
| `seed-builder` | izvršilac | Sonnet | `supabase/seed.sql`, `supabase/seed/` | Test podaci: više teretana, dve godine istorije, ivični slučajevi, `expected.json` |
| `member-importer` | izvršilac | Sonnet | `scripts/import/` | Čišćenje i uvoz tabele pilot teretane; uvoz tek posle potvrde korisnika |
| `phase-verifier` | kapija | **Opus** | ništa | Build, lint, typecheck, testovi, kriterijum „Gotovo je kada" → PROŠLO / PALO |

**Effort:** svi agenti rade sa `effort: max`.

**Zašto Opus:** greška ovih agenata je najskuplja i najteža za primetiti — procureli podaci druge teretane, izgubljena ili duplirana uplata, pogrešna brojka na osnovu koje vlasnik odlučuje, i lažno „gotovo". Ostali rade po jasnom spisku provera i Sonnet je dovoljan.

### 13.2 Tok rada u fazi

```
glavna sesija: migracija
  → migration-guard → access-tester
glavna sesija: tipovi → upiti → UI
  → finance-guard / reception-ux-reviewer / privacy-guard (šta faza dira)
pre commita
  → rules-reviewer
kraj faze
  → testeri relevantni za fazu → phase-verifier
```

Presuda PALO bilo kog agenta blokira sledeći korak dok se ne popravi i ponovo ne proveri.

### 13.3 Agenti po fazi (pored `migration-guard`, `rules-reviewer`, `access-tester` i `phase-verifier`, koji idu u svakoj fazi)

| Faza | Dodatni agenti |
|------|----------------|
| 0 — Temelji | `seed-builder` |
| 1 — Članovi i kartice | `member-importer`, `perf-verifier`, `privacy-guard` |
| 2 — Paketi i članarine | `finance-guard` |
| 3 — Check-in | `reception-ux-reviewer`, `perf-verifier` |
| 4 — Uplate i smene | `finance-guard`, `reception-ux-reviewer`, `reports-verifier` |
| 5 — Offline | `offline-chaos-tester` |
| 6 — Personalni treninzi | `finance-guard`, `reception-ux-reviewer`, `reports-verifier` |
| 7 — POS i zalihe | `finance-guard`, `reception-ux-reviewer`, `offline-chaos-tester` |
| 8 — Ormarići i oprema | `reception-ux-reviewer` |
| 9 — Leadovi | `reports-verifier` |
| 10 — SMS | `privacy-guard` |
| 11 — Izveštaji | `reports-verifier`, `perf-verifier`, `reception-ux-reviewer` |
| 12 — Multi-lokacija i super-admin | `privacy-guard` |
| 13 — Grupni treninzi | `reception-ux-reviewer` |

`seed-builder` se ponovo pokreće u svakoj fazi koja dodaje tabele.
