# Dokumentation – weekPlanner (Labb 2)

**Namn:** Johan Briger  
**Applikation:** weekPlanner – Aktivitetsöversikt för familjen  

---

## 1. Beskrivning av funktionalitet

För att förbättra användarupplevelsen och göra veckoplaneringen mer visuell och interaktiv har applikationen byggts med tre samverkande Vue-funktioner: **dynamisk kategorisering (`:class`)**, **tillståndsbaserad visuell feedback**, och en **dynamiskt stylad framstegsmätare (`:style`)**.

### Vad koden gör:

1. **Färgkodade aktiviteter (`:class`):**  
   När en aktivitet skapas tilldelas den en kategori (*idrott, utflykt, läxa, aktivitet*). Vue kopplar kategorinamnet dynamiskt till kortets CSS-klasser via `:class="[act.type, ...]"` vilket ger kortet en unik färg på vänsterkanten.

2. **Interaktiv status för avklarade uppgifter (`:class` & `v-model`):**  
   När användaren bockar i kryssrutan *"Klar"* uppdateras `act.completed` reaktivt via `v-model`. Vue lägger då dynamiskt till klassen `.is-completed` via villkorsstyrd class-binding (`{ 'is-completed': act.completed }`), vilket sänker opaciteten och drar en linje över rubriken.

````html
<div 
  v-for="act in sortedActivitiesByMember[member]" 
  :key="act.id"
  class="activity-card"
  :class="[act.type, { 'is-completed': act.completed }]"
>
  <h4>{{ act.title }}</h4>
  ...
</div>
````

3. **Dynamisk KPI-indikator (`:style`):**  
   I applikationens header beräknas andelen slutförda aktiviteter i en `computed`-variabel (`completionPercentage`). Kortet för slutförda aktiviteter använder inline Style Binding (`:style`) för att ändra bakgrundsfärgen till en grön nyans när 100 % av veckans aktiviteter är markerade som klara:

```html
:style="{ backgroundColor: completionPercentage === 100 && totalActivities > 0 ? 'rgba(16, 185, 129, 0.35)' : 'rgba(255, 255, 255, 0.15)' }"

````


## 2. Förslag på framtida förbättringar

Om jag hade vidareutvecklat applikationen i nästa steg hade jag prioriterat följande två förbättringar:

1. **Drag-and-Drop mellan dagar och kolumner:**  
   Idag måste användaren öppna redigeringsmodalen för att flytta en aktivitet till en annan dag eller ett annat barn. Att kunna dra och släppa ett kort direkt mellan olika kolumner hade gjort schemaläggningen avsevärt snabbare och mer intuitiv.

2. **Veckohantering och arkivering:**  
   Nuvarande vy visar alla aktiviteter i en och samma vy utan tidsavgränsning. En vidareutveckling hade varit att lägga till navigering mellan veckonummer (t.ex. V.37, V.38) samt automatisk arkivering av tidigare veckors genomförda aktiviteter.
