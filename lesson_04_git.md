# Урок 4 — Git: контроль версий

## Зачем это нужно

Git — это не просто "сохранялка кода". Это система которая позволяет: работать нескольким людям над одним проектом без хаоса, откатиться к любой предыдущей версии кода, понять кто и зачем изменил каждую строку, безопасно экспериментировать не ломая рабочий код. Без Git невозможно работать в команде. Без Git невозможно настроить CI/CD. Без Git тебя не возьмут ни на одну работу в IT.

---

## Часть 1 — Как Git работает изнутри

### Три области Git

```
Рабочая директория    Индекс (Stage)      Репозиторий (.git)
(Working Directory)   (Staging Area)      (Repository)

  файлы на диске   →   git add   →   git commit   →   история коммитов
```

- **Рабочая директория** — обычные файлы которые ты редактируешь
- **Индекс (staging area)** — "зона подготовки", файлы которые войдут в следующий коммит
- **Репозиторий** — база данных всех коммитов, хранится в папке `.git`

### Что такое коммит

Коммит — это снимок состояния проекта в определённый момент времени. Каждый коммит содержит:
- Уникальный хэш (SHA-1): `a3f9b2c...`
- Автора и дату
- Сообщение
- Ссылку на родительский коммит
- Снимок всех файлов

История коммитов — это цепочка: каждый коммит знает своего родителя.

### Ветки

Ветка — это просто указатель на коммит. Когда ты создаёшь новую ветку — Git просто создаёт новый указатель. Когда делаешь коммит в ветке — указатель двигается вперёд.

```
main:    A → B → C
                  ↑
feature:          C → D → E
```

`HEAD` — специальный указатель на текущий коммит (на то где ты сейчас находишься).

---

## Часть 2 — Базовые команды

### Настройка и инициализация

```bash
# Первоначальная настройка (один раз на машине)
git config --global user.name "Имя Фамилия"
git config --global user.email "email@example.com"
git config --global core.editor "nano"       # Редактор по умолчанию
git config --list                            # Посмотреть все настройки

# Создать новый репозиторий
git init                        # В текущей папке
git init my-project             # Создать папку и инициализировать

# Клонировать существующий репозиторий
git clone https://github.com/user/repo.git
git clone https://github.com/user/repo.git my-folder   # В конкретную папку
```

### Основной рабочий цикл

```bash
# 1. Посмотреть статус
git status

# 2. Добавить файлы в индекс
git add file.txt                # Конкретный файл
git add src/                    # Всю папку
git add .                       # Все изменённые файлы
git add -p                      # Интерактивно — выбрать части файла

# 3. Сделать коммит
git commit -m "Add user authentication"
git commit                      # Откроет редактор для сообщения

# Добавить и закоммитить одной командой (только уже отслеживаемые файлы)
git commit -am "Fix typo in README"
```

### Просмотр истории

```bash
git log                         # Полная история
git log --oneline               # Краткая: хэш и сообщение
git log --oneline --graph       # С деревом веток
git log --oneline -10           # Последние 10 коммитов
git log --author="username"     # Коммиты конкретного автора
git log -- file.txt             # История конкретного файла

# Посмотреть изменения
git diff                        # Что изменилось но не добавлено в индекс
git diff --staged               # Что добавлено в индекс (войдёт в коммит)
git diff HEAD~1                 # Изменения относительно предыдущего коммита
git show a3f9b2c                # Показать конкретный коммит
```

### Отмена изменений

```bash
# Отменить изменения в рабочей директории (до git add)
git restore file.txt            # Вернуть файл к последнему коммиту
git restore .                   # Отменить все изменения

# Убрать файл из индекса (после git add, до git commit)
git restore --staged file.txt

# Отменить последний коммит но сохранить изменения
git reset HEAD~1                # Коммит отменён, изменения остались в рабочей директории
git reset --soft HEAD~1         # Коммит отменён, изменения остались в индексе

# Отменить последний коммит и выбросить изменения (ОСТОРОЖНО)
git reset --hard HEAD~1

# Создать новый коммит который отменяет старый (безопасно для shared репозиториев)
git revert a3f9b2c
```

> ⚠️ `git reset --hard` удаляет изменения безвозвратно. Используй с осторожностью.

---

## Часть 3 — Ветки

```bash
# Создать ветку
git branch feature-login         # Создать
git checkout -b feature-login    # Создать и сразу переключиться
git switch -c feature-login      # Современный способ

# Переключиться между ветками
git checkout main
git switch main                  # Современный способ

# Список веток
git branch                       # Локальные ветки
git branch -a                    # Все ветки включая удалённые

# Удалить ветку
git branch -d feature-login      # Удалить (если слита)
git branch -D feature-login      # Принудительно удалить

# Переименовать ветку
git branch -m old-name new-name
```

### Слияние веток (merge)

```bash
# Находясь в main, влить ветку feature-login
git checkout main
git merge feature-login

# Типы merge:
# Fast-forward — если main не изменялся, просто двигает указатель вперёд
# Merge commit  — создаёт новый коммит слияния

# Принудительно создать merge commit даже при fast-forward
git merge --no-ff feature-login
```

### Конфликты при слиянии

Конфликт возникает когда два человека изменили одну и ту же строку по-разному:

```
<<<<<<< HEAD
print("Hello World")      # Твоя версия (main)
=======
print("Hello, World!")    # Версия из feature-login
>>>>>>> feature-login
```

Что делать:
1. Открыть файл с конфликтом
2. Убрать маркеры `<<<<<<<`, `=======`, `>>>>>>>`
3. Оставить нужный вариант (или объединить оба)
4. Сделать `git add file.txt`
5. Сделать `git commit`

### Rebase — альтернатива merge

```bash
# Находясь в feature-login, перебазироваться на main
git checkout feature-login
git rebase main

# Интерактивный rebase — редактировать историю
git rebase -i HEAD~3         # Изменить последние 3 коммита
```

Rebase переписывает историю — коммиты получают новые хэши. Никогда не делай rebase на ветках которые уже запушены и используются другими.

---

## Часть 4 — Работа с удалённым репозиторием

```bash
# Посмотреть удалённые репозитории
git remote -v

# Добавить удалённый репозиторий
git remote add origin https://github.com/user/repo.git

# Отправить изменения
git push origin main             # Отправить ветку main
git push -u origin main          # Первый push: установить upstream
git push                         # После установки upstream — просто push
git push origin feature-login    # Отправить другую ветку

# Получить изменения
git fetch origin                 # Скачать, но не сливать
git pull                         # fetch + merge
git pull --rebase                # fetch + rebase (чище история)

# Удалить ветку на удалённом репозитории
git push origin --delete feature-login
```

### .gitignore — что не коммитить

Создай файл `.gitignore` в корне проекта:

```gitignore
# Переменные окружения и секреты
.env
.env.local
*.key
*.pem

# Зависимости
node_modules/
vendor/

# Артефакты сборки
build/
dist/
*.o
*.exe

# IDE файлы
.idea/
.vscode/
*.swp

# Операционная система
.DS_Store
Thumbs.db

# Логи
*.log
logs/
```

---

## Часть 5 — Хорошие практики

### Как писать сообщения коммитов

Плохо:
```
fix
update
changes
fix bug
```

Хорошо:
```
Add user authentication via JWT
Fix null pointer exception in payment handler
Refactor database connection pooling
Update README with installation instructions
```

Формат:
- Первая строка — краткое описание (до 72 символов), глагол в настоящем времени
- Пустая строка
- Подробное описание если нужно — что и почему (не как)

```
Add rate limiting to API endpoints

Without rate limiting, a single client can overwhelm the server
with requests. This adds token bucket algorithm limiting each
IP to 100 requests per minute.

Fixes #123
```

### Branching стратегия — GitFlow упрощённо

```
main          — стабильный код, всегда работает, идёт в продакшен
dev           — интеграционная ветка, сюда сливают фичи
feature/xxx   — разработка новой функции
hotfix/xxx    — срочное исправление бага в продакшене
```

Рабочий процесс:
1. Создал ветку `feature/add-login` от `dev`
2. Разработал, закоммитил
3. Открыл Pull Request в `dev`
4. Код-ревью, исправления
5. Слияние в `dev`
6. Когда `dev` стабилен — слияние в `main`

---

## Практические задания

### Задание 4.1 — Первый репозиторий

```bash
# Настрой git если ещё не сделал
git config --global user.name "Твоё Имя"
git config --global user.email "твой@email.com"

# Создай репозиторий
mkdir ~/projects/todo-service
cd ~/projects/todo-service
git init

# Создай первый файл
echo "# TODO Service" > README.md
echo "Простой сервис управления задачами" >> README.md

# Первый коммит
git add README.md
git commit -m "Initial commit: add README"

# Посмотри историю
git log
```

**Что нужно сделать:** Создай ещё 3 файла (`main.go`, `handler.go`, `Makefile` — пока пустые), добавь их в разных коммитах с осмысленными сообщениями. Посмотри историю через `git log --oneline`.

---

### Задание 4.2 — Работа с ветками

```bash
cd ~/projects/todo-service

# Создай ветку для новой функции
git switch -c feature/add-tasks

# Внеси изменения
echo 'package main

type Task struct {
    ID   int
    Name string
    Done bool
}' > task.go

git add task.go
git commit -m "Add Task struct"

# Добавь ещё один коммит
echo 'func (t *Task) Complete() {
    t.Done = true
}' >> task.go

git add task.go
git commit -m "Add Complete method to Task"

# Посмотри разницу между ветками
git log --oneline --graph --all
```

**Что нужно сделать:**
1. Вернись в `main` и внеси изменение в `README.md` (добавь строку)
2. Слей `feature/add-tasks` в `main` через `git merge`
3. Посмотри как выглядит история после слияния
4. Удали ветку `feature/add-tasks`

---

### Задание 4.3 — Конфликты

```bash
cd ~/projects/todo-service

# Создай ветку
git switch -c feature/update-readme

# Измени README в этой ветке
echo "# TODO Service v2" > README.md
git add README.md
git commit -m "Update title in README"

# Вернись в main и тоже измени README
git switch main
echo "# TODO Service - Production Ready" > README.md
git add README.md
git commit -m "Update README title for production"

# Попробуй слить — будет конфликт
git merge feature/update-readme
```

**Что нужно сделать:**
1. Открой `README.md` и реши конфликт — оставь заголовок `# TODO Service`
2. Завершим слияние через `git add` и `git commit`
3. Объясни своими словами почему возник конфликт и как Git помогает его решить

---

### Задание 4.4 — GitHub

1. Зарегистрируйся на `https://github.com` если нет аккаунта
2. Создай новый репозиторий `todo-service` (без README — он уже есть локально)
3. Подключи локальный репозиторий к GitHub:

```bash
git remote add origin https://github.com/твой_ник/todo-service.git
git push -u origin main
```

4. Создай файл `.gitignore` с нужным содержимым, закоммить и запушь
5. Создай ветку `feature/api-endpoints` локально, сделай коммит, запушь ветку на GitHub
6. Посмотри на GitHub как выглядит репозиторий с несколькими ветками

---

### Задание 4.5 — Финальный проект урока

В репозитории `todo-service` сделай следующее:

1. Создай ветку `feature/task-storage`
2. Создай файл `storage.go` с содержимым:

```go
package main

var tasks []Task
var nextID = 1

func AddTask(name string) Task {
    task := Task{ID: nextID, Name: name, Done: false}
    tasks = append(tasks, task)
    nextID++
    return task
}

func GetTasks() []Task {
    return tasks
}
```

3. Сделай коммит с правильным сообщением
4. Добавь функцию `DeleteTask(id int)` в отдельном коммите
5. Влей ветку в `main` с флагом `--no-ff` (чтобы создать merge commit)
6. Запушь всё на GitHub
7. Через `git log --oneline --graph` покажи историю и объясни что означает каждая линия в графе

---

## Шпаргалка

| Команда | Что делает |
|---------|-----------|
| `git init` | Создать репозиторий |
| `git clone URL` | Клонировать репозиторий |
| `git status` | Статус файлов |
| `git add .` | Добавить всё в индекс |
| `git commit -m "msg"` | Сделать коммит |
| `git log --oneline` | Краткая история |
| `git diff` | Несохранённые изменения |
| `git switch -c branch` | Создать и переключиться в ветку |
| `git merge branch` | Слить ветку |
| `git push origin main` | Отправить на GitHub |
| `git pull` | Получить изменения |
| `git restore file` | Отменить изменения в файле |

---

## Ресурсы для изучения

- **Интерактивный тренажёр:** `https://learngitbranching.js.org` — лучший способ понять ветки визуально, есть русский язык
- **Официальная книга:** `https://git-scm.com/book/ru/v2` — бесплатно на русском
- **Шпаргалка:** `https://training.github.com/downloads/ru/github-git-cheat-sheet/`
- **Визуализация:** `https://visualizing-git.netlify.app` — смотри как команды меняют граф

---

## Как понять что урок пройден

- [ ] Понимаю разницу между рабочей директорией, индексом и репозиторием
- [ ] Умею создавать коммиты с осмысленными сообщениями
- [ ] Умею создавать ветки, переключаться между ними и сливать
- [ ] Умею решать конфликты при слиянии
- [ ] Настроил репозиторий на GitHub и умею делать push/pull
- [ ] Создал `.gitignore` и понимаю зачем он нужен
- [ ] Могу объяснить что такое `HEAD` и как двигаются указатели веток

---

*Следующий урок: Язык программирования Go — основы*
