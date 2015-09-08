<!-- Projektet skall också redovisas i form av en skriftlig rapport. Rapporten skall vara på ca 15 sidor, med en textmassa som oräknat figurer och källkodsexempel är på minst 10 sidor. (Det blir lätt mer Detta är lågt! Jag räknar med att bättre projekt fyller ut dessa sidor utan vidare. Det dubbla är inget extremt. Men jag räknar inte sidantal.)

Texten skall vara skriven med era egna ord. Eventuellt citerad text måste mycket tydligt markeras som citerad och får inte dominera rapporten.

Referenserna är viktiga! Ange tydligt vilka referenser ni baserar er på.

Rapporten bör vara snyggt utformad och lättläst.

Den bör vara inlämnad inom en vecka efter sista redovisningstillfället. Rekommenderat format: PDF.
 -->

-------------------------------------------------------------------------------

# Spelmotor i WebGL


## Introduktion

Projekten i denna kurs fokuserar oftast på en specifik avancerad teknik. För mig var det mer intressant att titta på hur man skulle kunna sätta ihop en spelmotor från början.

Mål med mitt projekt
- Bygga något med WebGL
- Lära mig om spelmotorkonstruktion

Vilka resurser som använts. Game Programming patterns har använts som referens, annars har det varit WebGL-specifikationen. Inspiration har också tagit till viss del från hur Goo Engine är uppbyggd.

Specifikt ville jag undvika en scengraf (varför? och var anledningen befogad? svaret är troligen nej eftersom jag inte hade någon aning om vilka problem jag skulle stöta på innan jag faktiskt började bygga själva spelen)


## Designmönster som använts och varför


### Game Loop / Update function

Detta är det enklaste mönstret som fortfarande är värt att lyftas fram. Det finns ingen spelmotor eller spel som jag känner till som inte har en uppdateringsfunktion. Det är funktionen som uppdaterar spelet mellan varje gång det ritas eller skriva ut på skärmen, och det händer antingen vid ett fixt intervall eller vid specifika events.

Alla som är intresserade av att skapa realtidsspel utan att använda en befintlig spelmotor (hmm ... rekommenderas inte om målet är att göra spel fort) kommer att programmera en game loop väldigt tidigt. I JavaScript kan man påbörja och komma långt som spelprogrammerare med endast följande kod. Detta är en aningen förenklad variant av det jag började med.

```js
    function update(time) { /* ... */ }
    function draw() { /* ... */ }

    function loop(time) {
      // Be om att köra funktionen igen
      requestAnimationFrame(loop);

      // Uppdatera spelets tillstånd
      update(time);

      // Rita spelet
      draw();
    }

    // Starta loopen
    requestAnimationFrame(loop);
```


### Finite State Machine (FSM)

FSM är ett vanligt sätt att hålla reda på tillstånd i diverse program. Det finns ett antal tillstånd något kan befinna sig i. Ett specifikt tillstånd har regler för när det flyttar till ett annat tillstånd. Vid en viss input reagerar tillståndet endast om det har en regel för den input den får.

Om vi tänker oss att input är knapptryckningar så kan spelarens figur reagera på olika sätt beroende på vilket tillstånd figuren är i när knapptryckningen händer. Det trevliga är att man kan skicka alla knapptryckningar till figuren och vara säker på att den bara reagerar på de kommandon som för tillfället är aktuella. Tillstånden dikterar inget mer än vilken input som orsakar en förflyttning till ett annat tillstånd. Det är upp till programmeraren att fylla tillstånden med annat.

I mitt fall la jag till tre saker som en spelprogrammerare skulle kunna använda för att konstruera ett beteende. När ett tillstånd går från inaktivt till aktivt och tvärtom så exekveras in- och utträdesfunktioner, om de finns. Under tiden däremellan, när tillståndet är aktivt, så kan en funktion exekveras vid varje uppdatering.


### Stack för scenes

Från början var idén att använda FSM för att hantera om spelet var pausat, vilken bana man var på, och vad som händer i corner cases (?), men jag upptäckte snart att det blev väldigt mycket att hålla reda på. När denna känsla infinner sig bör man andas in lugnt och försöka lista ut om det finns ett enklare sätt att lösa saker.

Misstaget var att varje bana då behövde känna till vilka andra banor som finns. Det är problematiskt när man bestämmer sig mitt i att en viss bana behöver tas bort, eller om man vill lägga till en bana i en serie av andra banor.

BILD PÅ STATE MACHINE

Nej, något bättre behövdes. Med inspiration av hur cocos2d hanterar "scenes" samt läsning utav kapitlet om "state" i GPP upptäckte jag något som kallas pushdown automata.

BILD PÅ STATE MACHINE VS PUSHDOWN AUTOMATA

Pushdown automata fungerar ungefär som en LIFO stack. Det som är aktivt ligger uppe på toppen och hanterar all input som kommer in. När något i scenen signalerar att det är dags att lägga av så kan scene plockas bort från stacken och återvända till vad som ligger underst i stacken ELLER så kan man omedelbart lägga på något på toppen, t. ex. när man byter bana. På så vis kan ett standardläge alltid ligga på botten av stacken, och ovanpå det kan man slänga på en bana. Utöver detta så kan spelaren få för sig att pausa spelet — då kan man helt enkelt lägga på "pausläget" överst i stacken.

KODSKILLNAD?

Nackdelen med denna approach är att om man inte är försiktig och bygger en för "hög stack" så äter man snabbt upp arbetsminnet.

### Entity / Component / Builder

#### Entity / Component
När man programmerar objektorienterat så finns ett antal sätt att återanvända kod. Det troligen vanligaste sätt att återanvända kod är genom arv. I spelvärlden kan detta se ut som att fiender härstammar från en klass `Enemy`, och utökar/beskriver en viss typ av fiende i subklasserna `BigEnemy` och `SmallEnemy`. När det är väldigt få antal variationer som i det förra exemplet så kan arv vara utmärkt men när antalet variationer ökar eller när en `Enemy` delar funktionalitet med saker som inte är fiender så finns en risk att antalet klasser blir svårhanterligt många. Dessutom finns det stor risk att vissa klasser innehåller funktionalitet som den inte behöver.

En lösning till detta är att använda designmönstrena Entity och Component. Dessa mönster tillsammans bryter ut funktionalitet i komponenter som kan kombineras ihop i en så kallad *entity* för att skapa olika typer av saker. 

Man kan föreställa sig en *entity* som en "grej" som existerar i spelvärlden. Exakt vad det är för något beror helt och hållet på vilka komponenter den innehåller. Det är kombinationen av komponenter som gör en *entity* till en fiende, pickup, vägg, pistolkula eller spelare.

Spelmotorn måste hantera alla *entitys* som existerar och be dem att uppdatera sig själva. Detta kan ske direkt i t. ex. en spelmotors huvudklass eller i en klass som har som sitt syfte att hålla reda på alla *entitys*.


#### Builder/Factory pattern
Eftersom fiendetyperna `BigEnemy` och `SmallEnemy` inte längre är sina egna klasser så kan de inte längre instansieras med en rad kod, e.g. `new BigEnemy()`. När man använder Entity/Component krävs det att man skapar en `Entity`-klass samt alla komponentklasser som ska användas. Dessutom behöver varje komponent fästas i en entity. Det kan lätt bli mycket kod som också upprepas i onödan.

För att råda bot på detta kan man använda designmönster som Builder eller Factory. Dessa enkapsulerar skapandet av mer komplicerade objekt, ungefär som ett recept. Då reduceras kodmängden som behövs för att skapa en viss sak till en enda funktionsanrop. I likhet ger man som kund på ett café sällan instruktioner steg-för-steg utan oftast tittar man på menyn och beställer t. ex. en cappucino. Vill man ha något utöver det vanliga får man beskriva mer i detalj, och om man beställer samma sak många gånger om tröttnar cafépersonalen till slut och lägger till din beställning som ett menyalternativ istället.


### Fysikmotorn???
Finns det några särskilda mönster som användes i fysikmotorn?


## Komponenter

- Spelmotorn
  - Entity
  - Component
    - CameraComponent
    - GraphicsComponent
      - Graphics3DComponent
      - GraphicsSpriteComponent
    - FsmComponent
    - KeyboardInputComponent
  - Renderer
    - CanvasRenderer
    - WebGLRenderer
  - PlixApp
  - Scene
  - Util

- Dependencies
  - fsm
    - FSM (en manager som hanterar transitions mellan states)
    - State (states som innehåller beteende)
  - physix
    - klasser
  	  - World
  	  - Body
  	  - Vec2
    - AABB-kollisioner
    - Krafter
    - Massa


## Spelexempel

En spelmotor är inte särskilt mycket på egna ben. Dess styrka visar sig först när man faktiskt börjar bygga spel med den.


### Pong, ett simpelt spel

Spelet passade till den spelmotorstruktur jag hade byggt upp.

Kodexempel här? Och sedan visa på hur det blev annorlunda. 

I detta skede fanns både en Canvas2D- och WebGL-renderare som kunde bytas ut med varandra med knappt märkbar skillnad i den renderade bilden.

Pong-spelet visade inte på de begränsningar som fanns (varför?) i och med att det var byggt med själva kapabiliteterna som redan fanns (vilket var ... ?). Svagheterna visade sig först när ett mer involverat exempel byggdes.


### Jump Dude

Ett mer involverat spelexempel utvecklades också för att se hur spelmotorn klarade sig i detta aningen mer avancerade exempel.

Kraven som visade sig under utvecklingen
- Bättre meddelanden genom spelet. Just nu finns bara stöd för entititeter att kontrollera internt. Det finns inget globalt sätt för entiteter att kommunicera med varandra.
- Trodde att jag skulle kunna använda min FSM till allt, men det gjorde det mesta lite krångligt.

De största problem som uppstod var med kommunikation mellan olika saker som hände i spelet. När något händer i en del av spelet bör det kunna påverka vad som händer i ett annat ställe. Till exempel kameran skakar till när spelarens figur landar på marken. Just nu är det aningen klumpigt inlagt med globala variabler och som callback från fysikmotorn.

Callbacken från fysikmotorn var intressant, eftersom fysikmotorn onekligen måste kunna meddela spelmotorn om händelser.


### Ett ord angående prioritering

Nope. Det fanns ingen mening att fokusera på hur snabb spemotorn var under utvecklingstiden. Det är något jag kände till redan innan, och jag var ganska bra på att hålla fingrarna borta från att försöka bättra på hastigheten. Det brukar kallas premature optimization när man fokuserar på prestanda innan behovet har uppstått.

Däremot finns ytterligare en fälla att trilla ned i. Och jag föll ner i den. Att planera och arkitekterna utifrån antaganden om vad som kommer vara viktigt eller inte gör att mycket tid spenderas på att bygga för dessa gissningar. Som det visade sig under utveckligens tid hade jag idéer om vad som var viktigt och inte. Det bet mig i baken och gjorde att jag grävde i fel hörn alldeles för länge. Detta misstag kallas premature architecture. Det är inte fel att arkitektera ifall man faktiskt vet med säkerhet att problemen kommer att uppstå. Om man är det minsta osäker så ska man bara skriva upp idén och återbesöka den när problemet verkar ha uppstått. Vet man av erfarenhet att en game loop behövs för spelet så är det helt okej att implementera den i förebyggande syfte. Men det är en hal stig att börja vandra och man bör vara mycket försiktig och observant, helt plötsligt har man jobbat en månad i tron om att man jobbat på spelet, men i själva verket ser det likadant ut som fyra veckor innan.

Det finns ett tankesätt som lyder "Make it work. Make it right. Make it fast.". Make it right syftar på att se till att driva bollen framåt, och det är helt okej att skriva kod som man helst inte talar högt om. Make it right syftar till när man ägnar sig åt att refaktorera den möjligtvis pinsamma kod, men som fungerar. Det sista steget är make it fast, och detta är något vi vill ägna oss åt först när vi identifierat att koden eller programmets prestanda påverkar programmets kvalitet.

