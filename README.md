# MMDB Merger

YAML-конфигурируемый инструмент для сборки и объединения MMDB баз (GeoLite2-City + GeoLite2-ASN) в единую оптимизированную базу с фильтрацией локалей (en/ru/local), валидацией, отчётами и Docker-интеграцией.

## Основные возможности
- Объединение GeoLite2-City и GeoLite2-ASN в единый `ip2geo.mmdb`.
- Конфигурация через YAML: источники, стратегии merge, фильтры, локали, валидация, отчёты.
- Фильтрация локалей: сохранение только `en`, `ru` и вычисляемого `local` с правилами по странам.
- Гибкие стратегии: `deep_merge`, `overlay`, `union`; конфликт-резолв: `merge`, `prefer_first`, `prefer_priority`.
- Валидация результата: обязательные поля, покрытие локалей/ASN, тестовые IP.
- Репорты: summary (JSON), statistics (HTML), locale_coverage (CSV).
- Docker и docker-compose для повторяемых сборок.

## Быстрый старт
```bash
# Требования
python >= 3.11

# Клонирование
git clone https://github.com/pv-udpv/mmdb-merger
cd mmdb-merger

# Установка зависимостей
pip install -r requirements.txt

# Запуск с дефолтным конфигом
python -m geoip_builder.cli merge-mmdb --config ./config/mmdb_merge.yaml
```

## Конфигурация
Пример `config/mmdb_merge.yaml`:
```yaml
version: "1.0"
name: "IP2Geo MMDB Merger"
description: "Merge GeoLite2-City and GeoLite2-ASN into unified ip2geo.mmdb"

sources:
  - id: "geolite2_city"
    name: "GeoLite2 City Database"
    url: "https://github.com/P3TERX/GeoLite.mmdb/raw/download/GeoLite2-City.mmdb"
    type: "mmdb"
    priority: 10
  - id: "geolite2_asn"
    name: "GeoLite2 ASN Database"
    url: "https://github.com/P3TERX/GeoLite.mmdb/raw/download/GeoLite2-ASN.mmdb"
    type: "mmdb"
    priority: 10

output:
  path: "./output/ip2geo.mmdb"
  database_type: "IP2Geo-City-ASN"
  description:
    en: "Unified GeoIP database with City and ASN data"
    ru: "Единая GeoIP база с данными о городах и ASN"
  record_size: 28
  ip_version: 6
  build_epoch: auto
  languages: [en, ru]

merge:
  strategy: "deep_merge"
  conflict_resolution: "merge"
  fields:
    - source: "geolite2_city"
      extract: [city, subdivisions, country, continent, location, postal, registered_country, represented_country, traits]
    - source: "geolite2_asn"
      extract: [autonomous_system_number, autonomous_system_organization]
      rename:
        autonomous_system_number: "asn.number"
        autonomous_system_organization: "asn.organization"

filters:
  locales:
    enabled: true
    keep: ["en", "ru", "local"]
    local_rules:
      RU: "ru"
      CN: "zh-CN"
      JP: "ja"
      AE: "ar"
      SA: "ar"
      default: "en"
  remove_empty: true
  remove_deprecated: true
  deduplicate_names: true

optimizations:
  compress_names: true
  merge_subdivisions: true
  merge_adjacent_networks: false

validation:
  enabled: true
  checks:
    - name: "Required fields"
      type: "required_fields"
      fields: ["country"]
    - name: "Locale completeness"
      type: "locale_check"
      min_coverage: 0.95
    - name: "ASN data"
      type: "field_presence"
      field: "asn.number"
      min_coverage: 0.80
  test_ips:
    - ip: "8.8.8.8"
      expected: { country_code: "US", asn: 15169, city_en: "Mountain View" }
    - ip: "77.88.55.80"
      expected: { country_code: "RU", asn: 13238, city_ru: "Москва" }
    - ip: "185.107.252.11"
      expected: { country_code: "RU", city_ru: "Щёлкино" }

reporting:
  enabled: true
  generate:
    - { type: "summary", format: "json", path: "./reports/merge_summary.json" }
    - { type: "statistics", format: "html", path: "./reports/merge_stats.html" }
    - { type: "locale_coverage", format: "csv", path: "./reports/locale_coverage.csv" }
  metrics: [total_networks, total_ips_covered, unique_countries, unique_cities, asn_coverage_percent, locale_coverage_percent, file_size_mb, build_time_seconds]
```

## Установка
```bash
pip install --no-cache-dir \
  maxminddb \
  mmdbwriter \
  pyyaml \
  requests \
  click \
  rich
```

## CLI
```bash
# Слияние MMDB файлов
geoip-builder merge-mmdb --config ./config/mmdb_merge.yaml
```

## Верификация результата
```python
import maxminddb
reader = maxminddb.open_database('./output/ip2geo.mmdb')
for ip in ['8.8.8.8','77.88.55.80','185.107.252.11']:
    r = reader.get(ip)
    print(ip, r.get('country', {}).get('iso_code'), r.get('asn',{}).get('number'))
reader.close()
```

## Docker
`docker/mmdb-merger/Dockerfile`:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir \
    maxminddb \
    mmdbwriter \
    pyyaml \
    requests \
    click \
    rich
COPY geoip_builder/ ./geoip_builder/
COPY config/ ./config/
ENTRYPOINT ["python", "-m", "geoip_builder.cli", "merge-mmdb"]
```

`docker-compose.yaml`:
```yaml
services:
  mmdb-merger:
    build: ./docker/mmdb-merger
    volumes:
      - ./config:/app/config
      - ./output:/app/output
      - ./cache:/app/cache
      - ./reports:/app/reports
    command: ["--config", "/app/config/mmdb_merge.yaml"]
```

Запуск:
```bash
docker-compose run mmdb-merger
```

## Валидация и отчёты
- Проверка тестовых IP из конфига (country_code, ASN, города).
- `./reports/merge_summary.json` — краткая сводка сборки.
- `./reports/merge_stats.html` — HTML-статистика метрик.
- `./reports/locale_coverage.csv` — покрытие локалей по записям.

## Дорожная карта
- Поддержка дополнительных источников (IP2Location, DB-IP).
- Слияние смежных префиксов с одинаковыми данными.
- Кастомные overrides по странам/городам.
- Публикация пакета в PyPI и Docker Hub.

## Лицензия
MIT
