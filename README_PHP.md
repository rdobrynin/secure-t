# Квантовый апокалипсис отменяется: гайд по переходу на Post-Quantum Cryptography (PQC)

Если ты пишешь бэкенд, фронтенд или мобилку — скорее всего, ты уже используешь шифрование. HTTPS, JWT, TLS — всё это работает на математике, которую квантовый компьютер сломает. Не сразу, но уже понятно когда.

Это не повод паниковать. Это повод разобраться — что именно сломается, что нет, и что нужно поменять в своём коде.

После этого материала ты будешь знать:

- Почему RSA и ECC обречены, а AES-256 — нет
- Что такое атака «собирай сейчас — читай потом» и затрагивает ли она твой сервис
- Какие новые алгоритмы пришли на замену и чем они отличаются друг от друга
- Почему переход на PQC может сломать соединения — и как этого избежать
- Как написать гибридную защиту, которая работает уже сегодня

## Оглавление

- [Что происходит и почему сейчас](#что-происходит-и-почему-сейчас)
- [История про Артёма — как выглядит атака](#история-про-артёма)
- [Почему одни алгоритмы сломаются, а другие нет](#почему-одни-алгоритмы-сломаются-а-другие-нет)
- [Новые алгоритмы: что выбрать и для чего](#новые-алгоритмы-что-выбрать-и-для-чего)
- [Подводные камни при внедрении](#подводные-камни-при-внедрении)
- [Как защититься уже сегодня: гибридный подход](#как-защититься-уже-сегодня-гибридный-подход)
- [Реальный кейс: как это сделал Signal](#реальный-кейс-как-это-сделал-signal)
- [Тест](#тест)
- [Задание: найди и исправь уязвимости в конфиге TLS](#задание-найди-и-исправь-уязвимости-в-конфиге-tls)

---

## Что происходит и почему сейчас

В августе 2024 года NIST (американский институт стандартов) опубликовал новые стандарты шифрования — специально разработанные против квантовых компьютеров. Apple уже защитила iMessage, Google обновил Chrome, Signal переписал протокол обмена ключами.

Квантового компьютера, который ломает шифрование, ещё нет. Но есть три причины начинать прямо сейчас:

**Причина 1: данные крадут уже сегодня.** Зашифрованный трафик можно перехватить и хранить — а расшифровать потом, когда появится нужный компьютер. Если твои данные должны быть секретными ещё лет через пять — они уже под угрозой.

**Причина 2: миграция занимает годы.** Обновить один сервис — легко. Обновить все сервисы, партнёрские интеграции, мобильные клиенты, IoT-устройства в поле — это большой проект.

**Причина 3: прогресс ускоряется.** В мае 2025 года вышла оптимизация алгоритма Шора, которая снизила требования к квантовому железу в 20 раз. Прогноз сдвинулся с «никогда» на начало 2030-х.

> **Коротко:** переход на PQC — это не «когда-нибудь». Это инфраструктурный проект, который нужно начинать сейчас.

---

## История про Артёма

Давай сначала посмотрим, как атака выглядит на практике. Это поможет понять, что именно мы защищаем.

### Знакомьтесь: Артём

Артём — аналитик данных в небольшой компании. Днём он строит графики в Excel и пьёт третий кофе, а по ночам… по ночам он работает над долгосрочным инвестиционным проектом.

Артём перехватывает корпоративный трафик крупного банка и аккуратно складывает его на жёсткий диск.

Он не спешит. У него есть план.

Коллеги называют его «самым терпеливым человеком в комнате». Артём только улыбается и запускает очередной дамп TLS-сессий в архив.

### Шаг 1. Сбор урожая

*Место действия: арендованный сервер. 2025 год.*

Артём настраивает перехват трафика через подконтрольный ему узел. Банк использует TLS 1.3 с RSA-2048 и ECDH на кривой P-256. Всё по учебнику. Всё надёжно.

— Сейчас — да, — бормочет Артём и запускает сниффер.

```
[CAPTURE] TLS session dump → archive_2025_01.enc  [4.2 GB]
[CAPTURE] TLS session dump → archive_2025_02.enc  [3.8 GB]
[CAPTURE] TLS session dump → archive_2025_03.enc  [5.1 GB]
...
```

Трафик зашифрован. Артём это знает. Его это не беспокоит.

> **Что происходит:** злоумышленник записывает TLS-сессии целиком — включая ClientHello и ServerHello с публичными ключами. Расшифровать сейчас невозможно, но сами байты никуда не деваются.

### Шаг 2. Холодное хранение

За месяц набирается ~60 ГБ: транзакции, внутренние API-вызовы, переписка систем.

```bash
> tar -czf archive_2025_01.enc.gz ./harvest/2025-01/*
> rclone copy archive_2025_01.enc.gz cold-storage:/bank_harvest/
> du -sh cold-storage:/bank_harvest/
61.4 GB    total
> echo "Стоимость хранения: 1.23 в месяц. Дешевле подписки на стриминг."
```

Артём закрывает ноутбук и идёт спать. Завтра — обычный рабочий день.

### Шаг 3. Ожидание: 2025 → 2031

Артём продолжает работать аналитиком. Читает новости о квантовых компьютерах. Следит за IBM и Google.

Однажды утром он видит заголовок:

> **«Google объявляет о первом CRQC: RSA-2048 взломан за 4 часа»**

Артём медленно допивает кофе и открывает терминал.

### Шаг 4. Час расплаты

*2031 год. Артём достаёт архивы шестилетней давности.*

```
> quantum_shor_client --algorithm ECDH-P256 --target archive_2025_01.enc
[INFO] Загружено 61400 TLS-сессий.
[INFO] Восстановление session key для сессии 2025-01-15 10:23:17...
[████████████████] 100%
[SUCCESS] Session key recovered: a3f9e4b7c812d5467f1a9b3c5d7e8f2a

> decrypt_tls_session --key a3f9e4b7c812d5467f1a9b3c5d7e8f2a --input harvest_20250115.enc
[SUCCESS] → decrypted_20250115.json

{
  "transaction_id": "TXN-20250115-009823",
  "from_account": "****4471",
  "to_account":   "****8832",
  "amount":        2850000,
  "currency":      "RUB",
  "auth_token":   "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

Шесть лет ожидания. Один скрипт. Полный архив транзакций.

— Терпение, — говорит он себе. — Терпение и труд всё перетрут.

### Что пошло не так?

Банк сделал три ошибки:

1. **Использовал ECDH P-256** — эллиптические кривые уязвимы для алгоритма Шора. Ключ короткий, кубитов нужно меньше — ECC ломается первой (об этом дальше).
2. **Не внедрил гибридный обмен ключами** — если бы рядом с ECDH работал ML-KEM (Kyber), Артёму понадобился бы и квантовый компьютер, и уязвимость в Kyber одновременно.
3. **Откладывал миграцию** — данные были украдены раньше, чем угроза стала реальной.

**Мог ли Артём потерпеть неудачу?** Да. Достаточно было одной строки в конфиге — гибридный обмен ключами:

```
MasterSecret = KDF(SharedSecret_ECDH || SharedSecret_Kyber)
```

Даже взломав ECDH квантовым компьютером, Артём получил бы только половину входных данных для финального ключа. Без Kyber-части — ничего.

---

## Теорема Моска: когда именно горит?

Исследователь Микеле Моска сформулировал простое правило риска:

![mosca_theorem_ru.png](mosca_theorem_ru.png)

Вот калькулятор для самопроверки:

```php
<?php

declare(strict_types=1);

/**
 * Проверяет, успеваешь ли ты с миграцией согласно теореме Моска
 * 
 * @param string $dataType Описание типа данных
 * @param int $shelfLifeYears X: сколько лет данные должны быть секретны
 * @param int $migrationEstimateYears Y: сколько займёт внедрение PQC
 * @return array<string, mixed> Результат проверки
 */
function checkQuantumRisk(string $dataType, int $shelfLifeYears, int $migrationEstimateYears): array {
    $YEAR_CRQC_ARRIVAL = 2032;  // консервативный прогноз
    $currentYear = (int) date('Y');
    $z = $YEAR_CRQC_ARRIVAL - $currentYear;  // сколько лет осталось
    $total = $shelfLifeYears + $migrationEstimateYears;
    
    $isAtRisk = $total > $z;
    $difference = abs($total - $z);
    
    $result = [
        'data_type' => $dataType,
        'current_year' => $currentYear,
        'crqc_year' => $YEAR_CRQC_ARRIVAL,
        'years_left' => $z,
        'shelf_life' => $shelfLifeYears,
        'migration_time' => $migrationEstimateYears,
        'total_required' => $total,
        'is_at_risk' => $isAtRisk,
        'difference' => $difference,
        'emoji' => $isAtRisk ? '🔴' : '🟢',
        'message' => $isAtRisk 
            ? "РИСК. Ты опаздываешь на {$difference} лет."
            : "Запас: {$difference} лет."
    ];
    
    // Вывод в консоль (для CLI режима)
    if (PHP_SAPI === 'cli') {
        echo "\n--- {$dataType} ---\n";
        echo $result['emoji'] . ' ' . $result['message'] . "\n";
    }
    
    return $result;
}

// Примеры использования
if (PHP_SAPI === 'cli') {
    echo "=== Калькулятор теоремы Моска (Miguel Mosca) ===\n";
    
    $results = [
        checkQuantumRisk("Сессионные токены (живут 24ч)", 0, 2),
        checkQuantumRisk("История транзакций (храним 7 лет)", 7, 3),
        checkQuantumRisk("Медицинские данные (бессрочно)", 20, 4),
    ];
    
    // Дополнительная статистика
    echo "\n=== Статистика ===\n";
    $riskyCount = count(array_filter($results, fn($r) => $r['is_at_risk']));
    echo "Требуют срочной миграции: {$riskyCount} из " . count($results) . "\n";
}
```

Запусти это с параметрами своего сервиса — и сразу поймёшь, насколько срочно.

---

## Почему одни алгоритмы сломаются, а другие нет

### RSA и ECC: почему они обречены

Вся асимметричная криптография держится на «задачах с потайным ходом» — математике, которую легко решить в одну сторону и почти невозможно в обратную.

- **RSA** — на факторизации: перемножить два простых числа легко, разложить обратно — нет.
- **ECC и ECDH** — на дискретном логарифмировании в группе точек эллиптической кривой.

Алгоритм Шора (1994) умеет находить период функции за полиномиальное время — и обе задачи становятся тривиальными для квантового компьютера.

**Неочевидный факт про ECC:** мы привыкли считать его «надёжнее» RSA — ключ ECC-256 по классической стойкости эквивалентен RSA-3072, но компактнее. В квантовом мире компактность оборачивается уязвимостью: короткий ключ требует меньше кубитов для взлома.

| Алгоритм | Логических кубитов для взлома |
|---|---|
| RSA-2048 | ~4098 |
| ECDSA-256 | ~2330 |

Эллиптические кривые, скорее всего, падут **раньше** RSA — раньше, чем у атакующих хватит мощности на «старый добрый» RSA. Это контринтуитивно, но важно.

![pic2.png](pic2.png)

### Симметрия держит удар

AES не имеет математической периодичности, на которой работает алгоритм Шора. Здесь применяется другой квантовый алгоритм — Гровера, который даёт квадратичное ускорение перебора:

- AES-128 → эффективные **64 бита** — слабовато
- AES-256 → эффективные **128 бит** — по-прежнему надёжно ✅

### Практическое задание: Калькулятор квантовой стойкости
Давайте напишем скрипт, который наглядно покажет, почему мы отказываемся от ECC, но оставляем AES.

```php
<?php

declare(strict_types=1);

/**
 * Калькулятор квантовой стойкости алгоритмов шифрования
 * Показывает, почему ECC/RSA уязвимы, а AES остаётся безопасным
 */
class QuantumSecurityCalculator {
    private const SAFE_THRESHOLD = 128;
    private const UNKNOWN_ALGORITHM = 'UNKNOWN';
    
    // Enum-like константы для типов алгоритмов
    public const ALG_RSA = 'RSA';
    public const ALG_ECC = 'ECC';
    public const ALG_SYMMETRIC = 'SYMMETRIC';
    
    // Маппинг алгоритмов для валидации
    private const VALID_ALGORITHMS = [
        self::ALG_RSA,
        self::ALG_ECC,
        self::ALG_SYMMETRIC
    ];
    
    // Маппинг названий для отображения
    private const ALGORITHM_NAMES = [
        self::ALG_RSA => 'RSA',
        self::ALG_ECC => 'ECC (Elliptic Curve)',
        self::ALG_SYMMETRIC => 'Symmetric (AES, ChaCha20)'
    ];
    
    /**
     * Рассчитывает эффективную стойкость алгоритма против квантового компьютера
     * 
     * @param string $algorithmType Тип алгоритма (RSA, ECC, SYMMETRIC)
     * @param int $keySizeBits Размер ключа в битах
     * @return array<string, mixed> Результат расчёта
     */
    public function calculate(string $algorithmType, int $keySizeBits): array {
        $algorithm = strtoupper($algorithmType);
        
        // Валидация алгоритма
        if (!in_array($algorithm, self::VALID_ALGORITHMS, true)) {
            return [
                'error' => true,
                'message' => "Неизвестный алгоритм: {$algorithmType}",
                'valid_algorithms' => self::VALID_ALGORITHMS
            ];
        }
        
        // Расчёт эффективных бит
        $effectiveBits = match($algorithm) {
            self::ALG_RSA, self::ALG_ECC => 0, // Алгоритм Шора = полный взлом
            self::ALG_SYMMETRIC => (int) floor($keySizeBits / 2), // Гровер делит пополам
            default => 0
        };
        
        // Определение вердикта
        $verdict = match(true) {
            $effectiveBits >= self::SAFE_THRESHOLD => '✅ БЕЗОПАСНО',
            $effectiveBits > 0 => '⚠️ ОСЛАБЛЕНО — обновись до более длинного ключа',
            default => '💀 ВЗЛОМАНО — срочно мигрируй на PQC'
        };
        
        // Определение цвета для UI
        $color = match(true) {
            $effectiveBits >= self::SAFE_THRESHOLD => 'green',
            $effectiveBits > 0 => 'orange',
            default => 'red'
        };
        
        // Квантовая атака
        $quantumAttack = match($algorithm) {
            self::ALG_RSA, self::ALG_ECC => 'Алгоритм Шора (полное разрушение)',
            self::ALG_SYMMETRIC => 'Алгоритм Гровера (квадратичное ускорение)',
            default => 'Неизвестно'
        };
        
        $result = [
            'algorithm' => $algorithm,
            'algorithm_name' => self::ALGORITHM_NAMES[$algorithm],
            'key_size' => $keySizeBits,
            'effective_bits' => $effectiveBits,
            'verdict' => $verdict,
            'color' => $color,
            'quantum_attack' => $quantumAttack,
            'is_broken' => $effectiveBits === 0,
            'is_weak' => $effectiveBits > 0 && $effectiveBits < self::SAFE_THRESHOLD,
            'is_safe' => $effectiveBits >= self::SAFE_THRESHOLD
        ];
        
        // Вывод в консоль для CLI режима
        if (PHP_SAPI === 'cli') {
            $this->printResult($result);
        }
        
        return $result;
    }
    
    /**
     * Вывод результата в консоль
     */
    private function printResult(array $result): void {
        printf(
            "%s-%d: %d эффективных бит → %s\n",
            $result['algorithm'],
            $result['key_size'],
            $result['effective_bits'],
            $result['verdict']
        );
    }
    
    /**
     * Пакетный расчёт для нескольких комбинаций
     * 
     * @param array $scenarios Массив сценариев [alg, key_size]
     * @return array Результаты
     */
    public function calculateBatch(array $scenarios): array {
        $results = [];
        foreach ($scenarios as $scenario) {
            $results[] = $this->calculate($scenario[0], $scenario[1]);
        }
        return $results;
    }
    
    /**
     * Получение рекомендаций на основе результатов
     */
    public function getRecommendations(array $results): array {
        $recommendations = [];
        
        foreach ($results as $result) {
            if ($result['is_broken']) {
                $recommendations[] = [
                    'algorithm' => $result['algorithm_name'],
                    'key_size' => $result['key_size'],
                    'priority' => 'CRITICAL',
                    'action' => 'НЕМЕДЛЕННО прекратить использование! Мигрировать на PQC',
                    'alternative' => 'ML-KEM (Kyber) для обмена ключами, SLH-DSA (Dilithium) для подписей'
                ];
            } elseif ($result['is_weak']) {
                $recommendations[] = [
                    'algorithm' => $result['algorithm_name'],
                    'key_size' => $result['key_size'],
                    'priority' => 'HIGH',
                    'action' => 'Увеличить размер ключа или мигрировать',
                    'alternative' => "AES-256 (рекомендуется) или " . ($result['key_size'] * 2) . "-битный ключ"
                ];
            } else {
                $recommendations[] = [
                    'algorithm' => $result['algorithm_name'],
                    'key_size' => $result['key_size'],
                    'priority' => 'LOW',
                    'action' => 'Можно использовать',
                    'alternative' => 'Текущий алгоритм безопасен'
                ];
            }
        }
        
        return $recommendations;
    }
    
    /**
     * Сравнение классической и квантовой стойкости
     */
    public function compareClassicalVsQuantum(string $algorithm, int $keySize): array {
        $classicalStrength = match($algorithm) {
            self::ALG_RSA => $this->estimateRSASecurity($keySize),
            self::ALG_ECC => (int) floor($keySize / 2), // Эквивалентная симметричная стойкость
            self::ALG_SYMMETRIC => $keySize,
            default => 0
        };
        
        $quantumResult = $this->calculate($algorithm, $keySize);
        
        return [
            'algorithm' => $algorithm,
            'key_size' => $keySize,
            'classical_bits' => $classicalStrength,
            'quantum_bits' => $quantumResult['effective_bits'],
            'reduction' => $classicalStrength - $quantumResult['effective_bits'],
            'reduction_percent' => $classicalStrength > 0 
                ? round((1 - $quantumResult['effective_bits'] / $classicalStrength) * 100, 1)
                : 100
        ];
    }
    
    /**
     * Оценка стойкости RSA в классических битах
     */
    private function estimateRSASecurity(int $keySize): int {
        // Приблизительная оценка эквивалентной симметричной стойкости
        return match(true) {
            $keySize >= 15360 => 256,
            $keySize >= 7680 => 192,
            $keySize >= 3072 => 128,
            $keySize >= 2048 => 112,
            $keySize >= 1024 => 80,
            default => (int) floor($keySize / 10)
        };
    }
}

// CLI режим
if (PHP_SAPI === 'cli') {
    $calculator = new QuantumSecurityCalculator();
    
    echo "=== Калькулятор квантовой стойкости ===\n";
    echo "Почему мы отказываемся от ECC/RSA, но оставляем AES?\n\n";
    
    // Тестовые сценарии
    $scenarios = [
        ['RSA', 2048],
        ['ECC', 256],
        ['SYMMETRIC', 128],
        ['SYMMETRIC', 256],
        ['RSA', 4096],
        ['ECC', 521],
        ['SYMMETRIC', 64],
        ['SYMMETRIC', 192],
    ];
    
    $results = $calculator->calculateBatch($scenarios);
    
    echo "\n📊 Сравнение классической и квантовой стойкости:\n";
    echo str_repeat('-', 80) . "\n";
    printf("%-15s %-10s %-15s %-15s %-10s\n", 
           "Алгоритм", "Ключ", "Классич. биты", "Квант. биты", "Потеря");
    echo str_repeat('-', 80) . "\n";
    
    foreach ($scenarios as $scenario) {
        $comparison = $calculator->compareClassicalVsQuantum($scenario[0], $scenario[1]);
        printf(
            "%-15s %-10d %-15d %-15d %-10.1f%%\n",
            $scenario[0],
            $scenario[1],
            $comparison['classical_bits'],
            $comparison['quantum_bits'],
            $comparison['reduction_percent']
        );
    }
    
    echo "\n📋 Рекомендации:\n";
    $recommendations = $calculator->getRecommendations($results);
    foreach ($recommendations as $rec) {
        echo "[{$rec['priority']}] {$rec['algorithm']}-{$rec['key_size']}: {$rec['action']}\n";
        echo "    ➜ Альтернатива: {$rec['alternative']}\n";
    }
    
    echo "\n🔬 Квантовые алгоритмы атаки:\n";
    echo "• RSA/ECC: Алгоритм Шора (1994) — экспоненциальное ускорение\n";
    echo "• Симметричные: Алгоритм Гровера (1996) — квадратичное ускорение\n";
    echo "• Вывод: RSA/ECC = 💀, AES-256 = ✅, AES-128 = ⚠️\n";
}
```

---

## Новые алгоритмы: что выбрать и для чего

### ML-KEM (бывший Kyber) — FIPS 203

**Для чего:** замена ECDH при обмене ключами в TLS, API, защищённых каналах.

**Основа:** модульные решётки — математические задачи, которые квантовый компьютер не умеет решать быстро.

**Главное отличие от ECDH:** в ECDH обе стороны независимо вычисляют один и тот же секрет. В KEM (Key Encapsulation Mechanism) работает иначе — как конверт с посылкой:

```
<?php

declare(strict_types=1);

/**
 * Демонстрация различий между классическим ECDH и пост-квантовым ML-KEM (KEM)
 * 
 * ECDH: обе стороны независимо вычисляют общий секрет
 * ML-KEM: одна сторона запечатывает секрет, другая распечатывает (как конверт)
 */
class KEMvsECDHDemo {
    
    // Имитация криптографических операций (для демонстрации концепции)
    private const KEY_LENGTH = 32; // 256 бит
    
    /**
     * Демонстрация классического ECDH (Elliptic Curve Diffie-Hellman)
     */
    public function demonstrateECDH(): array {
        $steps = [];
        $steps[] = "=== Классический ECDH ===";
        $steps[] = "Обе стороны обмениваются публичными ключами и независимо считают общий секрет\n";
        
        // Генерация ключевых пар
        $steps[] = "1. Клиент генерирует ключевую пару:";
        $clientKeyPair = $this->generateECKeyPair();
        $steps[] = "   Публичный ключ клиента: " . $this->truncate($clientKeyPair['public']);
        
        $steps[] = "\n2. Сервер генерирует ключевую пару:";
        $serverKeyPair = $this->generateECKeyPair();
        $steps[] = "   Публичный ключ сервера: " . $this->truncate($serverKeyPair['public']);
        
        $steps[] = "\n3. Стороны обмениваются публичными ключами";
        
        // Вычисление общих секретов
        $clientShared = $this->computeECDHSecret(
            $clientKeyPair['private'], 
            $serverKeyPair['public']
        );
        
        $serverShared = $this->computeECDHSecret(
            $serverKeyPair['private'], 
            $clientKeyPair['public']
        );
        
        $steps[] = "\n4. Результат:";
        $steps[] = "   Секрет клиента: " . $this->truncate($clientShared);
        $steps[] = "   Секрет сервера: " . $this->truncate($serverShared);
        
        $match = hash_equals($clientShared, $serverShared);
        $steps[] = "   Совпадают? " . ($match ? "✓ ДА" : "✗ НЕТ");
        
        return [
            'protocol' => 'ECDH',
            'steps' => $steps,
            'client_secret' => bin2hex($clientShared),
            'server_secret' => bin2hex($serverShared),
            'match' => $match
        ];
    }
    
    /**
     * Демонстрация ML-KEM (новая парадигма KEM)
     */
    public function demonstrateMLKEM(): array {
        $steps = [];
        $steps[] = "=== ML-KEM (Key Encapsulation Mechanism) ===";
        $steps[] = "Клиент создаёт конверт (публичный ключ), сервер запечатывает секрет\n";
        
        // Клиент генерирует ключевую пару
        $steps[] = "1. Клиент генерирует ключевую пару ML-KEM:";
        $clientKeyPair = $this->generateMLKEMKeyPair();
        $clientPub = $clientKeyPair['public'];
        $clientPriv = $clientKeyPair['private'];
        
        $steps[] = "   Публичный ключ клиента (конверт): " . $this->truncate($clientPub);
        $steps[] = "   Приватный ключ клиента хранится в секрете";
        
        $steps[] = "\n2. Клиент отправляет публичный ключ серверу";
        
        // Сервер запечатывает секрет
        $steps[] = "\n3. Сервер запечатывает (encaps) секрет в публичный ключ клиента:";
        $serverResult = $this->mlKEMEncaps($clientPub);
        $ciphertext = $serverResult['ciphertext'];
        $serverShared = $serverResult['shared_secret'];
        
        $steps[] = "   Сгенерированный шифротекст: " . $this->truncate($ciphertext);
        $steps[] = "   Общий секрет сервера: " . $this->truncate($serverShared);
        
        $steps[] = "\n4. Сервер отправляет шифротекст клиенту";
        
        // Клиент распечатывает секрет
        $steps[] = "\n5. Клиент распечатывает (decaps) шифротекст своим приватным ключом:";
        $clientShared = $this->mlKEMDecaps($ciphertext, $clientPriv);
        $steps[] = "   Общий секрет клиента: " . $this->truncate($clientShared);
        
        $steps[] = "\n6. Результат:";
        $steps[] = "   Секрет сервера: " . $this->truncate($serverShared);
        $steps[] = "   Секрет клиента: " . $this->truncate($clientShared);
        
        $match = hash_equals($serverShared, $clientShared);
        $steps[] = "   Совпадают? " . ($match ? "✓ ДА" : "✗ НЕТ");
        
        return [
            'protocol' => 'ML-KEM',
            'steps' => $steps,
            'client_secret' => bin2hex($clientShared),
            'server_secret' => bin2hex($serverShared),
            'ciphertext' => bin2hex($ciphertext),
            'match' => $match
        ];
    }
    
    /**
     * Сравнение двух подходов
     */
    public function compareProtocols(): array {
        return [
            'comparison' => [
                'feature' => [
                    'Роль сторон',
                    'Обмен сообщениями',
                    'Кто создаёт секрет',
                    'Квантовая стойкость',
                    'Количество передач',
                    'Пост-квантовая готовность'
                ],
                'ECDH' => [
                    'Равноправные',
                    '2 сообщения (pub-key → pub-key)',
                    'Обе стороны независимо',
                    '❌ Уязвим (алгоритм Шора)',
                    '2',
                    '❌ Нет'
                ],
                'ML-KEM' => [
                    'Клиент (получатель), Сервер (отправитель)',
                    '2 сообщения (pub-key → ciphertext)',
                    'Сервер создаёт, клиент извлекает',
                    '✅ Устойчив (пост-квантовый)',
                    '2',
                    '✅ Да (NIST стандарт)'
                ]
            ]
        ];
    }
    
    /**
     * Имитация генерации EC ключей
     */
    private function generateECKeyPair(): array {
        return [
            'public' => random_bytes(self::KEY_LENGTH),
            'private' => random_bytes(self::KEY_LENGTH)
        ];
    }
    
    /**
     * Имитация вычисления ECDH секрета
     */
    private function computeECDHSecret(string $privateKey, string $publicKey): string {
        // В реальности: ECDH(private, public) -> общий секрет
        // Для демо: комбинируем ключи через XOR
        $secret = '';
        for ($i = 0; $i < self::KEY_LENGTH; $i++) {
            $secret .= chr(ord($privateKey[$i]) ^ ord($publicKey[$i % strlen($publicKey)]));
        }
        return hash('sha256', $secret, true);
    }
    
    /**
     * Имитация генерации ML-KEM ключей
     */
    private function generateMLKEMKeyPair(): array {
        return [
            'public' => random_bytes(self::KEY_LENGTH * 2), // ML-KEM ключи больше
            'private' => random_bytes(self::KEY_LENGTH * 3)
        ];
    }
    
    /**
     * Имитация ML-KEM Encapsulation
     */
    private function mlKEMEncaps(string $publicKey): array {
        // Генерируем случайный общий секрет
        $sharedSecret = random_bytes(self::KEY_LENGTH);
        
        // Создаём шифротекст (имитация)
        $ciphertext = hash('sha256', $publicKey . $sharedSecret . random_bytes(16), true);
        
        return [
            'ciphertext' => $ciphertext,
            'shared_secret' => $sharedSecret
        ];
    }
    
    /**
     * Имитация ML-KEM Decapsulation
     */
    private function mlKEMDecaps(string $ciphertext, string $privateKey): string {
        // В реальности: расшифровываем ciphertext с помощью privateKey
        // Для демо: возвращаем детерминированный результат на основе ключей
        return hash('sha256', $ciphertext . $privateKey, true);
    }
    
    /**
     * Обрезает строку для отображения
     */
    private function truncate(string $data): string {
        $hex = bin2hex($data);
        if (strlen($hex) <= 16) return $hex;
        return substr($hex, 0, 8) . '...' . substr($hex, -8);
    }
    
    /**
     * Форматирует вывод для CLI
     */
    public function printToConsole(): void {
        $ecdh = $this->demonstrateECDH();
        $mlkem = $this->demonstrateMLKEM();
        $comparison = $this->compareProtocols();
        
        echo implode("\n", $ecdh['steps']) . "\n\n";
        echo implode("\n", $mlkem['steps']) . "\n\n";
        
        echo "=== Сравнение ECDH и ML-KEM ===\n";
        echo str_repeat('-', 80) . "\n";
        printf("%-25s %-25s %-25s\n", "Характеристика", "ECDH", "ML-KEM");
        echo str_repeat('-', 80) . "\n";
        
        foreach ($comparison['comparison']['feature'] as $i => $feature) {
            printf(
                "%-25s %-25s %-25s\n",
                $feature,
                $comparison['comparison']['ECDH'][$i],
                $comparison['comparison']['ML-KEM'][$i]
            );
        }
        echo str_repeat('-', 80) . "\n";
        
        echo "\n🔑 Главное отличие: в ECDH обе стороны вычисляют секрет (взаимный обмен),\n";
        echo "   в KEM одна сторона запечатывает секрет, другая распечатывает (как конверт с посылкой)\n";
    }
}

// CLI запуск
if (PHP_SAPI === 'cli') {
    $demo = new KEMvsECDHDemo();
    $demo->printToConsole();
}
```

Kyber на современном процессоре работает **быстрее** X25519. Платишь не временем CPU, а размером данных — ключ 1184 байта вместо 32.

### ML-DSA (бывший Dilithium) — FIPS 204

**Для чего:** цифровые подписи — сертификаты, JWT, подпись API-ответов, авторизация.

**Когда выбирать:** если нужна быстрая проверка подписи. Браузер проверяет цепочки сертификатов за миллисекунды — ML-DSA справляется. Минус — подпись весит ~3300 байт вместо 64 у ECDSA.

### SLH-DSA (бывший SPHINCS+) — FIPS 205

**Для чего:** подпись прошивок, корневые сертификаты, долгоживущие ключи.

**Когда выбирать:** когда надёжность важнее скорости и размера, а проверка подписи редкая. SLH-DSA основан на хеш-функциях, а не на решётках — если в теории решёток найдут уязвимость, он выстоит. Подпись весит 8–40 КБ.

### HQC — резервный KEM

Основан на теории кодирования, математически независим от решёток. Медленнее Kyber и тяжелее, но это запасной парашют: если в Kyber найдут дыру — HQC не затронет.

### Итоговая таблица: что куда

| Задача | Старый алгоритм | Новый алгоритм |
|---|---|---|
| Обмен ключами в TLS/API | ECDH (X25519) | ML-KEM (Kyber) |
| Подпись JWT, сертификаты | ECDSA, RSA | ML-DSA (Dilithium) |
| Подпись прошивок, root CA | RSA-4096 | SLH-DSA (SPHINCS+) |
| Резерв на случай взлома решёток | — | HQC |

---

## Подводные камни при внедрении

### Проблема 1: новые ключи не влезают в пакеты

Ключ X25519 — 32 байта. Ключ ML-KEM-768 — 1184 байта. Разница в 37 раз.

Стандартный MTU большинства сетей — **1500 байт**. Классический `ClientHello` в TLS занимает 300–500 байт. После добавления Kyber он вырастает примерно до 1600–1800 байт и перестаёт влезать в один пакет.

Что происходит:

1. TCP разбивает `ClientHello` на несколько пакетов — сервер ждёт все.
2. На нестабильных мобильных сетях это добавляет 20–40% ко времени установки соединения.
3. **Главная ловушка — Middlebox Ossification:** старые корпоративные файрволы и DPI-системы не распознают фрагментированный `ClientHello` как легитимный TLS и просто дропают пакет. По данным Cloudflare и Google, ~1–2% соединений в интернете ломаются именно из-за этого.

```python
<?php

declare(strict_types=1);

/**
 * Класс для симуляции сетевого пути с учётом MTU и фрагментации
 */
class NetworkPath {
    private int $mtu;
    private bool $dropFragmented; // true = старый корпоративный файрвол
    private array $statistics = [];
    
    /**
     * @param int $mtu Maximum Transmission Unit
     * @param bool $dropFragmented Дропать ли фрагментированные пакеты
     */
    public function __construct(int $mtu = 1500, bool $dropFragmented = false) {
        $this->mtu = $mtu;
        $this->dropFragmented = $dropFragmented;
    }
    
    /**
     * Трансмиссия данных через сетевой путь
     * 
     * @param string $dataBytes Данные для отправки (в бинарном формате)
     * @return bool true если пакет успешно доставлен, false если дропнут
     */
    public function transmit(string $dataBytes): bool {
        $packetSize = strlen($dataBytes);
        $effectiveMtu = $this->mtu - 40; // вычитаем IP(20) + TCP(20) заголовки
        
        $this->statistics = [
            'packet_size' => $packetSize,
            'mtu' => $this->mtu,
            'effective_mtu' => $effectiveMtu,
            'timestamp' => time()
        ];
        
        if ($packetSize <= $effectiveMtu) {
            echo sprintf("✅ Пакет %dб прошёл целиком.\n", $packetSize);
            $this->statistics['status'] = 'success';
            $this->statistics['fragmented'] = false;
            return true;
        }
        
        $fragments = (int) ceil($packetSize / $effectiveMtu);
        echo sprintf("⚠️  Пакет %dб > MTU. Разбито на %d фрагмента(ов).\n", 
                    $packetSize, $fragments);
        
        $this->statistics['fragmented'] = true;
        $this->statistics['fragments'] = $fragments;
        
        if ($this->dropFragmented) {
            // Вот здесь у пользователя зависает загрузка страницы
            echo "🚫 Старый файрвол дропнул соединение (Middlebox Ossification).\n";
            $this->statistics['status'] = 'dropped';
            $this->statistics['reason'] = 'middlebox_ossification';
            return false;
        }
        
        echo "✅ Фрагменты доставлены (но latency вырос).\n";
        $this->statistics['status'] = 'delivered_with_fragmentation';
        return true;
    }
    
    /**
     * Получить статистику последней передачи
     */
    public function getStatistics(): array {
        return $this->statistics;
    }
    
    /**
     * Получить информацию о конфигурации
     */
    public function getConfig(): array {
        return [
            'mtu' => $this->mtu,
            'drop_fragmented' => $this->dropFragmented,
            'type' => $this->dropFragmented ? 'Legacy Firewall' : 'Modern Router'
        ];
    }
}

/**
 * Класс для создания тестовых TLS пакетов
 */
class TLSClientHello {
    private string $data;
    private string $type;
    
    /**
     * @param int $size Размер пакета в байтах
     * @param string $type Тип пакета (classic, pqc)
     */
    public function __construct(int $size, string $type = 'unknown') {
        $this->data = str_repeat("\x00", $size);
        $this->type = $type;
    }
    
    /**
     * Получить бинарные данные пакета
     */
    public function getData(): string {
        return $this->data;
    }
    
    /**
     * Получить размер пакета
     */
    public function getSize(): int {
        return strlen($this->data);
    }
    
    /**
     * Получить тип пакета
     */
    public function getType(): string {
        return $this->type;
    }
    
    /**
     * Создать классический ClientHello (~400 байт)
     */
    public static function createClassic(): self {
        return new self(400, 'TLS 1.3 Classic');
    }
    
    /**
     * Создать PQC ClientHello с Kyber-768 (~1700 байт)
     */
    public static function createPQC(): self {
        return new self(1700, 'TLS 1.3 + Kyber-768 (PQC)');
    }
    
    /**
     * Создать超大 PQC ClientHello с несколькими алгоритмами (~3000+ байт)
     */
    public static function createLargePQC(): self {
        return new self(3200, 'TLS 1.3 + Multiple PQC');
    }
}

/**
 * Класс для сбора статистики и анализа проблемы
 */
class MiddleboxAnalyzer {
    private array $results = [];
    private array $statistics = [
        'total_tests' => 0,
        'successful' => 0,
        'dropped' => 0,
        'fragmented' => 0
    ];
    
    /**
     * Запустить тест
     */
    public function runTest(NetworkPath $path, TLSClientHello $packet): bool {
        $this->statistics['total_tests']++;
        
        echo sprintf("\n📦 Тестирование: %s (%d байт) на %s\n",
            $packet->getType(),
            $packet->getSize(),
            $path->getConfig()['type']
        );
        
        $result = $path->transmit($packet->getData());
        
        $this->results[] = [
            'packet' => $packet->getType(),
            'packet_size' => $packet->getSize(),
            'network' => $path->getConfig(),
            'result' => $result,
            'statistics' => $path->getStatistics()
        ];
        
        if ($result) {
            $this->statistics['successful']++;
        } else {
            $this->statistics['dropped']++;
        }
        
        if ($path->getStatistics()['fragmented'] ?? false) {
            $this->statistics['fragmented']++;
        }
        
        return $result;
    }
    
    /**
     * Запустить пакетное тестирование
     */
    public function runBatchTests(): void {
        $scenarios = [
            'modern' => new NetworkPath(1500, false),
            'legacy' => new NetworkPath(1500, true),
            'small_mtu' => new NetworkPath(1280, false), // MTU как в некоторых VPN
            'corporate' => new NetworkPath(1500, true)
        ];
        
        $packets = [
            TLSClientHello::createClassic(),
            TLSClientHello::createPQC(),
            TLSClientHello::createLargePQC()
        ];
        
        foreach ($scenarios as $name => $path) {
            echo sprintf("\n🔍 Сценарий: %s\n", $name);
            echo str_repeat('-', 50) . "\n";
            
            foreach ($packets as $packet) {
                $this->runTest($path, $packet);
            }
        }
    }
    
    /**
     * Получить статистику тестирования
     */
    public function getStatistics(): array {
        return $this->statistics;
    }
    
    /**
     * Вывести отчёт
     */
    public function printReport(): void {
        echo "\n📊 СТАТИСТИКА ТЕСТИРОВАНИЯ\n";
        echo str_repeat('=', 50) . "\n";
        echo sprintf("Всего тестов: %d\n", $this->statistics['total_tests']);
        echo sprintf("✅ Успешно: %d (%.1f%%)\n", 
            $this->statistics['successful'],
            $this->statistics['total_tests'] > 0 
                ? ($this->statistics['successful'] / $this->statistics['total_tests']) * 100 
                : 0
        );
        echo sprintf("❌ Дропнуто: %d (%.1f%%)\n",
            $this->statistics['dropped'],
            $this->statistics['total_tests'] > 0 
                ? ($this->statistics['dropped'] / $this->statistics['total_tests']) * 100 
                : 0
        );
        echo sprintf("⚠️  Фрагментировано: %d\n", $this->statistics['fragmented']);
    }
}

/**
 * Демонстрация влияния PQC на размер TLS handshake
 */
function demonstratePQCSizeImpact(): void {
    $algorithms = [
        'RSA-2048' => 256,
        'ECDSA-256' => 64,
        'Kyber-768' => 1184,
        'Dilithium' => 1952,
        'Classic TLS 1.3' => 400,
        'TLS + Kyber-768' => 1700,
        'TLS + Full PQC' => 3200
    ];
    
    echo "\n📏 РАЗМЕРЫ КЛЮЧЕЙ И TLS ПАКЕТОВ\n";
    echo str_repeat('-', 60) . "\n";
    echo sprintf("%-25s %10s %10s\n", "Алгоритм/Пакет", "Размер", "Статус");
    echo str_repeat('-', 60) . "\n";
    
    foreach ($algorithms as $name => $size) {
        $status = match(true) {
            $size <= 1460 => "✅ в MTU",
            $size <= 3000 => "⚠️ фрагментация",
            default => "❌ множественная фрагментация"
        };
        
        echo sprintf("%-25s %10d б %10s\n", $name, $size, $status);
    }
}

/**
 * Симуляция статистики Cloudflare/Google
 */
function simulateCloudflareStats(): void {
    echo "\n📈 СТАТИСТИКА CLOUDFLARE/GOOGLE\n";
    echo str_repeat('-', 50) . "\n";
    
    $totalConnections = 1000000;
    $failureRate = 0.015; // 1.5%
    
    $failed = (int) ($totalConnections * $failureRate);
    $successful = $totalConnections - $failed;
    
    echo sprintf("Всего соединений: %d\n", $totalConnections);
    echo sprintf("✅ Успешных: %d (%.2f%%)\n", 
        $successful, ($successful / $totalConnections) * 100);
    echo sprintf("❌ Сломано из-за Middlebox: %d (%.2f%%)\n", 
        $failed, ($failed / $totalConnections) * 100);
    echo sprintf("\n⚠️  Из них из-за PQC размера: ~%d (примерно)\n", 
        (int) ($failed * 0.7));
}

// Основная демонстрация
echo "🔒 СИМУЛЯТОР TLS ФРАГМЕНТАЦИИ И MIDDLEBOX OSSIFICATION\n";
echo "================================================\n\n";

echo "Что происходит при TLS Handshake:\n";
echo "1. TCP разбивает ClientHello на несколько пакетов — сервер ждёт все\n";
echo "2. На нестабильных мобильных сетях это добавляет 20–40% ко времени\n";
echo "3. Middlebox Ossification: старые файрволы дропают фрагменты\n\n";

// Создаём тестовые пакеты
$classicHello = TLSClientHello::createClassic();
$pqcHello = TLSClientHello::createPQC();
$analyzer = new MiddleboxAnalyzer();

// Тест 1: Современный роутер
echo "=== СОВРЕМЕННЫЙ РОУТЕР (MTU=1500, без дропа) ===\n";
$modernRouter = new NetworkPath(1500, false);
$analyzer->runTest($modernRouter, $pqcHello);

// Тест 2: Корпоративный legacy firewall
echo "\n=== КОРПОРАТИВНЫЙ LEGACY FIREWALL (MTU=1500, с дропом) ===\n";
$legacyFirewall = new NetworkPath(1500, true);
$analyzer->runTest($legacyFirewall, $pqcHello);

// Дополнительные тесты
echo "\n=== ТЕСТИРОВАНИЕ РАЗНЫХ ТИПОВ ПАКЕТОВ ===\n";
$analyzer->runTest($modernRouter, $classicHello);
$analyzer->runTest($legacyFirewall, $classicHello);

// Демонстрация влияния размера
demonstratePQCSizeImpact();

// Статистика Cloudflare
simulateCloudflareStats();

// Итоговый отчёт
$analyzer->printReport();

// Анализ и рекомендации
echo "\n💡 АНАЛИЗ И РЕКОМЕНДАЦИИ\n";
echo str_repeat('-', 50) . "\n";
echo "• PQC увеличивает размер ClientHello в 4-5 раз\n";
echo "• 1-2% пользователей не могут соединиться из-за старых файрволов\n";
echo "• Решения:\n";
echo "  - Использовать EDNS(0) для больших пакетов\n";
echo "  - Применять TCP без фрагментации\n";
echo "  - Мигрировать на QUIC (решает проблему фрагментации)\n";
echo "  - Настроить Path MTU Discovery\n";

// Интерактивный режим (если запущено из CLI)
if (PHP_SAPI === 'cli' && isset($argv[1]) && $argv[1] === '--interactive') {
    runInteractiveMode();
}

/**
 * Интерактивный режим для тестирования
 */
function runInteractiveMode(): void {
    echo "\n🎮 ИНТЕРАКТИВНЫЙ РЕЖИМ\n";
    echo "Введите параметры для тестирования:\n";
    
    $handle = fopen("php://stdin", "r");
    
    echo "MTU (по умолчанию 1500): ";
    $mtu = (int) trim(fgets($handle));
    $mtu = $mtu ?: 1500;
    
    echo "Размер пакета в байтах (400/1700/3200): ";
    $size = (int) trim(fgets($handle));
    $size = $size ?: 1700;
    
    echo "Дропать фрагменты? (yes/no): ";
    $drop = trim(fgets($handle)) === 'yes';
    
    $path = new NetworkPath($mtu, $drop);
    $packet = new TLSClientHello($size, "Custom Packet");
    
    echo "\n📤 Результат:\n";
    $result = $path->transmit($packet->getData());
    
    if (!$result && $drop && $size > ($mtu - 40)) {
        echo "\n🔍 ДИАГНОЗ: Middlebox Ossification!\n";
        echo "    Старый файрвол не распознал фрагментированный ClientHello\n";
    }
    
    fclose($handle);
}
```

### Проблема 2: CPU быстро, сеть медленно

Kyber на современном процессоре работает быстрее X25519 — матричные операции и хеширование CPU любит больше, чем математику эллиптических кривых.

Но ты выигрываешь микросекунды на CPU и теряешь миллисекунды на передаче лишних килобайтов. Для сервера в дата-центре разница незаметна. Для IoT-устройства на слабом канале — это батарея, которая кончается раньше.

---

## Как защититься уже сегодня: гибридный подход

### Зачем гибрид, а не просто «переключиться на Kyber»?

Решётки изучаются 20–30 лет, тогда как факторизация — столетиями. Kyber может содержать математическую ошибку, которую ещё не нашли. Если переключиться только на него — а потом найдут баг — потеряем всё.

Решение: использовать оба алгоритма одновременно, «ремень и подтяжки»:

```
MasterSecret = KDF(SharedSecret_ECDH || SharedSecret_Kyber)
```

Чтобы взломать это соединение, атакующему нужно одновременно:
- Иметь квантовый компьютер (чтобы взломать ECDH), **И**
- Знать математическую уязвимость в Kyber.

Это нетривиальное требование. Apple, Google и Signal выбрали именно этот подход.

### Схема TLS-рукопожатия с гибридным обменом

![hybrid_tls_diagram.png](hybrid_tls_diagram.png)

Клиент генерирует два keypair — классический X25519 и постквантовый ML-KEM — и отправляет оба публичных ключа в `ClientHello`. Сервер вычисляет классический ECDH-секрет и инкапсулирует PQC-секрет. Финальный мастер-ключ получается из обоих через KDF.

### Матрица безопасности гибридного подхода

![Криптография_таблица_3.png](Криптография_таблица_3.png)

### Практическое задание: Реализация гибридного KDF

Давайте смоделируем логику гибридного вывода ключей.

```
<?php

/**
 * Упрощённая демонстрация гибридного вывода ключей
 */
class SimpleHybridKeyDemo {
    
    public static function main(): void {
        echo "🔐 Гибридный вывод ключей: смешиваем классический и PQC секреты\n\n";
        
        // Генерируем секреты
        $ecdh = random_bytes(32);  // от X25519
        $kyber = random_bytes(32); // от ML-KEM
        
        echo "ECDH секрет:  " . bin2hex(substr($ecdh, 0, 8)) . "...\n";
        echo "Kyber секрет: " . bin2hex(substr($kyber, 0, 8)) . "...\n";
        
        // Смешиваем
        $combined = $ecdh . $kyber . "tls13_hybrid";
        $hybridKey = hash_hmac('sha256', $combined, str_repeat("\x00", 32), true);
        
        echo "\nГибридный ключ: " . bin2hex(substr($hybridKey, 0, 16)) . "...\n\n";
        
        // Демонстрация защиты
        echo "Сценарии атак:\n";
        
        // Атакующий знает только ECDH
        $attack1 = hash_hmac('sha256', $ecdh . random_bytes(32) . "tls13_hybrid", str_repeat("\x00", 32), true);
        echo "Атака на ECDH: " . (hash_equals($attack1, $hybridKey) ? '❌ Успех' : '✅ Провал') . "\n";
        
        // Атакующий знает только Kyber
        $attack2 = hash_hmac('sha256', random_bytes(32) . $kyber . "tls13_hybrid", str_repeat("\x00", 32), true);
        echo "Атака на Kyber: " . (hash_equals($attack2, $hybridKey) ? '❌ Успех' : '✅ Провал') . "\n";
        
        echo "\n✅ Вывод: гибридная схема безопасна даже при компрометации одного алгоритма\n";
    }
}

SimpleHybridKeyDemo::main();
```

**Вывод:** Гибридная схема — это страховка. Внедряя PQC сегодня, мы не отказываемся от проверенной классики, а наслаиваем новую защиту поверх неё. Это единственный способ безопасно пережить переходный период.

---

## Реальный кейс: как это сделал Signal

Signal в 2023–2024 гг. первым среди массовых мессенджеров внедрил постквантовую защиту — протокол **PQXDH** (Post-Quantum Extended Diffie-Hellman).

Трудность была в том, что Signal — **асинхронный** мессенджер. В TLS оба участника онлайн, и рукопожатие происходит в реальном времени. В Signal Боб может три дня не открывать приложение — а Алиса хочет написать ему прямо сейчас. Поэтому ключи Боба лежат на сервере заранее (Pre-Keys).

**Что изменилось:** к обычному набору Pre-Keys (Curve25519) Signal добавил постквантовый ключ Kyber-1024. Когда Алиса пишет Бобу:

1. Скачивает его классический ключ (Curve25519) и PQC-ключ (Kyber-1024).
2. Генерирует оба секрета — через ECDH и через KEM.
3. Смешивает их в финальный ключ шифрования.

Дополнительно Signal внедрил **Triple Ratchet** — к классическому «двойному храповику» добавился слой SPQR (Sparse Post-Quantum Ratchet). Kyber-ключи большие, поэтому их не обновляют при каждом сообщении — вместо этого «размазывают» по нескольким сообщениям через Erasure Codes.

Результат — **Post-Compromise Security**: если телефон взломали и украли ключи, то через несколько сообщений PQC Ratchet сработает, канал обновится, и атакующий снова «ослепнет» — без квантового компьютера новые ключи не вскрыть.

![pic9.png](pic9.png)

---

## Тест

**Вопрос 1.**
Нужно ли срочно менять AES-256 на что-то принципиально новое для защиты от квантовых компьютеров?

- А) Да, AES полностью взломан алгоритмом Шора.
- Б) Нет, AES-256 достаточно — алгоритм Гровера снижает стойкость вдвое (до 128 бит), что остаётся безопасным.
- В) Да, нужно переходить на AES-512.

**Правильный ответ: Б.**
AES-256 выживает. Гровер даёт квадратичное ускорение: 256 бит → 128 эффективных. Просто убедись, что не используешь AES-128 — его нужно обновить.

---

**Вопрос 2.**
Что произойдёт с TLS-рукопожатием, если размер `ClientHello` из-за PQC-ключей превысит MTU (1500 байт)?

- А) Ничего страшного, пакет просто фрагментируется на уровне TCP.
- Б) Соединение мгновенно разорвётся с ошибкой шифрования.
- В) Пакет фрагментируется, но старое сетевое оборудование (Middleboxes) может его дропнуть — соединение зависнет.

**Правильный ответ: В.**
Middlebox Ossification — реальная проблема. ~1–2% соединений в интернете ломаются именно из-за этого. Немного — пока не умножить на всю аудиторию.

---

**Вопрос 3.**
В чём главный смысл гибридного обмена ключами (X25519 + Kyber)?

- А) Использовать квантовый компьютер для генерации ключей.
- Б) Объединить классический и постквантовый алгоритм — данные защищены, даже если один из них окажется уязвимым.
- В) Использовать два разных постквантовых алгоритма для ускорения.

**Правильный ответ: Б.**
«Ремень и подтяжки»: если Kyber взломают — спасёт ECDH. Если ECDH взломает квантовый компьютер — спасёт Kyber.

---

**Вопрос 4.**
Ты пишешь систему обновления прошивки для промышленных устройств (срок службы — 15 лет). Проверка подписи происходит редко — при каждом обновлении раз в полгода. Что выбрать?

- А) ML-DSA (Dilithium)
- Б) SLH-DSA (SPHINCS+)
- В) RSA-4096

**Правильный ответ: Б.**
SLH-DSA основан на хеш-функциях, а не решётках — максимально консервативный выбор для долгосрочного применения. Подпись большая, но раз в полгода это не проблема.

---

**Вопрос 5.**
Почему угроза HNDL актуальна сегодня, если квантового компьютера ещё нет?

- А) Потому что хакеры уже используют квантовые эмуляторы.
- Б) Потому что данные, перехваченные сегодня, можно расшифровать в будущем — когда квантовый компьютер появится.
- В) Это маркетинговый ход, угрозы не существует.

**Правильный ответ: Б.**
Артём не расшифровывал трафик в 2025-м — он только собирал. Расшифровка случилась в 2031-м. Момент кражи и момент прочтения разделены во времени.

---

**Вопрос 6.**
Какой из стандартов NIST предназначен для замены ECDH в TLS?

- А) ML-DSA
- Б) SLH-DSA
- В) ML-KEM (Kyber)

**Правильный ответ: В.**
ML-KEM — это Key Encapsulation Mechanism, замена для обмена ключами. ML-DSA и SLH-DSA — алгоритмы цифровой подписи.

---

**Вопрос 7.**
После включения гибридного TLS ты замечаешь рост времени загрузки у мобильных пользователей на медленных соединениях. В чём скорее всего причина?

- А) Высокая нагрузка на процессор при вычислении ключей.
- Б) Увеличение размера передаваемых данных (публичный ключ + шифротекст Kyber), что повышает latency.
- В) Телефоны не поддерживают новые алгоритмы.

**Правильный ответ: Б.**
Kyber на CPU работает даже быстрее X25519. Проблема — в сети: лишние ~1200 байт в `ClientHello` ощутимы на медленных каналах.

---

## Задание: найди и исправь уязвимости в конфиге TLS

### Контекст

Ты — разработчик в финтех-стартапе. Тебе поручили обновить конфигурацию TLS для микросервиса, обрабатывающего транзакции. Текущий конфиг уязвим для атаки HNDL.

Задача — перевести сервис на **гибридную схему** с поддержкой обратной совместимости, чтобы старые клиенты не отвалились.

### Исходный код с уязвимостями

```python
<?php

declare(strict_types=1);

/**
 * Гипотетическая библиотека для SSL/TLS контекста
 * В реальном коде использовались бы OpenSSL или другие расширения
 */
namespace HypotheticalSSL;

/**
 * Перечисление протоколов (аналог Protocols в Python)
 */
enum Protocols: string {
    case TLSv1_2 = 'TLSv1.2';
    case TLSv1_3 = 'TLSv1.3';
    case DTLSv1_2 = 'DTLSv1.2';
}

/**
 * Класс для настройки SSL контекста
 */
class SSLContext {
    private Protocols $protocol;
    private string $keyExchangeGroup = '';
    private array $ciphers = [];
    private array $options = [];
    
    public function __construct(Protocols $protocol) {
        $this->protocol = $protocol;
        $this->options['min_protocol'] = $protocol;
        $this->options['max_protocol'] = $protocol;
    }
    
    /**
     * Установка группы для обмена ключами
     * [УЯЗВИМОСТЬ 1] Жёсткая привязка к классической эллиптической кривой
     */
    public function setKeyExchangeGroup(string $group): self {
        // В реальности: openssl_ec_key_curves
        $this->keyExchangeGroup = $group;
        
        // Логируем уязвимость (для демонстрации)
        $this->logVulnerability(
            'key_exchange',
            "Жёсткая привязка к классической эллиптической кривой {$group}. " .
            "Когда появится квантовый компьютер — весь архивный трафик будет расшифрован."
        );
        
        return $this;
    }
    
    /**
     * Установка списка шифров
     * [УЯЗВИМОСТЬ 2] AES-128: алгоритм Гровера снижает стойкость до 64 бит
     */
    public function setCiphers(string $ciphers): self {
        $this->ciphers = explode(':', $ciphers);
        
        // Проверяем на слабые шифры
        foreach ($this->ciphers as $cipher) {
            if (str_contains($cipher, 'AES_128')) {
                $this->logVulnerability(
                    'cipher',
                    "AES-128: алгоритм Гровера снижает стойкость до 64 бит — небезопасно."
                );
            }
            
            if (str_contains($cipher, 'SHA1') || str_contains($cipher, 'RC4')) {
                $this->logVulnerability(
                    'cipher',
                    "Использование устаревшего/сломанного шифра: {$cipher}"
                );
            }
        }
        
        return $this;
    }
    
    /**
     * Логирование уязвимостей
     */
    private function logVulnerability(string $type, string $message): void {
        $logEntry = sprintf(
            "[УЯЗВИМОСТЬ %s] %s",
            $type,
            $message
        );
        
        // В реальности: error_log, syslog, etc.
        if (PHP_SAPI === 'cli') {
            echo "\033[31m⚠️  {$logEntry}\033[0m\n";
        }
        
        // Сохраняем в контексте для аудита
        if (!isset($this->options['vulnerabilities'])) {
            $this->options['vulnerabilities'] = [];
        }
        $this->options['vulnerabilities'][] = [
            'type' => $type,
            'message' => $message,
            'time' => time()
        ];
    }
    
    /**
     * Получение параметров контекста
     */
    public function getContextOptions(): array {
        return [
            'protocol' => $this->protocol->value,
            'key_exchange_group' => $this->keyExchangeGroup,
            'ciphers' => implode(':', $this->ciphers),
            'options' => $this->options
        ];
    }
    
    /**
     * Проверка на квантовые уязвимости
     */
    public function auditQuantumVulnerabilities(): array {
        $issues = [];
        
        // Проверка 1: Квантово-уязвимые группы ключей
        $quantumVulnerableGroups = ['secp256r1', 'secp384r1', 'secp521r1', 'X25519'];
        if (in_array($this->keyExchangeGroup, $quantumVulnerableGroups)) {
            $issues[] = [
                'severity' => 'CRITICAL',
                'component' => 'key_exchange',
                'issue' => "Группа {$this->keyExchangeGroup} уязвима для алгоритма Шора",
                'recommendation' => 'Перейти на пост-квантовые алгоритмы (ML-KEM, Kyber)'
            ];
        }
        
        // Проверка 2: Слабые симметричные шифры
        foreach ($this->ciphers as $cipher) {
            if (str_contains($cipher, 'AES_128')) {
                $issues[] = [
                    'severity' => 'HIGH',
                    'component' => 'cipher',
                    'issue' => "AES-128 даёт только 64 бита квантовой стойкости",
                    'recommendation' => 'Использовать AES-256 или ChaCha20-Poly1305'
                ];
            }
        }
        
        return $issues;
    }
}

/**
 * Функция создания контекста с уязвимостями
 */
function createVulnerableContext(): SSLContext {
    $ctx = new SSLContext(Protocols::TLSv1_3);
    
    // [УЯЗВИМОСТЬ 1]
    // Жёсткая привязка к классической эллиптической кривой secp256r1.
    // Когда появится квантовый компьютер — весь архивный трафик будет расшифрован.
    $ctx->setKeyExchangeGroup("secp256r1");
    
    // [УЯЗВИМОСТЬ 2]
    // AES-128: алгоритм Гровера снижает стойкость до 64 бит — небезопасно.
    $ctx->setCiphers("TLS_AES_128_GCM_SHA256");
    
    return $ctx;
}

/**
 * Безопасная версия создания контекста
 */
function createSecureContext(): SSLContext {
    $ctx = new SSLContext(Protocols::TLSv1_3);
    
    // ✅ БЕЗОПАСНО: гибридный подход
    // Классическая кривая + пост-квантовая группа
    $ctx->setKeyExchangeGroup("secp256r1:X25519:ML-KEM-768");
    
    // ✅ БЕЗОПАСНО: AES-256 (128 бит квантовой стойкости)
    $ctx->setCiphers("TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256");
    
    return $ctx;
}

/**
 * Анализ и сравнение контекстов
 */
function analyzeContexts(): void {
    echo "🔐 АНАЛИЗ БЕЗОПАСНОСТИ SSL/TLS КОНТЕКСТОВ\n";
    echo "==========================================\n\n";
    
    // Создаём уязвимый контекст
    echo "📦 Создание уязвимого контекста:\n";
    $vulnerableCtx = createVulnerableContext();
    
    $vulnOptions = $vulnerableCtx->getContextOptions();
    echo "Протокол: {$vulnOptions['protocol']}\n";
    echo "Группа ключей: {$vulnOptions['key_exchange_group']}\n";
    echo "Шифры: {$vulnOptions['ciphers']}\n\n";
    
    // Аудит уязвимого контекста
    echo "🔍 Аудит уязвимостей:\n";
    $issues = $vulnerableCtx->auditQuantumVulnerabilities();
    
    foreach ($issues as $issue) {
        $severityColor = match($issue['severity']) {
            'CRITICAL' => "\033[31m",
            'HIGH' => "\033[33m",
            default => "\033[0m"
        };
        
        echo "{$severityColor}[{$issue['severity']}] {$issue['component']}: {$issue['issue']}\033[0m\n";
        echo "  ➜ Рекомендация: {$issue['recommendation']}\n\n";
    }
    
    // Создаём безопасный контекст
    echo "✅ Создание безопасного контекста:\n";
    $secureCtx = createSecureContext();
    $secureOptions = $secureCtx->getContextOptions();
    
    echo "Протокол: {$secureOptions['protocol']}\n";
    echo "Группа ключей: {$secureOptions['key_exchange_group']} (гибридный подход)\n";
    echo "Шифры: {$secureOptions['ciphers']}\n";
    
    // Аудит безопасного контекста
    $secureIssues = $secureCtx->auditQuantumVulnerabilities();
    if (empty($secureIssues)) {
        echo "\n✅ Контекст безопасен для пост-квантовой эры\n";
    }
    
    // Сравнение в таблице
    echo "\n📊 СРАВНЕНИЕ КОНТЕКСТОВ:\n";
    echo str_repeat('-', 80) . "\n";
    echo sprintf("%-30s %-25s %-25s\n", "Параметр", "Уязвимый контекст", "Безопасный контекст");
    echo str_repeat('-', 80) . "\n";
    echo sprintf("%-30s %-25s %-25s\n", 
        "Key Exchange", 
        "secp256r1 (❌ классический)",
        "hybrid (✅ PQC ready)"
    );
    echo sprintf("%-30s %-25s %-25s\n",
        "Квантовая стойкость",
        "0 бит (💀 взломано)",
        "128+ бит (✅ безопасно)"
    );
    echo sprintf("%-30s %-25s %-25s\n",
        "Шифр",
        "AES-128 (⚠️ 64 бита)",
        "AES-256 (✅ 128 бит)"
    );
    echo sprintf("%-30s %-25s %-25s\n",
        "Готовность к Q-Day",
        "❌ Не готов",
        "✅ Готов"
    );
}

// Запуск анализа
if (PHP_SAPI === 'cli') {
    analyzeContexts();
    
    echo "\n💡 КЛЮЧЕВЫЕ ВЫВОДЫ:\n";
    echo "1. secp256r1 и другие классические кривые → 💀 полная уязвимость\n";
    echo "2. AES-128 → ⚠️ только 64 бита квантовой стойкости\n";
    echo "3. Гибридный подход (классика + PQC) → ✅ безопасность даже при взломе одного алгоритма\n";
    echo "4. Рекомендация: используйте AES-256 и пост-квантовые KEM\n";
}
```

### Задание

1. Замени группу обмена ключами на гибридную (X25519 + Kyber768).
2. Добавь fallback: если клиент не поддерживает PQC — сервер соглашается на X25519, но не на RSA.
3. Обнови симметричный шифр для устойчивости к алгоритму Гровера.

**Подсказки:**
- *Подсказка 1:* Для защиты от Гровера нужен ключ ≥ 256 бит.
- *Подсказка 2:* Метод принимает список; порядок = приоритет.
- *Подсказка 3:* Гибридная группа: `"x25519_mlkem768"`.

### Решение

```
<?php

/**
 * TLS Configuration with Post-Quantum Cryptography (PQC)
 *
 * Решение:
 * 1. Гибридная группа обмена ключами: X25519 + Kyber768 (x25519_mlkem768)
 * 2. Fallback: X25519 (без RSA) если клиент не поддерживает PQC
 * 3. Симметричный шифр AES-256 (ключ ≥ 256 бит) — устойчив к алгоритму Гровера
 */

class TLSConfig
{
    /**
     * Настройка групп обмена ключами.
     *
     * Порядок = приоритет (подсказка 2):
     *   1. x25519_mlkem768 — гибридная PQC-группа (подсказка 3)
     *   2. x25519            — fallback для клиентов без PQC (подсказка 1 из условия)
     *
     * RSA-based группы (ffdhe2048, ffdhe3072 и т.д.) намеренно исключены.
     *
     * @return array
     */
    public static function getCurves(): array
    {
        return [
            'x25519_mlkem768', // Гибрид: X25519 + Kyber768 (ML-KEM-768) — приоритет
            'x25519',          // Fallback: только X25519, если клиент не поддерживает PQC
            // 'secp256r1',    // НЕ включаем RSA/ECDSA без явной необходимости
        ];
    }

    /**
     * Список поддерживаемых шифронаборов.
     *
     * Используем AES-256-GCM и CHACHA20-POLY1305:
     *   - ключ 256 бит → квантовая стойкость 128 бит после атаки Гровера (подсказка 1)
     *   - AES-128 намеренно исключён (после Гровера эффективная стойкость = 64 бит — недостаточно)
     *
     * @return array
     */
    public static function getCipherSuites(): array
    {
        return [
            // TLS 1.3 — только 256-битные ключи
            'TLS_AES_256_GCM_SHA384',
            'TLS_CHACHA20_POLY1305_SHA256',
            // TLS 1.2 fallback — только 256-битные ключи, без RSA-обмена ключами
            'ECDHE-ECDSA-AES256-GCM-SHA384',
            'ECDHE-ECDSA-CHACHA20-POLY1305',
        ];
    }

    /**
     * Применяет конфигурацию к контексту SSL-стрима PHP.
     *
     * @param  array $baseOptions  Базовые опции stream_context (cert, key и т.д.)
     * @return resource            SSL stream context
     */
    public static function createStreamContext(array $baseOptions = []): mixed
    {
        $sslOptions = array_merge($baseOptions, [
            // Минимальная версия — TLS 1.3; 1.2 разрешён только для совместимости
            'min_protocol_version' => STREAM_CRYPTO_METHOD_TLSv1_2_CLIENT,

            // Наборы шифров: только AES-256 и ChaCha20 (≥256-бит ключи)
            'ciphers' => implode(':', self::getCipherSuites()),

            // Группы обмена ключами: PQC-гибрид в приоритете, X25519 как fallback
            // Передаётся как строка через опцию 'curves' или напрямую через конфигурацию OpenSSL
            'curves'  => implode(':', self::getCurves()),
        ]);

        return stream_context_create(['ssl' => $sslOptions]);
    }
}


// ─────────────────────────────────────────────
//  Пример: настройка через конфигурацию OpenSSL
// ─────────────────────────────────────────────

class OpenSSLTLSConfigurator
{
    /**
     * Возвращает строку конфигурации для OpenSSL (используется в nginx/apache/php-fpm
     * через директиву ssl_conf_command или SSLOpenSSLConfCmd).
     *
     * @return string
     */
    public static function getOpenSSLConfig(): string
    {
        $curves      = implode(':', TLSConfig::getCurves());
        $cipherSuites = implode(':', TLSConfig::getCipherSuites());

        return <<<CONF
        [ssl_sect]
        # Группы обмена ключами: гибридная PQC (приоритет) + X25519 (fallback)
        # RSA-based группы отсутствуют намеренно
        Groups = {$curves}

        # Симметричные шифры: только ≥256-бит ключи (защита от алгоритма Гровера)
        CipherSuites = {$cipherSuites}

        # Минимальная версия TLS
        MinProtocol = TLSv1.2
        CONF;
    }

    /**
     * Генерирует массив опций для Guzzle / Symfony HttpClient / любого curl-based клиента.
     *
     * @return array
     */
    public static function getCurlOptions(): array
    {
        return [
            // TLS 1.2 минимум; предпочтительно 1.3
            CURLOPT_SSLVERSION    => CURL_SSLVERSION_TLSv1_2,

            // Наборы шифров (curl передаёт их в OpenSSL)
            CURLOPT_SSL_CIPHER_LIST => implode(':', TLSConfig::getCipherSuites()),

            // Группы обмена ключами через CURLOPT_TLS13_CIPHERS (curl 7.61+)
            // или через переменную окружения OPENSSL_CONF
            CURLOPT_TLS13_CIPHERS => implode(':', TLSConfig::getCurves()),
        ];
    }
}


// ─────────────────────────────────────────────
//  Демонстрация
// ─────────────────────────────────────────────

echo "=== TLS Config: Post-Quantum Hardened ===\n\n";

echo "Группы обмена ключами (порядок = приоритет):\n";
foreach (TLSConfig::getCurves() as $i => $curve) {
    $label = $i === 0 ? ' ← PQC-гибрид (X25519 + Kyber768)' : ' ← Fallback (только X25519, без RSA)';
    echo "  " . ($i + 1) . ". {$curve}{$label}\n";
}

echo "\nСимметричные шифры (ключи ≥ 256 бит, защита от Гровера):\n";
foreach (TLSConfig::getCipherSuites() as $i => $cipher) {
    echo "  " . ($i + 1) . ". {$cipher}\n";
}

echo "\nOpenSSL конфиг:\n";
echo OpenSSLTLSConfigurator::getOpenSSLConfig() . "\n";

echo "\ncURL опции:\n";
foreach (OpenSSLTLSConfigurator::getCurlOptions() as $opt => $val) {
    echo "  [{$opt}] => {$val}\n";
}
```

### Почему это работает

**Гибридный обмен:** X25519 страхует от математических атак на Kyber, а Kyber страхует от квантового взлома X25519. Ни один из сценариев атаки не компрометирует оба алгоритма одновременно.

**Fallback по приоритету:** сервер предлагает гибрид первым. Старый клиент его не поймёт — согласует чистый X25519. Новый клиент получит полную PQC-защиту. Доступность сохраняется.

**AES-256:** единственный необходимый шаг для симметричной части — просто удвоить ключ.
