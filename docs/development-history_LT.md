# Projekto kūrimo istorija

[English](development-history.md) | [Lietuvių](development-history_LT.md)  
[README (EN)](../README.md) | [README (LT)](../README_LT.md)

## Įvadas

Real-Time Q&A Assistant neprasidėjo kaip iš karto pilnai sukurta programa.

Projektas palaipsniui išaugo iš nedidelio audio-processing proof of concept į modulinę Windows desktop programą, galinčią fiksuoti system audio, transkribuoti kalbą, atpažinti užbaigtus klausimus, generuoti context-aware atsakymus ir beveik realiuoju laiku palaikyti klausimų istoriją.

Kiekvienas svarbesnis architektūrinis pakeitimas atsirado kaip atsakas į realią problemą, pastebėtą development arba testing metu.

Šiame dokumente aprašoma ši projekto evoliucija.

## 1 etapas — pradinis Audio Capture

Pirmasis tikslas buvo paprastas:

> Užfiksuoti Windows leidžiamą garsą ir padaryti jį prieinamą tolimesniam apdorojimui.

Programa naudoja Windows WASAPI loopback per PyAudioWPatch.

Tai leidžia programai fiksuoti garsą, kurį leidžia pats kompiuteris, o ne tiesiogiai įrašinėti mikrofoną.

Pirmasis sėkmingas etapas patvirtino, kad programa gali atlikti:

```text
Windows audio
→ WASAPI loopback
→ Python
→ raw audio frames
```

Šiame etape dar nebuvo continuous transcription, semantinio apdorojimo, question detection ar GUI history.

Tikslas buvo tik įrodyti, kad patikimas system-audio capture apskritai yra įmanomas.

## 2 etapas — Speech-to-Text

Kai audio capture jau veikė, buvo pridėtas speech-to-text apdorojimas.

Užfiksuotas audio buvo konvertuojamas į WAV formatą ir siunčiamas į speech-to-text API.

Svarbus techninis sprendimas buvo nerašyti laikinų WAV failų į diską.

Vietoje to programa sukuria audio buffer tiesiog RAM atmintyje.

Duomenų eiga tapo tokia:

```text
system audio
→ captured frames
→ in-memory WAV
→ speech-to-text
→ text
```

Taip atsirado pirmasis pilnas audio-to-text pipeline.

## 3 etapas — nuolatinis apdorojimas

Vienkartinis garso įrašymas ir vienas transcription request real-time programai buvo nepakankamas.

Kitas žingsnis buvo continuous audio processing.

Audio pradėtas skaidyti į valdomo dydžio chunks ir nuolat apdoroti tol, kol programa veikia.

Tai sukūrė naują problemą:

> Audio capture turi tęstis net tuo metu, kai ankstesnis audio jau yra apdorojamas.

Visiškai sequential dizainas ilgainiui reikštų, kad audio processing ir network requests pradėtų blokuoti vienas kitą.

Todėl atsirado background processing.

## 4 etapas — Worker Threads ir Queues

Programa buvo pertvarkyta naudojant atskirus worker threads.

Pagrindiniai workers tapo:

```text
Capture Worker
Transcription Worker
Answer Worker
```

Duomenims tarp atskirų processing etapų perduoti buvo įvestos queues.

```text
Capture Worker
    ↓
audio_queue
    ↓
Transcription Worker
    ↓
answer_queue
    ↓
Answer Worker
```

GUI atnaujinimai taip pat buvo atskirti naudojant:

```text
gui_queue
```

Tokia architektūra leido skirtingoms programos dalims veikti nepriklausomai.

Audio capture nebereikėjo laukti answer generation, o GUI nebebuvo tiesiogiai priklausomas nuo background network operacijų.

## 5 etapas — Audio overlap

Chunk-based speech processing sukėlė dar vieną praktinę problemą.

Žodis, esantis vieno chunk pabaigoje, galėjo būti perkirstas tarp dviejų chunks.

Pavyzdžiui:

```text
Chunk 1:
"...the amount of"

Chunk 2:
"data and how did you..."
```

Be papildomos apsaugos dalis kalbos galėjo būti prarasta.

Todėl maždaug viena sekundė audio iš ankstesnio chunk pabaigos buvo pradėta pridėti prie kito chunk pradžios.

Tai pagerino kalbos tęstinumą, tačiau atsirado kita problema:

> Speech-to-text rezultatuose pradėjo kartotis tie patys žodžiai.

## 6 etapas — Transcript overlap pašalinimas

Kad būtų pašalintas dėl audio overlap atsiradęs pasikartojantis tekstas, buvo pridėta transcript palyginimo logika.

Sistema lygina:

```text
end of previous transcript
        ↕
beginning of new transcript
```

Sutampantys žodžiai pašalinami prieš prijungiant naują tekstą.

Vien exact matching nepakako, nes speech-to-text rezultatai gali šiek tiek skirtis net transkribuojant tą patį audio.

Todėl buvo įvestas fuzzy matching.

Overlap logika tapo tolerantiška nedideliems skirtumams, tačiau kartu stengtasi agresyviai nepašalinti nesusijusio teksto.

Šis etapas gerokai pagerino tęstinumą tarp gretimų speech-to-text rezultatų.

## 7 etapas — Conversation Turn State

Continuous transcription pati savaime neatsakė į svarbų klausimą:

> Kurie transcript fragmentai priklauso tam pačiam žodžiu užduodamam klausimui?

Todėl programoje atsirado current conversation turn sąvoka:

```text
current_turn
```

Nauji transcript fragmentai kaupiami į esamą turn.

Turn išlieka aktyvus tol, kol sistema nusprendžia, kad kalbantis žmogus užbaigė klausimą.

Taip aiškiai atsiskyrė:

```text
raw transcription fragments
```

nuo:

```text
a logical spoken question
```

Aktyvaus conversation turn dydis taip pat buvo apribotas, kad ilgos sesijos metu context negalėtų augti be ribų.

## 8 etapas — Silence Detection

Natūralus pirmas būdas nustatyti klausimo pabaigą yra aptikti, kada žmogus nustoja kalbėti.

Todėl audio sluoksnis pradėjo stebėti RMS volume.

Kai audio tam tikrą laiką išlieka žemiau nustatyto threshold, sistema suformuoja confirmed silence boundary.

Tai padėjo nustatyti momentus, kai kalbantis žmogus tikriausiai nustojo kalbėti.

Tačiau testing parodė, kad vien silence nepakanka.

Žmogus gali tiesiog sustoti pagalvoti ir po to tęsti tą patį klausimą.

Pavyzdžiui:

```text
"Could you tell me about a project where..."
        ↓
pause
        ↓
"...you had to process a large amount of data?"
```

Jeigu kiekvieną silence laikytume klausimo pabaiga, gautume neužbaigtus klausimus.

Todėl reikėjo antro detection lygio.

## 9 etapas — Semantic Question Detection

Po confirmed silence buvo pridėtas semantic question-state detection.

Vietoje prielaidos, kad silence automatiškai reiškia klausimo pabaigą, semantiškai analizuojamas visas sukauptas conversation turn.

Detector grąžina vieną iš trijų būsenų:

```text
YES
WAIT
NO
```

Reikšmės:

```text
YES  → the question appears complete
WAIT → the speaker probably intends to continue
NO   → the current speech is not a completed question
```

Taip atsirado svarbus dviejų lygių modelis:

```text
silence detection
        ↓
semantic question detection
```

Silence sluoksnis nustato, kada jau prasminga tikrinti tekstą.

Semantic sluoksnis nustato, ar klausimas iš tikrųjų baigtas.

Multi-part questions testing parodė, kad `WAIT → YES` eiga yra patikimesnė už paprastą segmentation pagal silence.

## 10 etapas — Completed Question Snapshot

Kai semantic detector grąžina `YES`, aktyvus conversation turn užfiksuojamas kaip completed-question snapshot.

Tai garantuoja, kad vėliau ateinantis audio nebegalės pakeisti jau atpažinto klausimo.

Programa laukia tikro naujo teksto prieš resetindama aktyvų conversation turn.

Tai tapo svarbu silence-only laikotarpiais.

Jei turn būtų resetinamas iškart po `YES`, tęsiantis tylai galėtų atsirasti tušti turns.

Galutinis behavior tapo toks:

```text
question completed
→ snapshot stored
→ silence may continue
→ new real text arrives
→ start next conversation turn
```

## 11 etapas — Context-Aware Answer Generation

Kai užbaigtus klausimus jau buvo galima pakankamai patikimai atpažinti, pridėtas answer generation.

Answer-generation etapas naudoja:

```text
detected question
+
user / candidate context
+
scenario / job context
```

Tikslas nebuvo generuoti generic answers.

Sistema turi kurti atsakymus, paremtus vartotojo pateikta informacija, ir neišgalvoti nepatvirtintos patirties.

Question translation ir suggested-answer generation buvo sujungti į vieną LLM request.

Taip tas pats klausimas modeliui nesiunčiamas du kartus.

Output struktūra tapo:

```text
QUESTION_LT
ANSWER_EN
```

Ateityje šį dizainą galima adaptuoti kitoms kalboms arba kitokiam response format.

## 12 etapas — nepriklausomas Answer Worker

Answer generation gali užtrukti ilgiau nei audio transcription.

Jeigu answer generation būtų vykdomas tiesiogiai Transcription Worker viduje, incoming speech processing galėtų būti pristabdytas.

Todėl buvo sukurtas atskiras Answer Worker.

Transcription Worker dabar tereikia perduoti:

```text
completed question
→ answer_queue
```

ir jis iš karto gali toliau apdoroti naują audio.

Answer Worker model generation vykdo nepriklausomai.

Šis atskyrimas tapo svarbia real-time programos elgsenos dalimi.

## 13 etapas — Graphical Interface

Praktiniam programos naudojimui buvo sukurtas Tkinter GUI.

Interface rodo:

```text
detected question
translated question
suggested answer
```

Background workers tiesiogiai nekeičia Tkinter widgets.

Vietoje to jie siunčia struktūrizuotus events per:

```text
gui_queue
```

GUI thread periodiškai apdoroja šiuos events.

Taip visos Tkinter operacijos lieka main thread ir išvengiama nesaugaus GUI keitimo iš background threads.

## 14 etapas — Question History

Pradinė GUI versija rodė tik naujausią klausimą.

Praktinio naudojimo metu tapo aišku, kad vartotojas gali norėti grįžti prie ankstesnio klausimo, programai tuo pačiu metu ir toliau klausantis.

Todėl atsirado question-history model.

Kiekvienam klausimui suteikiamas unikalus:

```text
question_id
```

GUI saugo atskirus entries:

```text
question
translation
answer
status
```

Pridėta navigacija:

```text
Previous
Next
Latest
Question X / Y
```

Papildomas `follow_latest` state nustato, ar GUI automatiškai seka naujausią klausimą.

Jeigu vartotojas peržiūri ankstesnį klausimą, nauji klausimai ir toliau išsaugomi, tačiau GUI neperšoka nuo tuo metu peržiūrimo įrašo.

## 15 etapas — patikimas Question ir Answer susiejimas

Asynchronous answer generation sukėlė subtilią problemą.

Atsakymas gali būti sugeneruotas jau po to, kai vartotojas GUI perėjo prie kito klausimo.

Todėl pagal display position nebegalima patikimai nustatyti, kuriam klausimui priklauso konkretus atsakymas.

Sprendimas — visame pipeline naudoti unikalų Question ID.

```text
question_id
    ↓
question snapshot
    ↓
answer queue
    ↓
generated answer
    ↓
GUI history entry
```

Taip question-answer mapping tapo deterministinis ir nebepriklausė nuo timing.

## 16 etapas — API Usage Investigation

Development metu netikėtai didelis LLM input usage atskleidė svarbią reliability problemą.

Pakartotinis augančio text context apdorojimas kartu su netyčia vienu metu paleistais keliais programos procesais galėjo labai stipriai padidinti API usage.

Šis incidentas paskatino ne vien lokalią pataisą, o kelis architektūrinius pakeitimus.

Projektas gavo:

```text
bounded detector input
bounded answer input
bounded current conversation turn
single-instance protection
reduced redundant processing
```

Tai tapo viena svarbiausių optimization pamokų projekte:

> API pagrindu veikiančioje real-time programoje kontroliuoti, kada modelis kviečiamas, yra taip pat svarbu kaip kontroliuoti, kas jam siunčiama.

## 17 etapas — Single-Instance Protection

Testing parodė, kad netyčia paleidus daugiau nei vieną programos kopiją gali prasidėti dubliuotas audio processing ir dubliuoti API requests.

Todėl buvo pridėtas Windows file lock.

Programa dabar patikrina, ar kita jos instancija jau veikia.

Jeigu lock nepavyksta gauti, antras procesas užsidaro ir nepaleidžia antro processing pipeline.

Ši apsauga egzistuoja tik tol, kol veikia application process.

## 18 etapas — Pure-Silence STT Optimization

Audio sistema jau naudojo silence detection klausimo ribai nustatyti.

Ši informacija buvo panaudota ir unnecessary speech-to-text calls sumažinti.

Kiekvienam naujam audio chunk pažymima, ar jame buvo aptikta kalba.

Jei chunk yra tik silence:

```text
speech_detected = false
```

speech-to-text request nevykdomas.

Tačiau visas processing cycle nėra praleidžiamas.

Tai svarbu, nes būtent silent chunk gali būti tas momentas, kai pagaliau patvirtinama klausimo pabaiga.

Todėl galutinė logika tapo:

```text
pure silence
→ skip STT API call
→ keep question-boundary processing active
```

Real regression test patvirtino, kad ši optimization išlaiko `WAIT → YES` question detection ir kartu pašalina unnecessary STT calls tylos metu.

## 19 etapas — Queue ir State Cleanup

Kai pagrindinis pipeline tapo stabilus, buvo peržiūrėti vidiniai queue duomenys ir state variables.

Kai kurios reikšmės buvo likusios iš ankstesnių development etapų, nors jų programa jau nebenaudojo.

Unused queue metadata ir dead state buvo pašalinti.

Audio queue buvo supaprastinta iki:

```text
chunk number
audio frames
speech detected flag
confirmed-silence flag
```

Pašalinus nereikalingą state worker interface tapo paprastesnis ir aiškesnis.

## 20 etapas — Debug Transcript Cleanup

Development metu pilnas sukauptas transcript buvo naudingas transcript overlap behavior debug.

Transcript nuolat augo ir buvo vis iš naujo spausdinamas į terminalą.

Kai overlap ir conversation-turn logika tapo pakankamai stabili, full transcript pačiai programai jau nebebuvo reikalingas.

Todėl jis buvo pašalintas iš working pipeline.

Programa dabar saugo tik tą state, kurio realiai reikia processing.

Tai sumažino debug triukšmą ir pašalino nereikalingą ilgainiui augantį transcript.

## 21 etapas — Shutdown Improvements

Pradinė shutdown procedūra laukė worker threads naudodama blocking `join()` tiesiai Tkinter thread.

Funkciškai tai veikė, tačiau GUI galėjo atrodyti pakibęs, kol network-dependent workers baigdavo darbą.

Todėl shutdown flow buvo pakeistas.

Vietoje GUI thread blokavimo Tkinter periodiškai tikrina, ar background workers jau baigė darbą.

Shutdown seka tapo:

```text
stop capture
→ wait without blocking GUI
→ finish transcription
→ finish queued answers
→ release audio resources
→ destroy GUI
```

Programa taip pat užtikrina, kad single-instance lock būtų atleistas pasibaigus GUI runtime.

## 22 etapas — Final Regression Testing

Po optimization ir stability pakeitimų buvo atliktas kelių klausimų end-to-end regression scenarijus.

Testas apėmė:

```text
system audio capture
→ multi-chunk transcription
→ audio overlap
→ transcript merge
→ WAIT state
→ continued question
→ YES state
→ answer generation
→ silence STT skip
→ second question
→ second answer
→ GUI history
→ clean shutdown
```

Galutinis testas patvirtino:

- keli klausimai apdorojami nepriklausomai;
- silence-only chunks praleidžia STT requests;
- semantic question detection ir toliau veikia;
- Question IDs teisingai atskiria klausimus;
- sugeneruojamas vertimas ir suggested answer;
- history navigation išlieka funkcionali;
- programa švariai užsidaro;
- pašalintas debug transcript nėra reikalingas normaliam programos veikimui.

## Modularization

Projektui augant atskiros atsakomybės palaipsniui buvo išskirtos į atskirus modulius.

Privati implementation šiuo metu atskiria:

```text
audio
transcription
LLM interaction
worker orchestration
GUI
configuration
application lifecycle
context data
```

Ši modularization nebuvo iki galo suprojektuota dar prieš pradedant darbą.

Ji natūraliai susiformavo praktinio development metu, vis geriau aiškėjant atskirų komponentų atsakomybėms.

## Development Approach

Projektas buvo kuriamas incrementally.

Tipinis ciklas buvo:

```text
implement one small capability
        ↓
test it independently
        ↓
integrate it into the pipeline
        ↓
perform a controlled runtime test
        ↓
observe problems
        ↓
refine the architecture
```

Buvo vengiama didelių rewrites, jei konkrečią problemą buvo galima išspręsti mažu tiksliniu pakeitimu.

Features, neturinčios aiškios praktinės naudos, sąmoningai buvo atidedamos arba visai nepridedamos.

Pavyzdžiui, tiesioginė microphone integracija buvo ištestuota atskirai, tačiau į pagrindinę programą nebuvo įtraukta, nes dabartiniam pagrindiniam workflow reikalingas system audio.

## Pagrindinės išmoktos pamokos

Development metu atsirado kelios platesnės engineering pamokos.

### Real-Time nereiškia, kad viskas turi vykti vienu metu

Responsive pipeline galima sukurti atskiriant atsakomybes ir leidžiant nepriklausomiems etapams komunikuoti asynchronously.

### Silence naudinga, bet jos nepakanka

Audio silence yra geras trigger semantinei analizei, tačiau vien silence neleidžia patikimai nustatyti, ar žodžiu užduotas klausimas iš tikrųjų baigtas.

### Overlap išsprendžia vieną problemą, bet sukuria kitą

Audio overlap padeda neprarasti žodžių tarp chunks, tačiau tada reikia transcript de-duplication.

### Explicit State yra vertingas

Tokie state kaip:

```text
current_turn
question_id
follow_latest
silence_confirmed
new_turn_pending
```

leidžia programos behavior suprasti daug lengviau nei bandyti jį išvesti vien iš timing.

### API sąnaudos yra architektūros klausimas

Unnecessary model calls, duplicated processes ir uncontrolled context growth gali tapti realiomis sistemos problemomis.

Todėl API usage turi būti vertinamas jau projektuojant architektūrą, o ne tik po implementation.

### Debugging features neturi automatiškai tapti Product Features

Full transcript buvo naudingas kuriant overlap logiką, bet tapo nereikalingas, kai pipeline stabilizavosi.

Development tools ir production requirements turi būti atskiriami.

### Optimization turi remtis realiais požymiais

Programa nebuvo optimizuojama vien todėl, kad teoriškai kažką dar buvo galima optimizuoti.

Pakeitimai buvo daromi tada, kai testing parodė konkrečią jų priežastį.

## Dabartinė development būsena

Dabartinis prototipas turi pilną veikiantį processing pipeline:

```text
Windows system audio
        ↓
capture
        ↓
speech-to-text
        ↓
overlap removal
        ↓
conversation-turn accumulation
        ↓
silence confirmation
        ↓
semantic question detection
        ↓
completed-question snapshot
        ↓
context-aware answer generation
        ↓
GUI history
```

Sistema taip pat praėjo atskirą optimization ir stability etapą.

Dabartinis development dėmesys jau persikėlė nuo pagrindinio pipeline kūrimo į:

```text
documentation
portfolio presentation
controlled distribution
future user-driven customization
```

## Tolimesnė kryptis

Dabartinis prototipas sąmoningai vertinamas kaip veikiantis baseline, o ne kaip nekintantis galutinis produktas.

Tolimesnis development gali būti grindžiamas realiais vartotojų poreikiais.

Galimos kryptys:

- standalone Windows packaging;
- paprastesnė configuration ne techniniams vartotojams;
- saugus vartotojo API key valdymas;
- configurable response language ir style;
- papildomi contextual data sources;
- GUI customization;
- scenario-specific question-detection behavior;
- papildomi performance ir usability pagerinimai pagal realų naudojimą.

Naujos funkcijos turėtų būti pridedamos tik tada, kai jos suteikia aiškią praktinę vertę.

## Santrauka

Projektas iš paprasto audio-capture eksperimento išaugo į modulinę real-time AI processing programą.

Svarbiausia development seka buvo:

```text
capture
→ transcribe
→ continue
→ merge
→ accumulate
→ detect
→ snapshot
→ generate
→ navigate
→ optimize
→ stabilize
```

Galutinė architektūra atspindi problemas, kurios buvo realiai atrastos programą kuriant ir testuojant, o ne teorinį dizainą, sukurtą dar prieš prasidedant development.
