# Приклади · Examples

Реальні повідомлення SolarVault 3 Pro Max AC (24.11.2026). Серійний номер, токен, назву Wi‑Fi, IP- і MAC-адреси замінено.
Real SolarVault 3 Pro Max AC messages (24 Nov 2026). Serial number, token, Wi‑Fi name, IP and MAC addresses are replaced.

Кожен файл / Each file: `{"topic": "…", "payload": {…}}`.

| Файл / File | Що це / What |
| --- | --- |
| [`request-type25.json`](request-type25.json) | Запит стану станції / Station state request |
| [`status-type2.json`](status-type2.json) | Відповідь на 25 / Reply to 25 |
| [`request-type105.json`](request-type105.json) | Запит стану системи / System state request |
| [`event-type106.json`](event-type106.json) | Відповідь на 105 / Reply to 105 |
| [`request-type100-plugs.json`](request-type100-plugs.json) | Запит списку розеток / Smart plug list request |
| [`event-type101-plugs.json`](event-type101-plugs.json) | Відповідь: розеток немає / Reply: no plugs |
| [`event-type101-cts.json`](event-type101-cts.json) | Відповідь: лічильників немає / Reply: no meters |
| [`command-discharge-limit.json`](command-discharge-limit.json) | Команда: ліміт розряду 15 % / Command: discharge limit 15 % |
| [`event-type123-wrong-token.json`](event-type123-wrong-token.json) | Неправильний токен / Wrong token |

Надіслати приклад / Send an example:

```bash
jq -c .payload request-type25.json | mosquitto_pub -h <host> -u <user> -P <pass> -t "hb/device/<SN>/action" -s
```

Не забудьте підставити свій токен / Remember to put in your own token.
