# Weather Edge Hunter v4.3 — Resolution-Grade

Weather Edge Hunter е browser-based decision-support инструмент за анализ на Polymarket weather markets. Версия **v4.3 Resolution-Grade** е фокусирана върху най-важния проблем при температурните пазари: да не смесва прогнозни/proxy данни с официалните данни, по които реално се извършва резолюцията.

> **Важно:** Инструментът е за research и decision-support. Не гарантира печалба, не изпълнява автоматични сделки и не трябва да се използва като единствен източник за финансово решение.

## Какво е новото във v4.3

След реални тестове с London, Madrid, Guangzhou, Paris, Singapore, Manila и Milan беше установено, че точните 1°C buckets са много чувствителни към разлика между градска прогноза, моделна grid температура и действителното измерване на конкретната resolution station.

v4.3 въвежда следните защити:

- **Verified station observations са отделени от forecast/proxy данните.**
- **Open-Meteo не се третира като официална resolution истина.** Използва се само за forecast/projection.
- **LIVE BET се блокира без verified station feed.**
- **Resolution mode не приема proxy данни като официален settlement result.**
- **Whole-degree discrete bucket probabilities** вместо решение само по mean temperature.
- Ensemble members се преобразуват до температурни buckets и се изчислява реално `P(bucket)`.
- Показва се **Top bucket + probability**.
- Exact bucket market има **60% model-probability hard floor**.
- Default minimum executable edge: **12 percentage points**.
- Default minimum confidence: **75%**.
- Default maximum resolution risk: **20%**.
- Log export записва bucket distribution, top bucket и verified-station status.
- Dynamic Polymarket/NOAA station discovery остава част от системата.

## Основна логика

```text
Polymarket market
      ↓
Parse rules / target date / station / source / units
      ↓
Verified station observation layer
      +
Forecast ensemble layer
      ↓
Projected final daily high
      ↓
Discrete whole-degree bucket distribution
      ↓
P(bucket)
      ↓
Polymarket executable price
      ↓
Model edge
      ↓
Resolution adjustment
      ↓
Fees / slippage
      ↓
Executable edge
      ↓
Resolution Guard + Risk filters
      ↓
ENTER / WAIT / PASS
```

## Resolution-first принцип

Основният принцип на проекта е:

```text
Resolution first.
Forecast second.
Edge third.
Trade last.
```

Преди анализ на вероятност инструментът трябва да определи:

1. точната дата на пазара;
2. локалната timezone на station/market;
3. exact ICAO / NOAA `site=` station;
4. official resolution source;
5. температурната единица;
6. bucket / threshold логиката;
7. дали live data е официално/verified или само proxy;
8. дали има clarification / fallback resolution rule.

## Data layers

### 1. Official / verified station observations

Предназначение:

- текущи station observations;
- live high-so-far;
- validation преди LIVE decision;
- resolution evidence само когато exact official settlement source е наличен.

**Proxy данни не се маркират като verified observations.**

### 2. Forecast ensemble

Използва ensemble температурни модели за вероятностна оценка на дневния максимум.

Текущият модел layer включва:

- ECMWF IFS
- NCEP GEFS
- ICON Seamless

Forecast layer е отделен от official observation/resolution layer.

### 3. Discrete bucket model

При Polymarket weather markets не е достатъчно да знаем, че mean high е например `24.9°C`.

За exact markets трябва да оценим:

```text
P(23°C)
P(24°C)
P(25°C)
P(26°C)
P(27°C)
...
```

v4.3 преобразува ensemble members към whole-degree buckets и използва bucket probability при edge calculation.

## Resolution Guard

Resolution Guard блокира или понижава confidence при:

- непарсната target date;
- липсваща exact station;
- липсващ или неясен resolution source;
- proxy feed вместо verified station data;
- твърде висок resolution risk;
- stale data;
- clarification/additional-context risk;
- insufficient exact-bucket probability;
- слаб executable edge;
- прекален spread;
- ниска liquidity;
- session exposure / max positions ограничения.

## Default risk settings

```text
Minimum executable edge: 12 pp
Minimum confidence:       75%
Maximum spread:            5%
Minimum liquidity:       $1000
Maximum positions:          4
Default test stake:        $5
Maximum session exposure: $20
Maximum resolution risk:  20%
Slippage buffer:            1%
```

Тези стойности са research defaults, а не обещание за оптимален sizing.

## Date modes

Weather Edge Hunter класифицира market-а спрямо **локалната дата на station**, не спрямо часовника в България.

- `YESTERDAY / RESOLUTION`
- `TODAY / LIVE`
- `TOMORROW+ / FORECAST`

Това е критично при пазари в Азия и САЩ.

## Dynamic station database

Системата не разчита само на ръчно поддържан списък от градове.

При discovery тя се опитва да:

1. намери активните Polymarket weather markets;
2. прочете resolution rules;
3. извлече NOAA/ICAO `site=XXXX`;
4. свърже station metadata;
5. използва точните координати и timezone;
6. добави station към session registry.

Предварително известни stations могат да се използват като fallback, но exact market rule има приоритет.

## Интерфейс

Основни бутони:

- **SCAN NOW** — търси и анализира активни weather markets;
- **40 MIN MONITOR** — периодичен monitoring режим;
- **STOP** — спира monitoring;
- **API SELF-TEST** — проверява основните външни endpoints;
- **REFRESH STATIONS** — обновява station/forecast данните;
- **EXPORT LOG** — export на диагностичен JSON log.

## Препоръчителен workflow

При нов test session:

```text
1. API SELF-TEST
2. REFRESH STATIONS
3. SCAN NOW
4. Провери exact Polymarket rules
5. Провери station / source / date / unit
6. Провери дали station feed е VERIFIED
7. Провери bucket probability
8. Провери executable edge
9. ENTER / WAIT / PASS
10. Export log
11. След resolution: запис в Test & Track / Notion
```

## Test & Track

За всеки реален тест е препоръчително да се записват:

```text
Test ID
Agent version
Market
Station
Resolution source
Target date
Bucket / side
Entry price
Stake
Model probability
Executable edge
Market movement
Official daily maximum
Winning bucket
Final Polymarket resolution
P/L
Classification
Lesson
```

Препоръчителни final classifications:

- `Correct model + real edge`
- `Correct forecast, poor entry price`
- `Wrong forecast`
- `Resolution/source problem`
- `Data-quality failure`
- `Risk-management failure`

## Known limitations

v4.3 все още не е пълна institutional-grade система.

Остават важни задачи:

- надеждно browser/server ingestion на exact official NOAA/WRH resolution data;
- по-добър METAR/official station transport без CORS проблеми;
- station-specific bias correction;
- calibration по historical resolved Polymarket markets;
- realized-edge dashboard;
- order-book VWAP/depth вместо само basic price/spread;
- automated resolution ingestion;
- unified portfolio/correlation risk manager;
- persistent database вместо само browser/local export;
- systematic backtesting по city/station/season/weather regime.

## Security

Никога не качвайте в публичен GitHub:

- private keys;
- wallet seed phrases;
- exchange credentials;
- API secrets;
- Gemini/OpenAI/други private API keys.

Ако се използва Gemini key в интерфейса, той трябва да бъде временен и да не се commit-ва в repository.

## GitHub Pages deployment

Минимална структура:

```text
/
├── index.html
├── README.md
└── LICENSE
```

Стъпки:

1. Качи v4.3 HTML файла като `index.html` в root на repository.
2. Качи `README.md` и `LICENSE`.
3. GitHub → Settings → Pages.
4. Source: main branch / root.
5. Отвори GitHub Pages URL.
6. Направи hard refresh (`Ctrl+F5`).
7. Пусни `API SELF-TEST` преди реален тест.

## Version

Current release:

```text
Weather Edge Hunter v4.3
Resolution-Grade
```

## License

MIT License — виж файла `LICENSE`.

## Disclaimer

Този проект е експериментален research / decision-support software. Prediction markets носят риск от загуба на целия заложен капитал. Forecast data, station observations, market prices и resolution sources могат да бъдат непълни, закъснели, променени или недостъпни. Винаги проверявайте официалните Polymarket market rules и exact resolution source преди реална позиция.
