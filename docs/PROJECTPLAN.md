# Projectplan: Board Game Tournament Manager

Team van 2 personen · 4 weken · Java 21 (IntelliJ) + JavaFX + Visual Paradigm  
Base package: `be.uantwerpen.sd.project` · Startklasse: `ViewApp`

---

## 1. Scope en eisen

### Functionele eisen
1. **Players beheren (`Player`)**
   - Velden: `id`, `name`, `skillLevel` (rating), `description`, `age`
   - Acties: toevoegen, bekijken, bewerken, verwijderen (globaal beheer + toevoegen aan toernooi)
2. **Game Types beheren (`GameType`)**
   - Velden: `id`, `name`, `minPlayers`, `maxPlayers` (per match), `scoringRule` (punten voor winst / gelijkspel / verlies), `description`
   - Minstens **2 voorgedefinieerde game types** bij opstart (bv. *Chess* 2–2 spelers, *Monopoly* 2–4 spelers)
   - Acties: selecteren, aanmaken, bekijken, bewerken, verwijderen
3. **Tournaments beheren (`Tournament`)**
   - Velden: `name`, `gameType`, `numberOfRounds`, `players`, `date`, `description`, `pairingStrategy`
   - Acties: aanmaken, bewerken, verwijderen, bekijken, spelers toevoegen/verwijderen, basisinfo tonen
4. **Rondes, Matches & Standings**
   - Pairing-strategy kiezen (`RoundRobinPairing`, `RandomPairing`; optioneel `SwissPairing`)
   - Pairings per ronde genereren (rekening houdend met `minPlayers`/`maxPlayers` of bye)
   - Matchlijst en resultaten per ronde bekijken
   - Resultaat per match invoeren (winnaar of gelijkspel)
   - Automatische puntenberekening per speler via Observer
   - Tussenstand per ronde + eindranking

### Niet-functionele eisen
- **5 Design Patterns:**
  1. **MVC** (`model`, `view`, `controller`)
  2. **Singleton, thread-safe** (`TournamentRepository` met holder-idiom)
  3. **Observer** (buiten MVC: `Match` → `MatchObserver` → `StandingsCalculator`)
  4. **Builder** (`Tournament.Builder`)
  5. **Strategy** (`PairingStrategy` en `ScoringStrategy`)
- **UML-diagrammen in `docs/`:**
  1. **Use Case Diagram** (alle actoren + hoofd-use-cases)
  2. **Volledig Klassendiagram** (GUI mag geabstraheerd worden)
  3. **5 Mini-klassendiagrammen** (1 per pattern, enkel de klassen/relaties van dat pattern)
  4. **Sequence Diagram** (min. 1 kern-use-case, bv. resultaat invoeren → Observer → standings update)
- **Tests:**
  - Unit tests per module, met **volledige coverage op minstens 1 belangrijke klasse** (controleren via JaCoCo)
  - Thread-safety test voor Singleton (50 concurrent tasks)
  - **Minstens 1 integratietest** (`*IT.java` in `src/test/java/...`) die controller + model + repository doorloopt
- **Documentatie & AI-regels:**
  - `CREDITS.md` voor externe bronnen
  - `LLM_QUESTIONS.md` als logboek (*waarom, wanneer, waar*). Gebruik AI enkel voor pattern-uitleg en debugging; schrijf en begrijp alle code zelf.

---

## 2. Repo-setup & Packagestructuur

De multi-module Maven-repo staat klaar (`pom.xml` + `project/pom.xml` met Java 21, JUnit 5, Mockito, JaCoCo en JavaFX 21.0.3).

- **Tests draaien:** `mvn test`
- **App starten:** `mvn -pl project javafx:run` (of via `ViewApp` in IntelliJ)
- **Integratietests & coverage:** naamgeving `*IT.java` wordt door Surefire opgepikt; JaCoCo is geconfigureerd in de root `pom.xml`.

```text
project/src/main/java/be/uantwerpen/sd/project/
├── model/        Player, GameType, ScoringRule, Tournament (+ Builder), Round, Match, Standing
├── pairing/      PairingStrategy, RoundRobinPairing, RandomPairing
├── scoring/      ScoringStrategy, StandardScoring
├── standings/    MatchObserver, StandingsCalculator
├── repository/   TournamentRepository (Singleton; beheert tournaments, players & gameTypes)
├── controller/   MainController, PlayerController, GameTypeController, TournamentController, RoundController
├── view/         MainView, PlayersView, GameTypesView, TournamentView, RoundView, StandingsView
└── ViewApp.java  start van de JavaFX-applicatie
```

---

## 3. Rolverdeling

| Onderdeel | Persoon A (Structuur, Data & CRUD) | Persoon B (Spelverloop, Scoring & Logica) |
| --- | --- | --- |
| **Domein & Data** | `Player`, `GameType`, `ScoringRule`, `Tournament` (+ `Builder`), `TournamentRepository` (Singleton + 2 predefined `GameType`s) | `Match`, `Round`, `Standing`, `PairingStrategy` (`RoundRobin`, `Random`), `ScoringStrategy`, `MatchObserver`, `StandingsCalculator` |
| **GUI (MVC)** | `MainView`/`MainController`, `PlayersView`/`Controller`, `GameTypesView`/`Controller`, `TournamentView`/`Controller` | `RoundView`/`RoundController`, `StandingsView` |
| **Patterns** | MVC, Singleton, Builder | Strategy, Observer |
| **UML & Docs** | Use Case Diagram, Volledig Klassendiagram, mini-diagrammen (MVC/Singleton/Builder), `CREDITS.md` | Sequence Diagram (Observer), mini-diagrammen (Strategy/Observer), `LLM_QUESTIONS.md`, pattern-uitleg |
| **Tests** | Singleton multi-thread test, `Tournament.Builder` & CRUD/validatie tests (100% coverage op `Tournament` of `GameType`) | `RoundRobinPairingTest`, `RandomPairingTest`, `ScoringStrategyTest`, `StandingsCalculatorTest`, integratietest `*IT.java` |

---

## 4. Gedeelde interfaces (in Week 1 samen vastleggen)

Leg deze signatures vast vóór iemand begint te coderen, zodat A en B zonder conflicten parallel kunnen werken:

```java
// pairing/PairingStrategy.java
public interface PairingStrategy {
    List<Match> generatePairings(List<Player> players, List<Round> previousRounds, GameType gameType);
    String getName();
}

// model/Match.java (werkt voor 2 spelers én multiplayer game types + bye)
public class Match {
    private final List<Player> players;      // 1 speler = bye; 2..maxPlayers spelers = echte match
    private Player winner;                   // null bij nog niet gespeeld of gelijkspel
    private boolean draw;
    private boolean played;
    private final List<MatchObserver> observers = new ArrayList<>();

    public void addObserver(MatchObserver observer) { observers.add(observer); }

    public void setWinner(Player winner) {
        this.winner = winner;
        this.draw = false;
        this.played = true;
        observers.forEach(o -> o.onResultEntered(this));
    }

    public void setDraw() {
        this.winner = null;
        this.draw = true;
        this.played = true;
        observers.forEach(o -> o.onResultEntered(this));
    }
}

// scoring/ScoringStrategy.java
public interface ScoringStrategy {
    int pointsFor(Match match, Player player, GameType gameType);
}
```

---

## 5. Week-voor-week tijdlijn (Samen → Eigen weg → Integratie)

### Week 1: Afstemming, Skeleton & Basismodel

**1. Eerst samen (in de les — ± 2 uur)**
- Repo clonen in IntelliJ (openen via root `pom.xml`, SDK 21 verifiëren).
- Controleren dat `mvn test` en `mvn -pl project javafx:run` bij allebei werken op een leeg JavaFX-venster in `ViewApp`.
- Setup-PR nakijken en mergen naar `main` (`.gitignore`, `docs/`, `CREDITS.md`, `LLM_QUESTIONS.md`).
- **Samen ontwerpen in Visual Paradigm / op papier:**
  - Use Case Diagram v1 (alle functionaliteiten uit sectie 1)
  - Klassendiagram v1 + vaste methodesignatures (`Tournament`, `Player`, `GameType`, `Match`, `Round`, `PairingStrategy`, `ScoringStrategy`, `MatchObserver`).
- Afspreken hoe `bye`-punten tellen (bv. een bye telt als winst voor die speler).

**2. Daarna ieder zijn eigen weg (thuis, op eigen feature branch)**
- **Persoon A (`feature/core-models-singleton`):**
  - `Player` (`name`, `skillLevel`, `description`, `age` + `equals`/`hashCode` op `id`)
  - `GameType` & `ScoringRule` (`name`, `minPlayers`, `maxPlayers`, `scoringRule`, `description`)
  - `Tournament` met `Tournament.Builder` + stubs voor `nextRound()`, `getRounds()`, `getStandings()`, `getStandingsAfterRound(int)`
  - `TournamentRepository` (thread-safe Singleton, met opslag voor tournaments, players, gameTypes + 2 voorgedefinieerde `GameType`s bij initialisatie)
  - Leeg hoofdvenster (`MainView` + `MainController`) gekoppeld aan `ViewApp`
  - Unit tests: `Tournament.Builder` + multi-threaded Singleton test (50 threads)
- **Persoon B (`feature/pairing-and-match`):**
  - `Match`, `Round`, `Standing`
  - `MatchObserver` interface
  - `PairingStrategy` interface + `RoundRobinPairing` (circle method, ondersteunt oneven aantal spelers met bye en respecteert `GameType` groepsgrootte)
  - Unit tests: `RoundRobinPairingTest` (4 spelers = 3 rondes × 2 matches, elke speler speelt 1× tegen elk ander; 5 spelers = telkens 1 bye)

**3. Einde week 1 samen**
- Elkaars PR reviewen en mergen naar `main`.
- Check: `mvn test` groen + JavaFX-venster opent bij allebei.

---

### Week 2: CRUD Schermen & Tweede Strategy + Rondes

**1. Eerst samen (in de les)**
- Verse `main` pullen en controleren of A's modellen en B's `Match`/`Round`/`PairingStrategy` netjes samenwerken.
- Navigatiestructuur van `MainView` vastleggen (tabbladen of zijmenu: *Players*, *Game Types*, *Tournaments*, *Rounds & Standings*).

**2. Daarna ieder zijn eigen weg (thuis, op eigen feature branch)**
- **Persoon A (`feature/crud-views`):**
  - `PlayersView` & `PlayerController`: spelers aanmaken, bekijken, **bewerken** en verwijderen + validatie (`Alert` bij lege naam/ongeldige leeftijd)
  - `GameTypesView` & `GameTypeController`: game types bekijken (incl. 2 predefined), aanmaken, **bewerken** en verwijderen (`min <= max`)
  - `TournamentView` & `TournamentController`: toernooi aanmaken (via `Builder`), bekijken, bewerken, verwijderen, game type kiezen, pairing strategy kiezen, geregistreerde spelers toevoegen/verwijderen
  - Unit tests voor `Player` en `GameType` validatie/CRUD
- **Persoon B (`feature/random-pairing-and-round-ui`):**
  - `RandomPairing` (shuffelt spelers, groepeert op basis van `GameType`)
  - `Tournament.nextRound()` invullen in overleg (genereert matches via gekozen `PairingStrategy`, controleert `rounds.size() < numberOfRounds`)
  - `RoundView` & `RoundController`: knop *Genereer ronde*, matchlijst per ronde tonen, resultaat per match invoeren (winnaar / gelijkspel) + resultaten per ronde bekijken
  - Unit test: `RandomPairingTest` (geen dubbele spelers in een ronde)

**3. Einde week 2 samen**
- Elkaars PR reviewen en mergen naar `main`.
- Check: in de GUI kan je spelers en game types beheren, een toernooi aanmaken, spelers inschrijven en rondes met matches genereren en resultaten bekijken.

---

### Week 3: Scoring, Observer, Standings & Integratie

**1. Eerst samen (in de les)**
- Volledige flow testen op `main`: speler aanmaken → game type kiezen → toernooi starten → ronde 1 genereren → resultaat invoeren.
- Eventuele interfaces tussen `TournamentView`, `RoundView` en `StandingsView` bijschaven.

**2. Daarna ieder zijn eigen weg (thuis, op eigen feature branch)**
- **Persoon A (`feature/gui-integration-and-validation`):**
  - Navigatie & UX stroomlijnen tussen `TournamentView`, `RoundView` en `StandingsView`
  - Foutafhandeling & randgevallen in de GUI (bv. te weinig spelers voor gekozen `GameType.minPlayers`, proberen meer rondes te genereren dan `numberOfRounds`, duidelijke `Alert`-meldingen)
  - Volledige unit test coverage (100%) realiseren op één belangrijke domeinklasse (bv. `Tournament` of `GameType`, controleren met JaCoCo)
- **Persoon B (`feature/scoring-observer-standings`):**
  - `ScoringStrategy` + concrete implementatie (punten uit `GameType.getScoringRule()`, bv. winst 3, gelijkspel 1, verlies 0)
  - `StandingsCalculator implements MatchObserver`: registreert zich in `Tournament.nextRound()` als observer op elke `Match` en herberekent punten automatisch bij `setWinner()` / `setDraw()`
  - `getStandings()` (huidige eindranking) en `getStandingsAfterRound(int roundNumber)` (tussenstand per ronde), gesorteerd op punten aflopend en daarna op naam
  - `StandingsView`: tabel met keuzelijst voor tussenstand per ronde + algemene eindranking
  - Tests: `ScoringStrategyTest`, `StandingsCalculatorTest` (2 rondes met bekende uitslagen) + **1 integratietest** (`TournamentFlowIT.java`: controller + model + repository van aanmaak tot eindranking)

**3. Einde week 3 samen (Code Freeze)**
- Volledig eindscenario doorlopen; bugs noteren als GitHub issues en oplossen.
- **Code freeze** voor kernfeatures. Enkel als alles 100% stabiel is: optionele extra (bv. `SwissPairing` of undo via Command).

---

### Week 4: UML, Documentatie, Code Cleanup & Proefpresentatie

**1. Eerst samen (in de les)**
- Proefpresentatie houden: A legt B's patterns/code uit en B legt A's patterns/code uit.
- Eisencheck uit sectie 6 doorlopen op `main`.

**2. Daarna ieder zijn eigen weg (thuis)**
- **Persoon A:**
  - Visual Paradigm: volledig **Klassendiagram** (via *Tools → Code → Instant Reverse*, daarna opschonen), definitief **Use Case Diagram**, en **3 mini-klassendiagrammen** (MVC, Singleton, Builder) → `.vpp` + PNG in `docs/`
  - Documentatie & `CREDITS.md` afwerken, handleiding schrijven (hoe starten & gebruiken)
  - Code cleanup (`System.out.println` en dode code weg)
- **Persoon B:**
  - Visual Paradigm: **Sequence Diagram** voor het invoeren van een resultaat (`RoundView` → `RoundController` → `Match` → `StandingsCalculator` → `Standing`), en **2 mini-klassendiagrammen** (Strategy, Observer) → PNG in `docs/`
  - Pattern-verantwoording uitschrijven (probleem, oplossing, waar in code, waarom logisch)
  - `LLM_QUESTIONS.md` volledig nalopen en JaCoCo/tests finaal draaien

**3. Minstens 2 dagen vóór de deadline samen**
- Laatste `mvn test` / `mvn verify` check en app-walkthrough van begin tot eind.
- Controleren dat alle diagrammen in `docs/` exact overeenkomen met de finale code.
- Inleveren.

---

## 6. Eisencheck (vóór inlevering)

- [ ] **Players:** aanmaken, bekijken, bewerken, verwijderen (met naam, skill/rating, beschrijving, leeftijd)
- [ ] **Game Types:** min. 2 predefined + selecteren, aanmaken, bekijken, bewerken, verwijderen (met min/max spelers, scoring rule, beschrijving)
- [ ] **Tournaments:** aanmaken (via `Builder`), bewerken, verwijderen, bekijken, spelers koppelen
- [ ] **Rondes & Standings:** min. 2 `PairingStrategy`s, matchlijst per ronde, resultaten invoeren & bekijken, automatische punten via Observer, tussenstand per ronde + eindranking
- [ ] **5 Patterns:** MVC, Singleton (thread-safe), Observer (buiten MVC), Builder, Strategy
- [ ] **UML in `docs/`:** 1 Use Case Diagram, 1 volledig Klassendiagram, 5 mini-klassendiagrammen, 1 Sequence Diagram
- [ ] **Tests:** `mvn test` groen, Singleton multi-thread test, volledige coverage op ≥ 1 belangrijke klasse, ≥ 1 integratietest (`*IT.java`)
- [ ] **Git & Logs:** feature branches + PR's gebruikt, `CREDITS.md` en `LLM_QUESTIONS.md` compleet
