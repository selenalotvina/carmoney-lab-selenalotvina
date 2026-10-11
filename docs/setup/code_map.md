# Разбор решения по заявке

## 1. Как считается approve / review / reject

Точка входа — `AssessmentService::assess($payload)` в `backend/src/Domain/AssessmentService.php`. Внутри в этом порядке вызываются:

1. `ApplicationValidator::validate($payload)` — проверяет заявку по `backend/config/rules.php` (VIN, год, пробег, стоимость, сумма, срок) и возвращает нормализованный массив. При ошибке бросает `ValidationException`.
2. `LtvCalculator::calculate($input['requested_amount'], $input['market_value'])` — возвращает LTV в процентах.
3. `DecisionEngine::decide($ltv)` — сравнивает LTV с порогами `rules.php → ltv.approve_max` (60.0) и `rules.php → ltv.review_max` (85.0). Возвращает `DecisionEngine::APPROVE` / `REVIEW` / `REJECT`.
4. `VehicleAge::inYears($input['year'])` — отдельно добавляет `vehicle_age` в результат.
5. В `AssessmentService` фиксируется `approved_limit`: при `approve` равен запрошенной сумме, иначе 0.

`DecisionEngine` берёт пороги через конструктор (`new DecisionEngine($rules['ltv'])` — конфиг `approve_max`/`review_max`). Сам справочник `ltv_by_age` определён в `rules.php`, но в текущем коде не используется — это отмечено в комментарии к `AssessmentService` (задача LOAN-12).

`VinValidator` вызывается из `ApplicationValidator::validate()`. `VehicleAge::inYears()` вызывается там же при проверке года, а также напрямую из `AssessmentService::assess()` для поля `vehicle_age`.

## 2. Где бы встало правило «пробег <= 400 000 км, иначе review»

Самое естественное место — `DecisionEngine::decide()` (после расчёта LTV и до возврата строки): если пробег > 400 000, метод сразу возвращает `self::REVIEW`. Альтернатива — расширить сигнатуру `decide(float $ltv, ?int $mileage = null)` или сделать отдельный шаг в `AssessmentService::assess()` перед `decisionEngine->decide(...)`.

Что для этого уже есть:

- Валидированный `$input['mileage']` (тип `int`) из `ApplicationValidator::validate()`.
- Константы `DecisionEngine::APPROVE` / `REVIEW` / `REJECT`.
- Параметр `LtvCalculator` уже возвращает LTV; конфигурация порогов лежит в `rules.php → ltv`.

Чего не хватает:

- В `backend/config/rules.php` нет ключа для порога пробега, ведущего к `review` (есть только `vehicle.max_mileage_km = 500000` — это верхняя граница для валидации). Значение 400 000 нужно добавить в конфиг, чтобы не хардкодить.
- В `DecisionEngine` сейчас нет доступа ни к пробегу, ни к правилам пробега: конструктор принимает только `approve_max` / `review_max`. Нужно расширить вход (например, передавать массив правил или отдельный объект с порогом).
- В `AssessmentService::assess()` пробег не передаётся в `decisionEngine`; нужно явно прокинуть `$input['mileage']`.

## 3. Что в коде уже сейчас проверяется про пробег

Только в `ApplicationValidator::validate()` (`backend/src/Domain/ApplicationValidator.php`, строки 43–46): пробег должен быть >= 0 и <= `rules.vehicle.max_mileage_km` (сейчас 500 000). При превышении заявка падает с `ValidationException` и до `LtvCalculator`/`DecisionEngine` не доходит. Сам `mileage` больше нигде в коде не используется: в `AssessmentService`, `LtvCalculator`, `DecisionEngine` пробег не фигурирует.
