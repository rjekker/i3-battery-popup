#! /bin/bash

################################################################################
# A script that shows a battery warning on i3wm                                #
#                                                                              #
# It supports multiple batteries                                               #
# (like my thinkpad T450s has)                                                 #
#                                                                              #
# When tcl/tk (wish) is installed, it shows a nice popup                       #
# Which you can configure to show on all workspaces                            #
# by adding the following to your i3 config:                                   #
# "for_window [title="Battery Warning"] sticky enable"                         #
#                                                                              #
# By default, the script will show two messages:                               #
# One at 10% and one at 5% battery                                             #
#                                                                              #
# The script takes the following options:                                      #
# -L : The percentage at which the first popup shows (default: 10)             #
#                                                                              #
# -l : The percentage at which the second popup shows                          #
#      Default: half of the percentage given by -L                             #
#                                                                              #
# -m : The message to show to the User                                         #
#                                                                              #
# -t : The time interval the script waits before checking the battery again.   #
#      Give this a value in seconds: 5s, 10s, or in minutes: 5m                #
#      Default: 5m                                                             #
#                                                                              #
# -n : Use notify-send for message.                                            #
#                                                                              #
# -N : Don't use Tcl/Tk dialog. Use i3-nagbar.                                 #
#                                                                              #
# By R-J Ekker, 2016                                                           #
################################################################################

error () {
    echo $1 >&2
    echo "Exiting" >&2
    exit $2
}

while getopts 'L:l:m:t:s:F:i:nND' opt; do
    case $opt in
        L)
            [[ $OPTARG =~ ^[0-9]+$ ]] || error "${opt}: ${OPTARG} is not a number" 2
            UPPER_LIMIT="${OPTARG}"
            ;;
        l)
            [[ $OPTARG =~ ^[0-9]+$ ]] || error "${opt}: ${OPTARG} is not a number" 2
            LOWER_LIMIT="${OPTARG}"
            ;;
        m)
            MESSAGE="${OPTARG}"
            ;;
        n)
            USE_NOTIFY_SEND="y"
            ;;
        i)
            NOTIFY_ICON="${OPTARG}"
            ;;
        N)
            DONT_USE_WISH="-n"
            ;;
        t)
            [[ $OPTARG =~ ^[0-9]+[ms]?$ ]] || error "${opt}: ${OPTARG} is not a valid period" 2
            SLEEP_TIME="${OPTARG}"
            ;;
        D)
            # Print some extra info
            DEBUG="y"
            ;;
        F)
            # Redirect debugging info to logfile
            # if -D not specified this will log nothing
            LOGFILE="${OPTARG}"
            ;;
        :)
            error "Option -$OPTARG requires an argument." 2
            ;;
        \?)
            exit 2
            ;;
    esac
done

# This function returns an awk script
# Which prints the battery percentage
# It's an ugly way to include a nicely indented awk script here
get_awk_source() {
    cat <<EOF
BEGIN {
    FS="=";
}
\$1 ~ /ENERGY_FULL$/ {
    f += \$2;
}
\$1 ~ /ENERGY_NOW\$/ {
    n += \$2;
}
\$1 ~ /CHARGE_FULL$/ {
    f += \$2;
}
\$1 ~ /CHARGE_NOW\$/ {
    n += \$2;
}
END {
    print int(100*n/f);
}
EOF
}

is_battery_discharging() {
    cat $BATTERIES | grep STATUS=Discharging && return 0
    return 1
} >/dev/null

get_battery_perc() {
    cat $BATTERIES | awk -f <(get_awk_source)
}

show_popup() {
    WISH_SCRIPT="wm state . withdrawn; tk_messageBox -icon warning -title \"Battery Warning\" -message \"${1}\"; exit"
    echo "$WISH_SCRIPT" | wish
}

show_nagbar(){
    i3-msg "exec i3-nagbar -m \"${1}\""
}

show_notify(){
    GNOME_ICON="/usr/share/icons/gnome/scalable/status/battery-low-symbolic.svg"
    XFCE_ICON="/usr/share/icons/elementary-xfce/status/48/battery-low.png"
    # try to find nice notify icon
    if [[ -z $NOTIFY_ICON ]]; then
        if [[ -f $GNOME_ICON ]]; then
            NOTIFY_ICON="${GNOME_ICON}"
        elif [[ -f $XFCE_ICON ]]; then
            NOTIFY_ICON="${XFCE_ICON}"
        fi
    fi
    [[ -n $NOTIFY_ICON ]] && NOTIFY_OPT="-i ${NOTIFY_ICON}"
    notify-send -u critical ${NOTIFY_OPT} "${1}"
}

show_message(){
    if [[ -n $USE_NOTIFY_SEND ]] && which notify-send; then
        show_notify "$1"
    elif [[ -z $DONT_USE_WISH ]] && which wish; then
        show_popup "$1"
    else
        show_nagbar "$1"
    fi
} >&2

debug(){
    [[ -n $DEBUG ]] && echo "$1"
}

main (){
    # Setting defaults
    UPPER_LIMIT="${UPPER_LIMIT:-10}"
    UPPER_HALF=$(( $UPPER_LIMIT / 2 ))
    LOWER_LIMIT=${LOWER_LIMIT:-$UPPER_HALF}
    MESSAGE="${MESSAGE:-Warning: Battery is getting low}"
    SLEEP_TIME="${SLEEP_TIME:-5m}"

    BATTERIES=/sys/class/power_supply/BAT*/uevent

    debug "Upper ${UPPER_LIMIT}; Lower ${LOWER_LIMIT}; sleep ${SLEEP_TIME}"
    debug "Current: $(get_battery_perc)%"

    LIMIT="${UPPER_LIMIT}"
    # This will be set to "y" after first click
    # So we know when to stop nagging
    POPUP_CLICKED=""

    while true; do
        debug "Checking.. "

        PERC=$(get_battery_perc)
        debug "got ${PERC}%"

        if is_battery_discharging; then
            debug "Battery is discharging"

            if [[ $PERC -lt $LIMIT ]]; then
                debug "showing warning"
                show_message "${MESSAGE}"

                if [[ -z $POPUP_CLICKED ]]; then
                    # first click; set limit lower
                    POPUP_CLICKED="y"
                    LIMIT=${LOWER_LIMIT}
                else
                    # We clicked twice; No more popups
                    LIMIT=0
                fi
            fi
        else
            # restart messages, reset limits
            POPUP_CLICKED=""
            if [[ $PERC -gt $UPPER_LIMIT ]]; then
                LIMIT=${UPPER_LIMIT}
            else
                LIMIT=${LOWER_LIMIT}
            fi
        fi
        debug "sleeping ${SLEEP_TIME}; current limit ${LIMIT}%; ${POPUP_CLICKED:+Popup was clicked}"
        sleep "${SLEEP_TIME}"
    done
}


LOCK_FILE="${HOME}/.i3-battery-popup.lock"

release_lock() {
    # This shouldn't be necessary but it seems
    # the lock doesn't release on i3 exit
    flock --unlock 200
}


(
    if [[ -n $LOGFILE ]]; then
        exec >>"$LOGFILE" 2>&1
    fi

    flock -xn 200 || { show_message "$(basename ${0}): cannot acquire lock ${LOCK_FILE}"; exit 3; }
    trap release_lock EXIT

    main
) 200>"${LOCK_FILE}"
