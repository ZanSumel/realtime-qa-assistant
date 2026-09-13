# Testavimas

[English](testing.md) | [Lietuvių](testing_LT.md)  
[README (EN)](../README.md) | [README (LT)](../README_LT.md)

## Apžvalga

Testing buvo atliekamas nuolat viso Real-Time Q&A Assistant development metu.

Kadangi programa jungia audio capture, network APIs, asynchronous workers, semantic processing ir desktop GUI, testing neapsiribojo vien izoliuotomis funkcijomis.

Projekte buvo derinami:

- atskirų komponentų testai;
- mock pagrįsti testai;
- kontroliuojami audio testai;
- realūs end-to-end runtime testai;
- regression testing po architektūrinių pakeitimų;
- manual usability observation.

Pagrindinis tikslas buvo ne tik patikrinti, ar veikia pavieniai komponentai, bet ir įsitikinti, kad visas real-time processing pipeline elgiasi teisingai realistiškomis sąlygomis.

## Testing metodas

Projektas buvo testuojamas incrementally.

Tipinis development ciklas:

```text
implement
→ test isolated behaviour
→ integrate
→ run controlled scenario
→ observe
→ correct
→ regression test
```

Toks būdas padėjo aptikti problemas kuo arčiau to etapo, kuriame jos atsirado.

Kur praktiškai įmanoma, buvo vengiama didelių nepatikrintų pakeitimų grupių.

## Audio Capture Testing

Pirmasis testing etapas buvo skirtas Windows system-audio capture.

Tikslas — patikrinti, ar programa patikimai gauna kompiuterio leidžiamą audio per WASAPI loopback.

Buvo tikrinama:

```text
WASAPI loopback device discovery
audio stream initialization
correct channel configuration
sample rate handling
continuous frame capture
```

Testuojamoje aplinkoje buvo naudojamas Windows loopback device, susietas su aktyviu Realtek audio output.

Sėkmingas capture tapo pagrindu visam likusiam pipeline.

## Microphone Test

Microphone capture taip pat buvo ištestuotas atskirai.

Nedidelis standalone test patvirtino, kad built-in Realtek microphone gali būti aptiktas ir sėkmingai naudojamas.

Tačiau microphone integration sąmoningai nebuvo pridėta į pagrindinę programą.

Dabartinis workflow remiasi system audio, o microphone support pagrindiniam use case nebuvo reikalingas.

Todėl tai buvo laikoma patikrinta ateities galimybe, o ne dabartinio produkto funkcija.

## Speech-to-Text Testing

Speech-to-text testing patvirtino, kad užfiksuotą audio galima:

```text
converted to an in-memory WAV buffer
→ sent to the STT API
→ returned as usable text
```

Buvo tikrinama ir normali kalba, ir tylos periodai.

Vėliau transcription etapas buvo papildytas pure-silence skipping, todėl testing taip pat patvirtino, kad silent audio gali praleisti nereikalingus STT calls nesugadindamas tolimesnio processing.

## Chunk Processing Tests

Continuous audio apdorojamas chunks.

Testing buvo sutelktas į kelias praktines rizikas:

```text
speech crossing chunk boundaries
missing words between chunks
duplicated words caused by overlap
silence occurring at chunk boundaries
```

Programa naudoja maždaug vienos sekundės audio overlap tarp gretimų chunks.

Testing patvirtino, kad tai sumažina riziką prarasti speech ties chunk boundaries.

## Transcript Merge Testing

Audio overlap natūraliai sukuria dubliuotą transcript turinį.

Todėl merge logika buvo tikrinama naudojant gretimus transcript fragmentus su pilnai arba iš dalies pasikartojančiomis frazėmis.

Pavyzdys:

```text
Previous:
"...how did you process the large amount of data"

New:
"the large amount of data and how did you validate it"
```

Tikslas buvo išsaugoti tęsinį:

```text
"...how did you process the large amount of data and how did you validate it"
```

nesukuriant dubliuotų frazių.

Buvo tikrinamas ir exact, ir fuzzy overlap behavior.

## Žinomi Transcript netobulumai

Speech-to-text output ne visada būna lingvistiškai idealus.

Testing metu kartais buvo pastebimi tokie artefaktai:

```text
"Could you tell me about. a time..."
```

arba:

```text
"what steps you took? to improve..."
```

Šie artefaktai netrukdė semantic question detector arba answer-generation modeliui teisingai suprasti klausimo.

Todėl papildomas agresyvus transcript normalization nebuvo įvestas.

Dabartinis behavior laikomas priimtinu tol, kol būsimas testing neparodys realios user-facing problemos.

## Silence Detection Testing

Silence detection buvo testuojamas naudojant RMS audio level.

Tikslas buvo atskirti:

```text
active speech
short natural pauses
confirmed silence
```

Programa neturi vertinti kiekvienos trumpos pauzės kaip klausimo pabaigos.

Silence period turi išlikti žemiau configured threshold visą nustatytą confirmation duration, ir tik tada inicijuojamas question-boundary check.

Testing patvirtino, kad speech atsinaujinus silence state teisingai resetinamas.

## Semantic Question Detection Testing

Semantic question detection buvo tikrinamas naudojant žodinius klausimus su natūraliomis pauzėmis.

Vienas svarbiausių scenarijų buvo multi-part question.

Pavyzdinė eiga:

```text
first part of question
→ silence
→ WAIT

speaker continues
→ silence
→ YES
```

Toks behavior patvirtino, kad sistema automatiškai nelaiko kiekvienos pauzės klausimo pabaiga.

Derinys:

```text
confirmed silence
+
semantic YES / WAIT / NO detection
```

pasirodė patikimesnis nei silence-only segmentation.

## Current Turn Testing

`current_turn` state buvo testuojamas siekiant patvirtinti, kad transcript fragmentai lieka vienoje grupėje tol, kol klausimas semantiškai užbaigiamas.

Svarbios elgsenos:

```text
new transcript fragments append to the current turn
WAIT preserves the same turn
YES creates a question snapshot
continued silence does not create a new empty turn
new real text starts the next turn
```

Tai neleido neteisingai skaidyti ilgų arba multi-part spoken questions.

## Question Snapshot Testing

Kai klausimas klasifikuojamas kaip užbaigtas, jo tekstas užfiksuojamas.

Testing patvirtino, kad vėliau gaunamas speech nepakeičia jau aptikto klausimo.

Tai svarbu, nes transcription tęsiasi net tuo metu, kai answer generation dar gali būti vykdomas.

## Question ID Testing

Kiekvienam completed question suteikiamas unikalus numeric ID.

Testing patvirtino, kad ID išlieka susietas su tuo pačiu klausimu per visą asynchronous pipeline:

```text
question detection
→ GUI entry
→ answer queue
→ answer generation
→ GUI update
```

Tai tapo ypač svarbu įvedus question history ir navigation.

## Mock GUI Testing

Prieš pasikliaujant vien live audio, GUI history behavior buvo tikrinamas naudojant mock events.

Mock testing leido imituoti:

```text
multiple detected questions
answers arriving later
navigation to previous questions
new questions arriving while viewing history
returning to the latest question
```

Taip GUI state logiką buvo galima patikrinti nepriklausomai nuo speech-to-text ir network timing.

## History Navigation Testing

GUI turi:

```text
Previous
Next
Latest
Question X / Y
```

Testing patvirtino, kad:

- ankstesni klausimai lieka pasiekiami;
- naujesni klausimai teisingai pridedami;
- navigation nesugadina history state;
- answers atnaujina teisingą klausimą;
- interface gali grįžti į naujausią klausimą;
- nauji klausimai automatiškai nenutraukia vartotojo, kai jis peržiūri senesnę history dalį.

Būtent dėl tokio behavior buvo įvestas ir ištestuotas `follow_latest` state.

## Asynchronous Answer Testing

Answer generation vykdo atskiras worker.

Tests patvirtino, kad transcription pipeline gali tęsti darbą tuo metu, kai Answer Worker apdoroja anksčiau aptiktą klausimą.

Tai buvo svarbu, kad lėtesni model responses neužblokuotų incoming audio processing.

## Translation and Answer Testing

Question translation ir suggested-answer generation atliekami vienu LLM request.

Testing patvirtino, kad response gali būti išparsintas į:

```text
translated question
suggested answer
```

Answer generation taip pat buvo tikrinamas pagal pateiktą contextual information.

Tikslas buvo generuoti naudingus atsakymus, bet neišgalvoti nepatvirtintų faktų ar vartotojo patirties.

## Pure-Silence STT Regression Test

Vienas paskutinių optimizavimo žingsnių buvo STT calls praleidimas chunks, kuriuose neaptinkama speech.

Šiam pakeitimui reikėjo atidaus regression testing.

Kritinis reikalavimas buvo:

> STT praleidimas neturi praleisti question-boundary processing.

Real test parodė, kad silent chunks generuoja:

```text
STT SKIP
```

o semantic question-state flow veikia toliau normaliai.

Test patvirtino, kad programa vis tiek gali pereiti per:

```text
WAIT
→ additional speech
→ YES
```

net kai tarpinių silence-only chunks metu STT API nekviečiamas.

## Single-Instance Testing

Single-instance lock buvo pridėtas po to, kai development metu paaiškėjo, jog netyčia vienu metu veikiančios kelios programos instancijos gali dubliuoti audio processing ir API usage.

Testing patvirtino:

```text
first application instance
→ acquires the lock

second application instance
→ cannot acquire the lock
→ does not start another processing pipeline
```

Tai sumažino netyčinio dubliuoto API consumption riziką.

## Shutdown Testing

Shutdown behavior buvo testuojamas po to, kai blocking thread joins Tkinter main thread buvo pakeisti.

Galutinė shutdown eiga buvo tikrinama siekiant patvirtinti:

```text
stop request is registered
audio capture stops
workers finish cleanly
remaining queued work is handled
audio resources are released
GUI closes
single-instance lock is released
```

Stebėta runtime seka baigėsi normaliai, nepalikdama vizualiai pakibusios programos.

## Final End-to-End Regression Test

Po optimization ir cleanup etapo buvo atliktas galutinis real-world regression test.

Scenarijuje buvo du atskiri žodžiu pateikti klausimai.

Pirmajam klausimui reikėjo kelių audio chunks ir semantic continuation.

Stebėta eiga:

```text
speech
→ transcription
→ WAIT
→ continued speech
→ WAIT
→ YES
→ answer generated
```

Tos pačios sesijos metu silence-only chunks teisingai praleido nereikalingus STT calls.

Tada antras klausimas buvo apdorotas atskirai:

```text
new speech
→ new question
→ YES
→ separate answer
```

Pilnas test patikrino:

```text
system audio capture
continuous chunking
audio overlap
transcript overlap removal
current_turn accumulation
confirmed silence handling
semantic WAIT / YES states
question snapshot creation
question_id assignment
answer generation
question translation
GUI updates
history navigation
pure-silence STT skipping
clean shutdown
```

Sėkmingam veikimui nereikėjo jokio pašalinto debug-only funkcionalumo.

## API Usage Testing

API consumption buvo vertinamas kaip sistemos reliability dalis.

Ankstesnis development incidentas parodė, kad pakartotinai augantis context ir kelios vienu metu veikiančios programos instancijos gali netikėtai stipriai padidinti model input usage.

Nustačius priežastį, testing buvo sutelktas į tai, ar architektūra kontroliuoja API usage per:

```text
bounded current-turn context
bounded detector input
bounded answer input
single-instance protection
pure-silence STT skipping
one combined translation + answer LLM request
```

Tikslas nebuvo sumažinti kiekvieną įmanomą request, o pašalinti tuos requests ir context growth, kurie neturi praktinės vertės.

## Manual Usability Observation

Kai kuriuos sistemos aspektus naudingiau vertinti interaktyviai nei vien isolated automated tests.

Manual runtime observation buvo naudojamas įvertinti:

```text
perceived response delay
GUI responsiveness
question-history behaviour
whether incoming audio appeared to queue excessively
whether question detection felt natural
shutdown behaviour
```

Final test etape response latency buvo laikomas priimtinu prototipui.

Papildoma latency instrumentation nebuvo pridėta, nes nebuvo pastebėta queue backlog ar user-facing performance problemos, dėl kurios reikėtų gilesnio matavimo.

## Testing principai

Testing sprendimai rėmėsi keliais praktiniais principais.

### Testuoti realų pipeline

Mock tests naudingi isolated state logic, tačiau audio, network, timing ir threading behavior taip pat reikia realaus runtime testing.

### Po architektūrinių pakeitimų atlikti regression testing

Tokie pakeitimai kaip silence-based STT skipping gali paveikti iš pažiūros nesusijusią logiką.

Todėl processing pipeline pakeitimai turi būti tikrinami iš naujo per visą question-detection flow.

### Neoptimizuoti be realaus pagrindo

Vien tai, kad optimization techniškai įmanoma, nebuvo laikoma pakankama priežastimi keisti stabilų behavior.

Pakeitimai buvo prioritetizuojami tada, kai testing atskleidė konkrečią problemą.

### Išsaugoti jau veikiantį behavior

Cleanup ir refactoring neturi tyliai pakeisti question detection, history behavior arba asynchronous processing.

### Cost yra reliability dalis

Netikėtas API consumption nėra tik finansinė problema.

Jis gali rodyti nekontroliuojamą programos behavior, todėl turi būti vertinamas kaip engineering reliability problema.

## Dabartinė testų būsena

Dabartinis Windows prototipas sėkmingai praėjo end-to-end regression, apimantį pagrindinį workflow.

Patikrintos sritys:

```text
Windows system-audio capture
speech-to-text
chunk overlap
transcript merge
silence detection
semantic question detection
multi-part question handling
question snapshots
question IDs
context-aware answer generation
translation
asynchronous workers
GUI history
navigation
silence-call optimization
single-instance protection
shutdown
```

Microphone capture buvo ištestuotas atskirai, tačiau sąmoningai lieka už pagrindinio application workflow ribų.

## Likusios Testing galimybės

Tolimesnis development gali pagrįsti papildomą testing tokiose srityse:

- ilgesnės continuous runtime sesijos;
- skirtingi Windows audio devices;
- skirtingi microphones, jei microphone support bus integruotas;
- įvairios background-noise sąlygos;
- skirtingi kalbėjimo stiliai ir akcentai;
- network interruption handling;
- API timeout ir retry behavior;
- packaged standalone executable behavior;
- non-technical user testing;
- performance measurements skirtingoje hardware.

Šie testai nėra būtini dabartiniam prototipui parodyti, tačiau taptų vis svarbesni, jei projektas judėtų nuo techninio prototipo link platesnio platinimo.

## Santrauka

Testing vystėsi kartu su pačia programa.

Projektas perėjo nuo izoliuotų komponentų tikrinimo iki pilnų real-time regression scenarijų, apimančių:

```text
audio
→ transcription
→ state
→ semantic detection
→ asynchronous generation
→ GUI
→ shutdown
```

Galutiniai testai patvirtino, kad dabartinė architektūra gali nuosekliai apdoroti kelis žodžiu pateiktus klausimus, išlikti responsive, teisingai palaikyti question history, vengti nereikalingų silence-related API calls ir švariai užsidaryti.
