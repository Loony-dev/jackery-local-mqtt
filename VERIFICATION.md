# Журнал звірок · Verification log

Коли й з чим звіряли документацію. Нова звірка — новий рядок зверху.
When and against what the documentation was checked. Each new check adds a row at the top.

**Планово / Scheduled:** раз на квартал, а також після оновлення прошивки станції, застосунку Jackery чи офіційної інтеграції.
Once a quarter, and after station firmware, Jackery app or official integration updates.

**Наступна / Next:** лютий 2027 · February 2027

| Дата / Date | Станція / Station | Прошивка / Firmware | Застосунок Jackery / App | Офіційна інтеграція / Integration | Що звірено / What was checked |
| --- | --- | --- | --- | --- | --- |
| 2026-11-24 | SolarVault 3 Pro Max AC, без дод. батарей / no expansion | — ¹ | 2.x | v2.0.0 · `3cd17ff` (2026-11-13) | type 25→2, 105→106, 100→101 (без підпристроїв / no sub-devices), 123/401, команда / command `socDischgLimit`, `ongridStat` під час відключень / during outages, запити / requests type 2 і / and 22 без відповіді / unanswered |

¹ Станція не надсилає `softver` по MQTT. The station doesn't send `softver` over MQTT.

## Як звіряти · How to check

1. Запишіть трафік / Record traffic: `mosquitto_sub -h <host> -u <user> -P <pass> -t "hb/#" -v > capture.txt` (кілька хвилин, поки йде опитування / a few minutes while something polls the station).
2. Порівняйте поля з таблицями в README: нові, зниклі, інші одиниці / Compare fields with the README tables: new, missing, different units.
3. Перевірте зміни в [офіційній інтеграції](https://github.com/Jackery-Official/jackery/commits/main) з дати останньої звірки / Check the [official integration](https://github.com/Jackery-Official/jackery/commits/main) for changes since the last check.
4. Оновіть README обома мовами, дату вгорі та цей журнал / Update both READMEs, the date at the top and this log.
