# Disaster-recovery-Keepalived
задание 1
<img width="1920" height="984" alt="image" src="https://github.com/user-attachments/assets/80e8b2a6-212e-4985-89fa-b1fc3e176ea5" />
<img width="702" height="712" alt="2026-09-20_15-45-42" src="https://github.com/user-attachments/assets/c85892c5-7fd3-42a2-bf7a-b1d7aaca74e4" />
задание 2

! /etc/keepalived/keepalived.conf   —   УЗЕЛ 2 (BACKUP), 10.129.0.25

global_defs {
    router_id web-02
    enable_script_security
    script_user root
    vrrp_garp_master_refresh 5
}

vrrp_script check_web {
    script   "/etc/keepalived/check_web.sh"
    interval 3
    timeout  2
    fall     2
    rise     2
}

vrrp_instance VI_1 {
    state           BACKUP
    interface       eth0
    virtual_router_id 51            
    priority        100             
    advert_int      1

    authentication {
        auth_type PASS
        auth_pass Str0ngVRRP        
    }

    unicast_src_ip 10.129.0.25
    unicast_peer {
        10.129.0.21
    }

    virtual_ipaddress {
        10.129.0.100/24 dev eth0
    }

    track_script {
        check_web
    }

    notify_master "/usr/bin/logger -t keepalived 'STATE -> MASTER, VIP поднят'"
    notify_backup "/usr/bin/logger -t keepalived 'STATE -> BACKUP, VIP снят'"
    notify_fault  "/usr/bin/logger -t keepalived 'STATE -> FAULT, health-check провален'"
}


[Uploading keepalived! /etc/keepalived/keepalived.conf   —   УЗЕЛ 1 (MASTER), 10.129.0.21

global_defs {
    router_id web-01
    enable_script_security          
    script_user root
    vrrp_garp_master_refresh 5      
}

vrrp_script check_web {
    script   "/etc/keepalived/check_web.sh"
    interval 3          
    timeout  2          
    fall     2          
    rise     2          
}

vrrp_instance VI_1 {
    state           MASTER
    interface       eth0
    virtual_router_id 51           
    priority        150             
    advert_int      1
    ! nopreempt                     
    authentication {
        auth_type PASS
        auth_pass Str0ngVRRP
    }

   
    unicast_src_ip 10.129.0.21
    unicast_peer {
        10.129.0.25
    }

    virtual_ipaddress {
        10.129.0.100/24 dev eth0
    }

    track_script {
        check_web
    }

    notify_master "/usr/bin/logger -t keepalived 'STATE -> MASTER, VIP поднят'"
    notify_backup "/usr/bin/logger -t keepalived 'STATE -> BACKUP, VIP снят'"
    notify_fault  "/usr/bin/logger -t keepalived 'STATE -> FAULT, health-check провален'"
}
-master.conf…]()


[check_web.sh](https://github.com/user-attachments/files/32441580/check_web.sh)#!/usr/bin/env bash


set -u

PORT="${PORT:-80}"
DOCROOT="${DOCROOT:-/var/www/html}"
INDEX="${DOCROOT}/index.html"
HOST="${HOST:-127.0.0.1}"
TAG="check_web"

log() {
    logger -t "$TAG" -p daemon.warning -- "$1" 2>/dev/null
}


port_is_open() {
    if command -v ss >/dev/null 2>&1; then
        ss -H -ltn "sport = :${PORT}" 2>/dev/null | grep -q . && return 0
        return 1
    fi
    (exec 3<>"/dev/tcp/${HOST}/${PORT}") 2>/dev/null || return 1
    exec 3<&- 3>&-
    return 0
}

# --- 1. Порт веб-сервера доступен? -------------------------------------------
if ! port_is_open; then
    log "FAIL: порт ${PORT} недоступен"
    exit 1
fi

# --- 2. index.html существует и читаем? --------------------------------------
if [ ! -f "$INDEX" ] || [ ! -r "$INDEX" ]; then
    log "FAIL: файл ${INDEX} отсутствует или недоступен для чтения"
    exit 1
fi

# --- 3. Сервер реально отдаёт index.html? ------------------------------------
if ! curl -fsS --max-time 2 -o /dev/null "http://${HOST}:${PORT}/index.html"; then
    log "FAIL: http://${HOST}:${PORT}/index.html не отдаёт код 2xx"
    exit 1
fi

exit 0

