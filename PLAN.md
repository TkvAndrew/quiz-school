# План реализации сайта «Викторина про школу» на HTML + CSS + JS

## Цель
Сделать одной страницей (HTML) интерактивную викторину: 10 вопросов про школу,
в каждом вопросе 5 вариантов ответа. Данные хранятся **списками** и **словарями**,
при правильном ответе картинка меняется на надпись «Молодец, все правильно!».

## Итоговая структура файлов
```
vork in opencode/
├── quiz-school.html   # сам сайт (HTML + CSS + JS в одном файле)
├── PLAN.md            # этот план
├── README.md          # описание проекта
├── tunnel.log         # лог публичного туннеля (появляется при выкладке)
├── vb_python_lesson.py      # 🎲 генератор идей (консольная версия)
└── vb_python_lesson_2/      # 💡 генератор идей (веб-версия на Streamlit)
```

Запуск: просто открыть `quiz-school.html` двойным кликом в браузере.
Никаких установок и зависимостей не нужно.

---

## Шаги реализации

### Шаг 1. Разметка страницы
```html
<div class="card">
  <h1>📝 Викторина про школу</h1>
  <div id="start">…выбор сложности…</div>
  <div id="quiz" style="display:none">
    <p id="questionText"></p>
    <ul class="options" id="optionsList"></ul>
    <img id="mascot"> <div id="praise">Молодец, все правильно! 🎉</div>
    <div class="timebar"><div class="time-fill" id="timeFill"></div></div>
  </div>
  <div id="result" style="display:none">…итог…</div>
</div>
```
Три экрана (`start`, `quiz`, `result`) лежат в одной карточке и переключаются
через `style.display` — никаких перезагрузок страницы.

### Шаг 2. Данные — список словарей
```js
const QUIZ = [
  { question: "Как называется человек, который руководит школой?",
    options: ["Директор", "Библиотекарь", "Охранник", "Повар", "Уборщик"], correct: 0 },
  ...
];
```
- `QUIZ` — **список** из 10 словарей (объектов);
- у каждого словаря: вопрос, **список** `options` из 5 вариантов и индекс
  правильного ответа `correct`;
- варианты на странице строятся тоже списком `<ul><li><button>`.

### Шаг 3. Настройки сложности — словарь
```js
const LEVELS = {
  easy:   { name: "Лёгкий",  count: 5,  time: 8, penalty: 0 },
  medium: { name: "Средний", count: 10, time: 5, penalty: 0 },
  hard:   { name: "Сложный", count: 10, time: 3, penalty: 1 }
};
```
Один словарь описывает всю логику уровня: сколько вопросов, сколько секунд
на ответ и какой штраф за ошибку. Кнопки на стартовом экране читают
`btn.dataset.level` и передают ключ в `startGame(key)`.

### Шаг 4. Перемешивание при каждом запуске
```js
function shuffle(list) {
  const arr = list.slice();
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    const tmp = arr[i]; arr[i] = arr[j]; arr[j] = tmp;
  }
  return arr;
}
```
Алгоритм Фишера — Йетса вызывается дважды:
1. `shuffle(QUIZ)` — **другой порядок вопросов** при каждой загрузке страницы;
2. `shuffle(item.options…)` — **правильный ответ тоже попадает в случайную позицию**.

### Шаг 5. Отрисовка текущего вопроса
```js
function renderQuestion() {
  const item = deck[current];
  optionsList.innerHTML = "";
  item.options.forEach(function (text, i) {
    const li = document.createElement("li");
    const btn = document.createElement("button");
    btn.className = "opt-btn";
    btn.textContent = (i + 1) + ". " + text;
    btn.addEventListener("click", function () { check(btn, i); });
    li.appendChild(btn);
    optionsList.appendChild(li);
  });
  startAnswerTimer();     // запускаем табло времени
}
```
Каждый вопрос строится заново — старые кнопки убираются через `innerHTML = ""`.

### Шаг 6. Проверка ответа и главная фишка
```js
if (choice === deck[current].correct) {
  locked = true;
  score++;
  mascot.style.display = "none";    // КАРТИНКА прячется
  praise.style.display = "block";   // ПОЯВЛЯЕТСЯ надпись
  startAutoAdvance();               // через N секунд — следующий вопрос
} else {
  btn.classList.add("wrong");       // кнопка краснеет и выключается
}
```
**Правильный ответ:** картинка-смайлик исчезает → надпись «Молодец, все
правильно! 🎉», очко +1, кнопка «Дальше →».
**Неверный ответ:** кнопка краснеет, подсказка «Неверно, попробуй ещё раз!»
(на сложном уровне ещё и `−1 очко`, счёт не уходит в минус).

### Шаг 7. Табло времени (полоска снизу)
```js
answerTimer = setInterval(function () {
  timeLeft -= TICK;               // TICK = 100 мс
  if (timeLeft <= 0) { timeIsUp(); return; }
  paintTimebar();                 // ширина полоски = timeLeft / timeTotal
}, TICK);
```
- полоска уменьшается плавно, меняет цвет: зелёная → жёлтая (≤50%) → красная (≤25%);
- подпись «Осталось времени: N сек.» краснеет на последних 2 секундах;
- при правильном ответе табло **останавливается** («Ответ принят!»).

### Шаг 8. Время истекло → «Попробовать снова»
```js
function timeIsUp() {
  stopTimers();
  locked = true;
  document.querySelectorAll(".opt-btn").forEach(b => b.disabled = true);
  timeLabel.textContent = "Время вышло!";
  retryBtn.style.display = "inline-block";   // 🔁 Попробовать снова
}
retryBtn.addEventListener("click", function () { renderQuestion(); });
```
Вопрос не меняется: тот же номер, новое время, кнопки снова активны.

### Шаг 9. Автопереход с обратным отсчётом
```js
function startAutoAdvance() {
  const delay = LEVELS[level].time;           // 8 / 5 / 3 секунды
  let left = delay;
  countdown.textContent = "Следующий вопрос через " + left + " сек.";
  tickTimer = setInterval(...);               // 5 → 4 → 3 …
  autoTimer  = setTimeout(next, delay * 1000);
}
```
Кнопка «Дальше →» остаётся: если нажать её раньше, `stopTimers()` снимает
таймер и переход получается ровно один.

### Шаг 10. Экран результата и перезапуск
```js
function showResult() {
  finalScore.textContent = score + " / " + TOTAL;
  const part = score / TOTAL;                 // доля — работает и для 5, и для 10 вопросов
}
restartBtn.addEventListener("click", restart);      // 🔄 Ещё раз (новое перемешивание)
levelBtn.addEventListener("click", showStart);      // 🎚 Сменить сложность
```

### Шаг 11. Публикация в общую сеть (по желанию)
```bash
python -m http.server 8765 --bind 127.0.0.1
ssh -R 80:127.0.0.1:8765 nokey@localhost.run
```
Локальный сервер + SSH-туннель из состава Windows → публичный адрес
вида `https://….lhr.life/quiz-school.html`. Ничего дополнительно не
устанавливать не нужно. Ссылка живёт, пока включён компьютер.

---

## Критерии готовности
- [x] 10 вопросов, в каждом ровно 5 вариантов ответа.
- [x] Данные — списки и словари (`QUIZ`, `LEVELS`).
- [x] При правильном ответе картинка меняется на надпись «Молодец, все правильно!».
- [x] При каждом обновлении страницы вопросы и варианты перемешиваются.
- [x] В начале можно выбрать сложность (лёгкая / средняя / сложная).
- [x] Табло снизу: полоска уменьшается с остатком времени.
- [x] Если время истекло — появляется кнопка «🔁 Попробовать снова».
- [x] Через 5 секунд (по уровню) вопрос меняется на следующий.
- [x] Счёт, прогресс-бар, экран результата, перезапуск.
- [x] Ошибок в консоли браузера нет.

## Возможные улучшения (по желанию)
1. Сохранять лучший результат в `localStorage`, чтобы он не терялся.
2. Кнопка «Пропустить вопрос» и подсказка-50:50.
3. Таблица рекордов по нескольким игрокам.
4. Отдельные вопросы по темам: история школы, предметы, правила.
5. Звуки: фанфара при правильном ответе, тиканье на последних секундах.
6. Разнести код по файлам: `index.html` + `style.css` + `script.js` + `data.js`.
