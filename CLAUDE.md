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

Én fil: `index.html` (~740 linjer). Ingen byggesteg, ingen avhengigheter utover
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

## Innlogging: fire innstillinger som henger sammen

Dagens løsning er **6-sifret engangskode** (OTP), ikke magic link. Disse fire må endres
samlet — endrer du én alene, slutter innlogging å virke:

| Hvor | Innstilling |
|---|---|
| `index.html`, `verifyCode()` | Krever nøyaktig 6 siffer (`/^\d{6}$/`) |
| Supabase → Sign In / Providers → Email | **Email OTP Length = 6** |
| Supabase → Emails → maler | **Magic Link** og **Confirm signup** bruker begge `{{ .Token }}` |
| Supabase → Emails → SMTP | Resend: `smtp.resend.com:465`, bruker `resend`, fra `noreply@mail.peakpoint.no` |

Begge malene må endres, ikke bare Magic Link: `signInWithOtp` står med
`shouldCreateUser: true`, så nye brukere kan få Confirm signup-malen.

Rate limit for e-post er hevet til 100/time (prosjektomfattende). De øvrige grensene er
per IP-adresse og trenger ikke endring for 11 personer.

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

## Hvis magic link skal vurderes (åpent spørsmål fra eieren)

Dette er kartlagt, men ikke utført. Verifiser hvert punkt selv før du bygger:

1. Begge maler tilbake til `{{ .ConfirmationURL }}`.
2. `detectSessionInUrl` står i dag på `false` i `createClient` — må bli `true`.
3. **Site URL og Redirect URLs i Supabase er ikke verifisert konfigurert.** Da eieren
   testet magic link 2026-07-18, endte klikk på lenken på en feilside. Det peker mot
   manglende redirect-oppsett, men er ikke bekreftet — sjekk før du konkluderer.
4. `verifyCode()` og hele kode-inntastingen blir overflødig; appen må i stedet ta imot en
   sesjon ved retur.
5. Kjent risiko å undersøke: e-postskannere hos Gmail/Outlook kan forhåndsåpne lenker og
   brenne engangslenken før mottakeren rekker å klikke. Gjelder ikke koder.

## Arbeidsflyt

- **Verifiser i ekte nettleser, ikke bare med tester.** `python -m http.server 8777` i
  repoet, åpne `http://127.0.0.1:8777`. Den lokale siden bruker den **ekte** databasen —
  innlogging der lager en ekte bruker som må ryddes bort etterpå.
- Etter kodeendring i `index.html`: sjekk at JavaScript-en parser før publisering
  (klipp ut `<script>`-blokken og kjør `node --check`).
- Etter publisering: bekreft mot den ekte URL-en, ikke mot lokal fil.
- Slett testbrukere når testingen er ferdig — undersøkelsen skal inneholde 11 ekte svar.
- Blir konteksten lang: skriv tilstand til fil og start fersk økt. Ikke stol på
  auto-compaction.

## Konvensjoner

Brukergrensesnitt og kodekommentarer på engelsk. Kommentarer forklarer *hvorfor*, ikke
*hva*. Ingen emoji i koden. Commit-meldinger på engelsk, imperativ overskrift.
