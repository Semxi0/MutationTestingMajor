# Onderzoek naar mutation testing met Major

TL;DR

**Gebruik Major niet voor CNI-project.**


Kan Major mutanten genereren en testen binnen de Java-build van het CNI-project, en hoeveel moeite kost de installatie? Dit individuele onderzoek levert input voor de gezamenlijke ADR waarin Major wordt vergeleken met PIT, PIT + Descartes en LittleDarwin.

**Voorlopige uitkomst.** De gewone compilatie van de voorbeeldklasse `triangle/Triangle.java` slaagde met Temurin 21.0.12.1. De compilatie met de Major-plugin stopte met dezelfde fout als bij de eerdere proef met JDK 24.0.2. Er is nog geen geslaagde mutation run en dus ook geen meting van de mutation score of de snelheid.

## Werking en projectcontext

Major gebruikt een plugin voor de Java-compiler om tijdens het compileren kleine wijzigingen in de code aan te brengen. Zo kan een vergelijking als `<` veranderen in `<=`. Als een test door die wijziging faalt, heeft de test de mutant gedetecteerd (*killed*). Een mutant die de tests doorstaat, blijft *survived*. Major levert zelf drie bestanden mee voor het standalone-voorbeeld. In `triangle/Triangle.java` staat de code die een driehoek classificeert. `TriangleTest.java` is de broncode van het testprogramma. Het script `run.sh` compileert de bestanden en start het testprogramma. Bij succesvolle compilatie van `TriangleTest.java` maakt de compiler `TriangleTest.class` aan.

De officiële downloadpagina vermeldt Major 3.0.1 van 10 september 2024 als nieuwste release. Java 21 is op 19 september 2023 gemaakt. De startinstructies van Major noemen Java 8 en de changelog vermeldt Java 11-ondersteuning voor de mutator. De documentatie geeft geen volledige lijst van compatibele JDK-versies en vermeldt geen reden waarom ondersteuning voor nieuwere versies ontbreekt. Onze proeven geven een resultaat voor JDK 21.0.12.1 en 24.0.2. Voor andere versies, waaronder JDK 11, heb ik zelf geen testresultaten. Beveiliging valt buiten deze twee compilatieproeven.

Het SAD noemt in H6.4.4 Java 21 als runtime van het spel. Voor de gezamenlijke ADR moet ook de JDK van de daadwerkelijke CNI-build worden gecontroleerd, omdat Major tijdens het compileren draait (niet dat we major gaan gebruiken).

## Installatie en inhoud van de download

De officiële ZIP is uitgepakt op Windows. De proeven zijn in Git Bash uitgevoerd vanuit de map van het meegeleverde voorbeeld. Onderstaande mapindeling komt uit de officiële installatiehandleiding.

| Map | Inhoud en doel |
|---|---|
| `bin/` | Bevat onder meer `major`, `ant` en `mmlc`. Met `bin/major` compileer je de voorbeeldklasse met mutaties. Ant is een buildtool die stappen uit een `build.xml` uitvoert, zoals compileren en testen. Het standalone-voorbeeld gebruikt `run.sh`. |
| `config/` | Archief en broncode van de runtimeklasse die Major tijdens de uitvoering gebruikt. |
| `doc/` | Lokale handleiding, waaronder `major.html`. |
| `example/` | Voorbeelden met Ant en een standalone-voorbeeld. Elk heeft een `run.sh`. |
| `lib/` | Bibliotheken van Major. Het testprogramma heeft onder andere `major-rt.jar` nodig. |
| `mml/` | MML-regels bepalen waar Major mutaties mag aanbrengen en welke wijzigingen zijn toegestaan, bijvoorbeeld het vervangen van vergelijkingen in één methode. De MML-compiler (`bin/mmlc`) zet die regels om in een `.mml.bin`-bestand. De proef gebruikt het meegeleverde, al gecompileerde `tutorial.mml.bin`. |

Bron voor de mapindeling: [officiële installatiehandleiding van Major](https://mutation-testing.org/).

**AI-gebruik.** AI is gebruikt om de foutmeldingen uit de lokale Git Bash-logs te onderzoeken.

## Proef 1 – Meegeleverd voorbeeld op JDK 24

Open Git Bash in de uitgepakte map. Op de testmachine stond het voorbeeld op het volgende pad:

```bash
cd ~/Downloads/major-latest/major/example/standalone
javac -version
bash run.sh
```

De actieve compiler was `javac 24.0.2`. De laatste melding was `ClassNotFoundException: TriangleTest`. Om te zien welke stap als eerste mislukte, is het script opnieuw uitgevoerd met een trace:

```bash
bash -x run.sh 2>&1 | tee major-run.log
ls -l TriangleTest.class ../../lib/major-rt.jar
grep -inE 'javac|error|exception' major-run.log | head -40
```

De controle met `ls` vond **na afloop** wel `major-rt.jar`, maar geen `TriangleTest.class`. De laatste foutmelding verklaart waarom Java het testprogramma niet kon starten. De trace laat zien dat de eerste fout op een eerdere stap ontstond. Al voor het compileren van de test crashte de mutator bij een aanroep van een interne compilermethode:

```text
java.lang.NoSuchMethodError: 'boolean com.sun.tools.javac.code.Symbol$VarSymbol.isLocal()'
```

## Proef 2 – Hetzelfde voorbeeld op JDK 21

In dezelfde Git Bash-sessie is overgeschakeld op de al geïnstalleerde Temurin-JDK. Pas het pad aan als die op een andere plek staat.

```bash
export JAVA_HOME="$HOME/.jdks/temurin-21.0.12.1"
export PATH="$JAVA_HOME/bin:$PATH"
java -version
javac -version
bash -x run.sh 2>&1 | tee major-java21.log
```

De trace bevestigde compiler **21.0.12.1**. De stappen van het voorbeeld gaven de volgende resultaten:

1. **Gewone compilatie.** `javac triangle/Triangle.java` voltooide zonder fout.
2. **Compilatie met Major.** `../../bin/major --mml ../../mml/tutorial.mml.bin triangle/Triangle.java` stopte in `Mutator.visitVarDef` met dezelfde `NoSuchMethodError` als op JDK 24. De plugin verwacht hier een compilermethode die in deze omgeving niet met die signatuur beschikbaar is.
3. **Compilatie van de test.** Het script ging verder. Bij het compileren van `TriangleTest.java` kon de compiler de pakketten `major.mutation` en `triangle` niet vinden. Het script gebruikte `.:../../lib/major-rt.jar` als classpath. De dubbele punt is een vermoedelijke tweede oorzaak bij gebruik van de Windows-JDK. De variant met een puntkomma is nog niet succesvol getest.
4. **Uitvoering van de test.** Omdat stap 3 geen `TriangleTest.class` opleverde, eindigde de uitvoering met `ClassNotFoundException: TriangleTest`. De compilercrash in stap 2 trad al eerder op.

## Afweging voor de gezamenlijke ADR

Een praktisch pluspunt is dat de ZIP een klein voorbeeld, uitvoerscripts en een lokale handleiding bevat. Daardoor kon de compilerplugin los van de projectbuild worden getest. Het belangrijkste minpunt is dat de mutator al op dat voorbeeld vastloopt met beide geteste JDK’s. Als JDK 21 een vaste eis van de projectbuild is, raden we Major 3.0.1 voor die build af. Het genereren van mutanten is daarvoor een basisvoorwaarde. Een proef met JDK 11 kan nog laten zien hoe Major op een gedocumenteerde versie werkt. Zo’n proef vraagt geen downgrade van het project en verandert de uitkomst van de JDK 21-proef niet. De conclusie geldt voor het Java-deel van het CNI-project.

## Bronnen

- [Major – overzicht, installatie en voorbeelden](https://mutation-testing.org/)
- [Major – releases en changelog](https://mutation-testing.org/downloads/)
- [Major – tutorial bij het Triangle-voorbeeld](https://mutation-testing.org/tutorial.html)
- [Oracle – releasedatum van JDK 21](https://www.oracle.com/java/technologies/javase/21-relnote-issues.html)
- CNI Software Architecture Document, §6.4.4. Testbewijs: lokale Git Bash-uitvoer van de twee proeven.

## Concrete tabel
| Pluspunten van Major | Minpunten van Major |
|---|---|
| De ZIP bevat een handleiding, startscripts en een werkend voorbeeldproject om de installatie te testen. | De mutator crashte in onze proeven op zowel JDK 21 als JDK 24. Dat maakt Major 3.0.1 ongeschikt voor de geteste Java 21-opstelling. |
| Met MML kun je instellen welke code Major muteert en welke wijzigingen het mag maken. | Het voorbeeldscript gebruikt een classpath met `:`. Dat gaf op Windows vermoedelijk een extra probleem bij het compileren van de test. |