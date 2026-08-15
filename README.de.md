# DeepSeek-V4-Flash auf Ryzen AI Max+ 395: technisch erfolgreich gestartet – und trotzdem wieder entfernt

[English](README.md) | **Deutsch**

Versuch vom 14. August 2026  
Rekonstruiert am 15. August 2026 aus den erhaltenen Serverlogs, API-Antworten, Request-Mitschnitten und Benchmark-Ausgaben.

> Historischer Bericht: Dieses Setup wurde vollständig entfernt. Vom Versuch befinden sich weder Modelldateien noch ein Serverprozess oder eine OpenCode-Anbindung auf dem Rechner. Der Bericht enthält keine Benutzernamen, privaten Pfade oder Zugangsdaten.

## Kurzfassung

Ich habe das vollständige **DeepSeek-V4-Flash mit 284,33 Milliarden Parametern** lokal auf einem GMKtec EVO-X2 mit Ryzen AI Max+ 395, Radeon 8060S und 128 GB Unified Memory betrieben. Das ausgewählte Unsloth-GGUF `UD-IQ2_M` belegte in drei Shards 84,68 GiB und lief mit vollständigem GPU-Offload über einen angepassten llama.cpp-Build für ROCm/ROCDXG.

Das Ergebnis war technisch echt:

- **11,36 Generations-tok/s** in einem kurzen API-Test
- **8,66 Generations-tok/s** nach einem Prompt mit 10.415 Tokens
- korrekte Metadaten über `/v1/models`
- Multi-Turn-Chat, Toolauswahl, aufeinanderfolgende Tools und Fehlerbehandlung bestanden
- OpenCode wurde beim Senden von `reasoning_effort=max`, `temperature=1.0` und `top_p=0.95` mitgeschnitten
- das eingebettete DeepSeek-Jinja-/DSML-Tool-Template wurde tatsächlich verwendet

Als alltäglicher Coding-Agent überzeugte das Setup auf diesem Rechner trotzdem nicht. Das Modell belegte ungefähr 88,33 GiB im GPU-/Unified-Memory-Pool; für Windows, Browser und OpenCode blieben ungefähr 24,89 GiB. Eine größere Agentenanfrage mit etwa 8.000 Prompt-Tokens brauchte allein für die Prompt-Auswertung ungefähr 162,5 Sekunden. Der vollständige Lauf OpenCode → Modell mit 48 angebotenen Tools wurde während dieser minutenlangen Vorverarbeitung abgebrochen. Die Browser-/MCP-Kette wurde danach direkt geprüft – nicht nachträglich als erfolgreicher kompletter Modell-Agentenlauf dargestellt.

Das ehrliche Fazit lautet deshalb: **Das vollständige Modell lief und seine API-/Tool-Grundfunktionen funktionierten. Das gesamte lokale Agentenerlebnis war mir jedoch zu langsam, speicherhungrig und unzuverlässig. Ich habe den Aufbau entfernt und ungefähr 90 GB freigegeben.**

## Testsystem

- Mini-PC: GMKtec EVO-X2
- APU: AMD Ryzen AI Max+ 395
- GPU: Radeon 8060S, `gfx1151`
- Arbeitsspeicher: 128 GB Unified Memory
- Host: Windows 11
- Laufzeitumgebung: WSL Ubuntu
- GPU-Backend: ROCm/HIP 7.2.4 über AMD ROCDXG
- Offload: alle Modell-Layer auf das GPU-Backend

Zusätzlich wurde ein Vulkan-Build von llama.cpp geprüft. Unter WSL war dort jedoch nur `llvmpipe` sichtbar, nicht die Radeon-GPU. In genau diesem WSL-Aufbau war ROCm/ROCDXG deshalb das brauchbare Backend.

## Welches Modell tatsächlich lief

Der gewünschte API-Name lautete `deepseek-v4-flash-0731-local`, war aber nur ein lokaler Alias. Für den Versuch wurde das verfügbare Modell-Repository verwendet:

- GGUF-Repository: `unsloth/DeepSeek-V4-Flash-GGUF`
- Basismodell: `deepseek-ai/DeepSeek-V4-Flash`
- `general.name`: `Deepseek-V4-Flash`
- `general.basename`: `Deepseek-V4-Flash`
- `general.base_model.0.name`: `DeepSeek V4 Flash`
- `general.architecture`: `deepseek4`
- Größenbezeichnung: `256x8.4B`
- von llama.cpp gemeldete Parameter: **284.334.567.511**
- Quantisierung: Unsloth **UD-IQ2_M**, zur Laufzeit gemeldet als `IQ2_M - 2.7 bpw`
- GGUF `general.file_type`: `29`

Geladen wurden diese drei Shards:

| Datei | Bytes |
|---|---:|
| `DeepSeek-V4-Flash-UD-IQ2_M-00001-of-00003.gguf` | 5.256.864 |
| `DeepSeek-V4-Flash-UD-IQ2_M-00002-of-00003.gguf` | 49.956.780.160 |
| `DeepSeek-V4-Flash-UD-IQ2_M-00003-of-00003.gguf` | 40.964.890.464 |
| **Gesamt** | **90.926.927.488 Bytes / 84,68 GiB** |

Der sehr kleine erste Shard enthielt überwiegend Metadaten. Der vollständige Satz mit drei Teilen wurde geladen (`split.count=3`).

## llama.cpp-Runtime

- Versionsangabe: `0.1.0-dev`
- Build: `1`
- API-Build-ID: `b1-4c1a0af`
- Commit: [`4c1a0af40d88c7fbb3b15c85bf2e8016d1d5b64c`](https://github.com/ggml-org/llama.cpp/commit/4c1a0af40d88c7fbb3b15c85bf2e8016d1d5b64c)
- Commit-Titel: `llama : allow virtual igpu devices (#26953)`

Die wesentliche Serverkonfiguration war:

```text
--model /models/DeepSeek-V4-Flash-UD-IQ2_M-00001-of-00003.gguf
--alias deepseek-v4-flash-0731-local
--host 127.0.0.1 --port 8080
--ctx-size 65536 --parallel 1
--gpu-layers all --flash-attn on
--cache-type-k q4_0 --cache-type-v q4_0
--batch-size 512 --ubatch-size 128
--threads 16 --threads-batch 32
--load-mode dio
--jinja --reasoning-format deepseek
--cache-prompt --metrics --slots
--timeout 3600 --threads-http 4
```

Aktiv waren 65.536 Kontext-Tokens. In den Modellmetadaten war ein nativer Trainingskontext von 1.048.576 Tokens angegeben.

## Gemessene Leistung

Dies sind erhaltene Messwerte aus llama.cpp und der API, keine Schätzungen.

| Test | Prompt | Ausgabe | Prompt-Verarbeitung | Generation | Zeit |
|---|---:|---:|---:|---:|---:|
| Kurzer Reasoning-/API-Test | 391 Tokens | 124 Tokens | 74,37 tok/s | **11,36 tok/s** | 5,37 s TTFT; 16,08 s gesamt |
| Long-Context-Retrieval | 10.415 Tokens | 155 Tokens | 52,85 tok/s | **8,66 tok/s** | 197,05 s Prompt; 17,79 s Generation; 214,84 s gesamt |

Im Long-Context-Test wurden die beiden versteckten Marker `NORTHSTAR-173` und `SOUTHPORT-741` korrekt wiedergegeben. In einem ersten Versuch war die Antwort vor dem vollständigen Marker abgeschnitten worden; der Wiederholungstest bestand.

Eine realistischere, toolreiche Anfrage enthielt ungefähr 8.023 Prompt-Tokens. Allein die Prompt-Auswertung dauerte ungefähr **162,51 Sekunden**, entsprechend **49,37 tok/s**, bevor das Modell überhaupt antworten konnte. Diese Kaltstart-Latenz war im Alltag wichtiger als die reine Generationsrate.

## Speicherbedarf

- llama.cpp GPU-/Unified-Memory-Pool: ungefähr **90.453 MiB / 88,33 GiB**
- insgesamt während des geladenen Modells belegter Systemspeicher: ungefähr **98,76 GiB**
- verbleibender Speicher für Windows, OpenCode, Browser und weitere Tools: ungefähr **24,89 GiB**

Der Serverprozess selbst zeigte nur ungefähr 1,45 GiB RSS, weil der größte Teil des Modells im GPU-/Shared-Memory-Pool lag. RSS allein bildete den tatsächlichen Speicherbedarf deshalb nicht sinnvoll ab.

## Antwort von `/v1/models`

Die erhaltene Antwort identifizierte den lokalen Alias und meldete:

```json
{
  "id": "deepseek-v4-flash-0731-local",
  "owned_by": "llamacpp",
  "meta": {
    "vocab_type": 2,
    "n_vocab": 129280,
    "n_ctx": 65536,
    "n_ctx_train": 1048576,
    "n_embd": 4096,
    "n_params": 284334567511,
    "size": 90921582940,
    "ftype": "IQ2_M - 2.7 bpw"
  }
}
```

## Chat-Template, Reasoning und Tool-Encoding

Der Server verwendete das von Unsloth im GGUF eingebettete Jinja-Template und keinen allgemeinen ChatML-Ersatz:

- Tokenizer-Modell: `gpt2`
- Pre-Tokenizer: `joyai-llm`
- llama.cpp-Chatformat: `peg-native`
- Reasoning-Format: `deepseek`
- Generationspräfix: `<｜Assistant｜><think>`
- Rollenmarker: `<｜User｜>` und `<｜Assistant｜>`
- Tool-Call-Encoding: `｜DSML｜tool_calls`, `｜DSML｜invoke`, `｜DSML｜parameter`
- BOS: `<｜begin▁of▁sentence｜>`
- EOS: `<｜end▁of▁sentence｜>`

Das Template verarbeitete parallele Tool-Aufrufe und Objektargumente. Folgende API-Tests bestanden:

- Systemprompt und Multi-Turn-Chat
- Toolauswahl mit gültigen JSON-Argumenten
- aufeinanderfolgende Fortsetzung mit `find_project_files` und anschließend `read_file`
- Ausweichstrategie nach einem absichtlich fehlgeschlagenen `lookup_ticket` über `search_ticket_archive`

## Was OpenCode tatsächlich sendete

Ein Mitschnitt der OpenCode-Anfrage – nicht die Selbstauskunft des Modells – zeigte:

```text
reasoning_effort = max
temperature      = 1.0
top_p            = 0.95
max_tokens       = 8192
```

Der llama.cpp-Slot bestätigte `reasoning_format=deepseek`; die Generation begann mit `<think>`.

Diese Unterscheidung ist wichtig, weil das Modell später im Chat behauptete, kein ausdrückliches Chain-of-Thought-/Reasoning-Verfahren zu verwenden. Diese Selbstauskunft war falsch: Request und Server-Mitschnitt belegen, dass MAX aktiv war.

Es gab außerdem eine wesentliche Konfigurationsgrenze. Die offizielle DeepSeek-Modellkarte empfiehlt für den lokalen Betrieb `temperature=1.0` und `top_p=1.0` sowie mindestens 384K Kontext für Think Max. Verwendet wurden wegen der OpenCode-Konfiguration und des praktisch verfügbaren Speichers `top_p=0.95` und nur 64K Kontext. Es handelte sich somit um eine echte MAX-Anfrage, aber nicht um die offiziell empfohlene Think-Max-Umgebung mit vollem Kontext.

## Browser- und Vision-Versuch: Was bestand – und was nicht

Im Anschluss wurden Playwright MCP 0.0.79 und ein kleines, getrenntes Ollama-Modell `qwen3-vl:4b` für die Screenshot-Analyse angebunden. Ein zunächst erwogenes Vision-Hilfsmodell mit 31B Parametern war für diesen Rechner unnötig groß und wurde nicht verwendet.

Direkte End-to-End-Prüfungen der angebundenen Werkzeuge bestanden:

- Seitennavigation und Screenshots
- lokale Screenshot-Analyse
- Interaktion mit Button und Slider
- DOM-Prüfung
- Netzwerkaufruf von `/health`
- Konsolenprüfung
- Desktop- und Mobile-Viewport
- visueller Vorher-/Nachher-Vergleich
- kein horizontaler Überlauf auf Mobile

Der entscheidende Unterschied lautet jedoch: **Der vollständige Lauf DeepSeek → OpenCode → 48 angebotene Tools → Browser wurde nicht beendet.** Nach mehreren Minuten Prompt-Vorverarbeitung wurde er abgebrochen. Anschließend wurden MCP- und Vision-Kette direkt geprüft. Das belegt die Funktion der Tool-Infrastruktur, aber keinen erfolgreich von DeepSeek über OpenCode ausgeführten vollständigen Browser-Agentenlauf.

## Warum ich alles wieder entfernt habe

Der Aufbau bestand die technischen Einzeltests, enttäuschte jedoch in der entscheidenden Nutzung:

1. Das vollständige Modell belegte den größten Teil des Unified Memory.
2. Die Kaltverarbeitung großer OpenCode-Toolschemas dauerte mehrere Minuten.
3. Erste/MAX-Antworten wirkten dadurch zeitweise wie ein Stillstand.
4. Das Modell beschrieb den eigenen aktiven Reasoning-Modus falsch.
5. Bei einer Three.js-Landingpage empfahl es, die Datei mit PowerShell in Teilen zu schreiben, weil sie angeblich „zu groß für einen einzelnen Schreibaufruf“ sei. Das war keine reale Plattformgrenze, sondern eine schlechte Agentenentscheidung. Eine Datei entstand zwar, der Ablauf war aber verwirrend und das Ergebnis nicht überzeugend.
6. Der vollständige OpenCode-Browserlauf mit 48 Tools wurde nicht End-to-End nachgewiesen.

Deshalb wurden Server und alle drei GGUF-Shards, der angepasste llama.cpp-Build, das WSL-Projektverzeichnis, die DeepSeek-Provider-/Agentenkonfiguration in OpenCode, die temporäre Playwright-/Vision-Anbindung und das kleine Vision-Hilfsmodell entfernt. Port 8080 wurde freigegeben und ungefähr 90 GB Speicher wurden zurückgewonnen. Unabhängige Projekte und bereits vorhandene Modelle blieben erhalten.

## Vergleich mit dem späteren Qwen3.8-27B-Versuch

Der spätere [Bericht zu Qwen3.8-27B auf Ryzen AI Max+ 395](https://github.com/erstmalreden/qwen3.8-27b-ryzen-ai-max-395-benchmarks) dokumentiert 16,68 tok/s bei 64K und 16,45 tok/s bei 128K mit einem Q5-Modell und wesentlich kleinerem Working Set.

Gegenüber 11,36 tok/s im kurzen DeepSeek-Test und 8,66 tok/s nach dem 10,4K-Prompt war Qwen im kurzen Vergleich ungefähr 1,47-mal und im Langprompt-Vergleich ungefähr 1,90-mal schneller. Das sind praktische Orientierungspunkte, kein vollständig vergleichbarer Modellbenchmark: Prompts, Quantisierungen, Runtimes und Speichermetriken unterschieden sich.

## Lehren für ähnliche Hardware

- Ein Strix-Halo-System mit 128 GB kann dieses 284B-Modell in einer IQ2-Quantisierung tatsächlich laden und ausführen.
- „Es lädt“ und „die API besteht Tooltests“ sind deutlich niedrigere Hürden als „es ist ein angenehmer täglicher Coding-Agent“.
- Nicht nur Generations-tok/s messen, sondern auch die kalte Vorverarbeitung großer Toolschemas.
- Reasoning-Parameter am ausgehenden Request und im Serverlog prüfen; das Modell nicht nach seiner eigenen Laufzeitkonfiguration fragen.
- Bei Unified Memory Windows-Prozess-RSS und GPU-/Shared-Memory-Pool getrennt bewerten.
- Das Vision-Hilfsmodell klein halten, wenn das Hauptmodell bereits fast den gesamten Speicher belegt.
- Einen direkten MCP-Test nicht als erfolgreichen modellgesteuerten Agententest bezeichnen.
- Auf diesem Rechner lieferte ein kleineres, besser quantisiertes Modell die deutlich bessere Gesamterfahrung.

## Primärquellen

- [Offizielle Modellkarte von DeepSeek-V4-Flash](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash)
- [Unsloth DeepSeek-V4-Flash GGUF-Repository](https://huggingface.co/unsloth/DeepSeek-V4-Flash-GGUF)
- [Unsloth-Diskussion zur llama.cpp-Prompt-Cache-Kompatibilität](https://huggingface.co/unsloth/DeepSeek-V4-Flash-GGUF/discussions/6)
- [AMD-Anleitung für ROCm/ROCDXG unter WSL](https://rocm.docs.amd.com/projects/radeon-ryzen/en/latest/docs/install/installryz/wsl/howto_wsl.html)
- [Im Versuch verwendeter llama.cpp-Commit](https://github.com/ggml-org/llama.cpp/commit/4c1a0af40d88c7fbb3b15c85bf2e8016d1d5b64c)

Alle Ergebnisse stammen von einem einzelnen Rechner und einem bestimmten Softwarestand. Es handelt sich um dokumentierte Beobachtungen, nicht um allgemeingültige Leistungsversprechen.
