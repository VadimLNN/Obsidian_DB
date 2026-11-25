---
tags:
  - version-control/git
  - career/interview
  - devops
aliases:
  - Git Interview Notes
  - Конспект Git
---

# Git — конспект для технических собеседований

> [!abstract] Что это
> Цель этого конспекта — не выучить команды как заклинания, а понять **модель Git**. Если понимаешь, где находятся данные, куда указывает `HEAD`, что лежит в `index` и какие refs двигает команда, большинство вопросов на собеседовании решаются логикой.

---

## Оглавление

1. [Ментальная модель Git](#1-ментальная-модель-git)
2. [Working tree, index и commit](#2-working-tree-index-и-commit)
3. [Diff, файлы, ignore и clean](#3-diff-файлы-ignore-и-clean)
4. [Commits, objects и неизменяемость](#4-commits-objects-и-неизменяемость)
5. [Ветки, HEAD и refs](#5-ветки-head-и-refs)
6. [Merge и конфликты](#6-merge-и-конфликты)
7. [Rebase и переписывание истории](#7-rebase-и-переписывание-истории)
8. [Reset, restore, revert и восстановление](#8-reset-restore-revert-и-восстановление)
9. [Remotes: fetch, pull и push](#9-remotes-fetch-pull-и-push)
10. [Cherry-pick, stash и tags](#10-cherry-pick-stash-и-tags)
11. [Просмотр истории, поиск и диагностика](#11-просмотр-истории-поиск-и-диагностика)
12. [Внутренности Git](#12-внутренности-git)
13. [Config, attributes и hooks](#13-config-attributes-и-hooks)
14. [Worktree, submodules, sparse и partial clone](#14-worktree-submodules-sparse-и-partial-clone)
15. [Безопасные рабочие сценарии](#15-безопасные-рабочие-сценарии)
16. [Шпаргалка выбора команды](#16-шпаргалка-выбора-команды)

---

# 1. Ментальная модель Git

## Что такое Git

**Git — распределённая система контроля версий.** Обычный `clone` содержит не только рабочие файлы, а локальную базу Git-объектов и историю репозитория. Поэтому `commit`, `branch`, `log`, `diff`, `rebase` и множество других операций не требуют постоянного соединения с сервером.

Это ключевое отличие от классической централизованной VCS: центральный сервер в Git — важная часть командного workflow, но не обязательный центр самой модели данных.

> [!info] Git ≠ [[GitHub]]
> **Git** — система контроля версий, формат объектов, refs и команды для работы с историей.  
> **GitHub** — сервис вокруг Git: hosting, pull requests, issues, permissions, Actions/[[CI CD|CI]] и т. д.

## Три области, которые нужно держать в голове

```text
Working tree                 Index / Staging area                Repository
файлы на диске               будущий snapshot commit            .git
      │                                │                            │
      │ git add                        │ git commit                 │
      ├───────────────────────────────►├───────────────────────────►│
      │                                │                            │
      │◄──────── git restore ──────────│                            │
      │                                │◄──── restore/reset ────────│
```

### Working tree

Файлы, которые ты прямо сейчас видишь и редактируешь.

### Index / staging area

Промежуточный снимок того, **что попадёт в следующий commit**. Это не просто список файлов: Git хранит конкретные версии содержимого, выбранные для следующего commit.

### Repository

Локальная база объектов и refs внутри `.git`.

## Tracked и untracked

**Tracked-файл** уже известен Git. Он может быть:

- `unmodified` — совпадает с зафиксированным состоянием;
- `modified` — изменён в working tree;
- `staged` — новая версия подготовлена в index.

**Untracked-файл** существует на диске, но ещё не добавлен в index и не находится в текущем snapshot.

## Базовые команды

| Команда | Смысл |
|---|---|
| `git init` | создаёт новый репозиторий и `.git`; существующие файлы автоматически не коммитит |
| `git clone <url>` | получает репозиторий, создаёт remote `origin` и обычно checkout'ит default branch |
| `git status` | показывает состояние working tree/index и информацию о текущей ветке |
| `git add <file>` | помещает текущую версию файла в index |
| `git commit` | создаёт commit **из index** |
| `git commit -a` | автоматически stage'ит изменения/удаления уже tracked-файлов и делает commit; новые untracked не добавляет |

> [!important] Главная ловушка `git add`
> `git add` копирует **текущее состояние** файла в index. Если после этого снова изменить файл, staged-версия не обновится сама.
>
> Поэтому один файл может одновременно отображаться как **staged** и **modified**.

---

# 2. Working tree, index и commit

## Как Git получает следующий commit

Представь файл `app.js`:

```text
HEAD:       версия A
index:      версия B
working:    версия C
```

Если выполнить:

```bash
git commit
```

в commit попадёт **B**, потому что commit строится из index. Версия C останется незакоммиченной.

Это объясняет сразу несколько типовых вопросов собеседования:

- почему после изменения staged-файла нужно повторить `git add`;
- почему staged и unstaged изменения могут существовать одновременно;
- почему `git commit` не означает «закоммитить всё, что сейчас лежит на диске».

## Частичное staging

```bash
git add -p
```

позволяет выбирать отдельные hunks. Это нужно, если в одном файле случайно смешаны две логические задачи.

Правильный workflow атомарного commit:

```bash
git add -p
git diff --staged
git commit
```

Затем stage'ишь оставшуюся задачу отдельно.

### Почему атомарные commits полезны

Не только ради красивого `git log`. Их проще:

- review'ить;
- revert'ить;
- cherry-pick'ить;
- анализировать через `bisect`;
- переносить между ветками;
- связывать с конкретной причиной изменения.

## Amend последнего commit

```bash
git commit --amend
```

не «редактирует существующий commit». Git создаёт **новый commit**, а текущая ветка начинает указывать на него.

Добавить забытый файл в последний локальный commit:

```bash
git add forgotten-file
git commit --amend --no-edit
```

Изменить только сообщение:

```bash
git commit --amend
```

> [!warning] Amend опубликованного commit
> После `amend` object ID меняется. Если старый commit уже использовали другие разработчики, их история будет построена на другой версии commit. Поэтому published/shared history без необходимости не переписывают.

---

# 3. Diff, файлы, ignore и clean

## Что именно сравнивает `git diff`

| Команда | Сравнение |
|---|---|
| `git diff` | working tree ↔ index |
| `git diff --staged` | index ↔ `HEAD` |
| `git diff --cached` | то же, что `--staged` |
| `git diff HEAD` | working tree в целом ↔ `HEAD`, включая staged + unstaged tracked changes |
| `git diff A B` | snapshot `A` ↔ snapshot `B` |
| `git diff A..B` | практически то же сравнение snapshots A и B |
| `git diff A...B` | `merge-base(A,B)` ↔ `B` |

Обычный `git diff` **не показывает содержимое новых untracked-файлов** как часть diff.

## Restore

Отбросить незастейдженные изменения tracked-файла:

```bash
git restore file
```

Git восстанавливает working tree из index.

Убрать файл из staging, сохранив изменения на диске:

```bash
git restore --staged file
```

Обычно index возвращается к версии из `HEAD`, а working tree не уничтожается. Старый распространённый способ обычного unstaging — `git reset file`; результат в типичном случае похож, но `reset` — более широкая команда, которая также умеет двигать `HEAD`/branch, тогда как `restore` специально разделяет восстановление файлов и index.

Получить файл из другого commit:

```bash
git restore --source=<commit> -- path/to/file
```

Старый синтаксис:

```bash
git checkout <commit> -- path/to/file
```

> [!danger] `git restore .`
> Массово заменяет незастейдженные изменения tracked-файлов. То же исторически относится к `git checkout .`. Перед такими командами проверяй `git status` и `git diff`.

## `git rm` и `git mv`

```bash
git rm file
```

удаляет tracked-файл из working tree и stage'ит удаление.

```bash
git rm --cached file
```

убирает файл из index, но оставляет на диске. Это типичный способ перестать отслеживать файл, который теперь должен попадать под `.gitignore`.

```bash
git mv old new
```

удобная комбинация перемещения + staging.

### Git не хранит rename как отдельный исторический факт

Git хранит snapshots. При `diff` он обычно **эвристически обнаруживает**, что удалённый и добавленный файлы достаточно похожи, чтобы считать их rename/copy.

## `.gitignore`

`.gitignore` предназначен в первую очередь для **untracked** файлов.

Если `.env` уже tracked, запись:

```gitignore
.env
```

не перестанет показывать его изменения.

Нужно сначала убрать его из index:

```bash
git rm --cached .env
```

После этого правило ignore сможет работать.

Другие места для ignore-правил:

- `.git/info/exclude` — локально для конкретного repository;
- глобальный excludes-файл через `core.excludesFile`.

Проверить, почему путь игнорируется:

```bash
git check-ignore -v path
```

Принудительно добавить игнорируемый файл:

```bash
git add -f path
```

## `git clean`

Удаляет **untracked** данные.

Сначала всегда безопаснее посмотреть dry-run:

```bash
git clean -n
```

Удалить untracked-файлы и каталоги:

```bash
git clean -fd
```

Ignored-файлы обычный `clean -fd` не обязан удалять. Опции вроде `-x` делают операцию существенно опаснее.

> [!danger] Опасная комбинация
> `git clean -fdx` способен удалить и ignored-файлы: локальные `.env`, зависимости, build cache и другие данные, которые Git никогда не сохранял.

---

# 4. Commits, objects и неизменяемость

## Snapshot, а не «список изменений»

Концептуально каждый commit фиксирует **snapshot дерева проекта**. При этом Git не обязан физически копировать каждый неизменившийся файл: одинаковый контент может ссылаться на уже существующие objects.

## Что такое commit

Commit object содержит примерно:

```text
tree <root-tree-id>
parent <parent-commit-id>   # 0 у root, 1 обычно, 2+ у merge
author ...
committer ...

commit message
```

Commit связывает:

- корневой `tree` snapshot;
- parent/parents;
- автора;
- committer;
- временные данные;
- сообщение.

## Почему commit неизменяемый

Object ID вычисляется из содержимого объекта. Если изменить:

- snapshot;
- parent;
- автора/committer metadata;
- message;

получится **новый объект с новым ID**.

Именно поэтому `amend`, `rebase` и `cherry-pick` обычно создают новые hashes.

## Object ID

Git-объекты адресуются hash'ем их содержимого и заголовка. Исторически большинство repositories используют SHA-1; Git также поддерживает SHA-256 object format.

Короткий hash вроде:

```text
8a31f2c
```

не является глобально уникальным. Он достаточен только пока однозначно разрешается внутри конкретного repository. Чем больше objects, тем длиннее сокращение может понадобиться.

## Одинаковый snapshot ≠ одинаковый commit

Два commits могут иметь идентичный `tree`, но разные IDs из-за:

- разных родителей;
- разного message;
- author/committer metadata.

Отсюда важный вывод:

> **identity commit и логическое изменение — разные вещи.**

Один и тот же patch после `cherry-pick` или `rebase` может получить другой commit ID.

---

# 5. Ветки, HEAD и refs

## Branch — это указатель

Ветка не является копией проекта и не хранит отдельную полную историю.

```text
A──B──C  ← main
```

`main` — по сути ref, указывающая на `C`.

После нового commit:

```text
A──B──C──D  ← main
```

Git создал `D`, а ref `main` переместилась.

Именно поэтому ветки в Git считаются **лёгкими**.

## Что такое ref

Ref — читаемое имя, которое в итоге разрешается в object ID.

Типичные пространства:

```text
refs/heads/       локальные ветки
refs/remotes/     remote-tracking refs
refs/tags/        tags
```

Локальные branch refs логически находятся в `refs/heads/`, хотя Git может физически хранить refs и в упакованном виде (`packed-refs`).

## HEAD

Обычно `.git/HEAD` содержит symbolic ref:

```text
ref: refs/heads/main
```

То есть:

```text
HEAD → main → commit C
```

### Detached HEAD

Если checkout сделан прямо на commit:

```text
HEAD → commit B
main → commit C
```

HEAD больше не привязан к локальной branch. Коммиты делать можно, но branch pointer автоматически их не удерживает.

Сохранить сделанную там работу:

```bash
git switch -c rescue-branch
```

или:

```bash
git branch rescue-branch
```

Если уже переключился и «потерял» detached commits — смотри `git reflog`.

## Создание и переключение веток

Создать, но не переключаться:

```bash
git branch feature
```

Создать и переключиться:

```bash
git switch -c feature
```

Старый универсальный вариант:

```bash
git checkout -b feature
```

`switch` удобнее тем, что имеет более узкую семантику: работа именно с branch/checkout state, а не одновременно и восстановление файлов, как исторический `checkout`.

## Переименование и удаление

Переименовать текущую ветку:

```bash
git branch -m new-name
```

Безопасное удаление:

```bash
git branch -d feature
```

Git откажется, если считает, что commits ветки ещё не интегрированы.

Принудительное:

```bash
git branch -D feature
```

Удаление branch удаляет прежде всего **ref**, а не мгновенно все commit objects. Если objects ещё доступны через другие refs/reflog, они сохраняются.

## Revision syntax: `~` и `^`

```text
HEAD~1
```

означает первый parent `HEAD`.

```text
HEAD~3
```

три раза пройти по первому parent.

```text
HEAD^
HEAD^1
```

первый непосредственный parent.

Для merge commit:

```text
M^1   первый parent
M^2   второй parent
```

### Главное различие

- `~N` — N раз идти по **первому parent**;
- `^N` — выбрать **N-го parent конкретного commit**.

## Полезные команды по refs/веткам

Какие branches содержат commit:

```bash
git branch --contains <commit>
```

включая remote-tracking:

```bash
git branch -a --contains <commit>
```

Граф:

```bash
git log --graph --oneline --decorate --all
```

## Orphan branch

```bash
git switch --orphan docs-root
```

создаёт ветку с новой независимой корневой историей — без обычного parent из текущей ветки.

---

# 6. Merge и конфликты

## Что делает merge

Находясь в `main`:

```bash
git merge feature
```

означает: **интегрировать `feature` в текущую ветку `main`**.

Возможны три основных результата:

1. fast-forward;
2. настоящий merge commit;
3. остановка на конфликтах.

## Fast-forward

До merge:

```text
A──B  ← main
    \
     C──D  ← feature
```

Если `main` является ancestor `feature`, можно просто переместить указатель:

```text
A──B──C──D
          ↑
     main, feature
```

Новый merge commit не нужен.

Требовать только fast-forward:

```bash
git merge --ff-only feature
```

Если histories разошлись, команда завершится ошибкой.

Принудительно создать merge commit даже когда FF возможен:

```bash
git merge --no-ff feature
```

## Three-way merge

Если histories разошлись:

```text
      C──D  ← feature
     /
A──B
     \
      E──F  ← main
```

Git использует:

- `D` — одну сторону;
- `F` — вторую;
- `B` — их общий предок, **merge base**.

Найти merge base:

```bash
git merge-base main feature
```

Итоговый merge commit имеет как минимум двух parents.

## Конфликты

Конфликт возникает, когда Git не может однозначно выбрать итог:

- обе стороны несовместимо меняют одну область;
- одна удаляет файл, другая его меняет (`delete/modify`);
- обе стороны по-разному переименовывают один путь (`rename/rename`);
- другие неоднозначные структурные ситуации.

Во время обычного конфликтного merge index может хранить несколько stages пути:

```text
stage 1 → base
stage 2 → ours
stage 3 → theirs
```

При обычном merge:

- **ours** — текущая checkout-ветка;
- **theirs** — вливаемая сторона.

Выбрать целиком одну версию файла:

```bash
git restore --ours file
git restore --theirs file
```

старый вариант:

```bash
git checkout --ours file
git checkout --theirs file
```

После ручного разрешения:

```bash
git add resolved-file
git commit
```

или в поддерживаемом сценарии:

```bash
git merge --continue
```

Отменить незавершённый merge:

```bash
git merge --abort
```

> [!warning] Нет conflict ≠ merge корректен
> Git понимает строки, trees и patches, но не бизнес-логику. Две изменения могут автоматически слиться и при этом сломать API, invariant или тесты. Поэтому после merge нужны тесты.

## Почему ours/theirs при rebase сбивают с толку

Rebase не просто «мерджит мою feature в main». Он временно переигрывает commits поверх новой базы через merge machinery. Поэтому контекст `ours`/`theirs` может не совпадать с интуитивным «мой код / чужой код».

На собеседовании лучше не заучивать стороны вслепую, а проверять текущий контекст и diff.

## Squash merge

```bash
git merge --squash feature
```

подготавливает итоговые изменения feature в index/working tree, но **не создаёт merge commit** и не фиксирует второго parent. Commit выполняется отдельно.

Следствие: topology исходной feature не становится частью ancestry. Поэтому Git впоследствии может не считать исходные feature commits слитыми по ancestry, хотя итоговый код уже присутствует.

## Octopus merge

Merge может иметь более двух parents. Такой merge называют octopus. Обычно он применим для объединения нескольких независимых, неконфликтующих веток.

## Полезные команды

Только merge commits:

```bash
git log --merges
```

Diff feature относительно общей базы:

```bash
git diff main...feature
```

---

# 7. Rebase и переписывание истории

## Что делает rebase

На feature:

```bash
git rebase main
```

Допустим было:

```text
      C──D  ← feature
     /
A──B──E──F  ← main
```

Rebase берёт изменения `C` и `D` и переигрывает их поверх `F`:

```text
A──B──E──F──C'──D'  ← feature
```

`C'` и `D'` — **новые commits**.

Почему новые IDs? Их parent уже другой, следовательно изменилось содержимое commit object.

## Merge vs rebase

| | Merge | Rebase |
|---|---|---|
| существующие commits | сохраняет | переписывает переносимые commits |
| topology | сохраняет ветвление | может сделать историю линейной |
| shared history | обычно безопаснее | требует осторожности |
| результат файлов | может быть тем же | может быть тем же |

> [!rule] Практическое правило
> **Rebase локальной/private feature** обычно нормален.  
> **Rebase общей опубликованной истории**, на которую уже опираются другие, требует координации.

## Конфликтный rebase

После исправления текущего конфликта:

```bash
git add resolved-file
git rebase --continue
```

Полностью отменить:

```bash
git rebase --abort
```

Пропустить текущий commit:

```bash
git rebase --skip
```

`--skip` нужно использовать осознанно: изменения этого commit могут исчезнуть из результата.

### Почему rebase иногда конфликтует много раз

Merge обычно объединяет два tips одним three-way merge. Rebase применяет **каждый commit по очереди** к новой базе. Один и тот же концептуальный конфликт может поэтому всплывать на нескольких commits.

Полезен:

```bash
git rerere
```

который умеет запоминать разрешения повторяющихся конфликтов.

## Interactive rebase

```bash
git rebase -i HEAD~5
```

Типичный todo:

```text
pick   aaaa feature base
fixup  bbbb typo
reword cccc add validation
edit   dddd refactor API
squash eeee tests
```

Действия:

| Команда | Что делает |
|---|---|
| `pick` | оставить commit |
| `reword` | оставить изменения, изменить message |
| `edit` | остановиться и позволить изменить commit |
| `squash` | объединить с предыдущим и отредактировать итоговое сообщение |
| `fixup` | объединить с предыдущим, обычно без отдельного message |
| `drop` | удалить commit из переписываемой цепочки |

Объединить последние N локальных commits:

```bash
git rebase -i HEAD~N
```

Изменить message не последнего commit — выбрать `reword`. Это перепишет не только его, но и последующие commits, потому что меняется цепочка parents.

Удалить commit — `drop` или удалить строку из todo.

## `--onto`

Общий вид:

```bash
git rebase --onto <newbase> <upstream> <branch>
```

Смысл: взять commits `branch`, находящиеся после `<upstream>`, и переиграть их на `<newbase>`.

Очень полезно, если feature была начата от неправильной ветки.

## Fixup + autosquash

Создать исправление специально для существующего локального commit:

```bash
git commit --fixup <target-commit>
```

Потом:

```bash
git rebase -i --autosquash <base>
```

Git разместит `fixup!`/`squash!` commits рядом с целевыми.

## Rebase vs cherry-pick

Обе операции переигрывают patches и создают новые commits.

- `rebase` системно переносит **цепочку** на новую базу;
- `cherry-pick` явно переносит **выбранные commits**.

## Empty commit при rebase

Commit может стать empty, если:

- его изменения уже появились в новой базе;
- похожий patch уже был применён;
- изменения поглотились предыдущими разрешениями конфликтов.

Это одна из причин, почему «commit пропал после rebase» не всегда означает потерю данных.

## Перед сложным rebase

```bash
git status
git branch backup-before-rebase
```

Backup branch лучше, чем надеяться только на reflog.

---

# 8. Reset, restore, revert и восстановление

Это одна из самых важных тем интервью.

## Четыре разных действия

| Команда | Основная идея |
|---|---|
| `restore` | восстановить working tree/index для файлов |
| `reset` | передвинуть HEAD/branch и при необходимости index/working tree |
| `revert` | создать новый commit с обратным изменением |
| `reflog` | найти предыдущие положения refs/HEAD |

## `git reset`

Концептуально reset сначала двигает текущую branch/HEAD к указанному commit. Дальше режим определяет, что произойдёт с index и working tree.

### `--soft`

```bash
git reset --soft HEAD~1
```

```text
HEAD       ← перемещается
index      ← остаётся
working    ← остаётся
```

Итог: изменения отменённого commit остаются **staged**.

### `--mixed`

```bash
git reset --mixed HEAD~1
# или просто
git reset HEAD~1
```

```text
HEAD       ← перемещается
index      ← приводится к target
working    ← остаётся
```

Итог: изменения остаются **unstaged**.

### `--hard`

```bash
git reset --hard HEAD~1
```

```text
HEAD       ← target
index      ← target
working    ← tracked state target
```

Незакоммиченные tracked-изменения могут быть уничтожены.

> [!important] `reset --hard` и untracked
> `git reset --hard` обычно **не удаляет все untracked-файлы**. Для этого существует `git clean`. Но конфликтующие пути могут создавать особые случаи, поэтому считать `--hard` полностью безвредным для любого untracked состояния нельзя.

## Revert

```bash
git revert <commit>
```

создаёт **новый commit**, который инвертирует изменения старого.

Поэтому для уже опубликованной общей `main` типичный выбор:

```bash
git revert bad-commit
git push
```

а не `reset` + force push.

### Revert merge commit

Merge имеет несколько parents, поэтому Git нужно указать mainline:

```bash
git revert -m 1 <merge-commit>
```

`-m 1` говорит, относительно какого parent считать изменение merge.

## Reflog

```bash
git reflog
```

показывает локальную историю перемещений refs/HEAD: commit, checkout, reset, rebase и т. д.

Например после:

```bash
git reset --hard HEAD~3
```

первым делом:

```bash
git reflog
```

найти старый tip и закрепить его:

```bash
git branch rescue <old-commit>
```

или осознанно reset'нуться обратно.

### Восстановление удалённой ветки

Если branch ref удалили, но её tip был недавно доступен:

```bash
git reflog
git branch restored <commit>
```

### Почему reflog — не backup

- он локальный;
- записи имеют срок хранения;
- unreachable objects могут быть удалены GC;
- на другом clone нужного reflog может не быть.

## `ORIG_HEAD`

Некоторые операции сохраняют предыдущее положение HEAD в `ORIG_HEAD`. Это удобная аварийная ссылка, но нельзя рассчитывать, что она всегда содержит именно нужное старое состояние.

## Dangling objects и `fsck`

Если reflog не помог:

```bash
git fsck --lost-found
```

может показать dangling/unreachable objects.

**Dangling commit** — commit object существует в object database, но больше не достижим из обычных refs.

Пока GC не удалил object, его иногда можно восстановить.

`git reflog expire` управляет истечением reflog-записей. Агрессивное expire + `git gc` уменьшает шансы восстановления.

## Практические рецепты

### Убрать последний локальный commit, оставить staged

```bash
git reset --soft HEAD~1
```

### Убрать commit, оставить изменения unstaged

```bash
git reset --mixed HEAD~1
```

### Уничтожить последний commit и tracked changes

```bash
git reset --hard HEAD~1
```

только если данные точно не нужны.

### Убрать один файл из последнего неопубликованного commit

Логика:

1. вернуть нужный файл в правильное состояние index;
2. сохранить/оставить его изменения в working tree, если они нужны;
3. выполнить `git commit --amend`.

Один из инструментов для unstaging:

```bash
git restore --staged file
```

### Восстановить удалённый tracked-файл

```bash
git restore file
```

если удаление ещё не закоммичено.

> [!rule] Когда `reset --hard` наиболее уместен
> Когда работа локальная, опубликованная history не переписывается, текущее tracked-состояние точно не нужно и ты проверил `status`/backup ref.

---

# 9. Remotes: fetch, pull и push

## Remote

Remote — именованная конфигурация удалённого repository: URL + refspecs и связанные настройки.

`origin` — просто распространённое имя, которое обычно создаёт `clone`. Оно не имеет специального сетевого смысла.

Посмотреть remotes:

```bash
git remote -v
```

Fetch URL и push URL могут отличаться, например чтение по HTTPS, запись по [[SSH и передача файлов|SSH]].

## Remote-tracking branches

```text
main          локальная branch
origin/main   локальная remote-tracking ref
```

`origin/main` — не «ветка, живущая прямо на GitHub». Это локально сохранённое последнее известное состояние remote branch после fetch/push.

## Fetch

```bash
git fetch origin
```

получает Git objects и обновляет remote-tracking refs согласно refspec.

При этом локальная `main` сама двигаться не обязана.

Поэтому ситуация:

```text
origin/main обновился
main остался прежним
```

после `fetch` абсолютно нормальна.

## Pull

Концептуально:

```text
git pull = git fetch + интеграция
```

Интеграция может быть:

- merge;
- rebase;
- только fast-forward — в зависимости от флагов/config.

Поэтому многие предпочитают:

```bash
git fetch
git log ...
git diff ...
git merge origin/main
```

или rebase отдельно: больше контроля над тем, что именно пришло.

### Pull только fast-forward

```bash
git pull --ff-only
```

Если histories разошлись — Git откажется делать merge commit.

### Pull с rebase

```bash
git pull --rebase
```

после fetch переигрывает локальные commits поверх обновлённого upstream.

Это даёт более линейную локальную историю, но локальные commit IDs переписываются.

## Push

```bash
git push origin feature
```

передаёт необходимые objects и просит remote обновить соответствующую ref.

Первый push с установкой upstream:

```bash
git push -u origin feature
```

После этого обычные `git push`/`git pull` могут использовать tracking information.

## Upstream branch

Upstream — ветка, относительно которой локальная branch обычно синхронизируется/сравнивается.

Посмотреть:

```bash
git branch -vv
```

или для текущей:

```bash
git rev-parse --abbrev-ref --symbolic-full-name @{u}
```

### Ahead / behind

- **ahead N** — локально N commits, которых нет в upstream;
- **behind M** — в upstream M commits, которых нет локально;
- оба одновременно — histories diverged.

`git status` может быть `clean`, но branch при этом может быть ahead/behind: cleanliness working tree и синхронизация history — разные измерения.

## Non-fast-forward rejection

Если remote branch уже содержит commits, которых нет в твоём push tip, обычный push может быть отклонён:

```text
non-fast-forward
```

Причина: обновление ref сделало бы существующие remote commits недостижимыми из этой branch.

## Force push

```bash
git push --force
```

разрешает переписать remote ref даже non-fast-forward образом. Это может стереть чужую видимую работу из branch history.

Предпочтительнее после осознанного rebase собственной feature:

```bash
git push --force-with-lease
```

`--force-with-lease` проверяет ожидаемое состояние remote ref и обычно откажется, если она изменилась неожиданно.

Но это **не абсолютная защита**: гарантия зависит от актуальности ожидаемых refs и workflow автоматического fetch.

> [!danger] Shared `main`
> Бездумный `--force` в общей `main` — один из самых опасных Git-сценариев: можно переписать ref поверх чужих commits.

## Prune

Remote branch удалена, но локально всё ещё виден `origin/feature`?

```bash
git fetch --prune
```

обновит refs и удалит stale remote-tracking refs.

Отдельно:

```bash
git remote prune origin
```

чистит устаревшие remote-tracking refs этого remote.

`[gone]` в:

```bash
git branch -vv
```

обычно означает, что configured upstream этой локальной branch больше не существует среди известных remote refs.

## Удаление remote branch

```bash
git push origin --delete feature
```

Удаление server ref не удаляет автоматически твою локальную branch.

Перед удалением после PR стоит проверить, что нужная работа действительно интегрирована и никто не использует branch как единственный ref на важные commits.

## Refspec

Refspec задаёт сопоставление source ref → destination ref.

Типовая идея fetch:

```text
refs/heads/*  →  refs/remotes/origin/*
```

Префикс `+` позволяет non-fast-forward update соответствующей ref.

## Fork workflow

Часто:

```text
origin   → твой fork
upstream → основной repository
```

Настройка:

```bash
git remote add upstream <url>
```

Обновление:

```bash
git fetch upstream
```

Затем main/feature интегрируются с `upstream/main`, а собственные branches push'ятся в `origin`.

## `git ls-remote`

```bash
git ls-remote <url>
```

показывает refs и их object IDs на remote без полного clone.

## Shallow clone

```bash
git clone --depth 1 <url>
```

загружает ограниченную глубину commit history.

Плюс — быстрее/меньше данных. Минус — операции, которым нужна полная ancestry/history, могут работать ограниченно.

Углубить:

```bash
git fetch --deepen 50
```

получить полную историю:

```bash
git fetch --unshallow
```

---

# 10. Cherry-pick, stash и tags

## Cherry-pick

```bash
git cherry-pick <commit>
```

берёт patch указанного commit и применяет его поверх текущего `HEAD`, обычно создавая **новый commit**.

Подходит, когда нужен отдельный bugfix без merge всей branch — например backport в release.

Диапазон в линейной истории:

```bash
git cherry-pick A^..B
```

включает `A` и `B`.

Применить без автоматического commit:

```bash
git cherry-pick --no-commit <commit>
```

Полезно, если нужно применить несколько patches, поправить результат и закоммитить одним commit.

Отменить незавершённый конфликтный cherry-pick:

```bash
git cherry-pick --abort
```

## Stash

```bash
git stash
```

сохраняет рабочее состояние в Git objects/refs и очищает соответствующую часть working tree.

По умолчанию обычный stash не обязан включать untracked.

Включить untracked:

```bash
git stash -u
```

Включить также ignored:

```bash
git stash -a
```

Список:

```bash
git stash list
```

Применить и оставить запись:

```bash
git stash apply stash@{1}
```

Применить и при успехе удалить запись:

```bash
git stash pop stash@{1}
```

Создать branch от исходной базы stash и применить его там:

```bash
git stash branch rescue stash@{0}
```

Это особенно полезно, если current branch сильно изменилась и обычное применение stash конфликтует.

### WIP commit или stash?

**Stash** хорош для краткого временного переключения контекста.

**WIP commit** часто лучше, если работу нужно:

- надёжно связать с branch;
- push'нуть/синхронизировать;
- оставить на дни;
- явно видеть в истории разработки.

Stash не стоит рассматривать как долговременный backup.

## Tags

Tag — ref, обычно используемая для именования релизного состояния.

### Lightweight

Простая ref на object.

### Annotated

Отдельный tag object, содержащий tagger, дату, message и потенциально cryptographic signature.

Создать:

```bash
git tag -a v1.2.0 -m 'Release 1.2.0'
```

На конкретном commit:

```bash
git tag -a v1.2.0 <commit> -m 'Release 1.2.0'
```

Tags не обязательно отправляются обычным `git push`.

Конкретный tag:

```bash
git push origin v1.2.0
```

Все tags:

```bash
git push --tags
```

Annotated tags, связанные с отправляемыми commits:

```bash
git push --follow-tags
```

Удалить локально:

```bash
git tag -d v1.2.0
```

Удалить remote tag:

```bash
git push origin --delete v1.2.0
```

> [!warning] Release tags должны быть стабильными
> Перемещение уже опубликованного release tag ломает воспроизводимость: одно и то же имя версии начинает означать разные commits для разных пользователей/кэшей.

---

# 11. Просмотр истории, поиск и диагностика

## Базовый `log`

```bash
git log
```

показывает commits, достижимые из выбранной revision.

Один из самых полезных видов:

```bash
git log --oneline --decorate --graph --all
```

- `--oneline` — компактно;
- `--decorate` — branch/tag names;
- `--graph` — topology;
- `--all` — все локально известные refs.

## `git show`

```bash
git show <commit>
```

показывает metadata commit и обычно patch относительно parent.

## Диапазоны `..` и `...`

Здесь важно различать `log` и `diff`.

### В `git log`

```bash
git log A..B
```

= commits, достижимые из `B`, но не из `A`.

Практический вопрос «что есть в feature, но нет в main»:

```bash
git log main..feature
```

```bash
git log A...B
```

= симметрическая разность: commits, уникальные для одной из сторон.

Удобный вид:

```bash
git log --left-right A...B
```

### В `git diff`

```bash
git diff A B
git diff A..B
```

сравнивают snapshots A и B.

А:

```bash
git diff A...B
```

сравнивает `merge-base(A,B)` с `B`.

> [!important] Три точки не имеют одной универсальной семантики
> `A...B` в `log` и `diff` — не одно и то же действие.

## Blame

```bash
git blame file
```

показывает commit, которому Git атрибутирует последнее изменение каждой текущей строки.

Это **не доказательство авторства строки**. Refactor, форматирование, move/copy и особенности similarity могут изменить attribution.

## Поиск по истории: `-S` и `-G`

```bash
git log -S'functionName'
```

ищет commits, где изменилось **количество вхождений** строки.

```bash
git log -G'regex'
```

ищет commits, где diff содержит строки, matching regex.

Если нужно понять, когда исчез конкретный вызов функции, обычно начинают с `-S`, затем смотрят `git show` найденных commits; при необходимости используют `-G` и `blame`.

## `git grep`

```bash
git grep 'pattern'
```

ищет по tracked content repository.

## Bisect

Лучший инструмент для поиска commit, внёсшего regression, если известны:

- хороший commit;
- плохой commit;
- способ проверить состояние.

Ручной workflow:

```bash
git bisect start
git bisect bad <bad>
git bisect good <good>
# проверить предложенный commit
git bisect good   # или bad
...
git bisect reset
```

Автоматически:

```bash
git bisect run ./test-script.sh
```

Git использует exit code теста и делает [[Algorithms|бинарный поиск]].

## `rev-parse`

```bash
git rev-parse HEAD
```

разрешает revision в object ID. Команда широко используется в скриптах и для получения служебной информации.

## Проверка ancestry

Является ли `A` предком `B`:

```bash
git merge-base --is-ancestor A B
```

Результат передаётся exit code.

Это удобно, например, чтобы проверить, содержит ли release branch конкретный fix **по ancestry**.

Но после squash/cherry-pick исходный commit ID может не быть ancestor, даже если логически эквивалентный patch присутствует.

## Patch-equivalence

```bash
git cherry -v upstream head
```

помогает понять, какие patches из `head` уже имеют эквивалент в upstream, даже если commit hashes различаются.

Это полезно после cherry-pick/rebase.

## `git describe`

Создаёт человекочитаемое описание commit на основе ближайшего подходящего tag, расстояния и abbreviated hash.

## История конкретного файла

```bash
git log --follow -- path/to/file
```

пытается продолжать историю через rename. `--follow` имеет ограничения и применяется к одному пути.

---

# 12. Внутренности Git

Понимание internals сильно облегчает вопросы про ветки, reset и восстановление.

## Четыре основных object type

```text
blob
 tree
 commit
 annotated tag
```

## Blob

Blob хранит **байты содержимого файла**.

Он не знает:

- имя файла;
- путь;
- историю имени.

## Tree

Tree представляет каталог/snapshot структуры и содержит записи вида:

```text
mode  object-id  filename
```

Запись может указывать на:

- blob;
- другое tree;
- gitlink submodule.

Именно tree связывает **имя файла** с blob object.

## Commit

Commit указывает на root tree и parent/parents, плюс metadata/message.

```text
branch/ref → commit → tree → blobs/trees
```

## Annotated tag object

Хранит отдельный объект с metadata tag и ссылкой на другой Git object.

## Content-addressable storage

Object ID получается из содержимого объекта.

Следствие: два идентичных файла могут ссылаться на один и тот же blob object, даже если они находятся под разными именами/путями.

## `.git/objects`

Object database содержит loose objects и packfiles.

### Packfile

Packfile объединяет множество objects и применяет compression/delta compression для экономии места и эффективной передачи.

## Garbage collection

```bash
git gc
```

выполняет обслуживание: упаковку, оптимизацию и со временем очистку некоторых unreachable objects согласно правилам expiration.

Это объясняет, почему недавно удалённый commit часто можно восстановить: ref уже исчезла, но object ещё физически существует и может быть доступен через reflog.

## Reachability

Object **reachable**, если до него можно дойти от корней вроде refs/reflogs через ссылки объектов.

Если object становится unreachable, он не обязательно удаляется мгновенно. Позже expiration/GC может его убрать.

## Низкоуровневые команды

Посмотреть object:

```bash
git cat-file -p <object>
```

Вычислить hash содержимого:

```bash
git hash-object file
```

С соответствующими флагами `hash-object` умеет также записывать object.

Создать tree из index:

```bash
git write-tree
```

## Porcelain и plumbing

**Porcelain** — пользовательские высокоуровневые команды:

- `status`;
- `commit`;
- `switch`;
- `merge`.

**Plumbing** — низкоуровневые строительные блоки:

- `cat-file`;
- `hash-object`;
- `write-tree`;
- `update-index`;
- и др.

## `.git/index`

Бинарное представление staging area. Содержит paths, object IDs, modes и служебные данные. Во время конфликта один path может иметь несколько index stages.

## `.git/HEAD`

Обычно symbolic ref на текущую branch. В detached state содержит напрямую commit object ID.

## Symbolic ref

Ref, которая хранит имя другой ref вместо object ID. Классический пример:

```text
HEAD → refs/heads/main
```

Обычная branch ref в `refs/heads/` должна разрешаться в commit. Tags могут ссылаться и на другие Git object types.

## Почему branch «лёгкая»

Git разделяет:

- immutable objects с данными;
- дешёвые movable refs.

Создание branch — в основном создание маленькой ссылки на уже существующий commit, а не копирование проекта.

---

# 13. Config, attributes и hooks

## Уровни config

Типично:

```text
system  → для всей системы
global  → для пользователя
local   → для repository
```

Более специфичная настройка обычно переопределяет менее специфичную.

Локальный config обычно:

```text
.git/config
```

Посмотреть настройки вместе с источником:

```bash
git config --list --show-origin
```

Дополнительно scope:

```bash
git config --list --show-origin --show-scope
```

## Line endings и `core.autocrlf`

`core.autocrlf` участвует в преобразовании окончаний строк между repository и working tree в определённых workflows.

Проблема личного global config: у разных членов команды могут быть разные значения.

Поэтому для правил проекта часто надёжнее version-controlled `.gitattributes`.

## `.gitattributes`

Может задавать path-specific attributes, например:

- text normalization;
- `eol`;
- diff drivers;
- merge drivers;
- `export-ignore`;
- другие механизмы Git и внешних инструментов.

Главное преимущество командных правил — файл version-controlled и одинаков для clone'ов.

## Hooks

Git hook — скрипт на событие Git.

Примеры client-side:

```text
pre-commit
commit-msg
pre-push
```

Есть также server-side receive hooks.

> [!warning] Client hooks ≠ security boundary
> Пользователь может не установить hook, изменить или обойти его. Обязательные правила должны контролироваться CI/server-side protections, а hooks можно использовать как быстрый локальный feedback.

## Commit signing

Commit/tag можно криптографически подписывать поддерживаемым Git форматом.

Проверить подпись commit:

```bash
git verify-commit <commit>
```

Подпись подтверждает связь объекта с соответствующим signing key при корректной trust/config model, но сама по себе не доказывает корректность кода.

## Archive

```bash
git archive HEAD
```

создаёт архив snapshot без `.git`, например для поставки исходников.

## Bundle

`git bundle` упаковывает Git objects + refs в один файл, который можно использовать для offline-переноса repository/history.

## Maintenance

```bash
git maintenance
```

запускает/планирует задачи обслуживания, особенно полезные для крупных repositories.

---

# 14. Worktree, submodules, sparse и partial clone

## Worktree

`git worktree` позволяет иметь несколько working directories, использующих общую object database одного repository.

Например одновременно открыть `main` и `hotfix`:

```bash
git worktree add ../project-hotfix hotfix
```

Это удобнее второго полного clone, когда нужен параллельный checkout.

Одна и та же branch обычно не checkout'ится одновременно в нескольких linked worktrees, чтобы два рабочих дерева независимо не двигали одну branch ref.

## Submodule

Submodule — отдельный Git repository, на который superproject фиксирует конкретный commit.

Superproject хранит:

- **gitlink** → object ID commit submodule;
- `.gitmodules` → mapping path/URL и конфигурацию.

Поэтому изменение файлов внутри submodule сначала является изменением **в отдельном repository submodule**. Для superproject важен переход gitlink с одного commit ID на другой.

### Clone с submodules

```bash
git clone --recurse-submodules <url>
```

или после обычного clone:

```bash
git submodule update --init --recursive
```

### Почему submodule не должен автоматически брать latest main

Superproject фиксирует точный commit для воспроизводимости.

Если CI каждый раз брал бы «последний main submodule», один и тот же commit superproject мог бы собираться с разным dependency source.

Если обновил submodule до нового commit, в superproject нужно commit'нуть новый **gitlink**; если изменились настройки — также `.gitmodules`.

## Sparse checkout

Позволяет материализовать в working tree только часть tracked paths.

Полезно в monorepo, если работаешь, например, только с:

```text
backend/
```

Repository при этом логически знает о большем дереве.

## Partial clone

Partial clone может сохранить commit graph, но отложить загрузку части objects — например blobs — до момента, когда они понадобятся.

## Shallow vs partial clone

| Shallow clone | Partial clone |
|---|---|
| ограничивает глубину commit history | откладывает загрузку выбранных object types |
| ancestry может быть неполной | history graph может быть полной |
| `--depth` | filters/promisor remote |

Для большого monorepo sparse checkout и partial clone могут использоваться совместно: один уменьшает working set, второй — объём загружаемых objects.

## Rerere

`rerere` = **Reuse Recorded Resolution**.

Git запоминает способ, которым конфликт был разрешён, и может переиспользовать его при повторении похожего конфликта.

Особенно полезно при:

- длинных rebases;
- повторных merges;
- maintenance branches с повторяющимися conflicts.

---

# 15. Безопасные рабочие сценарии

Этот раздел объединяет типичные вопросы вида «что бы ты сделал, если…».

## 15.1 Забыл файл в последнем commit

Commit локальный:

```bash
git add file
git commit --amend --no-edit
```

Если commit уже shared, сначала подумай, оправдано ли rewrite.

## 15.2 Ошибка в message последнего commit

Локально:

```bash
git commit --amend
```

Published commit лучше не переписывать без необходимости.

## 15.3 Начал feature от неправильной базы

Если commits локальные:

```bash
git fetch
git rebase main
```

или более точно:

```bash
git rebase --onto newbase oldbase feature
```

## 15.4 Нужно срочно переключиться на hotfix

Варианты:

### Stash

```bash
git stash -u
git switch hotfix
```

### WIP branch/commit

Если работа ценная или переключение не на пять минут — зачастую лучше commit в отдельной branch.

## 15.5 Случайно сделал commit в `main`, ещё не push

Сначала **закрепи commit новой branch**, потом возвращай main.

```bash
git switch -c feature
git switch main
git reset --hard HEAD~1
```

Для нескольких commits принцип тот же: feature должна указывать на текущий tip, после чего `main` можно вернуть на upstream/исходный commit.

> [!important] Порядок важен
> Сначала создать ref, сохраняющую нужные commits. Потом выполнять destructive reset.

## 15.6 Удалил local branch с несмердженными commits

```bash
git reflog
git branch restored <old-tip>
```

## 15.7 После `reset --hard` пропала работа

Первое:

```bash
git reflog
```

Если работа была commit'нута или HEAD/ref когда-то на неё указывал — шанс хороший.

Если это были только незакоммиченные изменения working tree, Git мог никогда не записывать их как recoverable object.

## 15.8 `.env` добавлен в `.gitignore`, но остаётся tracked

```bash
git rm --cached .env
```

Но если внутри был секрет, который уже push'нут, это отдельная security-проблема.

## 15.9 Секрет уже попал в remote history

Удалить файл новым commit **недостаточно**.

Правильный порядок:

1. **rotate/revoke credential немедленно**;
2. считать старый secret compromised;
3. при необходимости переписать Git history специализированным инструментом;
4. координировать force update / clone cleanup;
5. проверить CI logs, artifacts, forks, caches и другие места утечки.

Почему rotation раньше history rewrite: удаление из Git не заставляет исчезнуть уже скопированный secret.

## 15.10 После rebase нужно обновить свою remote feature

Если rewrite разрешён workflow:

```bash
git fetch
git push --force-with-lease
```

Не использовать `--force` автоматически.

## 15.11 Нужен один bugfix из другой ветки

```bash
git cherry-pick <commit>
```

После этого проверить tests — одинаковый patch в новой базе может вести себя иначе.

## 15.12 Нужно отменить плохой commit в общей main

```bash
git revert <commit>
```

Так history остаётся append-only.

## 15.13 Убрать локальный commit, но продолжить редактировать

Unstaged:

```bash
git reset --mixed HEAD~1
```

Staged:

```bash
git reset --soft HEAD~1
```

## 15.14 Посмотреть remote перед интеграцией

```bash
git fetch
git log --oneline --graph --decorate --all
git log HEAD..origin/main
git diff HEAD...origin/main
```

И только потом выбрать merge/rebase.

## 15.15 Feature сильно отстала от main

**Merge main → feature**, если:

- branch уже shared;
- topology важна;
- rewrite нежелателен.

**Rebase feature → main**, если:

- feature приватная/локальная;
- команда предпочитает linear history;
- допустимо переписать feature commits.

## 15.16 Проверить старый commit без движения существующих веток

Быстро:

```bash
git switch --detach <commit>
```

Если нужно параллельно сохранить текущий рабочий каталог — `git worktree`.

## 15.17 `branch -d` отказывается удалять ветку

Git считает, что commits не были интегрированы в релевантную history.

Сначала проверить:

```bash
git log main..feature
git diff main...feature
```

Если работа действительно не нужна — только тогда `git branch -D`.

## 15.18 Merge прошёл без conflict, но тесты упали

Это **semantic conflict**: Git смог объединить текст, но не понимает совместимость API/логики.

На собеседовании это хороший ответ на вопрос «гарантирует ли успешный merge корректность программы?» — нет.

## 15.19 Найти regression

При наличии автоматического теста:

```bash
git bisect run <test-command>
```

Это лучше линейного перебора истории.

## 15.20 Найти, когда исчез вызов функции

Начать:

```bash
git log -S'functionName'
```

Если нужна regex-семантика diff:

```bash
git log -G'functionName\('
```

Далее `git show`, при необходимости `git blame`.

## 15.21 Одинаковый код, разные commit hashes

Вполне возможно. Commit ID включает не только tree, но и parents/metadata/message.

То же объясняет, почему cherry-pick создаёт новый ID.

## 15.22 Squash merge и ancestry

После squash итоговый code может быть в `main`, но исходные feature commits не становятся её ancestors.

Проверять только hash ancestry в таком workflow недостаточно; иногда нужно сравнивать patches/trees.

## 15.23 Сравнить итоговые snapshots двух branches

```bash
git diff branchA branchB
```

Если diff пустой, tips имеют одинаковый файловый результат, хотя commit histories могут различаться.

## 15.24 Untracked после reset hard

Обычно останется. `git clean` — отдельный механизм удаления untracked.

## 15.25 Staged + unstaged одного файла

Посмотреть staged:

```bash
git diff --staged
```

unstaged:

```bash
git diff
```

## 15.26 Одновременно нужны `main` и `release`

```bash
git worktree add ../release-tree release
```

## 15.27 Monorepo нужен только backend

Рассмотреть:

- sparse checkout — уменьшить working tree;
- partial clone — уменьшить загруженные objects.

## 15.28 Submodule после clone пустой

```bash
git submodule update --init --recursive
```

## 15.29 Проверить, содержит ли release конкретный fix

По commit ancestry:

```bash
git merge-base --is-ancestor <fix> <release-tip>
```

Но после cherry-pick/squash исходный commit ID может отсутствовать.

Для patch-equivalence:

```bash
git cherry -v <upstream> <head>
```

## 15.30 «Какая команда Git самая опасная?»

Лучший ответ: **опасность зависит от контекста**.

Частые кандидаты:

```text
git reset --hard
git clean -fdx
git push --force
history rewrite опубликованных commits
```

Первые две могут уничтожить локальные незакоммиченные данные; force/history rewrite — чужую/shared history.

## 15.31 Общий принцип безопасного history rewrite

Перед `rebase`, `reset`, amend старых commits или force push:

```text
1. Есть ли незакоммиченная работа?
2. Опубликованы ли эти commits?
3. Кто-то другой строит на них работу?
4. Есть ли backup ref?
5. Актуален ли remote state после fetch?
6. Можно ли решить задачу append-only способом (`revert`)?
```

## 15.32 «Пропал commit после rebase»

Порядок диагностики:

```bash
git log --oneline --graph --decorate --all
git reflog
```

Commit мог:

- получить новый ID;
- стать empty;
- быть распознан как уже применённый patch;
- быть skipped;
- остаться доступным только через старый reflog entry.

## 15.33 Почему `git pull` неожиданно создал merge commit

Локальная и upstream histories разошлись, а pull настроен на merge. После fetch Git интегрировал их обычным merge.

Избежать неожиданного merge:

```bash
git pull --ff-only
```

или осознанно выбрать:

```bash
git pull --rebase
```

## 15.34 Почему нельзя автоматически rebase'ить shared main

Потому что rebase меняет IDs опубликованных commits и topology. Если команда использует merge commits как часть workflow/audit, механический rebase ломает договорённости и синхронизацию.

## 15.35 Delete/modify conflict

От общего ancestor:

- branch A удаляет файл;
- branch B изменяет его.

Git не знает, что является правильным бизнес-решением: принять удаление или сохранить изменённую версию.

## 15.36 Rename/rename conflict

Один исходный path был переименован двумя сторонами по-разному, либо rename взаимодействует с другими структурными изменениями неоднозначно.

## 15.37 Почему rebase конфликтует каждый commit, а merge один раз

Rebase применяет sequence patches по одному. Merge рассматривает финальные tips одновременно относительно merge base.

## 15.38 Когда squash полезен

Полезен:

- много WIP/fixup commits;
- промежуточная history не несёт ценности;
- команда хочет один логический commit на PR.

Вреден, если отдельные commits важны для:

- `revert`;
- `bisect`;
- code review/audit;
- понимания причинности.

## 15.39 Release tag на не-HEAD commit

Можно:

```bash
git tag -a v2.0.0 <commit> -m 'Release 2.0.0'
```

Затем push tag.

## 15.40 «Working tree clean» — что это реально значит

Обычно означает, что нет показываемых изменений tracked-файлов относительно index/HEAD и нет показываемых untracked.

Но это **не означает**:

- что branch синхронизирована с remote;
- что ignored-файлы отсутствуют;
- что внешняя БД/сервисы/генерируемое состояние совпадают.

---

# 16. Шпаргалка выбора команды

## Отмена/исправление

| Ситуация | Обычно использовать |
|---|---|
| отменить unstaged изменения файла | `git restore <file>` |
| убрать файл из staging | `git restore --staged <file>` |
| убрать локальный commit, оставить staged | `git reset --soft HEAD~1` |
| убрать локальный commit, оставить unstaged | `git reset --mixed HEAD~1` |
| локально уничтожить commit + tracked changes | `git reset --hard ...` |
| отменить published commit | `git revert <commit>` |
| найти старый HEAD после reset/rebase | `git reflog` |

## Интеграция изменений

| Задача | Инструмент |
|---|---|
| сохранить topology и объединить ветки | `merge` |
| перенести private commit chain на новую базу | `rebase` |
| взять отдельный commit | `cherry-pick` |
| объединить WIP history в итоговое изменение | squash / interactive rebase |

## Remote

| Задача | Команда |
|---|---|
| получить remote state без интеграции | `git fetch` |
| получить + автоматически интегрировать | `git pull` |
| pull без merge commit | `git pull --ff-only` |
| pull с rebase локальных commits | `git pull --rebase` |
| отправить branch | `git push` |
| обновить rebased private branch | `git push --force-with-lease` |
| убрать stale remote refs | `git fetch --prune` |

## Диагностика

| Что выяснить | Команда |
|---|---|
| текущее состояние | `git status` |
| staged diff | `git diff --staged` |
| unstaged diff | `git diff` |
| граф | `git log --graph --oneline --decorate --all` |
| commits только в B | `git log A..B` |
| уникальные commits двух сторон | `git log --left-right A...B` |
| кто последним менял строки | `git blame` |
| где изменилось количество строки | `git log -S` |
| где diff match'ит regex | `git log -G` |
| найти regression | `git bisect` |
| проверить ancestry | `git merge-base --is-ancestor` |
| найти потерянный tip | `git reflog` |
| исследовать object | `git cat-file -p` |

---

# Финальная модель, которую стоит уметь проговорить на собеседовании

Если нужно объяснить Git в нескольких предложениях:

> Git хранит immutable content-addressed objects. Commit указывает на snapshot дерева и parent commits. Branches и tags — refs на objects, а `HEAD` определяет текущую позицию/ветку. Working tree — то, что редактируется на диске, index — отдельный snapshot будущего commit. Merge объединяет histories, rebase переписывает commits поверх новой базы, reset двигает refs и при необходимости index/working tree, а revert отменяет изменение новым commit. Remote-tracking refs отражают последнее известное состояние remote и обновляются через fetch. Большинство «потерянных» commit'ов после локального rewrite можно искать через reflog, пока objects не были очищены.

> [!tip] Что реально проверяют на собеседовании
> Не количество запомненных flags, а способность ответить на три вопроса:
> 1. **Что сейчас находится в working tree, index и repository?**
> 2. **Куда указывают HEAD и refs до и после команды?**
> 3. **Операция создаёт новый commit, двигает ref или уничтожает незакоммиченное состояние?**
>
> Если эти три пункта понятны, большую часть сложных Git-сценариев можно вывести без зубрёжки.
