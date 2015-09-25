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


## Designmönster och metoder som använts


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

Skillnad mellan Builder och Factory?

Factory
"factory method pattern requires the entire object to be built in a single method call, with all the parameters pass in on a single line. The final object will be returned."

Builder
"in essence a wrapper object around all the possible parameters you might want to pass into a constructor invocation. This allows you to use setter methods to slowly build up your parameter list."

(http://stackoverflow.com/questions/757743/what-is-the-difference-between-builder-design-pattern-and-factory-design-pattern)


### Fysikmotorn???
Finns det några särskilda mönster som användes i fysikmotorn?

#### Simulering
Varje simuleringssteg görs mer Euler-integration, dvs man matar in tiden det tog för en bild att ritas ut i de klassiska fysikaliska formlerna för mekanisk rörelse. Spelet lägger till krafter som fysikkroppen `body` summerar i `body.accumulatedForce`, och sedan beräknas kroppens acceleration, hastighet och position i fysikmotorns integrationssteg.

```js
  // Calculate acceleration
  switch(body.type) {
    case Body.DYNAMIC:
      body.acc.x = this.gravity.x + (body.accumulatedForce.x / body.mass);
      body.acc.y = this.gravity.y + (body.accumulatedForce.y / body.mass);
      break;
    case Body.KINEMATIC:
      body.acc.x = body.accumulatedForce.x / body.mass;
      body.acc.y = body.accumulatedForce.y / body.mass;
      break;
  }

  // Zero out accumulated force
  body.accumulatedForce.x = 0;
  body.accumulatedForce.y = 0;

  // Calculate velocity
  body.vel.x += body.acc.x * timestep;
  body.vel.y += body.acc.y * timestep;

  // Calculate position
  body.pos.x += body.vel.x * timestep;
  body.pos.y += body.vel.y * timestep;
```

De två typerna av fysikkroppar `Body.DYNAMIC` och `Body.KINEMATIC` bestämmer hur kroppen beter sig i fysiksimuleringen. En kropp med typen `Body.DYNAMIC` beter sig helt i linje med vad man förväntar sig av en fysikkropp. Man kan tänka sig att det är en "vanlig" fysikkropp. Typen `Body.KINEMATIC` påverkas inte av gravitationskrafter, och påverkas inte av dynamiska kroppar. En kollision mellan en `DYNAMIC` och en `KINEMATIC` hanteras som om `KINEMATIC` har oändligt stor massa. Skulle två `KINEMATIC` kollidera så hanteras denna kollision som fullständigt oelastisk.


#### Fixed timestep from variable duration
Inte direkt ett designmönster, utan mer en metod för att se till att fysikmotorn inte tar för stora steg i varje iteration. Ifall fysikmotorn tar för stora steg kan det hända att två fysikkroppar som skulle ha kolliderat med varandra förflyttas för långt och "missar" varandra. En lösning är att se till att fysiksimuleringen körs med tillräckligt litet tidssteg.

```js
  // physics: the physics engine
  // frametime: time for the frame to render
  var dt = 10,
  var timeleft = frametime;

  while(timeleft > 0) {
    var step;

    if(timeleft < dt) {
      step = timeleft;
    } else {
      step = dt;
    }

    physics.integrate(step);

    timeleft -= dt;
  }
```

Detta gör att fysiksimuleringen potentiellt kommer att köras mer än en gång per frame. Fördelen är att fysiksimuleringen blir mer precis. Dock tar det naturligtvis längre tid att köra mer än en fysiksimulering per frame, och därför kan denna metod orsaka en sänkning ökning i renderingstid, vilket i nästa steg orsakar att fysikmotorn potentiellt behöver köra ännu fler steg. I praktiken verkar det som att denna metod fungerar någorlunda stabilt.

Vill man komma förbi detta potentiella problem så finns mer avancerade metoder, se [Fix Your Timestep!](http://gafferongames.com/game-physics/fix-your-timestep/).


#### AABB-kollisioner
Axis-Aligned Bounding Box (AABB) är ett sätt att representera en area (eller volym i tredimensionella applikationer). Ett enkelt sätt att föreställa sig hur ett objekt representeras av en AABB är en rektangel på rutat papper. Om man ritar en figur på det rutade pappret så kan man också rita en rektangel som omsluter figuren med hjälp av rutnätet på pappret. Om man ritar fler figurer kommer den omslutande rektangelns topp och botten samt vänster- och högersida vara respektivt parallella med alla andra rektanglar på pappret. Om man använder denna representation av en kropp i fysikmotorn så är det förhållandevis enkelt att beräkna ifall en kropp överlappar (kolliderar) med en annan kropp.

```js
  dh1 = bodyA.right - bodyB.left;
  dh2 = bodyB.right - bodyA.left;
  dv1 = bodyA.top - bodyB.bottom;
  dv2 = bodyB.top - bodyA.bottom;

  if(dh1 <= 0 || dh2 <= 0 || dv1 <= 0 || dv2 <= 0) {
    // no overlap
  } else {
    // there is overlap
  }
```

Algoritmen baseras på fyra test där det gäller att om ett utav testen är sanna så är kroppen garanterad att inte överlappa med den andra kroppen. Om den första kroppens högersida befinner sig till vänster om den andra kroppens vänstersida så är det garanterat att de båda kropparnas AABB inte överlappar. De andra sidorna av kropparnas AABB jämförs på liknande sätt.


#### Bitmask Layers
Ibland vill man ha objekt som krockar med endast vissa saker. I spelet används det till små detaljer som flyger omkring men som inte bör påverka spelaren. Jag har använt bitmasking för att tillåta ett litet antal lager. Ett fysikkropp kan tillhöra en eller flera lager. I Jump Dude finns två lager; ett förgrunds- och ett backgrundslager. Förgrundlagret innehåller spelaren och allt som spelaren kan kollidera med, dvs mark, power-ups, fiender och så vidare. Bakgrundslagret innehållet det mesta som förgrundslagret innehåller förutom spelaren och fiender. Denna konfiguration gör att man kan lägga in fysikkroppar som studsar omkring men som inte påverkar spelaren eller fiender. Ett exempel är om spelaren slår sönder en låda och fragment av lådan studsar omkring. Vi vill inte att spelarens figur ska påverkas av dessa små lådfragment, men vi vill att de ska kollidera med resten av världen. Fragmenten finns alltså endast på bakgrundslagret. När spelaren och lådfragmenten överlappar ser fysikmotorn att spelaren inte tillhör ett lager som fragmentet också tillhör, så därför ignoreras den potentiella kollisionen.

```js
  if(bodyA.layer & bodyB.layer) {
    // The bodies share at least one layer
  } else {
    // The bodies do not share any layer
  }
```


#### Callback?
Callbacks används för att spelmotorn ska kunna reagera på något särskilt som händer. Det är aningen meningslöst att ha en fysikmotor utan något sätt att prata med spelmotorn. När en kollision har hänt körs callbackfunktionen på båda kropparna.

BILD PÅ CALLBACK



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
    - FSM (hanterar transitions mellan states)
    - State (states som innehåller beteende)
  - physix
    - klasser
  	  - World
  	  - Body
  	  - Vec2


## Spelexempel

En spelmotor är inte särskilt mycket på egna ben. Dess styrkor (och svagheter) visar sig först när man börjar bygga spel med den. En viktig distinktion som måste göras är att spelmotorns kod och spelets kod inte är densamma. Olika spel har användning för liknande funktionalitet, som t. ex. rendering, hantering av mus och tangentbord, fysik, ljud och så vidare, men det som skiljer två spel från varandra är spellogiken. Den hör inte hemma i spelmotorkoden.

De två spelexempel jag byggt importerar spelmotorn och konstruerar sedan alla delar som spelen består av med hjälp av de funktioner som finns tillgängliga i spelmotorn. Innan vi tittar närmare på spelen så kommer härnäst en översikt över hur spelmotorn är tänkt att användas.


### Plix: en spelmotor

De viktigaste komponenterna för att bygga ett spel med spelmotorn, Plix, är klasserna `PlixApp`, `Scene` och `Entity`. Vi börjar med `PlixApp`.


#### Grunderna

När man instansierar ett `PlixApp`-objekt så kommer det att söka i DOM:en efter elementet `<canvas id="game"></canvas>`. I detta `canvas`-element kommer alla rendering att ske. Utöver det så initieras spelmotorns tidshantering, mus- och tangentbordshantering startas, en scenstack allokeras och renderingsmotorn startas (och får en referens till just det `canvas`-element som nämnts).

```js
var game = new PlixApp();
```

Spelmotorn är nu instansierad, men det är inte startad än. Eftersom scenstacken är tom väntar `PlixApp`-instansen i variabeln `game` på att medlemsfunktionen `PlixApp.runWithScene(Scene s)` ska köras. För att göra det behöver vi först skapa ett `Scene`-objekt.

```js
var game = new PlixApp();

var myScene = new Scene();
```

Mer än så behövs inte för att skapa scenen. Och för att starta spelmotorn görs följande.

```js
var game = new PlixApp();

var myScene = new Scene();

game.runWithScene(myScene);
```

När denna kod körs så har spelmotorn startat. Med anropet `game.runWithScene(myScene)` så har `myScene` lagts på toppen av scenstacken i `game`, och den interna uppdateringsloopen är igång. Varje uppdatering säger `PlixApp` till `myScene` att uppdatera sig själv. Men om man kör denna kod som den är just nu så kommer inget särskilt att hända. Inget ritas ut i `<canvas>`-elementet eftersom `myScene` ännu inte innehåller något som kan ritas (eller uppdateras).

Det som saknas är ett eller flera `Entity`-objekt i `myScene`. En `Entity` är det som rör sig, reagerar och lever i spelvärlden.

```js
var game = new PlixApp();

var myScene = new Scene();

var myEntity = new Entity();
myScene.attachEntity(myEntity);

game.runWithScene(myScene);
```

Det vi gjort nu är skapat ett `Entity`-objekt i `myEntity` och lagt till det i `myScene` med `myScene.attachEntity(myEntity)`. Det betyder att nu när `game` ber `myScene` att uppdatera så kommer den att be `myEntity` att uppdatera sig. Skapar vi och lägger till fler `Entity`-objekt i `myScene` så gäller samma sak för dem. Händelseförloppet vid varje uppdatering är alltså; spelmotorn (`PlixApp`) ber den översta scenen i scenstacken att uppdateras och i sin tur ber scenen alla entitys den känner till att uppdateras.

Vi har upp tills nu pratat mycket om uppdateringar, men vad händer i en uppdatering?

I varje uppdatering skickar `PlixApp` till den översta scenen den mängd tid som passerat sedan förra uppdateringen. Den tidsmängden används för att uppdatera fysikmotorn (om den finns). Sedan körs för varje entity dess script (om det finns), och slutligen överförs entityns fysikkroppsposition till entityns transform  (detta gäller om entityn har en fysikkropp) för att entityn ska ritas ut på rätt plats i `<canvas>`-elementet.

`myEntity` har ingen fysikkomponent än, och innan vi tittar på den så ska vi titta närmre på hur vi uppdaterar en entity via dess `script`-attribut.

```js
var game = new PlixApp();

var myScene = new Scene();

var myEntity = new Entity();
myScene.attachEntity(myEntity);

var radius = 10;
myEntity.script = function(ent) {
  ent.transform.position.x = game.width/2 + radius * Math.cos(ent.scene.app.timeElapsed / 1000);
  ent.transform.position.y = game.height/2 + radius * Math.sin(ent.scene.app.timeElapsed / 1000);
}

game.runWithScene(myScene);
```

I en entitys `Entity.transform.position` finns dess `x`- och `y`-koordinatpositioner. Genom att tilldela `Entity.script` en funktion med argumentet `ent` så kommer entityn åt sig själv i funktionskroppen. I kodexemplet syns också att vi kommer åt (`PlixApp.timeElapsed`), vilket är den tid som fortlöpt sedan spelmotorn startades, genom `ent.scene.app.timeElapsed`. Alla `Entity`-objekt har tillgång till det `Scene`-objekt det tillhör genom sitt `Entity.scene`-attribut. Detsamma gäller på liknande vis för `Scene`-objekt och `PlixApp` genom `Scene.app`-attributet.

Ovanstående kod kommer att förflytta `myEntity` i en cirkelbana med radien `10` och mittpunkt i centrum av spelmotorns `<canvas>`-element.


#### Gör mer med komponenter

För att göra mer än att flytta omkring entitys lägger man till `Component`-objekt i entityn. Dessa innehåller information som spelmotorn eller spelet kan använda i diverse syften, t. ex. fysiksimulering, rendering, hålla reda på antal liv och så vidare. I förra sektionen nämndes fysikkomponenten som vi ska titta lite närmare på.

```js
var ent = new Entity();

var physicsOptions = {
  entity: ent,
  width: 2,
  height: 8,
  type: Body.DYNAMIC,  // or Body.KINEMATIC
  mass: 2
};

var pc = new PhysicsComponent(physicsOptions);
ent.addComponent(pc);
```

När `PhysicsComponent` skapas så gör den först något som alla `Component`-objekt gör; den lägger till `entity`-attributet som skickas med i `options`-objektet (i detta exempel `physicsOptions`). Detta för att ha tillgång till entityn i dess konstruktor. `PhysicsComponent` kontrollerar sedan om det finns en fysikvärld (`World`) och skapar den om den inte finns, följt av att skapa ett `Body`-objekt som registreras hos `World`. Notera att `Body` och `World` inte är en del av spelmotorn Plix utan är en del av fysikmotorn _physix_ som skrevs i samband med spelmotorn. Mer om det snart.

Slutligen lägger vi till fysikkomponenten `pc` till entityn `ent` med `ent.addComponent(pc)`. Nu är allt sammankopplat, och vi kan komma åt fysikkomponenten genom `ent.components.physics`. Precis vad `XXX` byts ut mot i `ent.components.XXX` beror på vad `Component`-klassen har angett i dess `ComponentXXX.type`-attribut. `PhysicsComponent` har angett `'physics'` och statemachine-komponenten `FSMComponent` har `'fsm'`.

Nu när `ent` har en `PhysicsComponent` så kommer spelmotorn att känna av det och göra sånt som behövs för en entity med fysikkomponent, t. ex. synkroniseras fysikvärldens position med entitetens transformposition. På samma sätt kommer spelmotorns renderingmotor använda `GraphicsComponent` om den existerar som instruktion för att rita något som är knutet till entityn. Det är i sammansättningen av dessa olika komponenter som en entity blir en fiende, spelare, vägg, power-up, etc.

Jag nämnde tidigare att klasserna `World` och `Body` som används i fysikkomponenten inte tillhör spelmotorn. Det stämmer, och vi skulle kunna skriva en `Box2DPhysicsComponent` som istället använder fysikmotorn _Box2D_. Det är tänkt att vi ska kunna skapa en mängd olika typer av spel med endast ett par få byggstenar; `PlixApp`, `Scene` och `Entity` som kryddas med `Component`-klasser för att bli unika spelelement.


### Spel 1: Pong

Det klassiska spelet pong har återskapats säkert tusentals gånger. Denna variant är mycket simpel. När spelet har laddats in visas en startskärm och när höger musknapp klickas hoppar spelet in i spelläget. En kvadrat studsar på väggar och spelarna försöker hindra kvadraten från att åka i mål, dvs träffa väggen bakom respektive spelares rektangel. När rektangeln träffar mål är matchen slut och spelet återvänder till startskärmen.


#### Initialisering

Själva huvudprogrammet `main.js` är så pass kort att vi kan titta på den i dess helhet. Spelmotorn `PlixApp` samt den kod som används för att "bygga" spelet, `PongFactory`, importeras. `PongFactory` innehåller en mängd funktioner med namn som `PongFactory.createXXX(...)`. Vi återkommer till dessa funktioner snart.

```js
define([
    'plix/PlixApp',
    'js/PongFactory'
], function(
    PlixApp,
    PongFactory
) {
    'use strict';

    // Create an app. It's idle until told to run.
    var app = new PlixApp();

    // Create main menu scene
    var menu = PongFactory.createMenu(app);

    // Run the app with main menu scene
    app.runWithScene(menu);
});
```

Vi instansierar spelmotorn med `var app = new PlixApp()`. Spelet ska börja med en menyskärm, och den skapas med `var menu = PongFactory.createMenu(app)`. Den här funktionen är en factory-funktion som returnerar ett `Scene`-objekt innehållande spelets meny. Under huven skapas ett antal `Entity`-objekt som placeras på skärmen för att visa texten "CLICK 2 START".

```js
// figLayout: 2D array with "grid-text"
...
var scene = new Scene('menu');

var ent;
for(var r = 0; r < figLayout.length; r++) {
  for(var c = 0; c < figLayout[r].length; c++) {
    if(figLayout[r][c]) {

      ent = new Entity();
      scene.attachEntity(ent);

      ent.transform.position.x = 10*c - 85 + (app.width / 2);
      ent.transform.position.y = 10*r - 100 + (app.height / 2);

    }
  }
}
...
```

Notera att variabeln `ent` innehåller en referens till den senast skapade entityn i loopen. Efter loopen definieras `script`-attributet på den sista entityn i loopen.

```js
...
// Last entity waits for mouse input
ent.script = function(ent) {
  if(app.input.mouse.leftButton) {
    var level = PongFactory.createLevel(app);
    app.pushScene(level);
  }
};
...
```

I detta fall undersöker funktionen om musens vänsterknapp befinner sig i aktivt (nedtryckt) läge och skapar i så fall spelets nästa scen med `var level = PongFactory.createLevel(app)` samt lägger scenen på toppen av scenstacken med `app.pushScene(level)`.

Nu när `var menu = PongFactory.createMenu(app)` har returnerat spelets menyscen så startas spelmotorn med `app.runWithScene(menu)` och vi får följande resultat i webbläsaren.

BILD PÅ SPELETS STARTSKÄRM


#### Matchläge

I förra sektionen nämndes att spelet övergår till matchläge när vänster musknapp klickas. Detta skedde genom att matchlägets scen skapas med `PongFactory.createLevel(PlixApp app)` och sedan lades överst på scenstacken, `PlixApp.pushScene(Scene s)`.

Vad händer egentligen när dessa funktioner kallas?

`PongFactory.createLevel(PlixApp app)` skapar en ny scen, och i scenen skapas sju objekt; fyra väggar varav två är mål, två spelarkontrollerade rektanglar och en kvadrat (eller "boll"). Alla dessa objekt är av typen `Entity`, och de "tillverkas" på liknande vis med hjälp av de byggstenar som finns i spelmotorn. För att förenkla skapandet har varje spelelement en factory-funktion som ser ut som `PongFactory.createXXX(Scene s, Object options)`, där `XXX` är spelelementet.

Den simplaste av alla element är den vanliga väggen. `PongFactory.createWall(...)` skapar en `Entity` och adderar sedan en `PhysicsComponent` med konfiguration i argumentet `options`.

```js
PongFactory.createWall = function(scene, options) {

  var wall = new Entity();
  scene.attachEntity(wall);

  wall.transform.position.x = options.x;
  wall.transform.position.y = options.y;

  var pc = new PhysicsComponent({
    tag: options.tag || 'wall',
    entity: wall,
    type: Body.KINEMATIC,
    width: options.width,
    height: options.height
  });
  wall.addComponent(pc);

  return wall;
};
```

Detta är grundstrukturen för att skapa nästan allt spelmotorn ska hålla reda på. Det `Scene`-objekt som är längst upp i scenstacken är aktivt och kommer att hålla reda på och uppdatera alla `Entity`-objekt i en intern `Entity`-lista när `PlixApp` säger till (kom ihåg att det bara är den översta scenen i scenstacken som blir ombedd av `PlixApp` att uppdateras; med andra ord ligger menyskärmens scen under matchlägets och "väntar" på att bli aktiv igen). Ett `Entity`-objekt skapas med `var wall = new Entity()` och registreras hos scenen med `scene.attachEntity(Entity e)`.

Efter att ha registrerats hos scenen konfigureras entityn och vi skapar de komponenter som behövs. I det här fallet ska en vägg skapas, och det behövs inte mer än en `PhysicsComponent` som läggs till i entityn med `wall.addComponent(Component c)`. Fysikkroppstypen är en `Body.KINEMATIC` eftersom väggen inte ska rubbas ur plats.

Funktionen `PongFactory.createWall(..)` anropas från `PongFactory.createLevel(...)` för att skapa både vanliga väggar och de som representerar mål. Det som skiljer den vanliga väggen från målväggen är att den senare har värdet `'goal1'` eller `'goal2'` i `options.tag`-argumentet. Det finns inget särskilt med dessa värden utan vi kommer senare använda dem för att undersöka om bollen har träffat ett mål eller en vägg.

```js
// Bottom goal
PongFactory.createWall(scene, {
    x: app.width/2,
    y: app.height + 10,
    width: app.width,
    height: 20,
    tag: 'goal2'  // goal-tag
});

// Left wall
PongFactory.createWall(scene, {
    x: -9,
    y: app.height/2,
    width: 20,
    height: app.height
    //tag: 'wall'  not needed since it's the default
});
```

Bollen skapas i factory-funktionen `PongFactory.createBall(Scene s, Object options)` nästan exakt likadant som väggarna, men med fysikkroppstypen `Body.DYNAMIC`. Detta gör att den studsar och kolliderar som en "vanlig" fysikalisk kropp. Vi tilldelar också en funktion till `onCollision`. Funktionen kommer att anropas när fysikmotorn registrerar en kollision som involverar bollens fysikkropp.

```js
// from: PongFactory.createBall(scene, options)
var pc = new PhysicsComponent({
  tag: 'ball',
  entity: ball,
  type: Body.DYNAMIC,
  width: options.width,
  height: options.height,
  onCollision: function(otherBody) {
    if(otherBody.tag === 'goal1' || otherBody.tag === 'goal2') {
      ball.scene.app.popScene();
    }
  }
});
```

När en kollision skett med en fysikkropp som har ett `tag`-attribut som antingen är `'goal1'` eller `'goal2'` så plockar vi bort den översta scenen ur scenstacken med `ball.scene.app.popScene()`.

Den sista typen av spelelement, de spelarkontrollerade rektanglarna, är lite mer intressanta eftersom de behöver reagera på tangentbordstryckningar. Factory-funktionen `PongFactory.createPaddle(Scene s, Object options)` börjar på samma sätt som för de andra elementen men lägger till två ytterligare komponenter; `KeyboardComponent` och `FSMComponent`.

```js
// from: PongFactory.createPaddle(scene, options)
// Add input component
var ic = new KeyboardInputComponent({
    entity: paddleEntity
});
paddleEntity.addComponent(ic);
```

`KeyboardComponent` behöver bara instansieras och sedan läggas till hos `paddleEntity`. Detta ger entityn tillgång till en kopia av tangentbordets tillstånd genom objektet `Entity.components.input.keys`. I koden som följer används tillgången till tangentbordet. En `FSMComponent` skapas och sätter upp det standard-`State` som spelarrektangeln har.

```js
// from: PongFactory.createPaddle(scene, options)
// Create FSM component
var fsm = new FSMComponent();
paddle.addComponent(fsm);

// Configure FSM
fsm.createState('default')
  .onEnter(function(ent) {
    ent.script = function() {
      var xVel = 0;
      xVel += ent.components.input.keys[options.keys.left] ? -0.5 : 0;
      xVel += ent.components.input.keys[options.keys.right] ? 0.5 : 0;
      ent.components.physics.body.vel.x = xVel;
    };
  });
fsm.enterState('default');
```

Notera att `options.keys.left` och `options.keys.right` är argument till `PongFactory.createPaddle(...)`. De två spelarnas rektanglar för Pong-spelet skapas med olika tangenter för att kontrollera rörelse.

```js
// from: PongFactory.createLevel(app)
// Create paddle 1
PongFactory.createPaddle(scene, {
  x: 100,
  y: app.height - 50,
  width: 100,
  height: 10,
  keys: { left: 'A', right: 'S' }
});

// Create paddle 2
PongFactory.createPaddle(scene, {
  x: 200,
  y: 50,
  width: 100,
  height: 10,
  keys: { left: 'K', right: 'L' }
});
```

Det sista som behöver göras nu när alla matchlägesobjekt har skapats är att sätta bollen i rörelse. Detta görs genom att applicera en kraft i fysikmotorn på bollens fysikkropp.

```js
// from: PongFactory.createLevel(app)
// Give the ball a little push
var force = {
  x: (Math.random() - 0.5) * 0.1,
  y: Math.sign(Math.random() - 0.5) * Math.max(Math.random() * 0.05, 0.01)
};
ball.components.physics.body.applyForce(new Vec2(force.x, force.y));
```

Nu uppdateras alla `Entity`-objekt i scenen och matchlägen pågår fram tills bollen har kolliderat med en utav målväggarna.


#### Resultat

Pong-spelet visade på en begränsning som finns när man använder en _FSM_. Användningen av `FSMComponent` skippas helt och hållet och ersättas med ett `Entity.script` som aldrig ändras. Anledningen är att vi endast har en state för vardera spelarrektangel. Från början var så inte fallet, men eftersom man bara kan befinna sig i ett state i taget blir det problematiskt. Om vi hade haft tre state (stilla, vänster- och högerförflyttning) så kommer specialfallet när en spelare håller in både höger- och vänsterknapp att bli svårt eller omöjligt att hantera.

Tekniskt sett så tillhör inte _fsm_-biblioteket spelmotorn, trots att den utvecklades i samband med den. Däremot var den faktiskt tilltänkt att hantera mycket av spellogiken, men visade sig inte särskilt passande till Pong-kontroller. I övrigt fungerar spelet bra och det krävs inte allt för mycket kod för att skapa det.


#### 2D-rendering med WebGL och Canvas2D

I detta skede fanns både en Canvas2D- och WebGL-renderare som kunde bytas ut med varandra med knappt märkbar skillnad i den renderade bilden. För den som är intresserad av att se hur Canvas2D och WebGL skiljer sig finns koden för [CanvasRenderer](https://github.com/marcusstenbeck/plix/blob/e9f58401f7ad317a388ed14425f9ab21f256e35d/src/plix/CanvasRenderer.js) och [WebGLRenderer](https://github.com/marcusstenbeck/plix/blob/e9f58401f7ad317a388ed14425f9ab21f256e35d/src/plix/WebGLRenderer.js).



### Spel 2: Jump Dude

Ett mer involverat spelexempel utvecklades också för att se hur spelmotorn klarade sig i detta mer avancerade exempel. I och med att vi i förra sektionen pratade mycket om hur vi med koden skapade spelet så tänker jag i denna sektion röra mig mindre steg-för-steg och diskutera ett antal utmaningar jag stötte på längs vägen. Dessa är alltså saker som uppstått under utvecklingen av detta lite mer involverade spelexempel.

Jag utvecklade motorn i samband med att göra Pong, men med Jump Dude så var utmaningen att bygga ett spel med de verktyg jag hade. Förutom att fixa buggar så var det bara 3D-grafik som tillkom i motorn. Fysikmotorn _physix_ fick också några buggfixar och uppgraderingar (layers, sensorkroppar, fixed timestep integrering, callback-manager).


#### Allmänt

Det är jävligt mycket kod i ett spel som inte är en del av spelmotorn. Alla små detaljer behöver någonstans att vara och ett spel ska kunna kännas väldigt annorlunda från ett annat.



#### Utmaning: När något händer med A, gör också något med B

- Bättre meddelanden genom spelet. Just nu finns bara stöd för entititeter att kontrollera internt. Det finns inget globalt sätt för entiteter att kommunicera med varandra.

De största problem som uppstod var med kommunikation mellan olika saker som hände i spelet. När något händer i en del av spelet bör det kunna påverka vad som händer i ett annat ställe. Till exempel kameran skakar till när spelarens figur landar på marken. Just nu är det aningen klumpigt inlagt med globala variabler och som callback från fysikmotorn.

Callbacken från fysikmotorn var intressant, eftersom fysikmotorn onekligen måste kunna meddela spelmotorn om händelser.


#### Utmaning: FSM är bra när man vill vara väldigt noggrann
- Trodde att jag skulle kunna använda min FSM till allt, men det gjorde det mesta lite krångligt.

I slutändan använde jag bara _fsm_ för spelarens fyra tillstånd: `grounded`, `jumping`, `finished` och `dead`. Det bler alldeles för pilligt att "dra alla kablar".

BILD PÅ STATE MACHINE FÖR SPELAREN

```js
fsm.createState('jumping')
  .onEnter(function(ent) {
      ent.script = function() {
          sideMove(ent);
          scaleJump(ent);
      };
  })
  .addTransition('ground', 'grounded')
  .addTransition('die', 'dead')
  .addTransition('finish', 'finished');
```

Kolla hur störigt det är att behöva lägga till alla transitions. Sjukt jobbigt att upprepa detta för varje state.

Vad använde jag istället? Det mesta i spelet är rätt enkelt, så det behövs inte mer än ett eller ett par tillstånd. Med så få tillstånd var det mindre omväg att helt enkelt hålla reda på sakerna (exempel).

När spelaren dör eller klarar en bana då? Det sånt jag hade tänkt använda _fsm_ till.

Vad händer när spelaren dör?
Vad händer när en bana avklaras?
app.funktioner() som håller reda på spelets tillstånd.

Det är klart att jag skulle kunnat bygga eller refaktorera spelmotorn för att bättre hantera dessa saker, men syftet var att sluta utveckla spelmotorn och istället utveckla ett spel!

"Give a man a game engine and he will deliver a game. Teach a man to make a game engine and he will never deliver anything."


#### Utmaning: Gör mer (eller mindre) med fysikmotorn

Fysikmotorn har en mängd användbara funktioner, och det vore trevligt att använda den till dess styrkor. Kollisionsdetektering är en av grundpelarna i en fysiksimulering, men om man kan vara flexibel med hur eller när den används så blir den ännu mer användbar för spel.

Inspiration har tagits från fysikmotorn Box2D, men jag är säker att många andra fysikmotorer har samma funktionalitet.


##### Bitmask-lager

Fysikmotorn innehåller en värld, och i den världen studsar saker omkring. Jag ville använda fysikmotorn, men ha vissa saker som inte påverkar eller påverkas av spelaren. En lösning är att ha två fysikvärldar och synkronisera objekt som befinner sig i båda.

Problem:
  Vilken fysikvärld är "rätt" om de skiljer sig?
  Att ha dubbla kopior kostar dubbelt minne. Vi vill hålla oss ifrån att onödigtvis ta upp plats i minnet.

Fysiklager to the rescue. Bitmask. I kollisionsdetekteringen ignoreras kollisioner mellan kroppar som inte delar åtminstone ett fysiklager med varandra.

```js
var pc = new PhysicsComponent({
    tag: options.tag || 'wall',
    entity: wall,
    type: Body.KINEMATIC,
    width: options.width,
    height: options.height,
    layer: PHYSICS_LAYER_ONE | PHYSICS_LAYER_TWO
  });
wall.addComponent(pc);
```


##### Sensorkroppar
Känna av överlapp men inte göra något mer än det.


#### Utmaning: Callbacks körs flera gånger om

Hade trubbel med callbacks som kördes flera gånger. Kollision mellan två objekt registreras mer än en gång i tidsstegshoppandet så fick se till att bara tillåta registrering av callback mellan två objekt en gång (A-B och B-A behandlas likadant).


#### Utmaning: Dynamisk text
Dynamisk text i WebGL (och GL i allmänhet) är knepigt. I webbläsaren kan man använda den inbyggda textrenderingen om man vill. Jag ville gärna att texten var en del av "världen", så jag behövde lösa det på annat vis.

Skapa entitys och arrangera dem så att de bildar bokstäver.


#### Utmaning: Att lägga till "känsla"
Mycket easing-funktioner.


#### 3D-rendering med WebGL

...



### Sammanfattning
Sammanfattning av projektet osv.


#### Att hantera saker som jag inte vet hur jag ska göra
Oftast skapar jag en funktion och döper den till ett namn av det jag vill att funktionen ska göra. Jag placerar funktionsanropet på den plats i koden där den verkar passa. Sedan implementerar jag funktionen. Detta samlar allt på en fokuserad plats och jag kan när det fungerar lista ut hur jag kan göra saker bättre. Om det är okej att det "bara fungerar" så lämnar jag det som det är. Fördelen är att det blir väldigt explicit vad dessa rader kod är tänkt att åstadkomma.

Make it work. Make it right. Make it fast.

#### Ett par ord angående prioritering

Nope. Det fanns ingen mening att fokusera på hur snabb spemotorn var under utvecklingstiden. Det är något jag kände till redan innan, och jag var ganska bra på att hålla fingrarna borta från att försöka bättra på hastigheten. Det brukar kallas premature optimization när man fokuserar på prestanda innan behovet har uppstått.

Däremot finns ytterligare en fälla att trilla ned i. Och jag föll ner i den. Att planera och arkitekterna utifrån antaganden om vad som kommer vara viktigt eller inte gör att mycket tid spenderas på att bygga för dessa gissningar. Som det visade sig under utveckligens tid hade jag idéer om vad som var viktigt och inte. Det bet mig i baken och gjorde att jag grävde i fel hörn alldeles för länge. Detta misstag kallas premature architecture. Det är inte fel att arkitektera ifall man faktiskt vet med säkerhet att problemen kommer att uppstå. Om man är det minsta osäker så ska man bara skriva upp idén och återbesöka den när problemet verkar ha uppstått. Vet man av erfarenhet att en game loop behövs för spelet så är det helt okej att implementera den i förebyggande syfte. Men det är en hal stig att börja vandra och man bör vara mycket försiktig och observant, helt plötsligt har man jobbat en månad i tron om att man jobbat på spelet, men i själva verket ser det likadant ut som fyra veckor innan.

Det finns ett tankesätt som lyder "Make it work. Make it right. Make it fast.". Make it right syftar på att se till att driva bollen framåt, och det är helt okej att skriva kod som man helst inte talar högt om. Make it right syftar till när man ägnar sig åt att refaktorera den möjligtvis pinsamma kod, men som fungerar. Det sista steget är make it fast, och detta är något vi vill ägna oss åt först när vi identifierat att koden eller programmets prestanda påverkar programmets kvalitet.

