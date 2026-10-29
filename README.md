## Tab-Autovervollständigung für Linux-Shells automatisch generieren

Tab-Autovervollständigung gehört heute zum Standard. crazy-complete erzeugt die Skripte für Bash, Fish und Zsh automatisch.

## Warum gute Autovervollständigung sich lohnt

Wer regelmäßig auf der Kommandozeile arbeitet, nutzt sie ganz selbstverständlich: die Tab-Autovervollständigung.
Ein Druck auf die Tabulatortaste genügt, um Dateinamen, Optionen, Unterbefehle oder andere Eingaben automatisch zu ergänzen.

Das spart Zeit, reduziert Tippfehler und macht selbst komplexe Programme deutlich einfacher bedienbar.
Besonders bei Werkzeugen mit vielen Optionen oder einer verschachtelten Unterbefehlsstruktur lohnt sich eine gute Autovervollständigung: Anwender müssen sich deutlich weniger Optionen merken und entdecken häufig Funktionen, die ihnen sonst verborgen geblieben wären.

Für Entwickler bedeutet das allerdings zusätzlichen Aufwand. Während die Kommandozeilenschnittstelle nur einmal implementiert wird, muss die Autovervollständigung häufig separat gepflegt werden.

## Das Problem: Drei Shells, drei unterschiedliche Completion-Systeme

Bash, Fish und Zsh unterstützen zwar alle Tab-Autovervollständigung, verfolgen dabei aber unterschiedliche Konzepte. Jede Shell besitzt ihre eigene Syntax, ihre eigenen APIs und unterschiedliche Fähigkeiten.

Wer alle drei Shells unterstützen möchte, muss entweder mehrere Completion-Skripte pflegen oder sich auf die Möglichkeiten einer einzelnen Shell beschränken. Änderungen an der Kommandozeile müssen anschließend an mehreren Stellen nachgezogen werden. Das manuelle Schreiben von Completion-Skripten erfordert außerdem Kenntnisse der jeweiligen Shell – und selbst dann ist man vor Fallstricken nicht gefeit.

Dazu kommt, dass es schwer ist, drei separate Skripte so zu schreiben, dass sie sich exakt gleich verhalten. Zum Verhalten gehören auch Options-Regeln – etwa versteckte oder sich ausschließende Optionen. Jede dieser Regeln muss in Bash, Fish und Zsh dieselbe Wirkung erzielen, ansonsten verhält sich dieselbe Eingabe je nach Shell unterschiedlich.

## Die Lösung: Autovervollständigung generieren lassen

crazy-complete verfolgt einen anderen Ansatz. Statt Shell-spezifische Skripte von Hand zu schreiben, wird die Kommandozeilenschnittstelle nur einmal deklarativ definiert. Aus dieser Definition erzeugt das Programm Completion-Skripte für Bash, Fish und Zsh.

Das Definitionsgerüst muss nicht von Hand beginnen: Aus dem Hilfetext des Programms erzeugt crazy-complete eine erste Definition, auf der die Vervollständigung aufbaut:

```sh
grep --help | crazy-complete --input-type=help yaml /dev/stdin -o grep.yaml
```

Eine Definition enthält beispielsweise:

- Optionen und ihre Parameter
- Positionsargumente
- Unterkommandos (beliebig verschachtelt)
- sich ausschließende und wiederholbare Optionen
- versteckte und finale Optionen
- Regeln, wann Optionen aktiviert werden
- Das Erfassen von Optionswerten für kontextabhängige Vervollständigungen

crazy-complete bringt bereits 43 eingebaute Vervollständiger mit – von Datei- und Verzeichnisnamen, Benutzer- und Gruppennamen, Prozessnamen und Prozess-IDs über Signalnamen, Hostnamen, IP-Adressen und Netzwerkschnittstellen bis zu Zeitzonen und Datumswerten. Meta-Vervollständiger kombinieren die eingebauten Vervollständiger miteinander und erzeugen daraus beispielsweise kommagetrennte Listen, wie sie die `-o`-Option im Beispiel unten benötigt. Für speziellere Anwendungsfälle können außerdem eigene Vervollständiger geschrieben werden, die während der Generierung eingebunden werden.

## Ein Beispiel

Die folgende Datei (`example.yaml`) definiert die Vervollständigung für das `example`-Kommando.

```yaml
# Definition des Hauptprogrammes "example"
prog: 'example'
options:
  - option_strings: ['-h', '--help']
    help: 'Show help and exit'
    final: true # Nach dieser Option werden keine weiteren angeboten

---

# Definition des Unterkommandos "example start"
prog: 'example start'
aliases: ['launch'] # Erlaube auch "example launch"
help: 'Start a program'
options:
  - option_strings: ['-v', '--verbose']
    help: 'Increase output verbosity'
    repeatable: true # Diese Option kann mehrfach verwendet werden

  - option_strings: ['--deprecated-option']
    hidden: true # Versteckte Option, da sie "deprecated" (veraltet) ist

  - option_strings: ['--preserve-env']
    help: 'List of environment variables to keep'
    complete: ['list', ['environment']]
    # Komma-separierte Liste von Umgebungsvariablen

  - option_strings: ['--priority']
    help: 'Set priority for process'
    complete: ['choices', ['low', 'normal', 'high', 'realtime']]
    nosort: true # Die Kandidaten sollen nicht alphabetisch sortiert werden

  # --foreground und --background schließen sich gegenseitig aus (-> groups)
  - option_strings: ['--foreground']
    help: 'Start process in foreground'
    groups: ['start_mode']

  - option_strings: ['--background']
    help: 'Start process in background'
    groups: ['start_mode']

  - option_strings: ['--pid-file']
    help: 'Write PID to file'
    complete: ['file']
    when: 'has_option --background'
    # Option soll nur angeboten werden, wenn --background angegeben wurde

  - option_strings: ['-o', '--options']
    help: 'Specify options'
    complete: 'complete_options' # Siehe Abschnitt %defines%

positionals:
  # Erstes Argument ist ein Befehl ("example launch ls ...")
  - number: 1
    complete: ['command']

  # Weitere Argumente sind Optionen für den Befehl ("example launch ls -a ...")
  - number: 2
    repeatable: true
    complete: ['command_arg']

---

# Definition des Unterkommandos "example view-log"
prog: 'example view-log'
help: 'View log file'
options:
  - option_strings: ['--pager']
    complete: ['command']
    optional_arg: true # Das Argument für --pager ist optional

---

# Der %defines% Abschnitt erlaubt das definieren von Konstanten, die in
# weiteren Abschnitten genutzt werden können.
prog: '%defines%'

complete_options: ['key_value_list', ',', '=', [
  ['lang',   'set language',  ['locale']],
  ['user',   'set user',      ['user']],
  ['group',  'set group',     ['group']],
]]
```

Die Completion-Skripte werden wie folgt generiert:

```sh
crazy-complete bash example.yaml -o example.bash
crazy-complete fish example.yaml -o example.fish
crazy-complete zsh example.yaml -o example.zsh
```

Die Autovervollständigung sieht in den Shells dann so aus:

**Bash**:

```
> example
launch    start     view-log

> example launch --
--background  --foreground  --options=  --preserve-env=  --priority=  --verbose

> example launch --priority
low  normal  high  realtime

> example launch --preserve-env HOME,
DISPLAY  LANG  LOGNAME  MAIL  OLDPWD  PATH  PWD  SHELL  SHLVL  TERM  USER

> example launch --options
group  lang  user

> example launch --options group=nobody,user=
bin  daemon  man  nobody  root  sshd

```

**Fish**:

```
> example
launch  (Start a program)  start  (Start a program)  view-log  (View log file)

> example launch --
--background    (Start process in background)
--foreground    (Start process in foreground)
--options       (Specify options)
…and 3 more rows

> example launch --priority
low     (Set priority for process)  high      (Set priority for process)
normal  (Set priority for process)  realtime  (Set priority for process)

> example launch --preserve-env HOME,
HOME,DISPLAY  HOME,LOGNAME  HOME,OLDPWD  HOME,PWD    HOME,SHLVL  HOME,USER
HOME,LANG     HOME,MAIL     HOME,PATH    HOME,SHELL  HOME,TERM

> example launch --options
group=  (set group)  lang=  (set language)  user=  (set user)

> example launch --options group=nobody,user=
…ody,user=bin      …ody,user=man      …ody,user=root
…ody,user=daemon   …ody,user=nobody   …ody,user=sshd

```

**Zsh**:

```
> example
launch    start  -- Start a program
view-log         -- View log file

> example launch --
--background    -- Start process in background
--foreground    -- Start process in foreground
--options       -- Specify options
--preserve-env  -- List of environment variables to keep
--priority      -- Set priority for process
--verbose       -- Increase output verbosity

> example launch --priority
low       normal    high      realtime

> example launch --preserve-env HOME,
DISPLAY  LANG  LOGNAME  MAIL  OLDPWD  PATH  PWD  SHELL  SHLVL  TERM  USER

> example launch --options
group  -- set group
lang   -- set language
user   -- set user

> example launch --options group=nobody,user=
bin  daemon  man  nobody  root  sshd

```

## Die Stärken von crazy-complete

Das Projekt legt Wert auf effiziente Skripte. Dabei wird Code für Features nur eingebunden, wenn diese tatsächlich genutzt werden – das generierte Skript bleibt so möglichst schlank.

Beim Verfassen der Definitionsdateien folgt crazy-complete ebenfalls nicht dem Prinzip „Garbage in, Garbage out": Fehlerhafte Definitionen werden frühzeitig erkannt und mit sprechenden Fehlermeldungen samt Zeilennummern ausgegeben, sodass die Ursache schnell gefunden wird.

Die Funktionsweise ist darüber hinaus umfassend abgesichert: Über 800 Tests stellen sicher, dass die generierten Skripte in Bash, Fish und Zsh die korrekte Vervollständigung liefern.

## Der Vergleich mit anderen Generatoren

Es existieren weitere Completion-Generatoren.

**Carapace** ist leistungsfähig und deckt alle drei Shells ab. Es setzt allerdings voraus, dass der Generator auf dem Zielsystem installiert ist – die erzeugten Skripte sind nur noch Wrapper, die carapace zur Laufzeit aufrufen. Für Systeme, auf denen man die Abhängigkeit nicht mitverteilen will, ist das eine Hürde.

**Complgen** erzeugt ebenfalls standalone-Skripte für alle drei Shells. Seine Definitionssprache basiert auf EBNF (ähnlich yacc/bison). Der generierte Shell-Code ist eine Finite-State-Maschine: Sie läuft langsamer, nutzt die nativen Completion-Features der Shells nicht und bietet keine Möglichkeit, sich ausschließende, versteckte oder bedingte Optionen zu modellieren oder Optionswerte für nachfolgende Vervollständigungen abzugreifen.

## Über den Autor

Benjamin Abendroth ist Entwickler und Linux-Enthusiast.

## Hintergrundinformationen (nicht Bestandteil des Artikels)

Ein paar Hintergrundinformationen zu crazy-complete, die bei der Einordnung des Projekts hilfreich sein könnten:

- Das Ziel ist es, möglichst viele Programme zu unterstützen.
  Es gibt nur wenige Ausnahmen, in denen crazy-complete aufgrund der Syntax nicht einsetzbar ist – etwa `find` mit seiner `find -exec {} \;`-Syntax.

- Die generierten Completion-Skripte sind standalone: Auf dem Zielsystem muss crazy-complete nicht installiert sein.
  Für die Generierung genügen Python 3 und PyYAML.

- Für jede unterstützte Shell (Bash, Fish und Zsh) existieren rund 200 "positive" Tests.
  Sie stellen sicher, dass die generierten Completion-Skripte bei der Vervollständigung die gültigen Ausgaben liefern.

- Zusätzlich gibt es etwa 230 "negative" Tests.
  Sie überprüfen, dass fehlerhafte Eingaben frühzeitig erkannt und aussagekräftige Fehlermeldungen ausgegeben werden.
  Das Werkzeug folgt bewusst nicht dem Prinzip "Garbage in, garbage out", sondern versucht, fehlerhafte Spezifikationen möglichst früh und präzise zu diagnostizieren.
  Fehlermeldungen enthalten immer Zeilenangaben.

- Das Projekt folgt dem Grundsatz "You only pay for what you use". Funktionen werden nur eingebunden, wenn sie auch gebraucht werden.
  Auch der generierte Code richtet sich danach, welche Features genutzt werden.

- Das Projekt orientiert sich stark an dem durchdachten Completion-System der Zsh. Insbesondere für Zsh werden die kompaktesten Completion-Dateien generiert.

- Die Entwicklung erfolgte vollständig ohne KI-generierten Quellcode.
  Sämtlicher Code sowie die Architektur und die Testfälle wurden manuell erstellt.

- Es ist lokalisiert: Aktuell wird neben Englisch nur Deutsch unterstützt.
