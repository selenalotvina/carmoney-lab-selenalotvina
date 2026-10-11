готов
1) Сервис предварительной оценки заявки на заём под ПТС: принимает данные заявки, считает LTV и возвращает approve / review / reject.
2) Makefile: help, up, down, ps, logs, install, test, lint, seed; docker-compose.yml: запуск backend через `php -S ...` и проверка базы healthcheck `mysqladmin ping`.
3) Решение считается в папке `backend/src/Domain/`, классом `DecisionEngine`.
модель: openai/gpt-5.6-luna
