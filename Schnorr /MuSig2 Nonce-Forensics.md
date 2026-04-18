# Schnorr / MuSig2 Nonce-Forensics: математическая модель системы

## Статус документа

Этот документ описывает **только подтверждённые и используемые компоненты системы**.  
В него включены:

- математическая модель BIP340-слоя;
- формализация nonce-forensics как задачи анализа семейства скрытых нонсов;
- метрики family compression и connectivity;
- protocol-valid линейная модель частичной подписи MuSig2;
- форматы данных и инварианты, которые уже проверяются кодом и отчётами.

В документ **не включены** неподтверждённые claims и недоказанные recovery-утверждения. Он фиксирует только то, что уже является ценным ядром проекта.


## Сравнение с ближайшими аналогами

| Аналог                                 | Что это такое                                                                                   | Что совпадает с вашей системой                                                                 | Чего у аналога нет                                                                                                                 | Вывод                                                                                                       |
| -------------------------------------- | ----------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **BIP340**                             | Стандарт Schnorr для secp256k1: x-only pubkeys, challenge, verification, test vectors           | Есть базовая алгебра подписи, even-Y канонизация, проверка через стандартную верификацию       | Нет forensic-пайплайна по семействам скрытых nonce-кандидатов, нет compression/connectivity-метрик, нет blind benchmark discipline | Это **фундамент**, но не аналог нашей системы ([GitHub][1])                                                 |
| **BIP327 (MuSig2)**                    | Стандарт MuSig2: session context, partial signatures, aggregate signature                       | Есть partial signatures, session context, совместимость с BIP340                               | Нет affine-forensics слоя для анализа `k_eff(d')`, нет ранжирования кандидатов, нет dedup/blind forensic discipline                | Это **протокольный слой**, а не аналитическая система ([GitHub][2])                                         |
| **`secp256k1-zkp` MuSig2**             | Reference/engineering library для MuSig2: pubnonce, aggnonce, session, partial sig verify/agg   | Есть protocol-valid execution path и проверка individual partial signatures                    | Нет массового forensic-анализа скрытых нонсов, нет интерпретируемых семейных метрик, нет blind corpus framework                    | Это **референсная библиотека исполнения**, а не исследовательский forensic engine ([GitHub][3])             |
| **MuSig2 paper**                       | Академическое описание двухраундовой multisig-схемы                                             | Совпадает математический объект: ordinary Schnorr output, key aggregation, concurrent sessions | Нет operational tooling для audit/forensics, нет bridge/reporting/corpus discipline                                                | Это **схема**, а не продуктовый анализатор ([eprint.iacr.org][4])                                           |
| **FROST / RFC 9591**                   | Двухраундовые threshold Schnorr signatures                                                      | Родство по линейности частичных вкладов и работе с nonce/sign shares                           | Это threshold-протокол другого класса; нет BIP340+MuSig2 forensic-пайплайна, нет affine hidden-nonce ranking по кандидатам секрета | Это **соседняя категория**, но не прямой аналог ([datatracker.ietf.org][5])                                 |
| **HNP / weak-nonce literature**        | Литература по Hidden Number Problem, lattice/FFT/Bleichenbacher-атакам на biased/partial nonces | Совпадает идея: истинный ключ выявляется через структуру/смещение семейства nonce-кандидатов   | Обычно нет BIP340 membership bridge, нет MuSig2 session-aware linearization, нет reproducible audit corpora и blind hygiene        | Это **ближайший научный предок**, но не интегрированная система уровня вашего стека ([ecc2017.cs.ru.nl][6]) |
| **Публичные BIP340/MuSig2 библиотеки** | Реализации signing/verifying для кошельков и SDK                                                | Дают совместимость, подписи, иногда partial sig verification                                   | Нет forensic ranking, нет connectivity/compression model, нет стерильного benchmark runner                                         | Это **runtime tooling**, а не forensic research platform ([GitHub][7])                                      |


---

## 2. Базовая алгебра

Работа ведётся в группе `secp256k1`.

- Поле: `F_p`, где
  `p = 2^256 - 2^32 - 977`
- Порядок группы:
  `n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141`
- Базовая точка: `G`

Все скалярные уравнения ниже понимаются по модулю `n`, если явно не указано иное.

Для точки `P = dG` в x-only представлении используется **канонический even-Y представитель** секрета:

`d = d0`, если `y(d0 G)` чётен, иначе `d = n - d0`.

Аналогично для нонса `k` в BIP340 используется канонический even-Y представитель точки `R = kG`.

---

## 3. Модель BIP340

### 3.1. Подпись

Для сообщения `m`, x-only публичного ключа `P_x` и нонса `k` подпись описывается как:

- `R = kG`
- `r = x(R)`
- `e = H_tag("BIP0340/challenge", r || P_x || m) mod n`
- `s = k + e d mod n`

Итоговая подпись — это пара `(r, s)`.

### 3.2. Проверка через восстановленную точку

Система использует эквивалентную форму проверки:

`R* = sG - eP`

Подпись принадлежит BIP340-многообразию тогда и только тогда, когда одновременно выполнены условия:

1. `0 <= r < p`
2. `0 <= s < n`
3. `R* != O`
4. `y(R*)` чётен
5. `x(R*) = r`

Именно эта форма реализована как **membership bridge**.

### 3.3. Что именно даёт bridge

Bridge переводит подпись из представления `(r, s, P, m)` в точку `R*`, которая:

- либо подтверждает строгую BIP340-согласованность;
- либо возвращает причину нарушения (`r_out_of_field`, `s_out_of_range`, `r_star_is_infinity`, `r_star_odd_y`, `r_star_x_mismatch`).

Это не только проверка валидности. Это **нормализующий переход**, делающий дальнейший forensic-анализ корректным: все последующие метрики вычисляются только на строках, прошедших этот bridge.

---

## 4. Скрытый нонс как функция от кандидата секрета

Главная идея системы — рассматривать нонс не как неизвестную константу, а как **функцию от кандидата секрета**.

Для каждой подписи `i` и любого кандидата `d'` определяется

`k_i(d') = s_i - e_i d' mod n`

Если `d' = d`, то:

`k_i(d) = k_i`

то есть получается истинный канонический скаляр нонса этой подписи.

Если `d' != d`, то семейство `k_i(d')` ведёт себя как псевдослучайное affine-смещение.

Отсюда возникает основная задача nonce-forensics:

> найти такие статистические и структурные функционалы `F`, для которых значение `F({k_i(d)})` для истинного секрета заметно отличается от `F({k_i(d')})` для ложных кандидатов.

Это и есть математическое ядро всей системы.

---

## 5. Группировка подписей

Система работает не с одной подписью, а с **группами подписей одного и того же x-only публичного ключа**.

Для группы `G(P_x)` с подписями `i = 1..m` строятся:

- значения `e_i`;
- значения `s_i`;
- семейство скрытых нонсов `k_i(d')`;
- derived-метрики поверх `k_i(d')`.

Именно групповая постановка делает возможным forensic-анализ. На одной строке почти никакой структурный сигнал не виден; на ансамбле строк он становится измеримым.

---

## 6. Family Compression Model

### 6.1. Определение

Пусть для кандидата `d'` вычислены значения

`K(d') = [k_1(d'), ..., k_m(d')]`

Система измеряет, насколько это семейство **сжато** по сравнению со случайным фоном.

В текущем ядре используются следующие функционалы.

### 6.2. Базовые compression-метрики

#### 1. Число различных нонсов

`U_k(d') = |{k_i(d')}|`

#### 2. Число коллизий

`C_k(d') = m - U_k(d')`

Для reuse-сценариев правильный `d` уменьшает `U_k` и увеличивает `C_k`.

#### 3. Семейство последовательных дельт

`Δ_i(d') = k_{i+1}(d') - k_i(d') mod n`, `i = 1..m-1`

#### 4. Число различных дельт

`U_Δ(d') = |{Δ_i(d')}|`

#### 5. Число повторяющихся дельт

`C_Δ(d') = (m - 1) - U_Δ(d')`

Для ladder-сценариев правильный `d` уменьшает число различных дельт.

### 6.3. Префиксные compression-метрики

Для заданного числа бит `b` рассматривается множество высоких битов:

`Pref_b(k) = floor(k / 2^(256-b))`

Тогда:

`U_pref_b(d') = |{Pref_b(k_i(d'))}|`

В системе используются:

- `U_pref_128`
- `U_pref_64`
- `U_pref_32`
- `U_pref_16`

Для short / prefix-bias сценариев правильный `d` уменьшает количество различных высоких префиксов.

### 6.4. Интерпретация

Compression-модель отвечает на вопрос:

> становится ли семейство скрытых нонсов более компактным, если подставить конкретный кандидат секрета?

Эта модель особенно сильна на сценариях:

- nonce reuse;
- ladder;
- short nonce;
- shared prefix bias.

---

## 7. Connectivity Model

Compression измеряет сжатие по глобальным агрегатам. Connectivity измеряет **локальную геометрию** семейства.

### 7.1. Два знаковых листа

Для семейства `K(d')` строятся две сортированные ветви:

- `K⁺(d') = sort({k_i(d')})`
- `K⁻(d') = sort({-k_i(d') mod n})`

Это позволяет учитывать symmetry/duality эффекты, возникающие из sign/parity представления.

### 7.2. Режимы связности

Рассматриваются четыре режима соседства:

- `++`
- `--`
- `+-`
- `-+`

Для каждой пары ветвей считается локальный счётчик разностей по ближайшим соседям в окне ширины `w`.

### 7.3. Локальная delta-support метрика

Для каждого режима строится счётчик:

`count_mode(δ)` = сколько раз локальная соседняя разность равна `δ`

Далее определяется:

`Support_mode(d') = max_δ count_mode(δ)`

И затем:

`Support_local(d') = max_mode Support_mode(d')`

Эта величина в коде соответствует `max_local_delta_support`.

### 7.4. Повторная масса дельт

Если одна и та же локальная разность повторяется во многих соседних парах, возникает дополнительный сигнал.

Определяется:

`Mass_repeat(d') = Σ_δ max(count(δ)-1, 0)`

Это соответствует `total_repeated_delta_mass`.

### 7.5. Префиксная поддержка

Для каждой ветви `K⁺`, `K⁻` и набора бит `b ∈ {128, 96, 64, 32, 16}` считается максимальная поддержка одного и того же префикса:

`TopPref_b^+(d') = max_u |{k in K⁺ : Pref_b(k)=u}|`
`TopPref_b^-(d') = max_u |{k in K⁻ : Pref_b(k)=u}|`

Затем:

`TopPref_b(d') = max(TopPref_b^+(d'), TopPref_b^-(d'))`

Ключевые реализованные метрики:

- `max_prefix128_support`
- `max_prefix96_support`
- `max_prefix32_support`
- `max_prefix16_support`

### 7.6. Интерпретация

Connectivity-модель отвечает на вопрос:

> возникает ли у семейства скрытых нонсов локальная повторяемая геометрия, не сводимая к простой глобальной коллизии?

Эта модель полезна, когда структура проявляется как:

- повторяемые локальные сдвиги;
- кластеры ближайших соседей;
- устойчивые префиксные листья;
- паритетно-симметричные семейства.

---

## 8. Protocol-Valid MuSig2 Model

### 8.1. Зачем нужен отдельный слой

Для MuSig2 финальная on-chain подпись скрывает внутреннюю структуру. Поэтому система работает на уровне **частичных подписей** и полного `session context`.

Ключевая ценность состоит в том, что частичная подпись участника может быть приведена к affine-форме, пригодной для того же типа анализа, что и в Schnorr.

### 8.2. Сырой нонс участника

В MuSig2 участник использует два сырьевых скаляра нонса:

- `k1_raw`
- `k2_raw`

Их публичное представление:

- `R1 = k1_raw G`
- `R2 = k2_raw G`
- `pubnonce = cbytes(R1) || cbytes(R2)`

### 8.3. Session values

Из полного session context система восстанавливает значения:

- агрегированный ключ `Q`;
- aggregate nonce `R`;
- nonce coefficient `b`;
- challenge `e`;
- key aggregation coefficient `a` для конкретного участника;
- tweak / parity factors из reference implementation.

### 8.4. Канонизация знака

Система явно учитывает parity-факторы.

Для aggregate key:

- `g = 1`, если `y(Q)` чётен,
- `g = n - 1`, если `y(Q)` нечётен.

Из reference context также берётся `gacc`.

Комбинированный фактор ключа:

`par_key = g * gacc mod n`

Для aggregate nonce `R`:

- `par_nonce = 1`, если `y(R)` чётен,
- `par_nonce = n - 1`, если `y(R)` нечётен.

Тогда эффективные компоненты нонса:

- `k1_eff = k1_raw`, если `par_nonce = 1`, иначе `n - k1_raw`
- `k2_eff = k2_raw`, если `par_nonce = 1`, иначе `n - k2_raw`

### 8.5. Эффективный нонс участника

Система использует exact affine-представление:

`k_eff = k1_eff + b * k2_eff mod n`

Это уже не произвольная эвристика, а восстанавливаемая из session context величина.

### 8.6. Эффективный коэффициент секрета

Для участника вычисляется:

`c = e * a * g * gacc mod n`

В коде это проходит через промежуточные значения:

- `key_coeff_unwrapped = e * a mod n`
- `combined_parity = g * gacc mod n`
- `c = key_coeff_unwrapped * combined_parity mod n`

### 8.7. Финальная affine-форма частичной подписи

Частичная подпись участника удовлетворяет равенству:

`s_i = k_eff,i + c_i d_i mod n`

где:

- `d_i` — секретный скаляр участника;
- `k_eff,i` — эффективный нонс строки;
- `c_i` — коэффициент, вычисляемый из полного session context.

Это ключевой математический результат системы на уровне MuSig2:

> частичная подпись MuSig2 допускает protocol-valid affine-линеаризацию, совместимую с тем же forensic-подходом, что и BIP340.

### 8.8. Что это даёт

После линеаризации можно определить MuSig2-аналог скрытого нонса:

`k_eff,i(d') = s_i - c_i d' mod n`

И далее применять те же классы функционалов:

- compression;
- prefix support;
- connectivity;
- dedup-aware group analysis.

---

## 9. Дедупликация как часть модели данных

Для MuSig2 публичные корпуса могут содержать повторные сырьевые пары нонсов. Система поэтому использует **dedup по raw nonce pairs** как часть data discipline.

Если две строки имеют одинаковую пару

`(k1_raw, k2_raw)`

они не считаются независимыми наблюдениями для forensics-уровня, даже если формально принадлежат разным тестовым контекстам.

Поэтому для адаптированного публичного partial-sig корпуса система фиксирует:

- число строк;
- число различных сообщений;
- число различных коэффициентов `c_i`;
- число различных effective nonce `k_eff`;
- число различных raw nonce pairs.

Это защищает анализ от искусственного усиления сигнала за счёт повторов источника.

---

## 10. Стерильная blind-модель данных

Хотя этот документ не формулирует recovery-claims, для исследовательской дисциплины система уже задаёт строгую blind-модель корпуса.

### 10.1. Разделение public и truth

Каждый blind case состоит из двух логически разделённых сущностей:

1. **public case** — только те поля, которые допустимы для solver / audit pipeline;
2. **truth manifest** — приватный эталон для внешнего verifier.

### 10.2. Инварианты стерильности

Публичный blind case обязан не содержать:

- `secret`
- `hex_d`
- `musig_raw_secret_hex`
- `musig_k1_raw_int`
- `musig_k2_raw_int`
- `musig_effective_k_int`

То есть public dataset моделируется как честный внешний наблюдаемый слой, а truth живёт отдельно.

### 10.3. Benchmark integrity

Benchmark runner:

- сканирует public cases на forbidden fields;
- использует truth только на этапе внешней верификации;
- маркирует `truth_access: false` для публичного слоя.

Даже там, где blind-recovery-claim не делается, эта часть системы уже ценна как **инфраструктура экспериментальной чистоты**.

---

## 11. Форматы данных

### 11.1. Строка BIP340-аудита

Нормализованная строка содержит, по меньшей мере:

- `pubkey_xonly_hex`
- `message_hash_hex`
- `r_int`
- `s_int`
- `e_int`
- `signature_verified`
- `membership_pass`
- `failure_reason`

### 11.2. Строка MuSig2 partial-signature корпуса

Нормализованная строка содержит:

- `pubkey_xonly_hex`
- `message_hash_hex`
- `s_int`
- `e_int`  
  (в MuSig2 это уже affine-коэффициент `c_i`)
- `musig_coeff_int`
- `musig_noncecoef_b_int`
- `musig_keyagg_coeff_int`
- `musig_group_parity_factor_int`
- `musig_nonce_parity_factor_int`
- `musig_aggnonce_hex`
- `signature_verified`
- `membership_pass`

В calibration / generator-режимах могут дополнительно храниться raw nonce fields, но public blind-корпус и dedup-public audit-корпус от них очищаются.

---

## 12. Ранги и функционалы

Система не использует один “магический скор”. Она работает как набор функционалов, каждый из которых отражает конкретную гипотезу о структуре.

### 12.1. Ранжирование по compression

Кандидаты сортируются так, чтобы вверху оказывались те, у которых:

- меньше `unique_k_count`;
- меньше `unique_delta_count`;
- меньше число различных высоких префиксов.

То есть предпочитаются кандидаты, у которых семейство скрытых нонсов выглядит более компактным.

### 12.2. Ранжирование по connectivity

Кандидаты сортируются так, чтобы вверху оказывались те, у которых:

- больше `max_local_delta_support`;
- больше `max_prefix128_support`;
- больше `max_prefix96_support`;
- больше `max_prefix32_support`;
- больше `total_repeated_delta_mass`.

То есть предпочитаются кандидаты, у которых семейство скрытых нонсов имеет более выраженную локальную повторяемость.

### 12.3. Многомодельность

Важное свойство системы: разные функционалы соответствуют разным leakage-гипотезам.

- reuse → коллизии;
- ladder → повтор дельт;
- short nonce → малое число высоких префиксов;
- prefix bias → максимальная поддержка общего high-prefix;
- mixed local families → connectivity.

Это делает систему не одной атакой, а **платформой гипотез**.

---

## 13. Что уже доказано кодом и отчётами

Ниже перечислены только те свойства, которые уже подтверждены артефактами проекта.

### 13.1. Корректность BIP340 bridge

Подтверждено на официальных векторах BIP340.

Артефакт:
- `reports/bip340_official_test5_report.json`

Зафиксировано:
- `9/9` ожидаемо валидных векторов проходят membership bridge;
- `10/10` ожидаемо невалидных векторов корректно отвергаются.

### 13.2. Корректность на реальных key-path Taproot подписях

Артефакты:
- `reports/mainnet_taproot_test56_report.json`
- `reports/mainnet_taproot_holdout_exact_report.json`

Зафиксировано:
- `60/60` real mainnet key-path подписей одновременно проходят exact verification и membership bridge;
- на holdout-корпусе `1359/1359` также проходят verification и membership.

Это подтверждает, что bridge корректен не только на synthetic/official data, но и на реальных корпусах.

### 13.3. Реализованная family compression модель

Артефакт-код:
- `scripts/family_compression_control.py`

Подтверждено, что в системе реализованы и используются:
- `unique_k_count`
- `collision_count`
- `unique_delta_count`
- `repeated_delta_count`
- `high128_unique_count`
- `high64_unique_count`
- `high32_unique_count`
- `high16_unique_count`
- `low16_unique_count`

### 13.4. Реализованная connectivity модель

Артефакт-код:
- `scripts/family_connectivity_scanner.py`

Подтверждено, что система реализует:
- двухлистовую модель `K⁺ / K⁻`;
- режимы `++`, `--`, `+-`, `-+`;
- `max_local_delta_support`;
- `total_repeated_delta_mass`;
- `max_prefix128_support`, `max_prefix96_support`, `max_prefix32_support`, `max_prefix16_support`.

### 13.5. Позитивные BIP340 calibration families

Артефакт:
- `reports/bip340_positive_controls_summary.json`

Подтверждено, что система калибруется на семьях:
- `short_nonce128`
- `short_nonce160`
- `ladder`
- `prefix_bias32`

Это формализует controlled nonce-аномалии как стандартный calibration layer.

### 13.6. Protocol-valid MuSig2 affine-модель

Артефакт-код:
- `scripts/bip327_partial_sig_to_candidate_jsonl.py`
- `scripts/poisoned_bip327_protocol_valid.py`

Подтверждено на уровне кода и генерации, что для каждой protocol-valid строки вычисляются:
- `musig_coeff_int = c_i`
- `musig_noncecoef_b_int = b_i`
- `musig_effective_k_int = k_eff,i`

и проверяется тождество:

` s_i = k_eff,i + c_i d_i mod n `

### 13.7. Protocol-valid MuSig2 calibration families

Артефакт:
- `reports/bip327_protocol_valid_controls_summary.json`

Подтверждено наличие четырёх protocol-valid семейств:
- `raw_nonce_reuse`
- `raw_nonce_ladder`
- `short_raw_nonce128`
- `prefix_raw_nonce32`

То есть MuSig2-слой проекта не является “словесным продолжением” BIP340, а имеет отдельный генератор protocol-valid контролей.

### 13.8. Dedup-public partial-signature audit корпус

Артефакт:
- `reports/bip327_partial_sig_adapter_report_dedup_public.json`

Подтверждено:
- `signature_count = 5`
- `distinct_raw_nonce_pairs = 5`
- `distinct_coefficients = 5`
- `distinct_effective_k = 5`
- `dedup_raw_nonce_pairs = true`

Это фиксирует, что публичный audit-корпус уже очищается от повторов сырьевых пар нонсов.

### 13.9. Стерильный blind case generation

Артефакт:
- `reports/blind_case_001_generation_report.json`

Подтверждено:
- `session_count = 20`
- `distinct_messages = 20`
- `distinct_nonce_pairs = 20`
- `distinct_noncecoef_b = 20`
- `distinct_aggnonces = 20`
- `all_signature_verified = true`
- `all_membership_pass = true`
- `public_contains_secret_fields = false`
- `truth_access = false`
- `sterile_ready = true`

Это важно как доказанный data-discipline слой.

### 13.10. Blind benchmark integrity layer

Артефакт:
- `reports/blind_benchmark_integrity_report.json`
- `harness/benchmark_runner.py`

Подтверждено:
- public blind cases сканируются на forbidden fields;
- `all_public_cases_clean = true`;
- `truth_access = false` для публичного benchmark-слоя.

Это ценно как независимый слой проверки экспериментальной чистоты.

---

## 14. Что именно ценно в системе

С точки зрения GitHub-позиционирования ценность системы состоит в следующем.

### 14.1. Это строгий BIP340 bridge

Система не просто “парсит подписи”, а реализует строгий membership bridge с диагностикой причин отказа.

### 14.2. Это affine nonce-forensics framework

Система переводит задачу анализа подписи в задачу анализа семейства скрытых нонсов:

`k_i(d') = s_i - e_i d' mod n`

Это даёт единый аналитический язык для BIP340 и MuSig2.

### 14.3. Это набор интерпретируемых функционалов, а не black-box скоринг

Каждая метрика имеет прозрачный смысл:
- коллизия;
- повтор дельт;
- shared prefix;
- локальная связность;
- parity-aware leaf structure.

### 14.4. Это protocol-valid MuSig2 linearization layer

Проект уже умеет корректно раскладывать частичную подпись MuSig2 в affine-форму

` s_i = k_eff,i + c_i d_i mod n `

с учётом key aggregation, nonce coefficient и parity factors.

### 14.5. Это disciplined data pipeline

Система уже содержит:
- dedup-public audit корпусы;
- sterile blind cases;
- truth manifest separation;
- benchmark integrity checks.

Это делает проект ценным не только как набор идей, но и как инфраструктуру для корректного криптографического R&D.

---

## 15. Краткая формула проекта

Самое компактное описание математической модели системы можно записать так.

### BIP340-слой

Для каждой строки:

- `e_i = H(r_i || P_x || m_i) mod n`
- `R_i^* = s_i G - e_i P`
- membership требует `R_i^* != O`, `y(R_i^*)` even, `x(R_i^*) = r_i`
- скрытый нонс для кандидата `d'`:
  `k_i(d') = s_i - e_i d' mod n`

### Forensics-слой

По семейству `K(d') = {k_i(d')}` вычисляются:

- compression functionals;
- prefix-support functionals;
- local-connectivity functionals.

### MuSig2-слой

Для строки partial signature из полного session context вычисляются:

- `k_eff,i = k1_eff,i + b_i k2_eff,i mod n`
- `c_i = e_i a_i g_i gacc_i mod n`
- `s_i = k_eff,i + c_i d_i mod n`
- скрытый эффективный нонс для кандидата `d'`:
  `k_eff,i(d') = s_i - c_i d' mod n`

То есть вся система строится вокруг одной общей идеи:

> подпись — это affine-ограничение, из которого можно восстановить семейство скрытых нонсов как функцию от кандидата секрета, а затем измерять его структуру.

---

## 16. Рекомендуемое GitHub-позиционирование

**Schnorr / MuSig2 Nonce-Forensics** is a research-grade affine analysis framework for BIP340 and protocol-valid MuSig2 partial signatures. It combines strict BIP340 membership validation, hidden-nonce family reconstruction, compression/connectivity metrics, protocol-valid MuSig2 linearization, and sterile benchmark data discipline.

**Schnorr / MuSig2 Nonce-Forensics** — это исследовательский affine-фреймворк для анализа BIP340-подписей и protocol-valid частичных подписей MuSig2, сочетающий строгий membership bridge, реконструкцию семейства скрытых нонсов, compression/connectivity-метрики, MuSig2-линеаризацию и стерильную benchmark-дисциплину.

---

## 17. Минимальный список ключевых артефактов

- `scripts/mainnet_taproot_test56.py`
- `scripts/family_compression_control.py`
- `scripts/family_connectivity_scanner.py`
- `scripts/bip327_partial_sig_to_candidate_jsonl.py`
- `scripts/poisoned_bip327_protocol_valid.py`
- `scripts/generate_musig2_blind_recovery_corpus.py`
- `harness/benchmark_runner.py`
- `reports/bip340_official_test5_report.json`
- `reports/mainnet_taproot_test56_report.json`
- `reports/mainnet_taproot_holdout_exact_report.json`
- `reports/bip340_positive_controls_summary.json`
- `reports/bip327_protocol_valid_controls_summary.json`
- `reports/bip327_partial_sig_adapter_report_dedup_public.json`
- `reports/blind_case_001_generation_report.json`
- `reports/blind_benchmark_integrity_report.json`

---

## 18. Итог

Подтверждённое ядро системы состоит не в одном solver-е и не в одном красивом ранге. Оно состоит в следующем:

1. строгая BIP340 membership-геометрия;
2. affine-реконструкция скрытых нонсов как функции от кандидата секрета;
3. интерпретируемые compression/connectivity-функционалы;
4. protocol-valid MuSig2 affine-линеаризация;
5. dedup-aware и sterile data discipline.

Именно это и является математической моделью системы в её текущем, доказанном и ценном состоянии.
