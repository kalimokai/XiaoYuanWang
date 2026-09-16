#!/bin/sh
# eportal.sh - 校园网 eportal 自动认证脚本（动态 queryString 版）
# 适用 OpenWrt
# 用法:
#   /root/eportal.sh          单次检测，离线则登录
#   /root/eportal.sh --login  强制登录一次
#   /root/eportal.sh --daemon 守护模式，循环检测

# ============ 用户配置 ============
USER='账号'
PASS='密码'
SERVICE='服务类型'
# ============ 用户配置 ============

PORTAL='http://10.1.1.185/eportal/InterFace.do?method=login'
CHECK_URL='http://connect.rom.miui.com/generate_204'
CHECK_INTERVAL=10
LOG='/tmp/eportal.log'
LOCKDIR='/tmp/eportal.lock.d'

log() {
    echo "$(date '+%F %T') $*" >> "$LOG"
}

is_online() {
    code=$(curl -s -m 5 -o /dev/null -w '%{http_code}' "$CHECK_URL" 2>/dev/null)
    [ "$code" = "204" ]
}

get_query() {
    hdr=$(mktemp)
    body=$(mktemp)

    curl -s -m 8 -D "$hdr" -o "$body" "$CHECK_URL" 2>/dev/null

    loc=$(grep -i '^Location:' "$hdr" | sed 's/^[Ll]ocation: //' | tr -d '\r')
    q=$(printf '%s' "$loc" | sed -n 's/.*[?&]queryString=\([^&]*\).*/\1/p')

    if [ -z "$q" ]; then
        final=$(curl -s -L -m 8 -o /dev/null -w '%{url_effective}' "$CHECK_URL" 2>/dev/null)
        q=$(printf '%s' "$final" | sed -n 's/.*[?&]queryString=\([^&]*\).*/\1/p')
    fi

    if [ -z "$q" ]; then
        q=$(sed -n "s/.*queryString=\([^\"'&<> ]*\).*/\1/p" "$body" | head -n1)
    fi

    rm -f "$hdr" "$body"
    printf '%s' "$q"
}

acquire_lock() {
    if mkdir "$LOCKDIR" 2>/dev/null; then
        trap 'rmdir "$LOCKDIR" 2>/dev/null' EXIT INT TERM
        return 0
    fi
    return 1
}

do_login() {
    q=$(get_query)

    if [ -z "$q" ]; then
        log "cannot get queryString, abort"
        return 1
    fi

    log "queryString(encoded once): $q"

    resp=$(curl -s -m 10 "$PORTAL" \
        -H 'Connection: close' \
        -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' \
        --data 'action=login' \
        --data-urlencode "userId=$USER" \
        --data-urlencode "password=$PASS" \
        --data-urlencode "service=$SERVICE" \
        --data-urlencode "queryString=$q" \
        --data 'operatorPwd=' \
        --data 'operatorUserId=' \
        --data 'validcode=' \
        --data 'passwordEncrypt=false' 2>&1)

    log "resp: $resp"

    if echo "$resp" | grep -q '"result":"success"'; then
        log "login success"
        return 0
    fi

    log "login fail or unknown response"
    return 1
}

single_run() {
    if is_online; then
        log "online, skip"
        return 0
    fi

    if ! acquire_lock; then
        log "another instance running, skip"
        return 1
    fi

    log "offline, try login"
    do_login
    return $?
}

force_login() {
    if ! acquire_lock; then
        log "another instance running, skip"
        return 1
    fi

    log "force login"
    do_login
    return $?
}

daemon_loop() {
    while :; do
        single_run
        sleep "$CHECK_INTERVAL"
    done
}

# ============ 入口 ============

case "$1" in
    --daemon|-d)
        daemon_loop
        ;;
    --login|-l)
        force_login
        ;;
    *)
        single_run
        ;;
esac
