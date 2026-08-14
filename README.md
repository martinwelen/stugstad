# 🏡 Stug-städlistan

En mobilanpassad avbockningslista för städuppgifter på stugan, där varje uppgift
har ett pengavärde. Barnen (Samuel & Benjamin) öppnar en URL i mobilen, bockar av
uppgifter och väljer vem som gjort vad. Föräldern kan lägga till/ta bort uppgifter
och nollställa avbockningar via ett PIN-skyddat admin-läge.

**Live:** https://martinwelen.github.io/stugstad/
**QR-kod:** `qr-stugstad.png` (skanna för att öppna appen)

> Driftsuppgifter (admin-PIN, Azure-detaljer, SAS, hur man nollställer/roterar)
> finns i den lokala filen **`DRIFT.md`** — den är avsiktligt *inte* incheckad,
> eftersom detta repo är publikt.

---

## Funktioner

**Barnvy**
- Lista med alla uppgifter och värde i kr.
- Bocka av → välj **Samuel** eller **Benjamin** → 🎉 konfetti.
- Löpande summa per barn högst upp.
- En uppgift kan bockas av en gång — **utom "Packa egen väska"** som båda kan
  bocka av var för sig (15 kr styck).
- Avbockade uppgifter visas överstrukna men ligger kvar, med **Ångra**-knapp.
- **Delad state** mellan alla mobiler (se Arkitektur).

**Admin-läge (kugghjul ⚙️ längst ner, 4-siffrig PIN)**
- Lägg till ny uppgift (namn + värde).
- Ta bort uppgift.
- Nollställ en avbockning.
- Nya/ändrade uppgifter dyker upp hos barnen inom några sekunder.

PIN:en ligger **inte** i klartext i källkoden — endast en SHA-256-hash lagras och
jämförs i webbläsaren.

---

## Arkitektur

Allt ligger i **en enda statisk `index.html`** (HTML + CSS + vanilla JS, inget
byggsteg, inga externa beroenden — konfettin är egen canvas-kod).

```
  Mobil (barn)  ─┐
  Mobil (barn)  ─┼──►  index.html på GitHub Pages  ──►  Azure Blob: state.json
  Mobil (förälder)┘         (statisk hosting)            (delad state via SAS)
```

- **Hosting:** GitHub Pages (branch `main`, rot `/`). Push → automatisk deploy.
- **Delad state:** en `state.json`-blob i **Azure Blob Storage**. Appen läser
  (GET) var 5:e sekund när fliken är synlig och nyss använd, och skriver (PUT)
  vid varje ändring. Åtkomst sker via en begränsad **SAS-token** (läs/skriv/skapa
  på just den containern) — barnen behöver aldrig logga in.
- **Konfliktlösning:** last-write-wins. Före varje skrivning läses färskaste
  state, ändringen appliceras och skrivs tillbaka.
- **Offline-skydd:** senaste kända listan cachas i `localStorage`, så appen visar
  alltid något även om nätet tillfälligt saknas.

### Datamodell (`state.json`)
```json
{
  "v": 2,
  "tasks": [
    { "id": "lok", "namn": "Skörda ...", "varde": 40 },
    { "id": "vaska", "namn": "Packa egen väska", "varde": 15, "multi": true }
  ],
  "done":      { "lok": "Samuel" },
  "multiDone": { "vaska": ["Samuel", "Benjamin"] }
}
```
- `done` — vanliga uppgifter: `{ uppgifts-id: barnets namn }`.
- `multiDone` — flervalsuppgifter (`multi:true`): `{ uppgifts-id: [barn, ...] }`.

---

## Vardagsanvändning

- **Lägga till/ta bort uppgifter, nollställa fusk:** gör i appen via ⚙️ → PIN.
  Inget behov av att röra koden.
- **Ändra i utseende/logik:** redigera `index.html`, `git commit` + `git push`.
  GitHub Pages bygger om automatiskt (~1 min).

Se `DRIFT.md` för hur man byter PIN, nollställer hela listan inför en ny helg,
och roterar SAS-token.

---

## Teknik i korthet
- Ren statisk sida, inga ramverk, inget byggsteg.
- Egen konfetti i `<canvas>`.
- SHA-256 (Web Crypto) för PIN-jämförelse.
- Azure Blob Storage som enkel delad key-value-lagring via SAS + CORS.
