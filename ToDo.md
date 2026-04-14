- cron который проверяет новую версию xray/hysteria/awg и шлёт уведомление, но не обновляет автоматически. Например, скрипт который сравнивает текущую версию с latest release на GitHub и пишет в лог/отправляет в Telegram.
- обновить версию Hysteria до самой свежей
- убрать полностью возможность подключаться не к 2hop схеме (артефакт -- xray_client_connect_link.txt, hysteria2_client_connect_link.txt убрать)
- баг:
vpnuser@hmpeisqism:~$ sudo /opt/xray/xray run -c /opt/xray/config.json
Xray 26.3.27 (Xray, Penetrates Everything.) d2758a0 (go1.26.1 linux/amd64)
A unified platform for anti-censorship.
2026/04/10 11:36:08.116827 [Info] infra/conf/serial: Reading config: &{Name:/opt/xray/config.json Format:json}
Failed to start: main: failed to load config files: [/opt/xray/config.json] > infra/conf: failed to build routing configuration > infra/conf: invalid field rule > infra/conf: failed to parse domain rule: geosite:category-ru-blocked > infra/conf: failed to load geosite: CATEGORY-RU-BLOCKED > infra/conf: code not found in geosite.dat: CATEGORY-RU-BLOCKED

из-за этого пришлось убрать direct проброс этой категории через мост на relay

- баг:
The feature gRPC transport (with unnecessary costs, etc.) is deprecated, not recommended for using and might be removed. Please migrate to XHTTP stream-up H2 as soon as possible.

- баг:
Когда в первый раз запускаешь плейбук, такая ошибка

TASK [xray_relay : Obtain Let's Encrypt certificate for relay domain] **************************************************
[ERROR]: Task failed: Module failed: non-zero return code
Origin: /home/h4zzkr/projects/bifrost/roles/xray_relay/tasks/nginx_relay.yaml:88:3

86     enabled: true
87
88 - name: Obtain Let's Encrypt certificate for relay domain
     ^ column 3

fatal: [relay-vps]: FAILED! => {"changed": true, "cmd": ["certbot", "certonly", "--webroot", "-w", "/var/www/blah.blah.ru", "-d", "blah.blah.ru", "-m", "googlodict@gmail.com", "--agree-tos", "--non-interactive"], "delta": "0:00:03.333472", "end": "2026-04-10 11:21:24.580488", "msg": "non-zero return code", "rc": 1, "start": "2026-04-10 11:21:21.247016", "stderr": "Saving debug log to /var/log/letsencrypt/letsencrypt.log\nAn unexpected error occurred:\nNo such authorization\nAsk for help or search for solutions at https://community.letsencrypt.org. See the logfile /var/log/letsencrypt/letsencrypt.log or re-run Certbot with -v for more details.", "stderr_lines": ["Saving debug log to /var/log/letsencrypt/letsencrypt.log", "An unexpected error occurred:", "No such authorization", "Ask for help or search for solutions at https://community.letsencrypt.org. See the logfile /var/log/letsencrypt/letsencrypt.log or re-run Certbot with -v for more details."], "stdout": "Account registered.\nRequesting a certificate for blah.blah.ru", "stdout_lines": ["Account registered.", "Requesting a certificate for blah.blah.ru"]}

Когда во второй раз запускаешь, всё ок



Что лучше сделать: трехзвенную схему (но на самом деле там будет 1 русский 1 загран сервер, просто у заграна будет ещё 1 ip, как писали в комментах) + traffic padding или 