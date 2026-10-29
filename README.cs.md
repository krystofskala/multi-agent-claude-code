# Osobní systém AI agentů nad Claude Code (česky, zkráceně)

Tým dlouho běžících sessions Claude Code: dělí si práci podle rolí, domlouvají se v jednom kanálu na Discordu, veškerý stav drží v obyčejných souborech a dny běží bez dozoru na dvou strojích.

- **Skutečný provoz, ne ukázka.** Spravuje moje vedlejší projekty i systém samotný: nástroje, skripty, revize, rešerše, noční úlohy.
- **Pod tím obyčejný Claude Code.** Žádný vlastní agentní framework. Každý agent je normální `claude` CLI s popisem role, pracovní složkou a vlastní pamětí.
- **Jediný zdroj pravdy jsou soubory.** Úkoly, stavy, rozhodnutí a měření jsou v Markdown trezoru, zpráva v kanálu agenta jen probudí.
- **Sám se zvedne.** Spadlá session naváže na stejnou konverzaci, plánované úlohy pouštějí systemd časovače, každá změna má zálohu a sepsaný postup, jak ji vrátit.
- **Měřený.** Každý den se zapisuje spotřeba tokenů po agentech a spouštěčích, režie koordinace a otázky, na které nikdo neodpověděl.

## Architektura

- **Orchestrátor** (1): priority, rozdělení práce mezi leady, třídění inboxu, ranní seznam rozhodnutí. Sám nepracuje.
- **Systémový chat** (1 na stroj): zakládá, restartuje a zavírá stálé sessions, vlastní systémové nástroje.
- **Leadi projektů**: vedou projekt, pouštějí krátké pracovní sessions a jako jediní píšou „ověřeno“.
- **Pracovní sessions**: čerstvá session, jeden soubor úkolu, výsledek, konec.
- **Kontrolní chaty** (na zavolání): logika, kód, design, ladění, poznámky. Krátký běh `claude -p` s rolí, jen hlásí nálezy, nic sám neopravuje.

**Člověk v rozhodování:** v samotném běhu nejsem, ale určuju směr: nová část systému potřebuje moje ok na návrh na 1 stránku a otevřená rozhodnutí čekají v ranním seznamu jako otázky A/B, které odpovím jednou reakcí. Agenti mezitím dělají jinou část.

**Stroje:** Linuxový server (Fedora, VM), sessions v tmuxu ve smyčce, která je po pádu znovu spustí. Windows notebook, sessions ve Windows Terminalu přes PowerShell, s cílenými pravidly povolení místo plného bypassu.

**Společný kanál:** agenti píšou přes webhook pod svým jménem a ikonou a čtou přes oficiální Discord plugin pro Claude Code. Nad pluginem je moje vrstva směrování jako opakovatelná úprava, která se nahodí při každém startu, takže ji aktualizace pluginu nesmaže (cena: závisí na podobě kódu pluginu, po aktualizaci se musí zkontrolovat). Pravidlo: budí se jen adresát – odpověď budí jen autora zprávy, `@Jméno` od jiného agenta budí jen jeho, zprávy z webhooku jsou označené jako od agenta (nikdy ode mě), moje reakce (🅰️/🅱️/👍) plugin doručí rovnou agentovi i po restartu a připomínka „otázka čeká 15 min, pokračuj jinou částí“ jde přímo agentovi.

**Stav:** soubory úkolů podle jedné šablony (účel, rozsah, „hotovo =“, co nedělat, výsledek), `stav.md` projektu, inbox a ranní seznam rozhodnutí. Kód ve vlastních git větvích / worktree. Složka poznámek se místo gitu synchronizuje mezi stroji a telefonem s historií verzí (zapisuje do ní víc agentů naráz a telefon potřebuje obyčejné soubory), každá úprava má navíc zálohu `.bak`.

**Dál:** zhruba 20 systemd časovačů (hlídky, filtr, který pustí kontrolu jen při skutečné změně plánu nebo stavu, max 5×/h, noční fronta, měření, indexace, zálohy); měření tokenů jen z čísel v záznamech Claude Code; Gemini jako druhý názor (posílá se jen posuzovaný text, výstup se vždy ověří); lokální hledání podle významu přes qmd jako pilot (nic neodchází ven, soubory s tajemstvím se vyřadí).

## Co jsem se naučil

- **Schvalovací vrstvy škodily.** Stupně oprávnění, deníky schválení a tlačítka hlavně brzdily. Zrušil jsem je, zůstaly zálohy, postupy vrácení a revize před nasazením.
- **Koordinace je drahá.** V měřený den: práce 44 %, koordinace 32 %, kontroly 24 % vážených tokenů. Nejvíc ušetřily krátké kontrolní sessions a doručování jen adresátovi.
- **Co žije jen v konverzaci, se po restartu ztratí.** Hlídač reakcí v session orchestrátora po restartu potichu přestal běžet a rozhodnutí nikdo neviděl. Přesunul jsem ho do pluginu s doháněním.
- **Připomínky mně nefungovaly.** Z 65 za deset dní jsem reagoval na 8. Teď jdou agentovi, ať pokračuje jinou částí.
- **Hledání podle významu grep doplňuje, nenahrazuje.** Ve zkoušce na 10 otázkách: qmd 3/10, grep 7/10. qmd ale našel 2, které grep ne, a za ~750 tokenů místo až ~20 000.
- **Soubory vítězí nad chatem.** Co bylo jen v chatu, se dřív nebo později ztratilo.

## Životní cyklus úkolu

Soubor úkolu → `přijato: <ID>` → kontrola plánu (malá změna = jedna kontrola) → záloha, test nebo zkouška na kopii, postup „Vrátit“ → revize kódu → nasazeno → „ověřeno“ (jen lead, po skutečné kontrole) → jeden řádek `hotovo` orchestrátorovi. Proti tichým chybám modelu tedy chrání testy, zkoušky nanečisto a nezávislá kontrola čerstvou session, která čte skutečný diff; automatické evaly výstupů modelu zatím nemám. Nová část systému: rešerše → návrh na 1 stránku → moje ok → pilot → týden měření.

## Spolehlivost

Restart po pádu na stejnou konverzaci, 3 pády za 10 min = stop + push; po restartu agent dočte zmeškané zprávy. Datované zálohy `.bak` a postup vrácení u každého úkolu. Noční běhy: večer max 3 bezpečné úkoly, ráno souhrn, zrušit jde jednou reakcí. Denní měření místo dojmů; pro představu jeden měřený den: 7 ověřených úkolů, 5× jsem musel čekat.

Kód je neveřejný, rád ho projdu na callu.
