
find / -name "*patroni*" -type f 2>/dev/null

Как посмотреть лист патрони
patronictl -c /etc/postgresql/patroni.yml list


patronictl -c  /etc/postgresql/patroni.yml  edit-config
Изменить конфиг
patronictl -c  /etc/postgresql/patroni.yml  show-config


