#!/bin/bash
set -e -o pipefail

# borrowed from
# http://stackoverflow.com/questions/17615881/simplest-method-to-convert-file-size-with-suffix-to-bytes/17616108#17616108
parseSize() {(
    local SUFFIXES=('' K M G T P E Z Y)
    local MULTIPLIER=1

    shopt -s nocasematch

    for SUFFIX in "${SUFFIXES[@]}"; do
        local REGEX="^([0-9]+)(${SUFFIX}i?B?)?\$"

        if [[ $1 =~ $REGEX ]]; then
            echo $((${BASH_REMATCH[1]} * MULTIPLIER))
            return 0
        fi

        ((MULTIPLIER *= 1024))
    done

    echo "$0: invalid size \`$1'" >&2
    return 1
)}

output() {
    echo "${*}"
}

log() {
    level=${1}
    shift
    MSG="${*}"
    timestamp=$(date +%Y%m%d_%H%M%S)
    case ${level} in
        "info")
            if ! ${QUIET}; then
                output "${timestamp} I: ${MSG}"
            fi
            ;;
        "warn")
            output "${timestamp} W: ${MSG}"
            ;;
        "error")
            output "${timestamp} E: ${MSG}"
            ;;
        "debug")
            output "${timestamp} D: ${MSG}"
            ;;
    esac
}

abort() {
    log "error" "${*}"
    exit 1
}

suspend_domain() {
    DOMAIN=${1}
    STATE=$(get_domain_state ${DOMAIN})
    if [ "${STATE}" == "running" ]; then
        # suspend the VM? (sync, pm-suspend)
        log "info" "Requesting ${DOMAIN} flush data to disk ..."
        ssh ${DOMAIN} sync
        # ${VIRSH} dompmsuspend ${DOMAIN} --target mem # requires QEMU agent
        log "info" "Suspending ${DOMAIN} ..."
        ${VIRSH} suspend ${DOMAIN} > /dev/null
    fi
}

resume_domain() {
    DOMAIN=${1}
    STATE=$(get_domain_state ${DOMAIN})
    if [ "${STATE}" == "paused" ]; then
        # resume the VM?
        # ${VIRSH} dompmwakeup ${DOMAIN} # requires QEMU agent
        log "info" "Resuming ${DOMAIN} ..."
        ${VIRSH} resume ${DOMAIN} > /dev/null
        # give some time to ensure the domain has resumed?
        sleep 5
    fi
}

get_domain_state() {
    DOMAIN=${1}
    echo $(${VIRSH} dominfo ${DOMAIN} | awk '/State:/ {print $2}')
}

get_domain_volumes() {
    DOMAIN=${1}
    VOLUMES=$(${VIRSH} dumpxml ${DOMAIN} | \
              grep "protocol='rbd" | \
              sed -r 's/.*name=.(.*)../\1/'
             )
    echo ${VOLUMES}
}

snapshot_domain_volumes() {
    for VOLUME in ${VOLUMES}; do
        SNAPSHOT=${VOLUME}@${MARKER}
        log "info" "Creating snapshot ${SNAPSHOT} ..."
        # check to see if a snapshot already exists
        if  rbd ls -l vms | grep -q ${SNAPSHOT#vms/}; then
            log "info" "Existing snapshot for ${SNAPSHOT} found."
        else
            # snapshot the volume -- rbd snap create rbd/foo@snapname
            rbd snap create ${SNAPSHOT}
        fi
    done
}

get_snapshot_size() {
    SNAPSHOT=${1}
    echo $(rbd ls -l vms | grep ${SNAPSHOT#vms/} | awk '{print $2}')
}

prune_backups() {
    PATTERN=${1}
    KEPT=0
    # list the provided pattern newest first
    BACKUPS=$(ls -1 --reverse ${PATTERN} 2>/dev/null || true)
    for BACKUP in ${BACKUPS}; do
        if [ ${KEPT} -lt ${MAX_BACKUPS} ]; then
            log "info" "Keeping ${BACKUP} ..."
            KEPT=$((${KEPT} + 1))
        else
            log "info" "Removing ${BACKUP} ..."
            rm -f ${BACKUP} ${BACKUP}.done
        fi
    done
}

backup_domain() {
    DOMAIN=${1}

    NEEDED=false

    EXT="img.lz4"
    VOLUMES=$(get_domain_volumes ${DOMAIN})
    log "info" "Found volumes: ${VOLUMES}"
    for VOLUME in ${VOLUMES}; do
        SNAPSHOT=${VOLUME}@${MARKER}
        DST="${BACKUP_LOCATION}/${SNAPSHOT}.${EXT}"
        if [ -f ${DST}.done ] && [ -f ${DST} ]; then
            log "info" "Existing backup for ${VOLUME} found."
        else
            NEEDED=true
        fi
    done

    if ${NEEDED}; then
        suspend_domain ${DOMAIN}
        snapshot_domain_volumes ${VOLUMES}
        resume_domain ${DOMAIN}
        for VOLUME in ${VOLUMES}; do
            SNAPSHOT=${VOLUME}@${MARKER}
            SNAPSHOT_SIZE=$(get_snapshot_size ${SNAPSHOT})
            DST="${BACKUP_LOCATION}/${SNAPSHOT}.${EXT}"
            if ${QUIET}; then
                PV='cat'
            else
                PV="pv --eta --progress --rate --bytes --size ${SNAPSHOT_SIZE}"
            fi
            if [ "$(parseSize ${SNAPSHOT_SIZE})" -lt ${BACKUP_LIMIT} ]; then
                NEEDED=true
                log "info" "Backing up ${VOLUME} ..."
                # map the snapshot
    #                 MAPPED_DEV=$(rbd map ${SNAPSHOT})
    #                 ddrescue --sparse ${MAPPED_DEV} ${DST}
    #                 rbd unmap ${MAPPED_DEV}

                # export the snapshot
                rbd export --no-progress ${SNAPSHOT} - | \
                    ${PV} | \
                    lz4 > ${DST}
            else
                log "info" "Skipping ${VOLUME}, it exceeds the backup limit."
            fi
            touch ${DST}.done
            prune_backups "${BACKUP_LOCATION}/${VOLUME}@*.${EXT}"

            # flatten the volume? -- rbd snap rm
            # {pool-name}/{image-name}@{snap-name}
            log "info" "Removing snapshot ${SNAPSHOT} ..."
            rbd snap rm ${VOLUME}@${MARKER}
        done
        backup_domain_config ${DOMAIN}
    fi
}

backup_domains() {
    DOMAINS="${*}"

    for DOMAIN in ${DOMAINS}; do
        log "info" "Working with ${DOMAIN} ..."
        backup_domain ${DOMAIN}
    done
}

backup_domain_config() {
    DOMAIN=${1}
    DST="${BACKUP_LOCATION}/vms/${DOMAIN}@${MARKER}.xml"
    log "info" "Dumping configuration for ${DOMAIN} ..."
    virsh dumpxml ${DOMAIN} > ${DST}
    prune_backups "${BACKUP_LOCATION}/vms/${DOMAIN}@*.xml"
}

get_duration() {
    START=${1}
    END=${2}

    DUR=$((${END} - ${START}))
    UNITS="secs"
    if [ ${DUR} -gt 3600 ]; then
        DUR=$(echo ${DUR} / 3600 | bc -l)
        UNITS="hours"
    elif [ ${DUR} -gt 600 ]; then
        DUR=$(echo ${DUR} / 60.0 | bc -l)
        UNITS="mins"
    fi
    echo "${DUR} ${UNITS}"
}

usage(){
    output ""
    output "USAGE: ${0} [options] DOMAINS..."
    output ""
    output "   -d LOC      where to store the backups"
    output "   -h          this usage information"
    output "   -l SIZE     maximum size of a volume to backup"
    output "   -m MAX      maximum number of backups to keep"
    output "   -n NICE     nice level for the backup processes"
    output ""
    exit 0
}

# =====================================================================

# CONSTANTS
MARKER=$(date +%Y%m%d)
VIRSH='virsh'

# DEFAULTS
MAX_BACKUPS=2
BACKUP_LIMIT=$(parseSize 1T)
BACKUP_LOCATION="/opt/backups"
QUIET=false


while getopts ":hm:l:d:n:q" opt; do
    case "${opt}" in
        h)
            usage
            ;;
        m)
            MAX_BACKUPS=${OPTARG}
            ;;
        l)
            BACKUP_LIMIT=$(parseSize ${OPTARG})
            ;;
        d)
            BACKUP_LOCATION=${OPTARG}
            ;;
        n)
            NICE="nice -n ${OPTARG}"
            ;;
        q)
            QUIET=true
            ;;
        *)
            abort "Unsupported option {${opt}}"
            ;;
    esac
done
shift $((OPTIND-1))


START=$(date +%s)
backup_domains "${*}"
END=$(date +%s)

DUR=$(get_duration ${START} ${END})
log "info" "Total backup time: ${DUR}"
