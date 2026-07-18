# Sarajevo trip survey

Spørreundersøkelse for 11 venner som skal til Sarajevo. Én statisk HTML-fil på GitHub
Pages, svar lagret i Supabase. Eieren er ikke utvikler — forklar enkelt før hvert
godkjenningsspørsmål, og vis alltid hva du faktisk har verifisert kontra antatt.

## 🔴 Må aldri røres

- **DNS-roten `peakpoint.no`** kjører eierens bedrifts-e-post (`post@peakpoint.no`) via
  Microsoft 365. Endres rot-MX eller rot-SPF, stopper jobb-e-posten hans. All e-post fra
  denne appen går på underdomenet `mail.peakpoint.no`, som er helt adskilt. Rører du DNS:
  verifiser rot-MX-en etterpå med et direkte oppslag.
- **Supabase service-nøkkelen skal aldri i frontend.** `SUPABASE_ANON_KEY` i `index.html`
  er offentlig med vilje — RLS beskytter dataene. Service-nøkkelen omgår RLS.
- **Ingen API-nøkler i repoet.** Resend-nøkkelen bor kun i Supabase sine SMTP-innstillinger.

## Arkitektur

Én fil: `index.html` (~850 linjer). Ingen byggesteg, ingen avhengigheter utover
Supabase-klienten fra CDN. **Deploy = `git push origin main`** → GitHub Pages plukker det
opp på 1–2 minutter.

Supabase-prosjekt `iqffgffgqgtwltdvvlfv` (`sarajevo-trip-survey`, region eu-north-1).
Tabell `sarajevo_survey_responses`: `user_id` (PK, FK → `auth.users`), `name`,
`answers` (jsonb), tidsstempler. RLS på: alle innloggede kan lese alle svar, men kun
endre sin egen rad.

Appen er én side som viser/skjuler seks views (`boot`, `auth`, `landing`, `form`, `done`,
`results`) via `show()`. `NAV_VIEWS` + `popstate` gjør at nettleserens tilbake-knapp går
ett steg innad i appen; `boot` og `auth` er bevisst utenfor, så Tilbake fra første skjerm
forlater siden normalt.

## Innlogging: fem ting som henger sammen

Dagens løsning er **lenke og 6-sifret kode i samme e-post**. Endrer du én av disse alene,
slutter innlogging å virke:

| Hvor | Innstilling |
|---|---|
| `index.html`, `createClient` | `detectSessionInUrl: true` |
| `index.html`, `signInWithOtp` | `emailRedirectTo: REDIRECT_TO` (= `location.origin+location.pathname`) |
| Supabase → URL Configuration | Site URL = GitHub Pages-URL-en. Redirect URLs må inneholde **både** den og `http://127.0.0.1:8777/` for lokal test |
| Supabase → Emails → maler | Begge peker på `{{ .RedirectTo }}?token_hash={{ .TokenHash }}&type=…` — `type=magiclink` i Magic Link, `type=signup` i Confirm signup |
| Supabase → Emails → SMTP | Resend: `smtp.resend.com:465`, bruker `resend`, fra `noreply@mail.peakpoint.no` |

Begge malene må endres, ikke bare Magic Link: `signInWithOtp` står med
`shouldCreateUser: true`, så nye brukere kan få Confirm signup-malen. Ulik `type` i de to
er ikke slurv — feil `type` gir avvist nøkkel.

**Lenke og kode er samme engangsnøkkel.** Supabase-dokumentasjonen: *"Email OTPs share an
implementation with Magic Links."* Brukes den ene, dør den andre. Levetid 1 time
(`Email OTP Expiration`, maks 24 t). Gjenbruk er ikke mulig — engangsbruk er hardkodet.
Det spiller liten rolle i praksis: `persistSession: true` gjør at én innlogging varer til
brukeren logger ut eller sletter nettleserdata.

Rate limit for e-post er hevet til 100/time (prosjektomfattende). De øvrige grensene er
per IP-adresse og trenger ikke endring for 11 personer. I tillegg finnes en sperre på ca.
60 s mellom hver e-post til *samme* adresse (`over_email_send_rate_limit`, HTTP 429) — en
utålmodig bruker som trykker flere ganger får den, og det er normalt.

## Åpen dør, med en planlagt lås

`shouldCreateUser: true` betyr at hvem som helst med lenken kan logge inn og svare — det
finnes ingen sperre mot andre enn de 11. Bevisst valgt, ikke en forglemmelse: eieren har
ikke alle e-postadressene på forhånd, og folk som taster feil adresse skal kunne rette det
selv uten å måtte mase på ham.

**Når alle 11 har logget inn, skal døra låses** — i Supabase, ikke i koden:
`Authentication → Sign In / Providers → Email → "Allow new users to sign up" = av`.
Eksisterende brukere kommer fortsatt inn, nye avvises. Kan skrus på igjen når som helst.
Lås ikke før alle er inne, ellers må eieren slippe folk inn manuelt.

Ikke løs dette med en liste over tillatte adresser i `index.html`. Sperren ville da ligget
hos den som skal stoppes, og du ville publisert 11 privatpersoners e-postadresser i et
offentlig repo.

## Knappen «Finish signing in» — ikke fjern den

Trykk på lenken viser en bekreftelsesknapp i stedet for å logge inn med en gang. Det ser
ut som et unødvendig ekstra trinn. Det er det ikke.

Microsofts e-postskanner åpner hver eneste lenke som sendes til Hotmail/Outlook-adresser.
Målt i prosjektets egen auth-logg 2026-07-18, med tre nivåer av mottiltak:

| Oppsett | Hva skanneren gjorde |
|---|---|
| Lenke rett til Supabase `/verify` | `GET` fra `104.47.*`, `135.232.*`, `72.153.*` — brant nøkkelen |
| Lenke til egen side, innløsning ved sidelast | **`POST /verify` fra `72.153.231.68` kl. 21:26:17** — skanneren kjører JavaScript |
| Lenke til egen side, innløsning ved knappetrykk | Ingen treff. E-post sendt 21:35:49, innlogging 21:37:33, ingen Microsoft-adresse i vinduet |

Skanneren kjører altså skript, men trykker ikke på knapper. Flytter du innløsningen
tilbake til sidelast, brenner Outlook lenkene til flere av de 11 igjen.

Skanneren traff historisk 25–60 s etter utsending. Tester du dette, **vent minst 2
minutter før du trykker** — ellers vinner du kappløpet og beviser ingenting.

**Nøkkelen ligger i query-strengen** (`?token_hash=…`), ikke bak `#`, så den sendes til
GitHub sine servere. Derfor står `<meta name="referrer" content="no-referrer">` i `<head>`
— uten den lekker adressen med nøkkel til CDN-en på linje 270. Engangsbruken er det som
gjør plasseringen forsvarlig: gjør du lenken gjenbrukbar, faller den begrunnelsen bort.

## Arbeidsflyt

- **Verifiser i ekte nettleser, ikke bare med tester.** `python -m http.server 8777` i
  repoet, åpne `http://127.0.0.1:8777`. Den lokale siden bruker den **ekte** databasen —
  innlogging der lager en ekte bruker som må ryddes bort etterpå.
- Etter kodeendring i `index.html`: sjekk at JavaScript-en parser før publisering
  (klipp ut `<script>`-blokken og kjør `node --check`).
- Etter publisering: bekreft mot den ekte URL-en, ikke mot lokal fil.
- **Feilsøk innlogging i auth-loggen, ikke ved å gjette.** Supabase MCP → `get_logs` med
  `service: "auth"` (siste 24 t). Svaret er for stort til å leses rått — parse `event_message`
  som JSON og se på `path`, `method`, `status`, `remote_addr`, `login_method`, `error_code`.
  `remote_addr` skiller mennesket fra skanneren, og `method` skiller lenke fra kode.
  Loggen har ligget noen sekunder etter sanntid; spør to ganger før du konkluderer med
  at noe *ikke* skjedde.
- Slett testbrukere når testingen er ferdig — undersøkelsen skal inneholde 11 ekte svar.
  Sjekk først: per 2026-07-18 finnes bare eierens egen konto, uten svar. Den skal stå.
- Blir konteksten lang: skriv tilstand til fil og start fersk økt. Ikke stol på
  auto-compaction.

## Konvensjoner

Brukergrensesnitt og kodekommentarer på engelsk. Kommentarer forklarer *hvorfor*, ikke
*hva*. Ingen emoji i koden. Commit-meldinger på engelsk, imperativ overskrift.
