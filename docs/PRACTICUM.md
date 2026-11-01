# Дисковое хранилище на Go: создаём, проверяем, дорабатываем

## 0. Что получится и как проходит работа

В конце у вас будет консольная программа `bin/kv`. Она сохраняет пары «ключ → значение» в папке на диске. Например, по ключу `city` можно записать `Москва`, закрыть программу и прочитать это значение при следующем запуске.

Готовое хранилище поддерживает запись, чтение, удаление, поиск по началу ключа, пакеты операций, снимки, статистику и сессии JSONL. В папке базы будут лежать журнал `wal.log`, снимок `snapshot.kv` и файл блокировки `LOCK`.

Сразу собирать готовую версию не будем. Мы последовательно пройдём шесть этапов:

| Этап | Что добавляем | Что проверяем |
|---|---|---|
| [1. Память](#stage-1) | словарь и команды | почему запись исчезает после завершения программы |
| [2. Журнал](#stage-2) | сохранение операций в файл | как пережить перезапуск и почему неполная запись мешает чтению |
| [3. Восстановление](#stage-3) | удаление только недописанного хвоста | можно ли вернуть доступ к предыдущим записям |
| [4. Контрольные суммы](#stage-4) | длину и CRC32 записи | как отличить неполную запись от повреждённой |
| [5. Снимок](#stage-5) | сохранение текущего состояния | как сократить журнал и почему двух писателей пока нельзя допускать |
| [6. Блокировка](#stage-6) | исключительное открытие базы | как предотвратить потерю записи при двух процессах |

На каждом этапе вы добавите или замените несколько файлов, соберёте программу и проведёте опыт. Перед запуском попробуйте предсказать результат. После него объясните, что изменилось и какая проблема ещё осталась. В следующих шагах будем возвращаться к этим проблемам.

Весь код и команды находятся в этом документе. Код дан готовыми блоками. Их можно копировать в Terminal, знать Go заранее не обязательно. Основная работа — пройти изменения по порядку, увидеть ограничения ранних версий и проверить исправления. Все этапы выполняются **в одной новой рабочей папке**. Промежуточные программы сохраняются отдельно, поэтому к их поведению можно вернуться.

Версии 1–5 нужны для опытов и намеренно неполны. Для дальнейшего использования предназначена версия после этапа 6. Все опыты создают отдельные временные базы. Значения — строки, текущее состояние базы хранится в памяти. Проверки аварий относятся к завершению процесса, физическое отключение питания не испытывается.

Прохождение нужно записать на видео. Нажмите `Shift+Command+5`, выберите область записи. В меню «Параметры» выберите микрофон, чтобы были слышны ваши объяснения, затем нажмите «Запись». Дальше выполняйте шаги по порядку. Чтобы закончить запись, нажмите кнопку остановки в строке меню macOS. Отдельного сценария видео или письменного отчёта нет.

### Инструменты и материалы

Нужны Mac, доступ в интернет для установки инструментов и Terminal. Go устанавливается в разделе 2, Command Line Tools — в разделе 3. Платные сервисы, Docker и внешняя база данных не используются.

- [Go: загрузки](https://go.dev/dl/) и [официальная инструкция](https://go.dev/doc/install). В этом задании используется установка в папку проекта, все команды приведены ниже.
- [Apple Command Line Tools](https://developer.apple.com/documentation/xcode/installing-the-command-line-tools/) — компилятор для запуска тестов с детектором гонок.
- [Краткое знакомство с Go](https://go.dev/tour/welcome/1) — справка, если хочется разобраться в синтаксисе готовых блоков.
- [Детектор гонок](https://go.dev/doc/articles/race_detector.html) и [fuzzing](https://go.dev/doc/tutorial/fuzz) — пояснения к проверкам раздела 11.
- [Спецификация хранилища](SPEC.md) — команды, форматы файлов и границы гарантий итоговой версии.

<a id="assessment"></a>

### Критерии оценивания

Основной маршрут — разделы 1–12 без дополнительных пунктов 11.6–11.7. В нём собирается весь инструмент. Уровни П и Э добавляют проверки уже работающего хранилища.

| Уровень | Что должно быть подтверждено |
|---|---|
| **Б−** | Собраны все шесть версий. Итоговая программа сохраняет данные после перезапуска, выполняет пакет и сессию из 11.1–11.2. В видео показан этот результат, но часть опытов с восстановлением, повреждением, снимком или блокировкой не выполнена либо не объяснена. Это неполное прохождение основного маршрута. |
| **Б** | Основной маршрут пройден полностью. Показаны опыты всех шести этапов, проверки 11.1–11.5 и сохранённый проект. `go vet` и тесты с `-race` проходят. Ожидаемые ошибки в опытах отличены от неисправностей программы. |
| **Б+** | Выполнено всё для Б. На своих ключе и значении повторён опыт 6.3. В видео объяснены все шесть механизмов из таблицы ниже, включая различие между отказом неверного пакета и восстановлением после аварии. |
| **П** | Выполнено всё для Б+. Пройден пункт 11.6: показаны три точки аварии при снимке и отказ создания временного файла. Объяснено, откуда восстанавливаются данные в каждом случае и почему не потеряна единственная актуальная копия. |
| **Э** | Выполнено всё для П. По пункту 11.7 добавлен собственный воспроизводимый сценарий с автоматическими проверками. До запуска сформулирован ожидаемый результат, после запуска объяснены наблюдения и границы проверенной гарантии. |

Для Б+ объясните на результатах своих опытов:

| Механизм | Вопрос |
|---|---|
| Память и журнал | Почему версия 1 теряет запись, а версия 2 читает её новым процессом? |
| Восстановление хвоста | Почему неполную последнюю запись можно отбросить, а полную запись с неверной CRC нельзя молча пропустить? |
| Контрольная сумма | Почему корректный JSON с другим значением принимался на этапе 3 и отклонялся на этапе 4? |
| Снимок | Почему журнал очищается после публикации снимка и откуда берутся данные после следующего запуска? |
| Блокировка | Почему второй процесс терял чужую запись и как `flock` предотвращает этот сценарий? |
| Пакет | Что показывает неверный пакет в 11.1? Почему для проверки аварии посреди записи дополнительно нужны тесты 11.3? |

Каждый следующий уровень включает требования предыдущего. Дополнительный опыт не компенсирует неработающую обязательную часть. Удалять готовые тесты или ослаблять их проверки ради успешного запуска нельзя. Сам по себе запуск готового кода без объяснения результатов не подтверждает П или Э. Числовые баллы здесь не назначаются.

## 1. Открываем терминал и папку проекта

### 1.1. Открыть Terminal

1. Нажмите `Command+Space`.
2. Введите `Terminal`.
3. Нажмите Return.

Откроется окно с приглашением ввода. После команды нажимайте Return. Блок из нескольких строк можно вставить целиком.

### 1.2. Перейти в папку задания

1. Найдите распакованную папку задания в Finder. Внутри должны быть `README.md` и папка `docs`.
2. В Terminal напечатайте `cd ` с пробелом после `cd`.
3. Перетащите папку из Finder в Terminal.
4. Нажмите Return.

`cd` ничего не печатает. Команда меняет рабочую папку. Проверьте её содержимое:

```bash
pwd
ls
```

**Результат:** путь к папке задания и список с `README.md` и `docs`. Исходники программы появятся позже, когда вы создадите их по инструкции.

### 1.3. Если задание ещё не скачано

Этот пункт нужен перед пунктом 1.2, если на компьютере пока нет папки задания.

1. На странице репозитория GitHub нажмите **Code**, затем **Download ZIP**.
2. В Finder нажмите `Option+Command+L`, чтобы открыть «Загрузки».
3. Дважды нажмите на скачанный ZIP-архив. Если браузер уже распаковал его, используйте появившуюся папку.
4. Перейдите в эту папку в Terminal по пункту 1.2.

**Результат:** задание распаковано, Terminal открыт в папке с `README.md` и `docs`. Название папки может содержать окончание `-main` или `-master`.

### 1.4. Запомнить путь и проверить Mac

Дальше работаем в одном окне Terminal:

```bash
project_dir="$PWD"
printf 'Папка проекта: %s\n' "$project_dir"
uname -s
uname -m
```

**Результат:** путь проекта, `Darwin` и архитектура Mac. Обычно это `arm64` на Apple Silicon или `x86_64` на Intel. Переменная `project_dir` позволит вернуться в эту папку позже.

Символ `$` внутри команд нужен: так оболочка подставляет значение переменной. Кавычки тоже копируйте. Строки с `#` в начале — комментарии.

## 2. Устанавливаем Go

### 2.1. Создать установщик

Этот скрипт скачивает Go с официального сайта, проверяет SHA-256 и устанавливает его в `.tools` внутри проекта. Сначала создайте файл:

```bash
mkdir -p 'scripts'
cat > 'scripts/install-go-macos.sh' <<'KV_LAB_SOURCE'
#!/bin/bash
# Local, checksum-verified installation. Never edits shell profiles or /usr/local.
set -euo pipefail
cd "$(dirname "$0")/.."
if [ "$(uname -s)" != Darwin ]; then
  echo 'This installer is for macOS; install Go from https://go.dev/dl/ on other systems.' >&2
  exit 1
fi
version=1.27.3
case "$(uname -m)" in
  arm64) arch=arm64; expected=dd3908cf20959d798c1e06ebe5b72213f0505ce01373b40ce3110975a6cd99c2 ;;
  x86_64) arch=amd64; expected=b6f8e931e4ab497fcde6da8ae4499a58f9d258e55c8a80824a516eae5950a750 ;;
  *) echo 'Unsupported Mac architecture' >&2; exit 1 ;;
esac
destination="$PWD/.tools/go$version"
if [ -e "$destination" ]; then
  # A native arm64 Go also runs when the invoking shell is under Rosetta.
  # Validate the runnable toolchain, not equality with that shell's uname.
  installed=''
  if [ -x "$destination/go/bin/go" ]; then
    installed=$("$destination/go/bin/go" version 2>/dev/null || true)
  fi
  case "$installed" in
    "go version go$version darwin/arm64"|"go version go$version darwin/amd64")
      echo "Already installed: $destination/go/bin/go"
      exit 0 ;;
  esac
  echo "Existing incomplete installation: $destination. Move it aside before retrying." >&2
  exit 1
fi
mkdir -p "$PWD/.tools"
scratch=$(mktemp -d "$PWD/.tools/download.XXXXXX")
trap 'rm -rf "$scratch"' EXIT
archive="$scratch/go.tar.gz"
if [ -n "${KV_GO_ARCHIVE:-}" ]; then
  cp "$KV_GO_ARCHIVE" "$archive"
else
  curl --fail --location --retry 3 --connect-timeout 15 --max-time 180 \
    "https://go.dev/dl/go$version.darwin-$arch.tar.gz" --output "$archive"
fi
actual=$(shasum -a 256 "$archive")
actual=${actual%% *}
if [ "$actual" != "$expected" ]; then
  echo "SHA-256 mismatch: expected $expected, got $actual" >&2
  exit 1
fi
tar -xzf "$archive" -C "$scratch"
"$scratch/go/bin/go" version
rm "$archive"
mv "$scratch" "$destination"
trap - EXIT
echo "Installed without sudo: $destination/go/bin/go"
echo 'For this terminal: export PATH="$PWD/.tools/go1.27.3/go/bin:$PATH"'
KV_LAB_SOURCE
printf 'Записан файл: scripts/install-go-macos.sh\n'
```

**Результат:** `Записан файл: scripts/install-go-macos.sh`. Файл создан. Команды `mkdir` и `cat` при успехе ничего не печатают.

На этом шаге Go ещё не скачивается. Установка начнётся после команды из следующего пункта.

### 2.2. Запустить установку

```bash
bash scripts/install-go-macos.sh
```

**Результат:** при первой установке появится прогресс скачивания, затем версия Go и строка `Installed without sudo`. Если Go уже установлен этим скриптом, появится `Already installed`.

При обрыве скачивания повторите эту же команду. При несовпадении SHA-256 скрипт не устанавливает полученный архив.

### 2.3. Включить Go в текущем терминале

```bash
export PATH="$project_dir/.tools/go1.27.3/go/bin:$PATH"
export GOTOOLCHAIN=local
go version
go env GOOS GOARCH GOROOT
command -v go
```

**Результат на Apple Silicon:**

```text
go version go1.27.3 darwin/arm64
darwin
arm64
.../.tools/go1.27.3/go
.../.tools/go1.27.3/go/bin/go
```

На Intel вместо `arm64` будет `amd64`. Многоточие обозначает путь к папке задания, вводить его не нужно. Команды `export` сами ничего не печатают.

Настройка действует только в текущем окне. В новом Terminal снова перейдите в папку проекта по пункту 1.2 и выполните:

```bash
project_dir="$PWD"
export PATH="$project_dir/.tools/go1.27.3/go/bin:$PATH"
export GOTOOLCHAIN=local
go version
```

**Результат:** версия `go1.27.3`. Переменную `GOROOT` отдельно задавать не нужно.

## 3. Подготавливаем инструменты для тестов

Детектор гонок Go использует C-компилятор. На Mac он поставляется с Command Line Tools. Полный Xcode для этого проекта не требуется.

### 3.1. Проверить Command Line Tools

```bash
xcode-select -p
xcrun --find clang
clang --version
```

**Результат:** пути к инструментам и версия Apple clang. Первый путь может быть `/Library/Developer/CommandLineTools`. Путь к полному Xcode тоже допустим.

### 3.2. Если инструменты не найдены

```bash
xcode-select --install
```

**Результат:** macOS откроет окно установки. Нажмите «Установить» и дождитесь завершения. Если система сообщает, что инструменты уже есть, переходите к следующему блоку.

```bash
xcrun --find clang
export CGO_ENABLED=1
go env CGO_ENABLED
```

**Результат:** путь к clang и число `1`. Если инструменты уже были установлены в пункте 3.1, этот блок тоже можно выполнить. `export` ничего не печатает.

## 4. Создаём рабочую папку

В папке задания находятся документы. Теперь создадим отдельную папку для вашей реализации. Выполните в том же окне Terminal:

```bash
work_dir=$(mktemp -d "${TMPDIR:-/tmp}/kv-workshop-build.XXXXXX")
cd "$work_dir"
go mod init example.com/kv-lab
printf 'Рабочая папка: %s\n' "$work_dir"
```

**Результат:** сообщение `creating new go.mod: module example.com/kv-lab` и путь к новой папке. В ней пока один файл — `go.mod`.

Переменная `project_dir` хранит путь к папке задания, `work_dir` — к вашей реализации. Не закрывайте это окно до завершения работы. Если окно закрыли, нужную папку можно снова перетащить из Finder после команды `cd `, как в пункте 1.2. Переменные придётся задать заново из её фактического пути.

Все команды создания файлов приведены ниже, прямо в соответствующих этапах. Каждый блок содержит команды `mkdir`, `cat`, весь текст файла и `printf`. Копируйте их вместе, включая строку `KV_LAB_SOURCE` в конце. Команда сама создаст файл. Если файл уже есть, его содержимое заменится.

<a id="stage-1"></a>

## 5. Этап 1. Хранилище только в памяти

### 5.1. Добавить код и собрать программу

Сначала нужна программа, которую можно запустить и проверить. Этот этап создаёт словарь в памяти и готовую оболочку команд. Журнала, снимков и блокировки пока нет. Начните с `Store` и `Batch`: запись сейчас меняет только `map`. Код CLI одинаков на всех этапах, чтобы команды проверки оставались знакомыми.

Выполните следующие блоки по порядку в папке `$work_dir`. Каждый блок создаст или заменит один файл.

#### 5.1.1. go.mod

Имя модуля позволяет частям проекта находить друг друга через `import`. Этот блок заменит `go.mod`, созданный командой `go mod init`.

```bash
mkdir -p '.'
cat > 'go.mod' <<'KV_LAB_SOURCE'
module example.com/kv-lab

go 1.23.0
KV_LAB_SOURCE
printf 'Записан файл: go.mod\n'
```

**Результат:** `Записан файл: go.mod`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 5.1.2. kv/errors.go

Общие типы операций, ошибок и статистики. Часть ошибок понадобится на следующих этапах.

```bash
mkdir -p 'kv'
cat > 'kv/errors.go' <<'KV_LAB_SOURCE'
// Package kv implements a local, durable string-to-string store for macOS/Linux.
// The entire live dataset resides in memory. All operations on a Store are
// serialized; a directory can be opened by only one process at a time.
package kv

import "errors"

var (
	ErrNotFound  = errors.New("key not found")
	ErrClosed    = errors.New("store is closed")
	ErrLocked    = errors.New("store is already open")
	ErrCorrupt   = errors.New("corrupt store")
	ErrInvalid   = errors.New("invalid input")
	ErrUncertain = errors.New("I/O outcome uncertain; close and reopen the store")
)

const (
	MaxKeyBytes      = 1024
	MaxValueBytes    = 1 << 20
	MaxBatchOps      = 1000
	MaxRecordBytes   = 16 << 20
	MaxSnapshotBytes = 256 << 20
)

// Op is one member of an atomic batch. A delete of a missing key is a no-op.
// Empty string values are valid. An empty key is not.
type Op struct {
	Type  string `json:"type"`
	Key   string `json:"key"`
	Value string `json:"value,omitempty"`
}

type Pair struct {
	Key   string `json:"key"`
	Value string `json:"value"`
}

type Stats struct {
	Keys               int    `json:"keys"`
	Sequence           uint64 `json:"sequence"`
	SnapshotSequence   uint64 `json:"snapshot_sequence"`
	WALBytes           int64  `json:"wal_bytes"`
	RecoveredTailBytes int64  `json:"recovered_tail_bytes"`
}
KV_LAB_SOURCE
printf 'Записан файл: kv/errors.go\n'
```

**Результат:** `Записан файл: kv/errors.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 5.1.3. kv/format.go

Проверка ключей и значений, затем применение операций к словарю. Дискового формата здесь ещё нет.

```bash
mkdir -p 'kv'
cat > 'kv/format.go' <<'KV_LAB_SOURCE'
package kv

import (
	"fmt"
	"unicode/utf8"
)

func validateKey(key string) error {
	if len(key) == 0 || len(key) > MaxKeyBytes || !utf8.ValidString(key) {
		return fmt.Errorf("%w: key must be UTF-8, 1..%d bytes", ErrInvalid, MaxKeyBytes)
	}
	return nil
}

func validateOps(ops []Op) error {
	if len(ops) == 0 || len(ops) > MaxBatchOps {
		return fmt.Errorf("%w: batch requires 1..%d operations", ErrInvalid, MaxBatchOps)
	}
	for _, op := range ops {
		if err := validateKey(op.Key); err != nil {
			return err
		}
		switch op.Type {
		case "set":
			if len(op.Value) > MaxValueBytes || !utf8.ValidString(op.Value) {
				return fmt.Errorf("%w: value must be UTF-8 and at most %d bytes", ErrInvalid, MaxValueBytes)
			}
		case "delete":
			if op.Value != "" {
				return fmt.Errorf("%w: delete cannot contain a value", ErrInvalid)
			}
		default:
			return fmt.Errorf("%w: unknown operation %q", ErrInvalid, op.Type)
		}
	}
	return nil
}

func apply(data map[string]string, ops []Op) {
	for _, op := range ops {
		if op.Type == "set" {
			data[op.Key] = op.Value
		} else {
			delete(data, op.Key)
		}
	}
}
KV_LAB_SOURCE
printf 'Записан файл: kv/format.go\n'
```

**Результат:** `Записан файл: kv/format.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 5.1.4. kv/store.go

Основной файл этапа. `Open` создаёт пустой словарь. `Batch` меняет его, а `Get` читает. После завершения процесса словарь исчезает. `Snapshot` пока возвращает сообщение о том, что снимки появятся на этапе 5.

```bash
mkdir -p 'kv'
cat > 'kv/store.go' <<'KV_LAB_SOURCE'
package kv

import (
	"fmt"
	"math"
	"sort"
	"strings"
	"sync"
)

// Stage 1 stores data only in memory. The directory is not used yet.
type Store struct {
	mu     sync.Mutex
	data   map[string]string
	seq    uint64
	closed bool
}

func Open(dir string) (*Store, error) {
	if dir == "" {
		return nil, fmt.Errorf("%w: directory is empty", ErrInvalid)
	}
	return &Store{data: make(map[string]string)}, nil
}
func (s *Store) stateError() error {
	if s.closed {
		return ErrClosed
	}
	return nil
}
func (s *Store) Get(key string) (string, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return "", err
	}
	if err := validateKey(key); err != nil {
		return "", err
	}
	v, ok := s.data[key]
	if !ok {
		return "", ErrNotFound
	}
	return v, nil
}

func (s *Store) Set(key, value string) error {
	return s.Batch([]Op{{Type: "set", Key: key, Value: value}})
}

func (s *Store) Delete(key string) error { return s.Batch([]Op{{Type: "delete", Key: key}}) }

func (s *Store) List(prefix string) ([]Pair, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return nil, err
	}
	pairs := make([]Pair, 0)
	for k, v := range s.data {
		if strings.HasPrefix(k, prefix) {
			pairs = append(pairs, Pair{k, v})
		}
	}
	sort.Slice(pairs, func(i, j int) bool { return pairs[i].Key < pairs[j].Key })
	return pairs, nil
}

func (s *Store) Batch(ops []Op) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return err
	}
	if err := validateOps(ops); err != nil {
		return err
	}
	if s.seq == math.MaxUint64 {
		return fmt.Errorf("%w: sequence exhausted", ErrInvalid)
	}
	apply(s.data, ops)
	s.seq++
	return nil
}
func (s *Store) Stats() (Stats, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return Stats{}, err
	}
	return Stats{Keys: len(s.data), Sequence: s.seq}, nil
}
func (s *Store) Snapshot() error {
	return fmt.Errorf("%w: snapshots are added at stage 5", ErrInvalid)
}
func (s *Store) Close() error {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.closed = true
	return nil
}
KV_LAB_SOURCE
printf 'Записан файл: kv/store.go\n'
```

**Результат:** `Записан файл: kv/store.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 5.1.5. internal/cli/cli.go

Готовая оболочка для команд из терминала. Она вызывает методы `Store` и печатает JSON. В справке уже перечислены команды итоговой версии, но снимки пока недоступны. Подробно читать этот файл для опыта с памятью не требуется.

```bash
mkdir -p 'internal/cli'
cat > 'internal/cli/cli.go' <<'KV_LAB_SOURCE'
// Package cli implements a testable CLI. Run never calls os.Exit.
package cli

import (
	"bufio"
	"bytes"
	"encoding/json"
	"errors"
	"flag"
	"fmt"
	"io"
	"os"

	"example.com/kv-lab/kv"
)

const usage = `kv-lab 1.0.0 — durable local key-value store

Usage: kv [-dir PATH] COMMAND [ARGS]
  set KEY VALUE       save a UTF-8 string, including an empty value
  get KEY             read a value (exit 3 when missing)
  delete KEY          delete a key (missing is a successful no-op)
  list [PREFIX]       sorted key/value pairs
  batch FILE|-        atomically apply a JSON array of operations
  snapshot            persist state and compact the WAL
  stats               key count, sequences and WAL size
  check               open, validate and recover an incomplete WAL tail
  shell               read JSON commands, one per line; EOF closes store
  help                show this help
  version             print the version

Flags must appear BEFORE the command. Default directory: ./data
All successful data responses are JSON. Errors go to stderr.
check and all other data commands perform normal recovery on open.
`

func exitCode(err error) int {
	switch {
	case errors.Is(err, kv.ErrInvalid):
		return 2
	case errors.Is(err, kv.ErrNotFound):
		return 3
	case errors.Is(err, kv.ErrLocked):
		return 4
	case errors.Is(err, kv.ErrCorrupt):
		return 5
	case errors.Is(err, kv.ErrUncertain):
		return 6
	default:
		return 1
	}
}

func report(w io.Writer, err error) int { _, _ = fmt.Fprintln(w, err); return exitCode(err) }
func badArgs() error                    { return fmt.Errorf("%w: wrong arguments; use kv help", kv.ErrInvalid) }

type request struct {
	Command string  `json:"command"`
	Key     string  `json:"key,omitempty"`
	Value   string  `json:"value,omitempty"`
	Prefix  string  `json:"prefix,omitempty"`
	Ops     []kv.Op `json:"ops,omitempty"`
}

func strictJSON(b []byte, dst any) error {
	d := json.NewDecoder(bytes.NewReader(b))
	d.DisallowUnknownFields()
	if err := d.Decode(dst); err != nil {
		return fmt.Errorf("%w: JSON: %v", kv.ErrInvalid, err)
	}
	var extra any
	if err := d.Decode(&extra); err != io.EOF {
		return fmt.Errorf("%w: trailing JSON", kv.ErrInvalid)
	}
	return nil
}

func execute(s *kv.Store, r request) (any, error) {
	ok := map[string]bool{"ok": true}
	switch r.Command {
	case "set":
		return ok, s.Set(r.Key, r.Value)
	case "get":
		v, err := s.Get(r.Key)
		return kv.Pair{Key: r.Key, Value: v}, err
	case "delete":
		return ok, s.Delete(r.Key)
	case "list":
		return s.List(r.Prefix)
	case "batch":
		return ok, s.Batch(r.Ops)
	case "snapshot":
		return ok, s.Snapshot()
	case "stats", "check":
		return s.Stats()
	default:
		return nil, fmt.Errorf("%w: unknown command %q", kv.ErrInvalid, r.Command)
	}
}

func validateRequest(r request) error {
	switch r.Command {
	case "set":
		if r.Prefix != "" || r.Ops != nil {
			return badArgs()
		}
	case "get", "delete":
		if r.Value != "" || r.Prefix != "" || r.Ops != nil {
			return badArgs()
		}
	case "list":
		if r.Key != "" || r.Value != "" || r.Ops != nil {
			return badArgs()
		}
	case "batch":
		if r.Key != "" || r.Value != "" || r.Prefix != "" {
			return badArgs()
		}
	case "stats", "check", "snapshot":
		if r.Key != "" || r.Value != "" || r.Prefix != "" || r.Ops != nil {
			return badArgs()
		}
	default:
		return fmt.Errorf("%w: unknown command %q", kv.ErrInvalid, r.Command)
	}
	return nil
}

func shell(s *kv.Store, in io.Reader, out io.Writer) error {
	scanner := bufio.NewScanner(in)
	scanner.Buffer(make([]byte, 4096), kv.MaxRecordBytes+1024)
	enc := json.NewEncoder(out)
	var lastErr error
	for scanner.Scan() {
		if len(bytes.TrimSpace(scanner.Bytes())) == 0 {
			continue
		}
		var r request
		err := strictJSON(scanner.Bytes(), &r)
		if err == nil {
			err = validateRequest(r)
		}
		var result any
		if err == nil {
			result, err = execute(s, r)
		}
		if err != nil {
			lastErr = err
			result = map[string]any{"ok": false, "error": err.Error(), "code": exitCode(err)}
		}
		if err = enc.Encode(result); err != nil {
			return err
		}
	}
	if err := scanner.Err(); err != nil {
		return fmt.Errorf("read shell input: %w", err)
	}
	return lastErr
}

// Run executes a command. It accepts redirected streams for deterministic tests.
func Run(args []string, in io.Reader, out, errOut io.Writer) (code int) {
	fs := flag.NewFlagSet("kv", flag.ContinueOnError)
	fs.SetOutput(errOut)
	dir := fs.String("dir", "./data", "store directory")
	fs.Usage = func() { _, _ = io.WriteString(errOut, usage) }
	if err := fs.Parse(args); err != nil {
		if errors.Is(err, flag.ErrHelp) {
			return 0
		}
		return 2
	}
	a := fs.Args()
	if len(a) == 0 {
		_, _ = io.WriteString(errOut, usage)
		return 2
	}
	if a[0] == "help" && len(a) == 1 {
		_, err := io.WriteString(out, usage)
		if err != nil {
			return report(errOut, err)
		}
		return 0
	}
	if a[0] == "version" && len(a) == 1 {
		_, err := io.WriteString(out, "kv-lab 1.0.0\n")
		if err != nil {
			return report(errOut, err)
		}
		return 0
	}
	r := request{Command: a[0]}
	switch a[0] {
	case "set":
		if len(a) != 3 {
			return report(errOut, badArgs())
		}
		r.Key, r.Value = a[1], a[2]
	case "get", "delete":
		if len(a) != 2 {
			return report(errOut, badArgs())
		}
		r.Key = a[1]
	case "list":
		if len(a) > 2 {
			return report(errOut, badArgs())
		}
		if len(a) == 2 {
			r.Prefix = a[1]
		}
	case "batch":
		if len(a) != 2 {
			return report(errOut, badArgs())
		}
		var src io.Reader = in
		if a[1] != "-" {
			f, err := os.Open(a[1])
			if err != nil {
				return report(errOut, err)
			}
			defer f.Close()
			src = f
		}
		b, err := io.ReadAll(io.LimitReader(src, kv.MaxRecordBytes+1))
		if err != nil {
			return report(errOut, err)
		}
		if len(b) > kv.MaxRecordBytes {
			return report(errOut, fmt.Errorf("%w: input file too large", kv.ErrInvalid))
		}
		if err = strictJSON(b, &r.Ops); err != nil {
			return report(errOut, err)
		}
	case "snapshot", "stats", "check", "shell":
		if len(a) != 1 {
			return report(errOut, badArgs())
		}
	default:
		return report(errOut, fmt.Errorf("%w: unknown command %q", kv.ErrInvalid, a[0]))
	}
	s, err := kv.Open(*dir)
	if err != nil {
		return report(errOut, err)
	}
	defer func() {
		if err := s.Close(); err != nil && code == 0 {
			code = report(errOut, err)
		}
	}()
	if r.Command == "shell" {
		if err = shell(s, in, out); err != nil {
			return report(errOut, err)
		}
		return 0
	}
	result, err := execute(s, r)
	if err != nil {
		return report(errOut, err)
	}
	if err = json.NewEncoder(out).Encode(result); err != nil {
		return report(errOut, err)
	}
	return 0
}
KV_LAB_SOURCE
printf 'Записан файл: internal/cli/cli.go\n'
```

**Результат:** `Записан файл: internal/cli/cli.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 5.1.6. cmd/kv/main.go

Точка запуска. Она передаёт аргументы и потоки ввода-вывода в обработчик команд.

```bash
mkdir -p 'cmd/kv'
cat > 'cmd/kv/main.go' <<'KV_LAB_SOURCE'
package main

import (
	"example.com/kv-lab/internal/cli"
	"os"
)

func main() { os.Exit(cli.Run(os.Args[1:], os.Stdin, os.Stdout, os.Stderr)) }
KV_LAB_SOURCE
printf 'Записан файл: cmd/kv/main.go\n'
```

**Результат:** `Записан файл: cmd/kv/main.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

**Сборка этапа**


```bash
gofmt -w kv internal cmd
go build -trimpath -o bin/kv ./cmd/kv
cp bin/kv bin/kv-01-memory
printf 'Собрана версия 1: память\n'
```

**Результат:** `Собрана версия 1: память`. Остальные команды при успехе ничего не выводят. `bin/kv` — текущая программа, `bin/kv-01-memory` — сохранённая копия этого этапа.

### 5.2. Записать и прочитать в одном процессе

```bash
memory_dir=$(mktemp -d "${TMPDIR:-/tmp}/kv-memory.XXXXXX")
printf '%s\n' \
  '{"command":"set","key":"city","value":"Москва"}' \
  '{"command":"get","key":"city"}' |
  ./bin/kv -dir "$memory_dir" shell
```

**Результат:**

```json
{"ok":true}
{"key":"city","value":"Москва"}
```

`printf` передал две команды одному процессу. После конца ввода сессия завершилась. Пока процесс работал, обе команды использовали один словарь.

### 5.3. Прочитать после завершения процесса

Перед запуском подумайте: где сейчас может находиться записанный город?

```bash
missing_code=0
./bin/kv -dir "$memory_dir" get city || missing_code=$?
printf 'Код завершения: %s\n' "$missing_code"
test "$missing_code" -eq 3
ls -A "$memory_dir"
```

**Результат:** `key not found` и код `3`. `test` ничего не напечатает, если код совпал. `ls` тоже не покажет файлов: программа ещё ничего не записывает на диск.

Это ожидаемая проблема первой версии. `Open` создаёт новую `map`, поэтому каждое открытие начинается с пустого состояния.

**Перед следующим этапом:** объясните, почему успешный `set` в одной сессии не помог следующему процессу. Нам нужно сохранять изменения вне памяти программы.

<a id="stage-2"></a>

## 6. Этап 2. Сохраняем операции в журнал

### 6.1. Добавить журнал

Теперь запись должна переживать завершение процесса. `Store` получит файл журнала. Новый `journal.go` будет записывать каждый пакет одной строкой JSON и читать строки при открытии. Пока любая незавершённая строка считается ошибкой.

Выполните следующие блоки по порядку в папке `$work_dir`. Каждый блок создаст или заменит один файл.

#### 6.1.1. kv/store.go

В `Open` появился файл, в `Close` — его закрытие. Главное изменение в `Batch`: сначала `writeRecord`, затем `apply`. Ошибка записи запрещает дальнейшие операции до переоткрытия.

```bash
mkdir -p 'kv'
cat > 'kv/store.go' <<'KV_LAB_SOURCE'
package kv

import (
	"errors"
	"fmt"
	"math"
	"os"
	"path/filepath"
	"sort"
	"strings"
	"sync"
)

// Stages 2-4 add a WAL but do not yet implement snapshots or locking.
type Store struct {
	mu            sync.Mutex
	data          map[string]string
	wal           *os.File
	seq           uint64
	recoveredTail int64
	closed        bool
	failed        error
}

func Open(dir string) (*Store, error) {
	if dir == "" {
		return nil, fmt.Errorf("%w: directory is empty", ErrInvalid)
	}
	if err := os.MkdirAll(dir, 0700); err != nil {
		return nil, err
	}
	f, err := os.OpenFile(filepath.Join(dir, "wal.log"), os.O_CREATE|os.O_RDWR, 0600)
	if err != nil {
		return nil, err
	}
	s := &Store{data: make(map[string]string), wal: f}
	if err = syncDir(dir); err == nil {
		err = s.replay()
	}
	if err != nil {
		_ = f.Close()
		return nil, err
	}
	return s, nil
}
func (s *Store) stateError() error {
	if s.closed {
		return ErrClosed
	}
	if s.failed != nil {
		return s.failed
	}
	return nil
}

func (s *Store) poison(err error) error {
	s.failed = fmt.Errorf("%w: %w", ErrUncertain, err)
	return s.failed
}

func (s *Store) Get(key string) (string, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return "", err
	}
	if err := validateKey(key); err != nil {
		return "", err
	}
	v, ok := s.data[key]
	if !ok {
		return "", ErrNotFound
	}
	return v, nil
}

func (s *Store) Set(key, value string) error {
	return s.Batch([]Op{{Type: "set", Key: key, Value: value}})
}

func (s *Store) Delete(key string) error { return s.Batch([]Op{{Type: "delete", Key: key}}) }

func (s *Store) List(prefix string) ([]Pair, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return nil, err
	}
	pairs := make([]Pair, 0)
	for k, v := range s.data {
		if strings.HasPrefix(k, prefix) {
			pairs = append(pairs, Pair{k, v})
		}
	}
	sort.Slice(pairs, func(i, j int) bool { return pairs[i].Key < pairs[j].Key })
	return pairs, nil
}

func (s *Store) Batch(ops []Op) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return err
	}
	if err := validateOps(ops); err != nil {
		return err
	}
	if s.seq == math.MaxUint64 {
		return fmt.Errorf("%w: sequence exhausted", ErrInvalid)
	}
	// A successful answer follows the WAL write and Sync.
	if err := s.writeRecord(ops); err != nil {
		return err
	}
	apply(s.data, ops)
	s.seq++
	return nil
}
func (s *Store) Stats() (Stats, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return Stats{}, err
	}
	info, err := s.wal.Stat()
	if err != nil {
		return Stats{}, err
	}
	return Stats{Keys: len(s.data), Sequence: s.seq, WALBytes: info.Size(), RecoveredTailBytes: s.recoveredTail}, nil
}
func (s *Store) Snapshot() error {
	return fmt.Errorf("%w: snapshots are added at stage 5", ErrInvalid)
}
func (s *Store) Close() error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.closed {
		return nil
	}
	s.closed = true
	return s.wal.Close()
}
func syncDir(dir string) error {
	f, err := os.Open(dir)
	if err != nil {
		return err
	}
	err = f.Sync()
	return errors.Join(err, f.Close())
}
KV_LAB_SOURCE
printf 'Записан файл: kv/store.go\n'
```

**Результат:** `Записан файл: kv/store.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 6.1.2. kv/journal.go

`writeRecord` сериализует пакет, записывает строку и вызывает `Sync`. `replay` читает сохранённые строки и восстанавливает словарь. Обратите внимание на ветку `err == io.EOF`: неполная строка сейчас мешает открыть всю базу.

```bash
mkdir -p 'kv'
cat > 'kv/journal.go' <<'KV_LAB_SOURCE'
package kv

import (
	"bufio"
	"encoding/json"
	"fmt"
	"io"
)

type record struct {
	Version int    `json:"version"`
	Seq     uint64 `json:"seq"`
	Ops     []Op   `json:"ops"`
}

func (s *Store) writeRecord(ops []Op) error {
	payload, err := json.Marshal(record{Version: 1, Seq: s.seq + 1, Ops: ops})
	if err != nil {
		return err
	}
	if len(payload) > MaxRecordBytes {
		return fmt.Errorf("%w: record too large", ErrInvalid)
	}
	line := append(payload, '\n')
	n, err := s.wal.Write(line)
	if err == nil && n != len(line) {
		err = io.ErrShortWrite
	}
	if err == nil {
		err = s.wal.Sync()
	}
	if err != nil {
		return s.poison(err)
	}
	return nil
}
func (s *Store) replay() error {
	reader := bufio.NewReader(s.wal)
	var offset int64
	for {
		line, err := reader.ReadBytes('\n')
		if err == io.EOF && len(line) == 0 {
			break
		}
		if err == io.EOF {
			return fmt.Errorf("%w: incomplete JSON line at %d", ErrCorrupt, offset)
		}
		if err != nil {
			return err
		}
		if len(line) > MaxRecordBytes+1 {
			return fmt.Errorf("%w: record too large", ErrCorrupt)
		}
		var rec record
		if err := json.Unmarshal(line, &rec); err != nil {
			return fmt.Errorf("%w: JSON: %v", ErrCorrupt, err)
		}
		if rec.Version != 1 || rec.Seq == 0 || rec.Seq != s.seq+1 {
			return fmt.Errorf("%w: sequence or version", ErrCorrupt)
		}
		if err := validateOps(rec.Ops); err != nil {
			return fmt.Errorf("%w: %v", ErrCorrupt, err)
		}
		apply(s.data, rec.Ops)
		s.seq = rec.Seq
		offset += int64(len(line))
	}
	_, err := s.wal.Seek(0, io.SeekEnd)
	return err
}
KV_LAB_SOURCE
printf 'Записан файл: kv/journal.go\n'
```

**Результат:** `Записан файл: kv/journal.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

**Сборка этапа**


```bash
go build -trimpath -o bin/kv ./cmd/kv
cp bin/kv bin/kv-02-wal
go doc -src ./kv Store.Batch
```

**Результат:** программа собрана, затем напечатан код `Batch`. Теперь перед изменением словаря вызывается `writeRecord`. Внутри неё запись попадает в файл, затем вызывается `Sync`.

### 6.2. Повторить опыт с перезапуском

```bash
wal_dir=$(mktemp -d "${TMPDIR:-/tmp}/kv-lines.XXXXXX")
./bin/kv -dir "$wal_dir" set city 'Москва'
./bin/kv -dir "$wal_dir" get city
cat "$wal_dir/wal.log"
```

**Результат:** запись вернёт `{"ok":true}`, новый процесс прочитает `Москва`. `cat` покажет строку примерно такого вида:

```json
{"version":1,"seq":1,"ops":[{"type":"set","key":"city","value":"Москва"}]}
```

Мы повторили тот же опыт, который не сработал на этапе 1. Разница — в файле: при запуске `replay` прочитал журнал и восстановил словарь. Номер `seq` позволяет проверить порядок записей.

### 6.3. Проверить со своим значением

В первой строке можете заменить `Казань` на свой город:

```bash
my_value='Казань'
./bin/kv -dir "$wal_dir" set favorite-city "$my_value"
./bin/kv -dir "$wal_dir" get favorite-city
```

**Результат:** успешная запись и выбранное вами значение при следующем запуске. Теперь в журнале две строки, поскольку каждая команда `set` добавляет запись.

### 6.4. Оставить в конце неполную запись

Что случится, если процесс успел записать только часть следующей строки? Воспроизведём такой файл вручную. Все команды хранилища из предыдущих блоков уже завершились.

```bash
wal_before=$(wc -c < "$wal_dir/wal.log" | tr -d '[:space:]')
printf '{"version":1' >> "$wal_dir/wal.log"
corrupt_code=0
./bin/kv -dir "$wal_dir" get city || corrupt_code=$?
printf 'Код завершения: %s\n' "$corrupt_code"
test "$corrupt_code" -eq 5
```

**Результат:** ошибка `corrupt store: incomplete JSON line ...` и код `5`. Обе предыдущие записи целы, но программа отказывается открыть базу из-за последней неполной строки.

Это воспроизведение **последствия недописанной записи**, а не настоящее завершение процесса сигналом. Проверки с настоящим `SIGKILL` будут в конце, когда соберём все механизмы.

**Перед следующим этапом:** нужно ли отбрасывать всю базу, если оборвалась только последняя запись? Папку `$wal_dir` пока оставьте как есть — следующая версия должна открыть именно её.

<a id="stage-3"></a>

## 7. Этап 3. Восстанавливаем журнал с неполным хвостом

### 7.1. Изменить чтение журнала

Меняется только чтение журнала. Полные строки восстанавливаются как раньше. Если в конце осталась строка без завершающего переноса, программа укорачивает файл до последней целой записи.

Выполните следующие блоки по порядку в папке `$work_dir`. Каждый блок создаст или заменит один файл.

#### 7.1.1. kv/journal.go

Сравните ветку `err == io.EOF` с предыдущим этапом. Вместо отказа она запоминает число отброшенных байт, вызывает `Truncate` и `Sync`. Полная строка с неверным JSON всё ещё вызывает ошибку.

```bash
mkdir -p 'kv'
cat > 'kv/journal.go' <<'KV_LAB_SOURCE'
package kv

import (
	"bufio"
	"encoding/json"
	"fmt"
	"io"
)

type record struct {
	Version int    `json:"version"`
	Seq     uint64 `json:"seq"`
	Ops     []Op   `json:"ops"`
}

func (s *Store) writeRecord(ops []Op) error {
	payload, err := json.Marshal(record{Version: 1, Seq: s.seq + 1, Ops: ops})
	if err != nil {
		return err
	}
	if len(payload) > MaxRecordBytes {
		return fmt.Errorf("%w: record too large", ErrInvalid)
	}
	line := append(payload, '\n')
	n, err := s.wal.Write(line)
	if err == nil && n != len(line) {
		err = io.ErrShortWrite
	}
	if err == nil {
		err = s.wal.Sync()
	}
	if err != nil {
		return s.poison(err)
	}
	return nil
}
func (s *Store) replay() error {
	reader := bufio.NewReader(s.wal)
	var offset int64
	for {
		line, err := reader.ReadBytes('\n')
		if err == io.EOF && len(line) == 0 {
			break
		}
		if err == io.EOF {
			s.recoveredTail = int64(len(line))
			if err := s.wal.Truncate(offset); err != nil {
				return err
			}
			if err := s.wal.Sync(); err != nil {
				return err
			}
			break
		}
		if err != nil {
			return err
		}
		if len(line) > MaxRecordBytes+1 {
			return fmt.Errorf("%w: record too large", ErrCorrupt)
		}
		var rec record
		if err := json.Unmarshal(line, &rec); err != nil {
			return fmt.Errorf("%w: JSON: %v", ErrCorrupt, err)
		}
		if rec.Version != 1 || rec.Seq == 0 || rec.Seq != s.seq+1 {
			return fmt.Errorf("%w: sequence or version", ErrCorrupt)
		}
		if err := validateOps(rec.Ops); err != nil {
			return fmt.Errorf("%w: %v", ErrCorrupt, err)
		}
		apply(s.data, rec.Ops)
		s.seq = rec.Seq
		offset += int64(len(line))
	}
	_, err := s.wal.Seek(0, io.SeekEnd)
	return err
}
KV_LAB_SOURCE
printf 'Записан файл: kv/journal.go\n'
```

**Результат:** `Записан файл: kv/journal.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

**Сборка этапа**


```bash
go build -trimpath -o bin/kv ./cmd/kv
cp bin/kv bin/kv-03-recovery
printf 'Собрана версия 3: восстановление хвоста\n'
```

**Результат:** сообщение о сборке. Сохранена отдельная программа `bin/kv-03-recovery`.

### 7.2. Открыть ту же базу

```bash
./bin/kv -dir "$wal_dir" stats
./bin/kv -dir "$wal_dir" get city
./bin/kv -dir "$wal_dir" get favorite-city
wal_after=$(wc -c < "$wal_dir/wal.log" | tr -d '[:space:]')
printf 'До неполной записи: %s байт, после восстановления: %s байт\n' "$wal_before" "$wal_after"
test "$wal_before" -eq "$wal_after"
```

**Результат:** первый ответ покажет `keys: 2`, `sequence: 2` и `recovered_tail_bytes: 12`. Оба значения читаются. Размер файла вернулся к размеру до добавления неполной строки. `test` при совпадении размеров ничего не печатает.

`recovered_tail_bytes` относится только к текущему открытию. Если снова вызвать `stats`, это поле будет 0: хвост уже удалён.

Проверим, что после восстановления можно продолжить запись:

```bash
./bin/kv -dir "$wal_dir" set country 'Россия'
./bin/kv -dir "$wal_dir" get country
```

**Результат:** `{"ok":true}`, затем значение `Россия`. Новая запись добавилась после восстановленных данных.

### 7.3. Найти проблему, которую JSON не замечает

Теперь целиком изменим значение внутри **копии** журнала, сохранив корректный JSON:

```bash
edited_dir=$(mktemp -d "${TMPDIR:-/tmp}/kv-edited.XXXXXX")
sed 's/Москва/Самара/g' "$wal_dir/wal.log" > "$edited_dir/wal.log"
./bin/kv -dir "$edited_dir" get city
```

**Результат:** `Самара`. Программа приняла изменённое значение, хотя команду `set city Самара` мы не выполняли. JSON остался правильным, номера записей тоже.

**Перед следующим этапом:** проверка формата и проверка целостности — разные задачи. Нужен способ заметить изменение байтов полной записи. Для этого добавим контрольные суммы.

<a id="stage-4"></a>

## 8. Этап 4. Проверяем целостность записей

### 8.1. Добавить бинарный заголовок и CRC32

Корректный JSON ещё не означает, что байты не изменились. Добавим заголовок записи с длиной и контрольными суммами. Журнал становится бинарным. Для него в практикуме создаётся новая папка данных.

Выполните следующие блоки по порядку в папке `$work_dir`. Каждый блок создаст или заменит один файл.

#### 8.1.1. kv/format.go

`encodeFrame` создаёт запись из заголовка и JSON. `readFrame` проверяет длину и CRC32. Неполная запись возвращает `io.ErrUnexpectedEOF`, полная повреждённая — `ErrCorrupt`. В этом файле также описан формат будущего снимка.

```bash
mkdir -p 'kv'
cat > 'kv/format.go' <<'KV_LAB_SOURCE'
package kv

import (
	"bytes"
	"encoding/binary"
	"encoding/json"
	"fmt"
	"hash/crc32"
	"io"
	"unicode/utf8"
)

const headerSize = 16

type record struct {
	Version int    `json:"version"`
	Seq     uint64 `json:"seq"`
	Ops     []Op   `json:"ops"`
}

type snapshot struct {
	Version int               `json:"version"`
	Seq     uint64            `json:"seq"`
	Data    map[string]string `json:"data"`
}

// frame: magic(4), payload length(4 LE), CRC32 of first 8 bytes(4 LE),
// CRC32 of payload(4 LE), JSON payload. Header CRC prevents a damaged length
// from being mistaken for a legitimately interrupted final payload.
func encodeFrame(magic string, payload []byte) []byte {
	b := make([]byte, headerSize+len(payload))
	copy(b, magic)
	binary.LittleEndian.PutUint32(b[4:8], uint32(len(payload)))
	binary.LittleEndian.PutUint32(b[8:12], crc32.ChecksumIEEE(b[:8]))
	binary.LittleEndian.PutUint32(b[12:16], crc32.ChecksumIEEE(payload))
	copy(b[16:], payload)
	return b
}

// readFrame returns io.EOF for a clean end, io.ErrUnexpectedEOF only for a
// partial frame. Fully present invalid frames are never silently discarded.
func readFrame(r io.Reader, magic string, limit uint32) ([]byte, error) {
	var h [headerSize]byte
	if _, err := io.ReadFull(r, h[:]); err != nil {
		return nil, err
	}
	if string(h[:4]) != magic || binary.LittleEndian.Uint32(h[8:12]) != crc32.ChecksumIEEE(h[:8]) {
		return nil, fmt.Errorf("%w: invalid %s frame header", ErrCorrupt, magic)
	}
	n := binary.LittleEndian.Uint32(h[4:8])
	if n == 0 || n > limit {
		return nil, fmt.Errorf("%w: frame length %d", ErrCorrupt, n)
	}
	b := make([]byte, int(n))
	if _, err := io.ReadFull(r, b); err != nil {
		if err == io.EOF {
			err = io.ErrUnexpectedEOF
		}
		return nil, err
	}
	if crc32.ChecksumIEEE(b) != binary.LittleEndian.Uint32(h[12:16]) {
		return nil, fmt.Errorf("%w: payload checksum mismatch", ErrCorrupt)
	}
	return b, nil
}

func decodeJSON(b []byte, dst any) error {
	d := json.NewDecoder(bytes.NewReader(b))
	d.DisallowUnknownFields()
	if err := d.Decode(dst); err != nil {
		return fmt.Errorf("%w: JSON: %v", ErrCorrupt, err)
	}
	var extra any
	if err := d.Decode(&extra); err != io.EOF {
		return fmt.Errorf("%w: trailing JSON", ErrCorrupt)
	}
	return nil
}

func validateKey(key string) error {
	if len(key) == 0 || len(key) > MaxKeyBytes || !utf8.ValidString(key) {
		return fmt.Errorf("%w: key must be UTF-8, 1..%d bytes", ErrInvalid, MaxKeyBytes)
	}
	return nil
}

func validateOps(ops []Op) error {
	if len(ops) == 0 || len(ops) > MaxBatchOps {
		return fmt.Errorf("%w: batch requires 1..%d operations", ErrInvalid, MaxBatchOps)
	}
	for _, op := range ops {
		if err := validateKey(op.Key); err != nil {
			return err
		}
		switch op.Type {
		case "set":
			if len(op.Value) > MaxValueBytes || !utf8.ValidString(op.Value) {
				return fmt.Errorf("%w: value must be UTF-8 and at most %d bytes", ErrInvalid, MaxValueBytes)
			}
		case "delete":
			if op.Value != "" {
				return fmt.Errorf("%w: delete cannot contain a value", ErrInvalid)
			}
		default:
			return fmt.Errorf("%w: unknown operation %q", ErrInvalid, op.Type)
		}
	}
	return nil
}

func apply(data map[string]string, ops []Op) {
	for _, op := range ops {
		if op.Type == "set" {
			data[op.Key] = op.Value
		} else {
			delete(data, op.Key)
		}
	}
}
KV_LAB_SOURCE
printf 'Записан файл: kv/format.go\n'
```

**Результат:** `Записан файл: kv/format.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 8.1.2. kv/journal.go

Вместо строк теперь используются `encodeFrame` и `readFrame`. Восстановление по-прежнему удаляет только неполный хвост. Полную запись с неверной контрольной суммой пропускать нельзя.

```bash
mkdir -p 'kv'
cat > 'kv/journal.go' <<'KV_LAB_SOURCE'
package kv

import (
	"encoding/json"
	"fmt"
	"io"
)

func (s *Store) writeRecord(ops []Op) error {
	payload, err := json.Marshal(record{Version: 1, Seq: s.seq + 1, Ops: ops})
	if err != nil {
		return err
	}
	if len(payload) > MaxRecordBytes {
		return fmt.Errorf("%w: record too large", ErrInvalid)
	}
	frame := encodeFrame("KVW1", payload)
	n, err := s.wal.Write(frame)
	if err == nil && n != len(frame) {
		err = io.ErrShortWrite
	}
	if err == nil {
		err = s.wal.Sync()
	}
	if err != nil {
		return s.poison(err)
	}
	return nil
}
func (s *Store) replay() error {
	var offset int64
	for {
		payload, err := readFrame(s.wal, "KVW1", MaxRecordBytes)
		if err == io.EOF {
			break
		}
		if err == io.ErrUnexpectedEOF {
			info, err := s.wal.Stat()
			if err != nil {
				return err
			}
			s.recoveredTail = info.Size() - offset
			if err = s.wal.Truncate(offset); err != nil {
				return err
			}
			if err = s.wal.Sync(); err != nil {
				return err
			}
			break
		}
		if err != nil {
			return err
		}
		var rec record
		if err = decodeJSON(payload, &rec); err != nil {
			return err
		}
		if rec.Version != 1 || rec.Seq == 0 || rec.Seq != s.seq+1 {
			return fmt.Errorf("%w: sequence or version", ErrCorrupt)
		}
		if err = validateOps(rec.Ops); err != nil {
			return fmt.Errorf("%w: %v", ErrCorrupt, err)
		}
		apply(s.data, rec.Ops)
		s.seq = rec.Seq
		offset += int64(headerSize + len(payload))
	}
	_, err := s.wal.Seek(0, io.SeekEnd)
	return err
}
KV_LAB_SOURCE
printf 'Записан файл: kv/journal.go\n'
```

**Результат:** `Записан файл: kv/journal.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

**Сборка этапа**


```bash
go build -trimpath -o bin/kv ./cmd/kv
cp bin/kv bin/kv-04-checksums
printf 'Собрана версия 4: контрольные суммы\n'
```

**Результат:** сообщение о сборке и сохранённая копия программы.

Старый текстовый журнал имеет другой формат. Для бинарного журнала создаём **новую папку**. Автоматического преобразования в этой работе нет. Старые папки можно читать сохранённой версией `bin/kv-03-recovery`.

```bash
frame_dir=$(mktemp -d "${TMPDIR:-/tmp}/kv-frames.XXXXXX")
./bin/kv -dir "$frame_dir" set city 'Москва'
./bin/kv -dir "$frame_dir" get city
od -An -tx1 -N16 "$frame_dir/wal.log"
```

**Результат:** значение сохраняется и читается. Последняя команда покажет 16 байт заголовка в шестнадцатеричном виде. Первые четыре — `4b 56 57 31`, то есть `KVW1`.

### 8.2. Убедиться, что восстановление хвоста осталось

```bash
printf 'KVW' >> "$frame_dir/wal.log"
./bin/kv -dir "$frame_dir" stats
./bin/kv -dir "$frame_dir" get city
```

**Результат:** `recovered_tail_bytes: 3`, затем `Москва`. Последние три байта не составляют даже полный заголовок, поэтому программа убрала только их.

### 8.3. Повредить полную запись

Создадим ещё одну отдельную копию и заменим `Москва` на `Самара`, как в предыдущем опыте. Оба слова занимают 12 байт в UTF-8. Длина записи и правильность JSON сохранятся, контрольная сумма останется старой:

```bash
damaged_dir=$(mktemp -d "${TMPDIR:-/tmp}/kv-damaged.XXXXXX")
cp "$frame_dir/wal.log" "$damaged_dir/wal.log"
value_offset=$(LC_ALL=C grep -abo 'Москва' "$damaged_dir/wal.log" | cut -d : -f 1)
printf 'Самара' | dd of="$damaged_dir/wal.log" bs=1 seek="$value_offset" count=12 conv=notrunc
before_hash=$(shasum -a 256 "$damaged_dir/wal.log")
corrupt_code=0
./bin/kv -dir "$damaged_dir" get city || corrupt_code=$?
printf 'Код завершения: %s\n' "$corrupt_code"
test "$corrupt_code" -eq 5
after_hash=$(shasum -a 256 "$damaged_dir/wal.log")
test "$before_hash" = "$after_hash"
printf 'Повреждение обнаружено, файл не изменён при открытии\n'
```

**Результат:** поиск определит начало значения в файле, затем `dd` сообщит о записи 12 байт. Хранилище вернёт `payload checksum mismatch` и код `5`. Последняя строка подтвердит, что открытие не обрезало повреждённый файл. Обе команды `test` при успехе молчат.

На этапе 3 изменение корректного JSON оставалось незамеченным. Теперь несоответствие контрольной суммы вызывает ошибку. Убирать такую запись как недописанный хвост нельзя: мы не знаем, какие подтверждённые данные в ней были.

CRC32 обнаруживает случайные изменения. Это не защита от человека, который намеренно изменит и данные, и их контрольную сумму.

**Перед следующим этапом:** формат стал надёжнее, но каждое изменение всё ещё увеличивает журнал. Даже сто замен одного ключа оставят сто записей.

<a id="stage-5"></a>

## 9. Этап 5. Создаём снимок состояния

### 9.1. Добавить снимки

В следующей версии работа с журналом и снимком собрана в одном `store.go`. Сохраним прежний файл для сравнения и удалим его из пакета:

```bash
cp kv/journal.go journal-before-snapshot.go.txt
rm kv/journal.go
printf 'Старая версия сохранена в journal-before-snapshot.go.txt\n'
```

**Результат:** указанное сообщение. Копия остаётся в рабочей папке, а `kv/journal.go` удалён. Расширение `.txt` не позволяет Go включить сохранённую копию в сборку. До выполнения следующих блоков проект временно не собирается.

Сохраним текущее состояние целиком, чтобы не хранить всю историю замен. Новый `store.go` объединяет работу с журналом и снимком. Формат журнала остаётся таким же, как на этапе 4.

Выполните следующие блоки по порядку в папке `$work_dir`. Каждый блок создаст или заменит один файл.

#### 9.1.1. kv/store.go

Основное новое — `loadSnapshot` и `Snapshot`. Сначала публикуется снимок, затем очищается журнал. `replay` теперь учитывает номера записей, уже вошедших в снимок. Остальные операции сохраняют прежний смысл. В `Open` есть место для блокировки, но следующий файл пока её не включает.

```bash
mkdir -p 'kv'
cat > 'kv/store.go' <<'KV_LAB_SOURCE'
package kv

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"math"
	"os"
	"path/filepath"
	"sort"
	"strings"
	"sync"
)

type walFile interface {
	io.Reader
	io.Writer
	io.Seeker
	Truncate(int64) error
	Sync() error
	Close() error
	Stat() (os.FileInfo, error)
}

// Store owns an exclusive directory lock until Close. Do not copy a Store.
type Store struct {
	mu            sync.Mutex
	dir           string
	lock          *os.File
	wal           walFile
	data          map[string]string
	seq           uint64
	snapshotSeq   uint64
	recoveredTail int64
	closed        bool
	failed        error
	// Tests install controlled crash points; never configured via environment
	// variables or CLI in the production binary.
	testHook func(string)
}

func Open(dir string) (*Store, error) {
	if dir == "" {
		return nil, fmt.Errorf("%w: directory is empty", ErrInvalid)
	}
	if err := os.MkdirAll(dir, 0700); err != nil {
		return nil, err
	}
	lock, err := os.OpenFile(filepath.Join(dir, "LOCK"), os.O_CREATE|os.O_RDWR, 0600)
	if err != nil {
		return nil, err
	}
	if err = lockFile(lock); err != nil {
		_ = lock.Close()
		return nil, err
	}
	s := &Store{dir: dir, lock: lock, data: make(map[string]string)}
	ok := false
	defer func() {
		if !ok {
			if s.wal != nil {
				_ = s.wal.Close()
			}
			_ = lock.Close()
		}
	}()
	if err = s.loadSnapshot(); err != nil {
		return nil, err
	}
	flags := os.O_RDWR | os.O_CREATE
	// A published snapshot implies the WAL was already created. Its absence
	// is not an interrupted append and must not become silent data loss.
	if s.snapshotSeq > 0 {
		flags = os.O_RDWR
	}
	s.wal, err = os.OpenFile(filepath.Join(dir, "wal.log"), flags, 0600)
	if err != nil {
		return nil, fmt.Errorf("open WAL: %w", err)
	}
	if err = syncDir(dir); err != nil {
		return nil, fmt.Errorf("sync directory: %w", err)
	}
	if err = s.replay(); err != nil {
		return nil, err
	}
	ok = true
	return s, nil
}

func (s *Store) loadSnapshot() error {
	f, err := os.Open(filepath.Join(s.dir, "snapshot.kv"))
	if errors.Is(err, os.ErrNotExist) {
		return nil
	}
	if err != nil {
		return err
	}
	defer f.Close()
	payload, err := readFrame(f, "KVS1", MaxSnapshotBytes)
	if err != nil {
		return fmt.Errorf("%w: snapshot: %v", ErrCorrupt, err)
	}
	var tail [1]byte
	if n, err := f.Read(tail[:]); n != 0 || err != io.EOF {
		return fmt.Errorf("%w: trailing snapshot bytes", ErrCorrupt)
	}
	var snap snapshot
	if err := decodeJSON(payload, &snap); err != nil {
		return err
	}
	if snap.Version != 1 || snap.Data == nil {
		return fmt.Errorf("%w: snapshot version or data", ErrCorrupt)
	}
	for k, v := range snap.Data {
		if err := validateOps([]Op{{Type: "set", Key: k, Value: v}}); err != nil {
			return fmt.Errorf("%w: snapshot: %v", ErrCorrupt, err)
		}
	}
	s.data, s.seq, s.snapshotSeq = snap.Data, snap.Seq, snap.Seq
	return nil
}

func (s *Store) replay() error {
	var offset int64
	var previous uint64
	first := true
	for {
		payload, err := readFrame(s.wal, "KVW1", MaxRecordBytes)
		if err == io.EOF {
			break
		}
		if err == io.ErrUnexpectedEOF {
			info, err := s.wal.Stat()
			if err != nil {
				return err
			}
			s.recoveredTail = info.Size() - offset
			if err = s.wal.Truncate(offset); err != nil {
				return err
			}
			if err = s.wal.Sync(); err != nil {
				return err
			}
			break
		}
		if err != nil {
			return fmt.Errorf("WAL offset %d: %w", offset, err)
		}
		var rec record
		if err = decodeJSON(payload, &rec); err != nil {
			return fmt.Errorf("WAL offset %d: %w", offset, err)
		}
		if rec.Version != 1 || rec.Seq == 0 {
			return fmt.Errorf("%w: WAL version or sequence at %d", ErrCorrupt, offset)
		}
		if err = validateOps(rec.Ops); err != nil {
			return fmt.Errorf("%w: WAL operation: %v", ErrCorrupt, err)
		}
		if first {
			if rec.Seq > s.seq && rec.Seq-s.seq != 1 {
				return fmt.Errorf("%w: WAL starts after a missing sequence", ErrCorrupt)
			}
		} else if previous == math.MaxUint64 || rec.Seq != previous+1 {
			return fmt.Errorf("%w: non-consecutive WAL sequence", ErrCorrupt)
		}
		first, previous = false, rec.Seq
		if rec.Seq > s.snapshotSeq {
			if s.seq == math.MaxUint64 || rec.Seq != s.seq+1 {
				return fmt.Errorf("%w: replay sequence gap", ErrCorrupt)
			}
			apply(s.data, rec.Ops)
			s.seq = rec.Seq
		}
		offset += int64(headerSize + len(payload))
	}
	// A snapshot may have been published before the old WAL was truncated.
	// If it covers every WAL record, no stale prefix needs to be retained.
	if s.snapshotSeq > 0 && s.seq == s.snapshotSeq {
		if err := s.wal.Truncate(0); err != nil {
			return err
		}
		if err := s.wal.Sync(); err != nil {
			return err
		}
	}
	_, err := s.wal.Seek(0, io.SeekEnd)
	return err
}

func (s *Store) stateError() error {
	if s.closed {
		return ErrClosed
	}
	if s.failed != nil {
		return s.failed
	}
	return nil
}

func (s *Store) poison(err error) error {
	s.failed = fmt.Errorf("%w: %w", ErrUncertain, err)
	return s.failed
}

func (s *Store) hook(stage string) {
	if s.testHook != nil {
		s.testHook(stage)
	}
}

func (s *Store) Get(key string) (string, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return "", err
	}
	if err := validateKey(key); err != nil {
		return "", err
	}
	v, ok := s.data[key]
	if !ok {
		return "", ErrNotFound
	}
	return v, nil
}

func (s *Store) Set(key, value string) error {
	return s.Batch([]Op{{Type: "set", Key: key, Value: value}})
}
func (s *Store) Delete(key string) error { return s.Batch([]Op{{Type: "delete", Key: key}}) }

// Batch durably commits all operations as one WAL record, then changes memory.
// A crash before a success response may still leave a committed record.
func (s *Store) Batch(ops []Op) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return err
	}
	if err := validateOps(ops); err != nil {
		return err
	}
	if s.seq == math.MaxUint64 {
		return fmt.Errorf("%w: sequence exhausted", ErrInvalid)
	}
	b, err := json.Marshal(record{Version: 1, Seq: s.seq + 1, Ops: ops})
	if err != nil {
		return err
	}
	if len(b) > MaxRecordBytes {
		return fmt.Errorf("%w: encoded batch exceeds %d bytes", ErrInvalid, MaxRecordBytes)
	}
	frame := encodeFrame("KVW1", b)
	n, err := s.wal.Write(frame)
	if err != nil {
		return s.poison(err)
	}
	if n != len(frame) {
		return s.poison(io.ErrShortWrite)
	}
	s.hook("wal-written")
	if err = s.wal.Sync(); err != nil {
		return s.poison(err)
	}
	s.hook("wal-synced")
	apply(s.data, ops)
	s.seq++
	return nil
}

// List returns an independent, lexicographically sorted snapshot of matching
// pairs. Empty prefix matches everything; it is not a key and may be empty.
func (s *Store) List(prefix string) ([]Pair, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return nil, err
	}
	pairs := make([]Pair, 0)
	for k, v := range s.data {
		if strings.HasPrefix(k, prefix) {
			pairs = append(pairs, Pair{k, v})
		}
	}
	sort.Slice(pairs, func(i, j int) bool { return pairs[i].Key < pairs[j].Key })
	return pairs, nil
}

func (s *Store) Stats() (Stats, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return Stats{}, err
	}
	info, err := s.wal.Stat()
	if err != nil {
		return Stats{}, err
	}
	return Stats{len(s.data), s.seq, s.snapshotSeq, info.Size(), s.recoveredTail}, nil
}

// Snapshot publishes a complete checksummed state before discarding the WAL.
// Never truncate the WAL before rename and directory sync have succeeded.
func (s *Store) Snapshot() error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if err := s.stateError(); err != nil {
		return err
	}
	b, err := json.Marshal(snapshot{Version: 1, Seq: s.seq, Data: s.data})
	if err != nil {
		return err
	}
	if len(b) > MaxSnapshotBytes {
		return fmt.Errorf("%w: snapshot exceeds %d bytes", ErrInvalid, MaxSnapshotBytes)
	}
	tmp := filepath.Join(s.dir, "snapshot.tmp")
	f, err := os.OpenFile(tmp, os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0600)
	if err != nil {
		return err
	}
	frame := encodeFrame("KVS1", b)
	n, err := f.Write(frame)
	if err == nil && n != len(frame) {
		err = io.ErrShortWrite
	}
	if err != nil {
		_ = f.Close()
		return err
	}
	s.hook("snapshot-written")
	if err = f.Sync(); err != nil {
		_ = f.Close()
		return err
	}
	if err = f.Close(); err != nil {
		return err
	}
	s.hook("snapshot-synced")
	if err = os.Rename(tmp, filepath.Join(s.dir, "snapshot.kv")); err != nil {
		return err
	}
	s.hook("snapshot-renamed")
	if err = syncDir(s.dir); err != nil {
		return s.poison(err)
	}
	s.snapshotSeq = s.seq
	s.hook("snapshot-published")
	if err = s.wal.Truncate(0); err != nil {
		return s.poison(err)
	}
	s.hook("wal-truncated")
	if _, err = s.wal.Seek(0, io.SeekStart); err != nil {
		return s.poison(err)
	}
	if err = s.wal.Sync(); err != nil {
		return s.poison(err)
	}
	s.hook("snapshot-complete")
	return nil
}

// Close is idempotent. Every successful write was already synced by Batch.
func (s *Store) Close() error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if s.closed {
		return nil
	}
	s.closed = true
	return errors.Join(s.wal.Close(), s.lock.Close())
}

func syncDir(dir string) error {
	f, err := os.Open(dir)
	if err != nil {
		return err
	}
	err = f.Sync()
	return errors.Join(err, f.Close())
}
KV_LAB_SOURCE
printf 'Записан файл: kv/store.go\n'
```

**Результат:** `Записан файл: kv/store.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 9.1.2. kv/lock_unix.go

Это намеренная временная заглушка: `lockFile` возвращает успех. Она позволит увидеть, что случается с двумя писателями до добавления защиты.

```bash
mkdir -p 'kv'
cat > 'kv/lock_unix.go' <<'KV_LAB_SOURCE'
//go:build darwin || linux

package kv

import "os"

// Stage 5 intentionally permits multiple writers. Stage 6 replaces this stub.
func lockFile(f *os.File) error { return nil }
KV_LAB_SOURCE
printf 'Записан файл: kv/lock_unix.go\n'
```

**Результат:** `Записан файл: kv/lock_unix.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 9.1.3. kv/lock_other.go

Сообщение о неподдерживаемой платформе. На macOS используется предыдущий файл.

```bash
mkdir -p 'kv'
cat > 'kv/lock_other.go' <<'KV_LAB_SOURCE'
//go:build !darwin && !linux

package kv

import (
	"fmt"
	"os"
)

func lockFile(_ *os.File) error { return fmt.Errorf("kv-lab supports macOS and Linux only") }
KV_LAB_SOURCE
printf 'Записан файл: kv/lock_other.go\n'
```

**Результат:** `Записан файл: kv/lock_other.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

**Сборка этапа**


```bash
go build -trimpath -o bin/kv ./cmd/kv
cp bin/kv bin/kv-05-snapshot
go doc -src ./kv Store.Snapshot
```

**Результат:** сборка завершится без вывода, затем появится код `Snapshot`. Формат журнала не изменился, поэтому продолжаем с базой `$frame_dir` из этапа 4.

В `Snapshot` найдите порядок действий:

1. Запись во временный файл.
2. Синхронизация и закрытие файла.
3. Замена опубликованного снимка.
4. Синхронизация каталога.
5. Очистка и синхронизация журнала.

### 9.2. Накопить историю одного ключа

```bash
for value in 1 2 3 4 5; do
  ./bin/kv -dir "$frame_dir" set counter "$value"
done
./bin/kv -dir "$frame_dir" stats
./bin/kv -dir "$frame_dir" get counter
```

**Результат:** пять ответов `{"ok":true}`. В базе два ключа, `sequence: 6`, журнал занимает больше нуля байт. Значение `counter` равно `5`, хотя в файле хранятся все пять его версий.

### 9.3. Создать снимок и проверить данные

```bash
./bin/kv -dir "$frame_dir" snapshot
./bin/kv -dir "$frame_dir" stats
wc -c "$frame_dir/wal.log"
./bin/kv -dir "$frame_dir" get city
./bin/kv -dir "$frame_dir" get counter
```

**Результат:** `snapshot` вернёт `{"ok":true}`. `sequence` и `snapshot_sequence` равны `6`, `wal_bytes` равен `0`. Команда `wc` также покажет размер 0. Оба значения по-прежнему читаются: `Москва` и `5`.

Новый процесс восстановил их из `snapshot.kv`. Журнал больше не нужен для воспроизведения старой истории.

### 9.4. Продолжить изменения после снимка

```bash
./bin/kv -dir "$frame_dir" set counter '6'
./bin/kv -dir "$frame_dir" delete city
./bin/kv -dir "$frame_dir" list
./bin/kv -dir "$frame_dir" check
```

**Результат:** список содержит только `counter` со значением `6`. `sequence: 8`, `snapshot_sequence: 6`. Новые записи в журнале дополняют снимок, а удалённый город не появился снова при открытии.

### 9.5. Обнаружить потерю записи при двух писателях

Снимок и контрольные суммы не защищают от двух процессов с разными копиями словаря. Проведём опыт с двумя настоящими экземплярами `kv shell`. Вспомогательная программа создаст свою пустую базу и выполнит действия в таком порядке:

| Шаг | Процесс A | Процесс B |
|---|---|---|
| 1 | открывает базу и пишет `base=0` | ещё не запущен |
| 2 | остаётся открытым | открывает ту же базу и читает журнал |
| 3 | ждёт | пишет `left=B` и получает успех |
| 4 | пишет `rght=A` и получает успех | остаётся открытым |
| 5 | оба процесса закрываются | новый процесс читает `left` |

Что, по вашему прогнозу, прочитает последний процесс?

Создайте программу, которая выполнит этот сценарий. Она запускает два процесса и ждёт ответа на каждую команду, чтобы получить указанный порядок действий:

```bash
mkdir -p 'tools/experiment'
cat > 'tools/experiment/main.go' <<'KV_LAB_SOURCE'
// experiment coordinates two real CLI processes without timing assumptions.
package main

import (
	"bufio"
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"flag"
	"fmt"
	"io"
	"os"
	"os/exec"
	"path/filepath"
	"time"
)

type session struct {
	cmd    *exec.Cmd
	in     io.WriteCloser
	out    *bufio.Scanner
	errors bytes.Buffer
	closed bool
}

func start(ctx context.Context, binary, dir string) (*session, error) {
	s := &session{cmd: exec.CommandContext(ctx, binary, "-dir", dir, "shell")}
	var err error
	if s.in, err = s.cmd.StdinPipe(); err != nil {
		return nil, err
	}
	out, err := s.cmd.StdoutPipe()
	if err != nil {
		_ = s.in.Close()
		return nil, err
	}
	s.out = bufio.NewScanner(out)
	s.cmd.Stderr = &s.errors
	if err = s.cmd.Start(); err != nil {
		_ = s.in.Close()
		_ = out.Close()
		return nil, err
	}
	return s, nil
}

func (s *session) request(name, command string) error {
	fmt.Printf("%s → %s\n", name, command)
	if _, err := fmt.Fprintln(s.in, command); err != nil {
		return err
	}
	if !s.out.Scan() {
		if err := s.out.Err(); err != nil {
			return err
		}
		return io.EOF
	}
	fmt.Println("Ответ:", s.out.Text())
	var response map[string]any
	if err := json.Unmarshal(s.out.Bytes(), &response); err != nil {
		return err
	}
	if response["ok"] == false {
		return fmt.Errorf("command failed: %s", s.out.Text())
	}
	return nil
}

func (s *session) close() error {
	if s.closed {
		return nil
	}
	s.closed = true
	_ = s.in.Close()
	return s.cmd.Wait()
}

func (s *session) stop() {
	if !s.closed {
		_ = s.cmd.Process.Kill()
		_ = s.close()
	}
}

func experiment(binary, expected string) error {
	if expected != "lost" && expected != "locked" {
		return fmt.Errorf("-expect must be lost or locked")
	}
	binary, err := filepath.Abs(binary)
	if err != nil {
		return err
	}
	dir, err := os.MkdirTemp("", "kv-two-writers-")
	if err != nil {
		return err
	}
	fmt.Println("Отдельная база эксперимента:", dir)
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	a, err := start(ctx, binary, dir)
	if err != nil {
		return err
	}
	defer a.stop()
	if err = a.request("Процесс A", `{"command":"set","key":"base","value":"0"}`); err != nil {
		return err
	}
	b, err := start(ctx, binary, dir)
	if err != nil {
		return err
	}
	defer b.stop()
	// Waiting for an answer ensures that B has opened and replayed the WAL.
	if err = b.request("Процесс B", `{"command":"stats"}`); err != nil {
		waitErr := b.close()
		var exit *exec.ExitError
		if expected != "locked" || !errors.As(waitErr, &exit) || exit.ExitCode() != 4 {
			return fmt.Errorf("unexpected second-process failure: %v, %v: %s", err, waitErr, b.errors.String())
		}
		fmt.Print("Процесс B: ", b.errors.String())
		if err = a.close(); err != nil {
			return err
		}
		out, err := exec.CommandContext(ctx, binary, "-dir", dir, "get", "base").CombinedOutput()
		if err != nil || string(out) != "{\"key\":\"base\",\"value\":\"0\"}\n" {
			return fmt.Errorf("reopen failed: %v: %s", err, out)
		}
		fmt.Print("После закрытия A: ", string(out))
		fmt.Println("ПРОВЕРЕНО: второй процесс получил код 4, после закрытия первого база открылась.")
		return nil
	}
	if expected == "locked" {
		return fmt.Errorf("second writer was accepted despite expected locking")
	}
	if err = b.request("Процесс B", `{"command":"set","key":"left","value":"B"}`); err != nil {
		return err
	}
	// Equal-length records expose the overwrite at A's stale file offset.
	if err = a.request("Процесс A", `{"command":"set","key":"rght","value":"A"}`); err != nil {
		return err
	}
	if err = b.close(); err != nil {
		return err
	}
	if err = a.close(); err != nil {
		return err
	}
	out, err := exec.CommandContext(ctx, binary, "-dir", dir, "get", "left").CombinedOutput()
	var exit *exec.ExitError
	if !errors.As(err, &exit) || exit.ExitCode() != 3 {
		return fmt.Errorf("expected lost key with exit 3, got %v: %s", err, out)
	}
	fmt.Print("Чтение left новым процессом: ", string(out))
	out, err = exec.CommandContext(ctx, binary, "-dir", dir, "list").CombinedOutput()
	if err != nil || string(out) != "[{\"key\":\"base\",\"value\":\"0\"},{\"key\":\"rght\",\"value\":\"A\"}]\n" {
		return fmt.Errorf("unexpected surviving data: %v: %s", err, out)
	}
	fmt.Print("Оставшиеся записи: ", string(out))
	fmt.Println("ПРОВЕРЕНО: оба set получили успех, но подтверждённый ключ left потерян.")
	return nil
}

func main() {
	binary := flag.String("bin", "", "absolute or relative path to the built kv binary")
	expected := flag.String("expect", "", "lost at stage 5, locked at stage 6")
	flag.Parse()
	if *binary == "" || flag.NArg() != 0 {
		fmt.Fprintln(os.Stderr, "usage: go run tools/experiment/main.go -bin PATH -expect lost|locked")
		os.Exit(2)
	}
	if err := experiment(*binary, *expected); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
KV_LAB_SOURCE
printf 'Записан файл: tools/experiment/main.go\n'
```

**Результат:** `Записан файл: tools/experiment/main.go`. Файл создан. Команды `mkdir` и `cat` при успехе ничего не печатают.

Теперь запустите опыт:

```bash
go run ./tools/experiment -bin "$PWD/bin/kv" -expect lost
```

**Результат:** программа покажет команды обоих процессов и успешные ответы на запись. В конце будет `key not found` для `left` и строка:

```text
ПРОВЕРЕНО: оба set получили успех, но подтверждённый ключ left потерян.
```

Процесс A сохранил старую позицию в файле. Его запись перезаписала запись B. Ключи `left` и `rght` специально имеют одинаковую длину, поэтому полная запись заменила другую полную запись. Контрольная сумма осталась правильной и не помогла обнаружить потерю.

Вспомогательная программа ждёт ответ каждой команды, прежде чем отправить следующую. Результат не зависит от случайно подобранных задержек. Её исходник вы только что создали в `tools/experiment/main.go`. Она использует интерфейс CLI и не меняет код хранилища.

**Перед следующим этапом:** нужен один владелец папки данных. `mutex` защищает вызовы внутри одного экземпляра `Store`, но не согласует два разных процесса.

<a id="stage-6"></a>

## 10. Этап 6. Запрещаем второго писателя

### 10.1. Включить блокировку и добавить тесты

Заменим заглушку настоящим `flock` и повторим опыт с двумя процессами. Затем добавим тесты и примеры итоговой программы. После этого этапа будут доступны все возможности итоговой версии.

Выполните следующие блоки по порядку в папке `$work_dir`. Каждый блок создаст или заменит один файл.

#### 10.1.1. kv/lock_unix.go

Неблокирующий `flock` возвращает ошибку, если база уже открыта. Ядро освобождает его при закрытии файла или завершении процесса. Остальной код хранилища менять не требуется.

```bash
mkdir -p 'kv'
cat > 'kv/lock_unix.go' <<'KV_LAB_SOURCE'
//go:build darwin || linux

package kv

import (
	"errors"
	"fmt"
	"os"
	"syscall"
)

func lockFile(f *os.File) error {
	if err := syscall.Flock(int(f.Fd()), syscall.LOCK_EX|syscall.LOCK_NB); err != nil {
		if errors.Is(err, syscall.EWOULDBLOCK) || errors.Is(err, syscall.EAGAIN) {
			return ErrLocked
		}
		return fmt.Errorf("flock: %w", err)
	}
	return nil
}
KV_LAB_SOURCE
printf 'Записан файл: kv/lock_unix.go\n'
```

**Результат:** `Записан файл: kv/lock_unix.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 10.1.2. kv/store_test.go

Проверки операций, неполных и повреждённых записей, блокировки и ошибок ввода-вывода.

```bash
mkdir -p 'kv'
cat > 'kv/store_test.go' <<'KV_LAB_SOURCE'
package kv

import (
	"bytes"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"math"
	"math/rand"
	"os"
	"path/filepath"
	"reflect"
	"strings"
	"sync"
	"testing"
)

func mustOpen(t *testing.T, dir string) *Store {
	t.Helper()
	s, err := Open(dir)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(func() { _ = s.Close() })
	return s
}
func must(t *testing.T, err error) {
	t.Helper()
	if err != nil {
		t.Fatal(err)
	}
}
func value(t *testing.T, s *Store, k, want string) {
	t.Helper()
	got, err := s.Get(k)
	must(t, err)
	if got != want {
		t.Fatalf("%s: got %q want %q", k, got, want)
	}
}
func write(t *testing.T, p string, b []byte) { t.Helper(); must(t, os.WriteFile(p, b, 0600)) }
func read(t *testing.T, p string) []byte {
	t.Helper()
	b, err := os.ReadFile(p)
	must(t, err)
	return b
}
func recFrame(t *testing.T, seq uint64, ops []Op) []byte {
	t.Helper()
	b, err := json.Marshal(record{1, seq, ops})
	must(t, err)
	return encodeFrame("KVW1", b)
}

func TestBasicPersistence(t *testing.T) {
	dir := t.TempDir()
	s := mustOpen(t, dir)
	must(t, s.Set("name", "Анна\n\"quoted\""))
	must(t, s.Set("empty", ""))
	must(t, s.Set("gone", "x"))
	must(t, s.Set("name", "Борис"))
	must(t, s.Delete("gone"))
	must(t, s.Delete("missing"))
	if _, err := s.Get("gone"); !errors.Is(err, ErrNotFound) {
		t.Fatal(err)
	}
	must(t, s.Close())
	must(t, s.Close())
	s = mustOpen(t, dir)
	value(t, s, "name", "Борис")
	value(t, s, "empty", "")
	if _, err := s.Get("gone"); !errors.Is(err, ErrNotFound) {
		t.Fatal(err)
	}
	stats, err := s.Stats()
	must(t, err)
	if stats.Sequence != 6 || stats.Keys != 2 {
		t.Fatal(stats)
	}
}

func TestBatchAtomicAndValidation(t *testing.T) {
	s := mustOpen(t, t.TempDir())
	must(t, s.Batch([]Op{{Type: "set", Key: "x", Value: "1"}, {Type: "set", Key: "y", Value: "2"}, {Type: "delete", Key: "x"}, {Type: "set", Key: "x", Value: "3"}}))
	value(t, s, "x", "3")
	value(t, s, "y", "2")
	before, err := s.Stats()
	must(t, err)
	bad := [][]Op{nil, {}, {{Type: "set", Key: "z", Value: "oops"}, {Type: "bogus", Key: "x"}}, {{Type: "set", Key: ""}}, {{Type: "delete", Key: "x", Value: "bad"}}, {{Type: "set", Key: "\xff"}}, {{Type: "set", Key: "x", Value: "\xff"}}, {{Type: "set", Key: strings.Repeat("x", MaxKeyBytes+1)}}, {{Type: "set", Key: "x", Value: strings.Repeat("x", MaxValueBytes+1)}}, make([]Op, MaxBatchOps+1)}
	for i, ops := range bad {
		if err := s.Batch(ops); !errors.Is(err, ErrInvalid) {
			t.Fatalf("%d: %v", i, err)
		}
	}
	after, err := s.Stats()
	must(t, err)
	if before != after {
		t.Fatalf("invalid input mutated state: %v %v", before, after)
	}
	if _, err := s.Get("z"); !errors.Is(err, ErrNotFound) {
		t.Fatal(err)
	}
	if _, err := s.Get(""); !errors.Is(err, ErrInvalid) {
		t.Fatal(err)
	}
	big := make([]Op, 17)
	for i := range big {
		big[i] = Op{Type: "set", Key: fmt.Sprint(i), Value: strings.Repeat("x", MaxValueBytes)}
	}
	if err := s.Batch(big); !errors.Is(err, ErrInvalid) {
		t.Fatal(err)
	}
}

func TestLimitsAndList(t *testing.T) {
	s := mustOpen(t, t.TempDir())
	k := strings.Repeat("k", MaxKeyBytes)
	v := strings.Repeat("v", MaxValueBytes)
	must(t, s.Set(k, v))
	value(t, s, k, v)
	must(t, s.Set("p:2", "two"))
	must(t, s.Set("p:1", "one"))
	got, err := s.List("p:")
	must(t, err)
	if !reflect.DeepEqual(got, []Pair{{"p:1", "one"}, {"p:2", "two"}}) {
		t.Fatal(got)
	}
	got[0].Value = "changed"
	value(t, s, "p:1", "one")
	empty, err := s.List("none")
	must(t, err)
	if empty == nil || len(empty) != 0 {
		t.Fatal(empty)
	}
}

func TestSnapshotAndAppend(t *testing.T) {
	dir := t.TempDir()
	s := mustOpen(t, dir)
	must(t, s.Set("a", "1"))
	must(t, s.Set("gone", "x"))
	must(t, s.Delete("gone"))
	must(t, s.Snapshot())
	stats, err := s.Stats()
	must(t, err)
	if stats.WALBytes != 0 || stats.SnapshotSequence != 3 {
		t.Fatal(stats)
	}
	must(t, s.Set("a", "2"))
	must(t, s.Set("b", "3"))
	must(t, s.Close())
	s = mustOpen(t, dir)
	value(t, s, "a", "2")
	value(t, s, "b", "3")
	if _, err := s.Get("gone"); !errors.Is(err, ErrNotFound) {
		t.Fatal(err)
	}
	must(t, s.Snapshot())
	must(t, s.Snapshot())
	must(t, s.Close())
	s = mustOpen(t, dir)
	value(t, s, "a", "2")
	stats, err = s.Stats()
	must(t, err)
	if stats.Sequence != 5 {
		t.Fatal(stats)
	}
}

func TestEmptySnapshot(t *testing.T) {
	dir := t.TempDir()
	s := mustOpen(t, dir)
	must(t, s.Snapshot())
	must(t, s.Close())
	s = mustOpen(t, dir)
	must(t, s.Set("x", "y"))
	value(t, s, "x", "y")
}

func TestEveryIncompleteTail(t *testing.T) {
	base := recFrame(t, 1, []Op{{Type: "set", Key: "a", Value: "saved"}})
	partial := recFrame(t, 2, []Op{{Type: "set", Key: "b", Value: "new"}, {Type: "delete", Key: "a"}})
	for cut := 0; cut < len(partial); cut++ {
		t.Run(fmt.Sprint(cut), func(t *testing.T) {
			dir := t.TempDir()
			write(t, filepath.Join(dir, "wal.log"), append(append([]byte{}, base...), partial[:cut]...))
			s := mustOpen(t, dir)
			value(t, s, "a", "saved")
			if _, err := s.Get("b"); !errors.Is(err, ErrNotFound) {
				t.Fatal(err)
			}
			st, err := s.Stats()
			must(t, err)
			if st.WALBytes != int64(len(base)) || st.RecoveredTailBytes != int64(cut) {
				t.Fatal(st)
			}
			must(t, s.Set("c", "after"))
			must(t, s.Close())
			s = mustOpen(t, dir)
			value(t, s, "c", "after")
		})
	}
}

func TestCorruptWALIsNotRepaired(t *testing.T) {
	good := recFrame(t, 1, []Op{{Type: "set", Key: "x", Value: "y"}})
	cases := map[string][]byte{}
	for _, pos := range []int{0, 4, 8, 12, headerSize + 4} {
		b := append([]byte{}, good...)
		b[pos] ^= 1
		cases[fmt.Sprintf("flip-%d", pos)] = b
	}
	cases["gap"] = recFrame(t, 2, []Op{{Type: "set", Key: "x", Value: "y"}})
	cases["duplicate"] = append(append([]byte{}, good...), good...)
	cases["zero-seq"] = recFrame(t, 0, []Op{{Type: "set", Key: "x", Value: "y"}})
	cases["bad-op"] = recFrame(t, 1, []Op{{Type: "what", Key: "x"}})
	cases["json"] = encodeFrame("KVW1", []byte("{"))
	cases["extra-json"] = encodeFrame("KVW1", []byte(`{"version":1,"seq":1,"ops":[]} {}`))
	cases["version"] = encodeFrame("KVW1", []byte(`{"version":2,"seq":1,"ops":[{"type":"set","key":"x"}]}`))
	cases["unknown-field"] = encodeFrame("KVW1", []byte(`{"version":1,"seq":1,"ops":[],"extra":1}`))
	for name, b := range cases {
		t.Run(name, func(t *testing.T) {
			dir := t.TempDir()
			path := filepath.Join(dir, "wal.log")
			write(t, path, b)
			s, err := Open(dir)
			if s != nil {
				_ = s.Close()
				t.Fatal("opened corrupt data")
			}
			if !errors.Is(err, ErrCorrupt) {
				t.Fatal(err)
			}
			if !bytes.Equal(read(t, path), b) {
				t.Fatal("corrupt WAL was modified")
			}
			// Failed Open must release the process lock.
			write(t, path, nil)
			s = mustOpen(t, dir)
			must(t, s.Close())
		})
	}
}

func TestCorruptSnapshot(t *testing.T) {
	cases := [][]byte{nil, []byte("KVS"), encodeFrame("KVS1", []byte(`{"version":2,"seq":1,"data":{}}`)), encodeFrame("KVS1", []byte(`{"version":1,"seq":1,"data":null}`)), encodeFrame("KVS1", []byte(`{"version":1,"seq":1,"data":{"":"bad"}}`)), append(encodeFrame("KVS1", []byte(`{"version":1,"seq":1,"data":{}}`)), 0)}
	for i, b := range cases {
		t.Run(fmt.Sprint(i), func(t *testing.T) {
			dir := t.TempDir()
			write(t, filepath.Join(dir, "snapshot.kv"), b)
			_, err := Open(dir)
			if !errors.Is(err, ErrCorrupt) {
				t.Fatal(err)
			}
		})
	}
}

func TestSnapshotOldWALAndStaleTemp(t *testing.T) {
	dir := t.TempDir()
	s := mustOpen(t, dir)
	must(t, s.Set("x", "1"))
	must(t, s.Delete("x"))
	old := read(t, filepath.Join(dir, "wal.log"))
	must(t, s.Snapshot())
	must(t, s.Close())
	write(t, filepath.Join(dir, "wal.log"), old)
	write(t, filepath.Join(dir, "snapshot.tmp"), []byte("interrupted"))
	s = mustOpen(t, dir)
	if _, err := s.Get("x"); !errors.Is(err, ErrNotFound) {
		t.Fatal(err)
	}
	must(t, s.Set("after", "3"))
	must(t, s.Close())
	s = mustOpen(t, dir)
	value(t, s, "after", "3")
	must(t, s.Snapshot())
	if _, err := os.Stat(filepath.Join(dir, "snapshot.tmp")); !errors.Is(err, os.ErrNotExist) {
		t.Fatal(err)
	}
}

func TestLockCloseAndMissingWAL(t *testing.T) {
	dir := t.TempDir()
	s := mustOpen(t, dir)
	if _, err := Open(dir); !errors.Is(err, ErrLocked) {
		t.Fatal(err)
	}
	must(t, s.Set("x", "y"))
	must(t, s.Snapshot())
	must(t, s.Close())
	checks := []func() error{func() error { _, e := s.Get("x"); return e }, func() error { return s.Set("x", "z") }, func() error { return s.Delete("x") }, s.Snapshot, func() error { _, e := s.Stats(); return e }, func() error { _, e := s.List(""); return e }}
	for _, f := range checks {
		if !errors.Is(f(), ErrClosed) {
			t.Fatal("closed store accepted operation")
		}
	}
	s = mustOpen(t, dir)
	must(t, s.Close())
	must(t, os.Remove(filepath.Join(dir, "wal.log")))
	if opened, err := Open(dir); err == nil {
		_ = opened.Close()
		t.Fatal("missing WAL accepted")
	}
}

func TestConcurrentOperations(t *testing.T) {
	s := mustOpen(t, t.TempDir())
	var wg sync.WaitGroup
	for i := 0; i < 8; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			for j := 0; j < 20; j++ {
				k := fmt.Sprintf("%d/%d", i, j)
				if err := s.Set(k, k); err != nil {
					t.Error(err)
					return
				}
				if _, err := s.Get(k); err != nil {
					t.Error(err)
				}
				if _, err := s.List(""); err != nil {
					t.Error(err)
				}
				if j%10 == 0 {
					if err := s.Snapshot(); err != nil {
						t.Error(err)
					}
				}
			}
		}(i)
	}
	wg.Wait()
	st, err := s.Stats()
	must(t, err)
	if st.Keys != 160 || st.Sequence != 160 {
		t.Fatal(st)
	}
}

func TestRandomModel(t *testing.T) {
	dir := t.TempDir()
	s := mustOpen(t, dir)
	model := map[string]string{}
	rng := rand.New(rand.NewSource(42))
	for i := 0; i < 300; i++ {
		ops := make([]Op, 1+rng.Intn(4))
		for j := range ops {
			k := fmt.Sprint(rng.Intn(20))
			if rng.Intn(3) == 0 {
				ops[j] = Op{Type: "delete", Key: k}
				delete(model, k)
			} else {
				v := fmt.Sprint(rng.Int())
				ops[j] = Op{Type: "set", Key: k, Value: v}
				model[k] = v
			}
		}
		must(t, s.Batch(ops))
		if i%13 == 0 {
			must(t, s.Snapshot())
		}
		if i%7 == 0 {
			must(t, s.Close())
			s = mustOpen(t, dir)
		}
		pairs, err := s.List("")
		must(t, err)
		got := map[string]string{}
		for _, p := range pairs {
			got[p.Key] = p.Value
		}
		if !reflect.DeepEqual(got, model) {
			t.Fatalf("step %d", i)
		}
	}
}

type faultyWAL struct {
	walFile
	fail string
}

func (f faultyWAL) Write(b []byte) (int, error) {
	if f.fail == "write" {
		n, _ := f.walFile.Write(b[:len(b)/2])
		return n, io.ErrUnexpectedEOF
	}
	if f.fail == "short" {
		return f.walFile.Write(b[:len(b)/2])
	}
	return f.walFile.Write(b)
}
func (f faultyWAL) Sync() error {
	if f.fail == "sync" {
		return errors.New("injected sync failure")
	}
	return f.walFile.Sync()
}
func (f faultyWAL) Truncate(n int64) error {
	if f.fail == "truncate" {
		return errors.New("injected truncate failure")
	}
	return f.walFile.Truncate(n)
}

func TestIOFailurePoisonsStore(t *testing.T) {
	for _, kind := range []string{"write", "short", "sync", "truncate"} {
		t.Run(kind, func(t *testing.T) {
			dir := t.TempDir()
			s := mustOpen(t, dir)
			must(t, s.Set("base", "saved"))
			s.wal = faultyWAL{s.wal, kind}
			var err error
			if kind == "truncate" {
				err = s.Snapshot()
			} else {
				err = s.Set("maybe", "value")
			}
			if !errors.Is(err, ErrUncertain) {
				t.Fatal(err)
			}
			if _, err = s.Get("base"); !errors.Is(err, ErrUncertain) {
				t.Fatal(err)
			}
			if err = s.Set("no", "write"); !errors.Is(err, ErrUncertain) {
				t.Fatal(err)
			}
			must(t, s.Close())
			s = mustOpen(t, dir)
			value(t, s, "base", "saved")
			must(t, s.Set("after", "ok"))
		})
	}
}

func TestSequenceExhausted(t *testing.T) {
	s := mustOpen(t, t.TempDir())
	s.seq = math.MaxUint64
	if err := s.Set("x", "y"); !errors.Is(err, ErrInvalid) {
		t.Fatal(err)
	}
}

func FuzzFrame(f *testing.F) {
	f.Add(encodeFrame("KVW1", []byte(`{"version":1,"seq":1,"ops":[]}`)))
	f.Add([]byte("KVW1"))
	f.Add([]byte{})
	f.Fuzz(func(t *testing.T, b []byte) {
		payload, err := readFrame(bytes.NewReader(b), "KVW1", 65536)
		if err == nil {
			var rec record
			_ = decodeJSON(payload, &rec)
		}
	})
}

func BenchmarkSetSync(b *testing.B) {
	s, err := Open(b.TempDir())
	if err != nil {
		b.Fatal(err)
	}
	defer s.Close()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		if err := s.Set("key", fmt.Sprint(i)); err != nil {
			b.Fatal(err)
		}
	}
}
KV_LAB_SOURCE
printf 'Записан файл: kv/store_test.go\n'
```

**Результат:** `Записан файл: kv/store_test.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 10.1.3. kv/crash_test.go

Аварийное завершение дочернего процесса в заданных точках записи и снимка. Вспомогательный `TestCrashHelper` при обычном запуске пропускается.

```bash
mkdir -p 'kv'
cat > 'kv/crash_test.go' <<'KV_LAB_SOURCE'
package kv

import (
	"bufio"
	"bytes"
	"errors"
	"fmt"
	"os"
	"os/exec"
	"testing"
	"time"
)

// TestCrashHelper runs only as a child launched by startAt. The parent waits
// for an exact stage marker, then sends SIGKILL. No timing guesses or defers.
func TestCrashHelper(t *testing.T) {
	if os.Getenv("KV_LAB_HELPER") != "1" {
		t.Skip("subprocess only")
	}
	s, err := Open(os.Getenv("KV_LAB_DIR"))
	must(t, err)
	stage := os.Getenv("KV_LAB_STAGE")
	stop := func() {
		fmt.Println("READY")
		for {
			time.Sleep(time.Hour)
		}
	}
	if stage == "lock" {
		stop()
	}
	s.testHook = func(got string) {
		if got == stage {
			stop()
		}
	}
	if stage == "wal-written" || stage == "wal-synced" {
		must(t, s.Batch([]Op{{Type: "set", Key: "a", Value: "new"}, {Type: "set", Key: "b", Value: "new"}, {Type: "delete", Key: "gone"}}))
	} else {
		must(t, s.Snapshot())
	}
	t.Fatal("crash point not reached")
}

func startAt(t *testing.T, dir, stage string) (*exec.Cmd, *bytes.Buffer) {
	t.Helper()
	cmd := exec.Command(os.Args[0], "-test.run=^TestCrashHelper$", "-test.timeout=20s")
	cmd.Env = append(os.Environ(), "KV_LAB_HELPER=1", "KV_LAB_DIR="+dir, "KV_LAB_STAGE="+stage)
	var stderr bytes.Buffer
	cmd.Stderr = &stderr
	out, err := cmd.StdoutPipe()
	must(t, err)
	must(t, cmd.Start())
	t.Cleanup(func() {
		if cmd.ProcessState == nil {
			_ = cmd.Process.Kill()
			_ = cmd.Wait()
		}
	})
	ready := make(chan bool, 1)
	go func() {
		scanner := bufio.NewScanner(out)
		for scanner.Scan() {
			if scanner.Text() == "READY" {
				ready <- true
				return
			}
		}
		ready <- false
	}()
	select {
	case ok := <-ready:
		if !ok {
			_ = cmd.Wait()
			t.Fatalf("helper exited: %s", stderr.String())
		}
	case <-time.After(10 * time.Second):
		_ = cmd.Process.Kill()
		_ = cmd.Wait()
		t.Fatalf("helper timeout: %s", stderr.String())
	}
	return cmd, &stderr
}

func killChild(t *testing.T, cmd *exec.Cmd) {
	t.Helper()
	must(t, cmd.Process.Kill())
	if err := cmd.Wait(); err == nil {
		t.Fatal("child exited without SIGKILL")
	}
}

func TestCrashRecovery(t *testing.T) {
	stages := []string{"wal-written", "wal-synced", "snapshot-written", "snapshot-synced", "snapshot-renamed", "snapshot-published", "wal-truncated", "snapshot-complete"}
	for _, stage := range stages {
		t.Run(stage, func(t *testing.T) {
			dir := t.TempDir()
			s := mustOpen(t, dir)
			must(t, s.Set("base", "saved"))
			must(t, s.Set("gone", "old"))
			must(t, s.Snapshot())
			isWrite := stage == "wal-written" || stage == "wal-synced"
			if !isWrite {
				must(t, s.Batch([]Op{{Type: "set", Key: "a", Value: "new"}, {Type: "set", Key: "b", Value: "new"}, {Type: "delete", Key: "gone"}}))
			}
			must(t, s.Close())
			cmd, _ := startAt(t, dir, stage)
			killChild(t, cmd)
			s = mustOpen(t, dir)
			value(t, s, "base", "saved")
			value(t, s, "a", "new")
			value(t, s, "b", "new")
			if _, err := s.Get("gone"); !errors.Is(err, ErrNotFound) {
				t.Fatal(err)
			}
			must(t, s.Set("after", "works"))
			must(t, s.Close())
			s = mustOpen(t, dir)
			value(t, s, "after", "works")
		})
	}
}

func TestCrossProcessLockAndRelease(t *testing.T) {
	dir := t.TempDir()
	cmd, _ := startAt(t, dir, "lock")
	if s, err := Open(dir); !errors.Is(err, ErrLocked) {
		if s != nil {
			_ = s.Close()
		}
		t.Fatalf("expected lock: %v", err)
	}
	killChild(t, cmd)
	s := mustOpen(t, dir)
	must(t, s.Set("reopened", "yes"))
}
KV_LAB_SOURCE
printf 'Записан файл: kv/crash_test.go\n'
```

**Результат:** `Записан файл: kv/crash_test.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 10.1.4. kv/edge_test.go

Граничные размеры и ошибки создания снимка, включая самое первое сохранение.

```bash
mkdir -p 'kv'
cat > 'kv/edge_test.go' <<'KV_LAB_SOURCE'
package kv

import (
	"bytes"
	"encoding/binary"
	"errors"
	"hash/crc32"
	"os"
	"path/filepath"
	"testing"
)

func TestFrameLengthBounds(t *testing.T) {
	for _, n := range []uint32{0, MaxRecordBytes + 1, ^uint32(0)} {
		b := encodeFrame("KVW1", []byte("x"))
		binary.LittleEndian.PutUint32(b[4:8], n)
		binary.LittleEndian.PutUint32(b[8:12], crc32.ChecksumIEEE(b[:8]))
		if _, err := readFrame(bytes.NewReader(b), "KVW1", MaxRecordBytes); !errors.Is(err, ErrCorrupt) {
			t.Fatal(n, err)
		}
	}
}

func TestSnapshotFailureBeforePublication(t *testing.T) {
	for _, blocked := range []string{"snapshot.tmp", "snapshot.kv"} {
		t.Run(blocked, func(t *testing.T) {
			dir := t.TempDir()
			s := mustOpen(t, dir)
			must(t, s.Set("x", "saved"))
			path := filepath.Join(dir, blocked)
			must(t, os.Mkdir(path, 0700))
			if err := s.Snapshot(); err == nil {
				t.Fatal("expected filesystem error")
			}
			// Nothing authoritative was replaced: the instance remains usable.
			value(t, s, "x", "saved")
			must(t, s.Set("y", "also saved"))
			must(t, os.Remove(path))
			must(t, s.Snapshot())
			must(t, s.Close())
			s = mustOpen(t, dir)
			value(t, s, "x", "saved")
			value(t, s, "y", "also saved")
		})
	}
}

func TestCrashFirstSnapshot(t *testing.T) {
	for _, stage := range []string{"snapshot-written", "snapshot-synced", "snapshot-renamed", "snapshot-published", "wal-truncated", "snapshot-complete"} {
		t.Run(stage, func(t *testing.T) {
			dir := t.TempDir()
			s := mustOpen(t, dir)
			must(t, s.Set("x", "saved"))
			must(t, s.Close())
			cmd, _ := startAt(t, dir, stage)
			killChild(t, cmd)
			s = mustOpen(t, dir)
			value(t, s, "x", "saved")
			must(t, s.Set("y", "after"))
			must(t, s.Close())
			s = mustOpen(t, dir)
			value(t, s, "y", "after")
		})
	}
}

func TestSnapshotWithNewRecordsAndIncompleteTail(t *testing.T) {
	dir := t.TempDir()
	s := mustOpen(t, dir)
	must(t, s.Set("old", "snapshot"))
	must(t, s.Snapshot())
	must(t, s.Set("new", "wal"))
	must(t, s.Close())
	path := filepath.Join(dir, "wal.log")
	b := read(t, path)
	partial := recFrame(t, 3, []Op{{Type: "delete", Key: "old"}, {Type: "delete", Key: "new"}})
	write(t, path, append(b, partial[:len(partial)-3]...))
	s = mustOpen(t, dir)
	value(t, s, "old", "snapshot")
	value(t, s, "new", "wal")
	must(t, s.Set("next", "seq3"))
	must(t, s.Close())
	s = mustOpen(t, dir)
	st, err := s.Stats()
	must(t, err)
	if st.Sequence != 3 {
		t.Fatal(st)
	}
}

func TestSnapshotChecksumDamage(t *testing.T) {
	dir := t.TempDir()
	s := mustOpen(t, dir)
	must(t, s.Set("x", "saved"))
	must(t, s.Snapshot())
	must(t, s.Close())
	path := filepath.Join(dir, "snapshot.kv")
	b := read(t, path)
	b[len(b)-1] ^= 1
	write(t, path, b)
	if _, err := Open(dir); !errors.Is(err, ErrCorrupt) {
		t.Fatal(err)
	}
	if !bytes.Equal(read(t, path), b) {
		t.Fatal("damaged snapshot modified")
	}
}
KV_LAB_SOURCE
printf 'Записан файл: kv/edge_test.go\n'
```

**Результат:** `Записан файл: kv/edge_test.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 10.1.5. internal/cli/cli_test.go

Проверки ответов, аргументов, кодов завершения и сессии JSONL.

```bash
mkdir -p 'internal/cli'
cat > 'internal/cli/cli_test.go' <<'KV_LAB_SOURCE'
package cli

import (
	"bytes"
	"encoding/json"
	"os"
	"path/filepath"
	"strings"
	"testing"

	"example.com/kv-lab/kv"
)

func run(t *testing.T, dir, in string, args ...string) (int, string, string) {
	t.Helper()
	var out, err bytes.Buffer
	code := Run(append([]string{"-dir", dir}, args...), strings.NewReader(in), &out, &err)
	return code, out.String(), err.String()
}
func success(t *testing.T, dir, in string, args ...string) string {
	t.Helper()
	code, out, err := run(t, dir, in, args...)
	if code != 0 {
		t.Fatalf("%v: %d %s", args, code, err)
	}
	return out
}

func TestCommands(t *testing.T) {
	dir := t.TempDir()
	success(t, dir, "", "set", "name", "Анна")
	out := success(t, dir, "", "get", "name")
	var p kv.Pair
	if err := json.Unmarshal([]byte(out), &p); err != nil || p.Value != "Анна" {
		t.Fatal(out, err)
	}
	success(t, dir, `[{"type":"set","key":"empty","value":""},{"type":"delete","key":"name"}]`, "batch", "-")
	if code, _, _ := run(t, dir, "", "get", "name"); code != 3 {
		t.Fatal(code)
	}
	success(t, dir, "", "get", "empty")
	success(t, dir, "", "list")
	success(t, dir, "", "list", "em")
	success(t, dir, "", "snapshot")
	success(t, dir, "", "check")
	success(t, dir, "", "stats")
	success(t, dir, "", "delete", "missing")
	path := filepath.Join(t.TempDir(), "batch.json")
	if err := os.WriteFile(path, []byte(`[{"type":"set","key":"file","value":"ok"}]`), 0600); err != nil {
		t.Fatal(err)
	}
	success(t, dir, "", "batch", path)
	success(t, dir, "", "get", "file")
}

func TestBadArgsAndNoSideEffects(t *testing.T) {
	for _, args := range [][]string{{}, {"wat"}, {"set", "x"}, {"get"}, {"delete"}, {"list", "a", "b"}, {"snapshot", "x"}, {"stats", "x"}, {"check", "x"}, {"batch"}, {"shell", "x"}, {"-bad"}, {"help", "x"}, {"version", "x"}} {
		dir := filepath.Join(t.TempDir(), "absent")
		code, _, _ := run(t, dir, "", args...)
		if code != 2 {
			t.Fatalf("%v: %d", args, code)
		}
		if _, err := os.Stat(dir); !os.IsNotExist(err) {
			t.Fatal("invalid invocation created data")
		}
	}
	for _, in := range []string{"{", `[] {}`, `[{"type":"set","key":"x","unknown":1}]`} {
		if code, _, _ := run(t, t.TempDir(), in, "batch", "-"); code != 2 {
			t.Fatal(in, code)
		}
	}
}

func TestHelpAndVersion(t *testing.T) {
	for _, args := range [][]string{{"help"}, {"version"}, {"-h"}} {
		dir := filepath.Join(t.TempDir(), "absent")
		code, _, _ := run(t, dir, "", args...)
		if code != 0 {
			t.Fatal(code)
		}
		if _, err := os.Stat(dir); !os.IsNotExist(err) {
			t.Fatal("created store")
		}
	}
}

func TestShell(t *testing.T) {
	dir := t.TempDir()
	in := "\n" + `{"command":"set","key":"a","value":"1"}` + "\n" + `{"command":"get","key":"a"}` + "\n" + `{"command":"snapshot"}` + "\n"
	out := success(t, dir, in, "shell")
	if len(strings.Split(strings.TrimSpace(out), "\n")) != 3 {
		t.Fatal(out)
	}
	code, out, _ := run(t, dir, `{"command":"get","key":"missing"}`+"\n"+`{"command":"set","key":"b","value":"2"}`+"\n", "shell")
	if code != 3 || !strings.Contains(out, `"ok":false`) || !strings.Contains(out, `"ok":true`) {
		t.Fatal(code, out)
	}
	success(t, dir, "", "get", "b")
	for _, in := range []string{`{"command":"bogus"}`, `{"command":"set","key":"x","ops":[]}`, `{"command":"get","key":"x","value":"ignored"}`, `{"command":"stats","key":"ignored"}`, `{"command":"list","value":"ignored"}`, `{"command":"batch","key":"ignored"}`, `{`} {
		if code, _, _ := run(t, dir, in, "shell"); code != 2 {
			t.Fatal(in, code)
		}
	}
}

func TestExitCodes(t *testing.T) {
	dir := t.TempDir()
	s, err := kv.Open(dir)
	if err != nil {
		t.Fatal(err)
	}
	if code, _, _ := run(t, dir, "", "stats"); code != 4 {
		t.Fatal(code)
	}
	_ = s.Close()
	if err := os.WriteFile(filepath.Join(dir, "wal.log"), bytes.Repeat([]byte{0}, 16), 0600); err != nil {
		t.Fatal(err)
	}
	if code, _, _ := run(t, dir, "", "check"); code != 5 {
		t.Fatal(code)
	}
	if code, _, _ := run(t, t.TempDir(), "", "batch", "/does/not/exist"); code != 1 {
		t.Fatal(code)
	}
}
KV_LAB_SOURCE
printf 'Записан файл: internal/cli/cli_test.go\n'
```

**Результат:** `Записан файл: internal/cli/cli_test.go`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 10.1.6. examples/batch.json

Четыре операции одного пакета. На новой базе после него останутся три ключа.

```bash
mkdir -p 'examples'
cat > 'examples/batch.json' <<'KV_LAB_SOURCE'
[
  {"type":"set","key":"user:1","value":"Анна"},
  {"type":"set","key":"user:2","value":"Борис"},
  {"type":"set","key":"empty","value":""},
  {"type":"delete","key":"old"}
]
KV_LAB_SOURCE
printf 'Записан файл: examples/batch.json\n'
```

**Результат:** `Записан файл: examples/batch.json`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

#### 10.1.7. examples/session.jsonl

Готовая последовательность команд для одной сессии. Каждая строка — отдельный JSON-объект.

```bash
mkdir -p 'examples'
cat > 'examples/session.jsonl' <<'KV_LAB_SOURCE'
{"command":"set","key":"city","value":"Москва"}
{"command":"get","key":"city"}
{"command":"batch","ops":[{"type":"set","key":"a","value":"1"},{"type":"set","key":"b","value":"2"}]}
{"command":"list"}
{"command":"snapshot"}
{"command":"stats"}
KV_LAB_SOURCE
printf 'Записан файл: examples/session.jsonl\n'
```

**Результат:** `Записан файл: examples/session.jsonl`. Файл создан или заменён. Сами `mkdir` и `cat` при успехе ничего не выводят.

**Сборка этапа**


```bash
gofmt -w kv internal cmd
go build -trimpath -o bin/kv ./cmd/kv
cp bin/kv bin/kv-06-final
./bin/kv version
```

**Результат:** `kv-lab 1.0.0`. Готовая программа находится в `bin/kv`, рядом сохранены все шесть версий.

### 10.2. Повторить опыт с двумя процессами

```bash
go run ./tools/experiment -bin "$PWD/bin/kv" -expect locked
```

**Результат:** второй процесс получит `store is already open` и код `4`. После закрытия первого процесса база снова откроется. В конце:

```text
ПРОВЕРЕНО: второй процесс получил код 4, после закрытия первого база открылась.
```

Исправление проверено тем же сценарием открытия, который до этого допускал потерю данных.

### 10.3. Попробовать блокировку вручную

Эту проверку удобно показать в двух окнах Terminal. В первом:

```bash
session_dir=$(mktemp -d "${TMPDIR:-/tmp}/kv-manual-session.XXXXXX")
printf '%q -dir %q stats\n' "$PWD/bin/kv" "$session_dir"
```

**Результат:** готовая команда с полными путями для второго окна. Выделите напечатанную строку и скопируйте через `Command+C`.

В первом окне выполните:

```bash
./bin/kv -dir "$session_dir" shell
```

**Результат:** программа ничего не напечатает и будет ждать ввода. Вставьте эту строку и нажмите Return:

```json
{"command":"stats"}
```

**Результат:** JSON с нулевыми значениями статистики. Сессия остаётся открытой.

1. Нажмите `Command+N`, чтобы открыть второе окно Terminal.
2. Вставьте скопированную команду `stats` и нажмите Return.
3. Появится `store is already open`.
4. В первом окне на пустой строке нажмите `Control+D`. Вернётся приглашение оболочки.
5. Во втором окне нажмите стрелку вверх и Return, чтобы повторить `stats`.

**Результат:** теперь статистика читается. Файл `LOCK` остаётся, но блокировка уже освобождена. Удалять этот файл вручную не нужно.

Продолжайте в первом окне, где сохранены переменные и открыта рабочая папка.

## 11. Проверяем итоговую программу

### 11.1. Пакет и пустое значение

```bash
batch_dir=$(mktemp -d "${TMPDIR:-/tmp}/kv-final-batch.XXXXXX")
cat examples/batch.json
./bin/kv -dir "$batch_dir" batch examples/batch.json
./bin/kv -dir "$batch_dir" list 'user:'
./bin/kv -dir "$batch_dir" get empty
./bin/kv -dir "$batch_dir" stats
```

**Результат:** `cat` покажет четыре операции. Пакет вернёт успех. В списке будут `user:1` со значением `Анна` и `user:2` со значением `Борис`. Ключ `empty` существует со значением `""`. Всего три ключа, `sequence: 1`: весь пакет попал в одну запись журнала.

Проверим отклонение пакета целиком:

```bash
invalid_code=0
printf '%s\n' '[{"type":"set","key":"should-not-exist","value":"x"},{"type":"wrong","key":"x"}]' |
  ./bin/kv -dir "$batch_dir" batch - || invalid_code=$?
printf 'Код неверного пакета: %s\n' "$invalid_code"
test "$invalid_code" -eq 2
missing_code=0
./bin/kv -dir "$batch_dir" get should-not-exist || missing_code=$?
test "$missing_code" -eq 3
./bin/kv -dir "$batch_dir" stats
```

**Результат:** ошибка неизвестной операции и код `2`. Ключ `should-not-exist` не найден. Ключей по-прежнему три, номер записи остаётся `1`. Первая операция неверного пакета тоже не выполнялась. Команды `test` при совпадении кодов ничего не выводят.

### 11.2. Готовая сессия JSONL

```bash
jsonl_dir=$(mktemp -d "${TMPDIR:-/tmp}/kv-final-jsonl.XXXXXX")
cat examples/session.jsonl
./bin/kv -dir "$jsonl_dir" shell < examples/session.jsonl
./bin/kv -dir "$jsonl_dir" get city
```

**Результат:** сначала текст команд, затем ответы. В конце сессии три ключа и пустой журнал после снимка. Отдельное чтение после закрытия сессии вернёт `Москва`.

### 11.3. Настоящее аварийное завершение

```bash
go test -count=1 -run 'TestCrashRecovery|TestCrashFirstSnapshot|TestCrossProcessLockAndRelease' -v ./kv
```

**Результат:** тесты с `PASS` и итоговая строка `ok`. Здесь уже проверяется настоящее завершение дочерних процессов через `SIGKILL`: 14 точек записи и снимка, а также освобождение блокировки после аварии. После каждого сбоя данные открываются заново и сравниваются с ожидаемыми.

Ранние опыты показывали отдельные проблемы. Эти тесты проверяют совместную работу журнала, восстановления, снимка и блокировки.

### 11.4. Общая проверка

```bash
go vet ./...
go test -count=1 -race -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
```

**Результат:** `go vet` при успехе молчит. Тесты показывают `ok`, затем появляется таблица покрытия функций. `FAIL` и `DATA RACE` быть не должно. Вспомогательный `TestCrashHelper` пропускается при обычном запуске, потому что запускается отдельно дочерним процессом. У `cmd/kv` и вспомогательной программы `tools/experiment` нет собственных unit-тестов.

### 11.5. Случайные входы и время записи

```bash
go test -run '^$' -fuzz '^FuzzFrame$' -fuzztime=5s -parallel=2 ./kv
```

**Результат:** число проверенных входов и примерно через пять секунд `PASS`. Число зависит от Mac и нагрузки.

```bash
go test -run '^$' -bench '^BenchmarkSetSync$' -benchtime=100x ./kv
```

**Результат:** строка `BenchmarkSetSync` с числом `100` и временем в `ns/op`. Это измерение вашего запуска, одинакового значения на всех компьютерах ждать не нужно.

<a id="advanced"></a>

### 11.6. Уровень П: сбой во время создания снимка

Этот пункт выполняется после основной проверки. Сначала повторите три точки аварии по три раза:

```bash
go test -count=3 -run '^TestCrashRecovery$/(snapshot-written|snapshot-published|wal-truncated)$' -v ./kv
sed -n '/func (s \*Store) Snapshot()/,/^}/p' kv/store.go
```

**Результат:** девять успешных запусков под-тестов и `PASS`, затем код метода `Snapshot`. В каждом случае дочерний процесс завершается через `SIGKILL`, а новый процесс открывает базу и проверяет значения.

Сопоставьте место остановки с порядком действий в `Snapshot`:

| Точка | Что успело произойти | Что используется при восстановлении |
|---|---|---|
| `snapshot-written` | временный файл записан, новый снимок ещё не опубликован | прежний снимок и журнал |
| `snapshot-published` | новый снимок опубликован и каталог синхронизирован, журнал ещё не очищен | новый снимок, уже учтённые записи журнала повторно не применяются |
| `wal-truncated` | журнал очищен | новый снимок |

Теперь проверьте обычную ошибку файловой системы. Создадим каталог на месте будущего временного файла. Из-за этого снимок не сможет записаться, но предыдущие данные должны остаться доступными.

```bash
mkdir -p experiments
cat > experiments/snapshot-failure.sh <<'KV_LAB_SOURCE'
#!/bin/bash
set -euo pipefail
cd "$(dirname "$0")/.."
kv_bin="$PWD/bin/kv"
case_dir=$(mktemp -d "${TMPDIR:-/tmp}/kv-snapshot-failure.XXXXXX")
"$kv_bin" -dir "$case_dir" set city Москва
mkdir "$case_dir/snapshot.tmp"
snapshot_code=0
"$kv_bin" -dir "$case_dir" snapshot || snapshot_code=$?
printf 'Код неудачного снимка: %s\n' "$snapshot_code"
test "$snapshot_code" -eq 1
test "$("$kv_bin" -dir "$case_dir" get city)" = '{"key":"city","value":"Москва"}'
"$kv_bin" -dir "$case_dir" set country Россия
rmdir "$case_dir/snapshot.tmp"
"$kv_bin" -dir "$case_dir" snapshot
test "$(wc -c < "$case_dir/wal.log" | tr -d '[:space:]')" -eq 0
test "$("$kv_bin" -dir "$case_dir" get city)" = '{"key":"city","value":"Москва"}'
test "$("$kv_bin" -dir "$case_dir" get country)" = '{"key":"country","value":"Россия"}'
printf 'PASS: после отказа и повторного снимка оба значения доступны\nБаза: %s\n' "$case_dir"
KV_LAB_SOURCE
printf 'Записан файл: experiments/snapshot-failure.sh\n'
bash experiments/snapshot-failure.sh
```

**Результат:** ожидаемая ошибка создания `snapshot.tmp`, код `1`, затем успешный снимок и строка `PASS`. Проверки `test` молчат при успехе. Если условие не выполнено, скрипт остановится до `PASS`. Путь в конце позволяет найти базу опыта.

В видео объясните, почему ошибка до публикации снимка не должна уничтожать журнал. Покажите, что повторный снимок очистил журнал, а оба значения читаются отдельными запусками программы. Отдельно поясните, почему ни этот опыт, ни `SIGKILL` не проверяют отключение питания компьютера.

<a id="expert"></a>

### 11.7. Уровень Э: собственный сценарий проверки

Для этого уровня нужно составить одну дополнительную проверку из знакомых команд CLI. Менять Go-код для неё не требуется. Проверка должна сочетать несколько операций и проверять новый случай. Простая замена `Москва` на другой город в готовом опыте не считается новым сценарием.

Можно выбрать один из вариантов:

1. **Несколько действий с одним ключом внутри пакета.** Запишите начальное значение. Одним пакетом выполните `set`, `delete`, снова `set` для этого ключа. Создайте снимок и прочитайте ключ новым процессом. Должно вернуться значение последнего `set`. Затем отправьте пакет с допустимой первой операцией и неверной последней. После отказа значение должно остаться прежним. Проверьте также, что номер последовательности не вырос от отклонённого пакета.
2. **Ключ на границе допустимой длины.** Сравните ключи из 512 и 513 букв `я`: это 1024 и 1026 байт в UTF-8. Первый должен сохраняться и читаться после снимка. Второй должен отклоняться с кодом `2`. Проверьте, что отказ не изменил количество ключей и номер последовательности.

До выполнения назовите ожидаемые значения, коды завершения и поля `stats`. Для скрипта возьмите за основу опыт из 11.6:

```bash
cp experiments/snapshot-failure.sh experiments/custom-scenario.sh
open -e experiments/custom-scenario.sh
```

**Результат:** TextEdit откроет копию скрипта. Исходный опыт останется в `snapshot-failure.sh`.

В открытом файле оставьте начальные строки до создания `case_dir` включительно. Замените последующие команды на выбранный сценарий. Для пакетов используйте формат из 11.1, для сравнения значений и кодов — команды `test` из исходного скрипта. Добавьте проверку каждого ожидаемого результата и строку `printf 'PASS: собственный сценарий\n'` в конце. Нажмите `Command+S`, затем вернитесь в Terminal.

```bash
bash -n experiments/custom-scenario.sh
bash experiments/custom-scenario.sh
```

**Результат:** проверка синтаксиса ничего не выводит. Сам сценарий выводит ответы CLI и доходит до `PASS` только после выполнения всех проверок. Одного просмотра вывода без `test` недостаточно: при неверном результате скрипт должен завершаться с ошибкой.

Покажите в видео содержимое скрипта, его запуск и объяснение результата. Укажите, какую гарантию проверили и что осталось за пределами опыта. Например, успешная проверка границы длины ключа ничего не говорит об устойчивости к аварии. Скрипт останется в `experiments` и попадёт в сохранённый проект из раздела 12. Готовые Go-тесты после добавления своего сценария должны по-прежнему проходить.

## 12. Что в итоге создано

В `$work_dir` находится ваша реализация. Она прошла путь от словаря, исчезающего при закрытии, до дискового хранилища с восстановлением и исключительным доступом к базе.

- `bin/kv` — итоговая программа.
- `bin/kv-01-memory` … `bin/kv-06-final` — версии для сравнения.
- `kv` — код хранения и тесты.
- `internal/cli` и `cmd/kv` — команды и точка запуска.
- `examples` — примеры пакета и сессии.
- `tools/experiment` — сценарий опыта с двумя процессами.

Откройте результат в Finder:

```bash
open "$work_dir"
```

**Результат:** Finder покажет рабочую папку.

Сохраните проект в постоянное место:

```bash
saved_dir=$(mktemp -d "$HOME/Desktop/kv-result.XXXXXX")
cp -R "$work_dir/." "$saved_dir/"
printf 'Проект сохранён: %s\n' "$saved_dir"
```

**Результат:** на Рабочем столе появится папка с уникальным окончанием. В ней исходники, тесты и собранные программы. Go остаётся в папке задания.

Учебные базы лежат отдельно. Для сохранения исправной базы из этапов 4–6 завершите открытые сессии через `Control+D`, затем выполните:

```bash
cp -R "$frame_dir" "$saved_dir/database"
"$saved_dir/bin/kv" -dir "$saved_dir/database" list
```

**Результат:** в сохранённом проекте появится папка `database`. В ней ключ `counter` со значением `6`. Программа читает снимок и последующие изменения журнала.

На этом основные шаги завершены. Попробуйте объяснить своими словами, какая проверка на каждом этапе не проходила раньше и почему теперь проходит.

## 13. Если что-то пошло не так

### Go не найден

В текущем окне выполните:

```bash
export PATH="$project_dir/.tools/go1.27.3/go/bin:$PATH"
export GOTOOLCHAIN=local
go version
```

**Результат:** `go1.27.3`. Если окно было закрыто и `project_dir` больше не задана, снова перейдите в папку задания по пункту 1.2 и повторите пункт 1.4, затем настройку Go из раздела 2.

### Go не видит проект

```bash
cd "$work_dir"
pwd
ls -l go.mod
```

**Результат:** путь к вашей реализации и найденный `go.mod`. В новом окне перейдите в сохранённую папку через Finder по пункту 1.2.

### После этапа 5 функции объявлены дважды

Проверьте, остался ли старый файл:

```bash
ls kv
```

**Результат:** список файлов пакета. После перехода к этапу 5 в нём не должно быть `journal.go`. Если он есть, выполните блок из пункта 9.1 и повторите сборку из пункта 9.1 после создания всех файлов этапа.

### База сообщает о повреждении

В пунктах 6.4 и 8.3 ошибка ожидается. Для обычной работы используйте исправную `$frame_dir` после этапа 4. Текстовые журналы `$wal_dir` и `$edited_dir` читаются только сохранённой версией этапа 3:

```bash
./bin/kv-03-recovery -dir "$wal_dir" get city
```

**Результат:** `Москва`. Итоговая версия не преобразует этот ранний формат автоматически.

### Terminal ждёт продолжения после вставки кода

Убедитесь, что вставили отдельную завершающую строку `KV_LAB_SOURCE` без пробелов перед ней и нажали Return. Если часть блока потеряна, нажмите `Control+C` и повторите **весь блок текущего этапа**. При повторе он заменит только указанный файл. После завершения появится `Записан файл: ...`.
