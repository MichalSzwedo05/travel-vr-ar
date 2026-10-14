# Plan realizacji projektu — travel-vr-ar

Aplikacja interaktywna dla biura podróży (VR + AR + Web) umożliwiająca klientowi
fotorealistyczny wybór wakacji. Dokument źródłowy: specyfikacja projektowa
(„projekt wdrozeniowy”).

---

## 1. Cel i zakres

**Cel:** stanowisko w biurze podróży (+ przeglądarka Chrome), na którym klient
przechodzi przez: ankietę AI → płynny wirtualny spacer po propozycjach →
eksplorację szczegółowych map → drukowaną/cyfrową ofertę.

**Zakres (MVP):**
- ankieta AI z zapisem odpowiedzi w czasie rzeczywistym,
- ciągła scena spaceru wirtualnego (fotorealizm),
- tryb eksploracji map i lokacji,
- generator podsumowania oferty (daty lotów, hotele, atrakcje, plan wycieczki),
- build natywny PC + port WebGL (Chrome),
- moduł AR (rozszerzony po MVP — patrz Ryzyka).

**Poza zakresem MVP:** system rezerwacji/płatności, panel administracyjny,
wersje mobilne (iOS/Android), wielojęzyczność (faza 4).

---

## 2. Architektura (wysoki poziom)

```
┌─────────────────────────────── Unity (PC / WebGL) ───────────────────────────────┐
│  UI Ankiety │ Scena spaceru │ Eksploracja map │ Widok podsumowania │ Wydruk PDF │
└───────────────┬──────────────────────────────────────────────────────────────────┘
                │ HTTPS (REST / WebSocket)
┌───────────────▼──────────────────┐   ┌─────────────────────────────────────────┐
│ Backend (API)                    │   │ Zewnętrzne usługi                      │
│  - sesje ankiet + archiwizacja   │   │  - LLM (generowanie pytań/analiza)     │
│  - katalog destynacji/hoteli     │◄──┤  - API lotów/hoteli (lub baza danych)  │
│  - silnik rekomendacji           │   │  - mapy (tiles / pakiet offline)       │
│  - generator oferty (PDF)        │   └─────────────────────────────────────────┘
└───────────────┬──────────────────┘
                │
        ┌───────▼────────┐
        │ Baza danych    │  odpowiedzi, preferencje, sesje, katalog ofert
        └────────────────┘
```

**Uzasadnienie decyzji:**
- **Unity** — wymaganie silnika; eksport natywny (Windows x64) + WebGL.
- **Backend oddzielny od klienta** — AI i archiwizacja nie mogą zależeć od
  buildu WebGL (limity, sekrety API nie mogą trafić do paczki przeglądarkowej).
- **WebGL zoptymalizowany pod Chrome** — rozmiar tekstur LOD, Draco/KTX2,
  `WEBGL_threads`, brak ciężkich shaderów RT.

---

## 3. Technologie

| Warstwa | Wybór | Uwagi |
|---|---|---|
| Silnik | Unity 2022.3 LTS lub Unity 6 LTS | stabilny eksport WebGL |
| Rendering | URP + HDRP-sample (fotorealizm) | hybrid: URP na WebGL, quality-gate PC |
| VR | OpenXR (PC VR) | headset wg decyzji biura (patrz pytania otwarte) |
| AR | AR Foundation / Vuforia | faza 4, poza MVP |
| Backend | Node.js (NestJS) **lub** FastAPI | jeden język zespołu — do ustalenia |
| Baza danych | PostgreSQL | sesje, odpowiedzi, katalog |
| AI | API LLM (np. OpenAI/Anthropic) + warstwa rekomendacji | fallback: reguły, gdy API niedostępne |
| Wydruk | PDF (server-side) | szablony oferty |
| CI/CD | GitHub Actions | build PC + WebGL na tag |
| Hosting WebGL | GitHub Pages / nginx biura | demo dla klienta |

---

## 4. Etapy i kamienie milowe

### Faza 0 — Fundament (tydzień 1)
- [ ] Repozytorium, struktura projektu Unity (`Assets/`, `Packages/`, `ProjectSettings/`),
      `.gitignore`, włączenie LFS dla `Assets/**/*.psd|tga|fbx` (fotorealizm = duże pliki).
- [ ] Wybór renderingu: URP z quality-gate (profil „PC High” / „WebGL Medium”).
- [ ] Pusty build testowy: PC + WebGL — pipeline dostarczania zweryfikowany.
- [ ] Szkielet backendu (auth sesji, healthcheck) + środowisko dev (Docker Compose).
- [ ] Szablon README/CODING standards, branching (`main` + `feat/*` + PR).

**Kryterium wyjścia:** „Hello world” działa w PC i w Chrome, CI buduje oba artefakty.

### Faza 1 — Inteligentna ankieta AI (tygodnie 2–4)
- [ ] Projekt danych: `Session`, `Answer`, `Preference` (archiwizacja od 1. pytania).
- [ ] Endpointy API: `POST /sessions`, `PATCH /sessions/{id}/answers`, `GET /summary`.
- [ ] Warstwa LLM: generowanie/pytania + klasyfikacja preferencji (destynacja,
      budżet, sezon, styl podróżowania, komfort).
- [ ] UI ankiety w Unity (wizard, progress bar, tryb offline-safe: kolejka zapisu).
- [ ] Idempotentna synchronizacja odpowiedzi (ponowne wysłanie, retry).
- [ ] Testy: jednostkowe (backend), playmode (UI ankiety), kontraktowe API.

**Kryterium wyjścia:** pełna ankieta przechodzi, dane lądują w DB po każdym kroku.

### Faza 2 — Ciągły spacer wirtualny (tygodnie 5–9)
- [ ] Silnik sceny: jeden „streamowany” level (additive scenes / Addressables),
      płynne przejścia bez ekranu ładowania między lokalizacjami.
- [ ] Import i optymalizacja assetów (LOD, lightmaps, refl-probes) — realistyczne
      oświetlenie, post-processing (bloom, tonemapping, SSAO na PC).
- [ ] Kamera/sterowanie: tryb biurowy (gamepad/klawiatura + teleport),
      płynny ruch bez motion sickness, opcja VR (OpenXR).
- [ ] Integracja z ankieta: kolejność lokacji wg rekomendacji AI.
- [ ] Telemetria: czas w lokacji, checkpointy (do personalizacji oferty).
- [ ] Performance budget: 60 FPS PC / 30–60 FPS WebGL (profilowanie co sprint).

**Kryterium wyjścia:** przejście całej trasy bez ładowania, quality gate FPS osiągnięty.

### Faza 3 — Eksploracja szczegółowa (tydzień 10–11)
- [ ] Menu „Szczegółowe mapy”: siatka lokacji z miniaturami, filtry wg preferencji.
- [ ] Tryb swobodnego oglądania (orbit / walkthrough) dla wybranej lokacji.
- [ ] Warstwa informacyjna (hot-spoty: opis, zdjęcia, cena orientacyjna).
- [ ] Nawigacja: powrót do spaceru głównego i z powrotem (stan zachowany).

### Faza 4 — Podsumowanie i wydruk oferty (tydzień 12–13)
- [ ] Agregacja danych: daty lotów, hotele, atrakcje, draft planu dnia po dniu.
- [ ] Generator PDF (szablon brandowany biura, wersja PL/EN).
- [ ] Eksport cyfrowy (HTML/share) + wydruk bezpośrednio ze stanowiska.
- [ ] Testy akceptacyjne z klientem (UAT na danych przykładowych).

### Faza 5 — AR i wdrożenie (tydzień 14–16)
- [ ] **AR (minimalny zakres MVP):** np. skan wizytówki/kodu → overlay oferty
      na telefonie lub podgląd „jak to będzie wyglądać” — do doprecyzowania.
- [ ] Twarda optymalizacja WebGL pod Chrome (rozmiar paczki < 50 MB gzip,
      initial load < 15 s na sprzęcie biurowym).
- [ ] Pakiet wdrożeniowy: instalator PC, wersja web, instrukcja obsługi stanowiska.
- [ ] Monitoring: logi backendu, crash reports (Unity Lighthouse/Sentry),
      analityka użycia ankiety.
- [ ] Alpha → Beta → Go-live (pilot na 1 stanowisku).

---

## 5. Struktura repozytorium

```
travel-vr-ar/
├── README.md
├── plan.md                  ← ten plik
├── .gitignore
├── client/                  ← projekt Unity (tutaj przeniesiony z Assets itd.)
│   ├── Assets/
│   ├── Packages/
│   └── ProjectSettings/
├── server/                  ← backend + API
│   └── ...
├── docs/                    ← specyfikacja, decyzje architektoniczne (ADR)
└── .github/workflows/       ← CI: build PC, build WebGL, testy
```

---

## 6. Ryzyka i mitygacja

| Ryzyko | Wpływ | Mitygacja |
|---|---|---|
| Fotorealizm × WebGL (wydajność) | wysoki | quality-profile, LOD/KTX2, budżet FPS od Fazy 2, wcześniejszy prototyp WebGL |
| Koszt/limit API LLM w ruchu biurowym | średni | cache rekomendacji, fallback regułowy, retry/circuit breaker |
| Duże assety w gicie | średni | Git LFS + polityka rozmiarów, alternatywnie Artifactory |
| Scope creep (AR, płatności, mobilne) | średni | twarde granice MVP (sekcja 1), kolejne fazy po UAT |
| Brak decyzji dot. headsetu VR | średni | kontroler FreeFly mode działa bez HMD; OpenXR po decyzji |

---

## 7. Pytania otwarte (do clienta / zespołu)

1. **VR:** czy stanowisko ma obsługiwać headset (Quest / PCVR), czy tylko ekran 2D?
2. **AR:** jaki konkretnie scenariusz AR jest oczekiwany w MVP?
3. **Dane:** katalog ofert (loty/hotele) — z API zewnętrznej agregatora czy własna DB?
4. **AI:** preferowany dostawca LLM i budżet na tokeny?
5. **Marka:** materiały graficzne/logo do PDF oferty, termin pilotażu.
6. **Zespół:** ile osób, jaki podział (Unity / backend / QA)?

---

## 8. Definition of Done (projekt)

- [ ] klient przechodzi pełny journey bez pomocy obsługującego,
- [ ] odpowiedzi ankiet zapisywane i odtwarzalne po restarcie aplikacji,
- [ ] spacer bez zacięć ≥ 60 FPS na PC biurowym, WebGL akceptowalny w Chrome,
- [ ] oferta generuje się w PDF < 5 s i zawiera 4 wymagane sekcje,
- [ ] build PC + web budowane automatycznie w CI, dokumentacja wdrożenia gotowa.
