# agenttrace — handoff для внешнего проекта

Ты (агент другого проекта) хочешь сжать историю работы у себя в проекте до коротких
трейсов с сохранением reasoning. Вот что нужно знать.

## Что это и где

`agenttrace` — CLI, который сжимает логи сессий кодинг-агентов (Claude Code, Codex CLI,
pi, Cursor IDE/agent, qwen, kimi, minimax/mcode) в короткие Markdown-трейсы: сохраняет
мысли/гипотезы/развороты агента, выбрасывает bulk (вывод инструментов, ретраи, шаблонный
шум). Публичная копия: `github.com/minmax/agenttrace`. Локальная рабочая копия:
**`/Users/pavel.karataev/ai/agenttrace`** (Node ≥ 22.19, `npm install`, без сборки — всё
через tsx, `npm run sd -- <command>`).

## Как получить трейсы своего проекта (2 шага, без LLM-ключей)

Тебе нужен режим **work-сервера** — программа сама НЕ зовёт LLM, сжатие делаешь ты
(агентом), программа выдаёт работу пачками и машинно валидирует ответы.

### 1. Инициализировать state и собрать очередь

```bash
cd /Users/pavel.karataev/ai/agenttrace
npm run sd -- work init --state /tmp/my-traces \
  --claude-root ~/.claude/projects --qwen-root ~/.qwen/projects   # корни под твои источники
```

- `--state <dir>` — каталог кампании (registry, скелеты, очередь, сайдкары, трейсы, банк).
- Корни: `--claude-root/--codex-root/--pi-root/--qwen-root/--kimi-code-sessions/
  --kimi-sessions/--minimax-root` (multiple). Без флага `--no-cursor` подхватится и Cursor.
- Программа найдёт сессии по cwd внутри логов (только твой проект и поддеревья), построит
  детерминированные скелеты и очередь окон (~20k ток каждое). Порядок — newest-first.

### 2. Работать по циклу claim → сжать → submit

```bash
# взять одну пачку (одно окно pass1 одной сессии)
npm run sd -- work claim --state /tmp/my-traces --layer pass1 --worker my-name > /tmp/job.json
# …прочитать ходы из /tmp/job.json (поле job.turns[].text), сжать…
# собрать ответ {"blocks":[…],"digest":{…}} и сдать по stdin:
npm run sd -- work submit --state /tmp/my-traces <jobId> < /tmp/answer.json
# pass2 (арки рассуждения + вердикт + банк) откроется по сессии, когда ВСЕ её окна сжаты:
npm run sd -- work claim --state /tmp/my-traces --layer pass2 --worker my-name
```

Отказ (`ok:false` + `errors[]`) = машинно-исправимый список: чини указанное и пересдавай
(≤3 попыток, затем `work release <jobId>`). Статус/финализация:

```bash
npm run sd -- work status  --state /tmp/my-traces   # JSON: очередь, done, лизы
npm run sd -- work finalize --state /tmp/my-traces  # трейсы + банк уроков + метрики
```

Результат: `traces/<project>/*.md` (приватные, с `@L<line>` якорями в исходный лог) и
`traces-repo/traces/<project>/*.md` (repo-bound: без якорей и приватных путей — можно
коммитить в публичную репу), банк в `bank/INDEX.md`.

## Контракт pass1 (что сдаётся в submit)

Один JSON на окно:
```json
{"blocks": [{"turnIndex": N, "anchor": {"fromLine": F, "toLine": T},
             "action": "≤240 симв — что делалось на ходе",
             "thoughts": [{"kind": "H|ALT|?|PIVOT|INSIGHT|ERR-R",
                           "source": "thinking|narration", "text": "≤320 симв суть",
                           "q": "<id из [q...] метки хода>"}]}],
 "digest": {"goal": "≤200", "currentBelief": "≤300", "openHypotheses": []}}
```
Правила, которые валидируются: anchor — точное покрытие ходов окна; склейка подряд
идущих незначимых ходов разрешена (anchor покрывает диапазон, внутри не должно быть
thinking); ход с [THINKING] ⇒ thoughts непустой (это главный носитель reasoning);
факты (тесты, exit-коды) сверяются с машинными — расхождение = dispute. Весь текст —
третье лицо, намерения, не дословные цитаты; language трейса — английский.

pass2 сдаёт arcs (H/PIVOT/ERR-R со статусами confirmed/refuted/open), verdict по
машинным фактам и ≤3 переносимых уроков в банк (strategy/guardrail).

## Если нужен готовый промпт воркера

Канонические шаблоны: `~/sd-run/prompts/pass1-worker.md` и `pass2-worker.md` (подставь
`{WORKER}` и `{MAX_BATCHES}`). Полную кампанию (реестр 34k+ сессий) живёт `~/sd-run`
— не пиши туда из чужого проекта, используй свой `--state`.

## Границы

- `distill`/`refine` (скрипт сам зовёт LLM через OpenAI-compatible endpoint) — альтернатива,
  когда у тебя есть ключ: `npm run sd -- refine <project-dir> --out traces --base-url …`.
- Оригинальные логи НЕ коммитятся: repo-bound flavor существует именно для этого.
- Детальная архитектура: `ARCHITECTURE.md` и `AGENTS.md` в репо.
