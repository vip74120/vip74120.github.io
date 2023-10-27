#! /bin/bash
###################################################################
#                                                                 #
# Script to read the releveant parameters to build a more recent  #
# database for linuxcnc pc hardware                               #
#                                                                 #
# I provide the script as is. No warrenties are granted, not      #
# even that the script is fit for the iuntended purpose.          #
#                                                                 #
# I put it under the LGPL Version 3. a copy of this license can   #
# found here: https://www.gnu.org/licenses/lgpl-3.0.html          #
#                                                                 #
###################################################################
#                                                                 #
# Targets, I hope to meet:                                        #
# . Comparability: hopefully this proposal leads to comparable    #
#   latency test data.                                            #
# . Repeatabilty: given the fast change in PC hard/software       #
#   probably hard to achieve, if asked to be consistent.          #
# . Still, we intend to run cnc machines with potential of        #
#   danger using stock pc hardware. Therefore we should at least  #
#   try to figure out what suits, is acceptable and/or reliable   #
#   enough. Could result in a potential conflict of goals though. #
#                                                                 #
# Todo:                                                           #
# o option to just get the hw data and skip latency test          #
# o option to just do latency test with latency summary only      #
# o skip the .lat file by default but leave an option to have it  #
# o made sure "abort" in UserOptions now really aborts            #
# V 20200922                                                      #
# . some debugging and prettier outputs (still more is needed)    #
#                                                                 #
# V 20200930                                                      #
# . added script "block-snd-modules" if snd modules are loaded    #
#                                                                 #
# V 20200922                                                      #
# . some debugging and prettier outputs (still more is needed)    #
# . added code to list parports and their config                  #
#   if there are no parports, the default is to run no basethread #
#   generated all the same                                        #
# . added a hint what output to place to the results thread       #
# . added logging of cpu temperatures, ie to monitor cpu          #
#   throttling                                                    #                                                                 #
#                                                                 #
# V 20200624                                                      #
#   added lookup for sound/pcspkr chipset                         #
#   Message when CTRL-C is pressed to standby for a graceful exit #
#   Some info on progress when installing software                #
#   clearer output when doing latency test                        #
#   I put the script under LGPL license                           #
#                                                                 #
# V 20200619-c                                                    #
# . general rework, using functions instead of spaghetti code,    #
# . with the help of "dialog" added :                             #
#   .  choice of base thread or no base thread                    #
#   .  choice of time to run                                      #
#   .  option for base thread at 100000                           #
# .  firefox as default browser                                   #
# .  option to quit by pressing ^C                                #
#                                                                 #
# V 20200615                                                      #
# initial version                                                 #
#                                                                 #
###################################################################

# basic variables
version=20200930

# Packages not normally installed, add whatever is needed 
packages="dialog xdotool dmidecode lshw firefox lm-sensors bc"
missing=""

# Settings for latency-histogram
basebin="--bbinsize 1000"
servobin="--sbinsize 1000"
baseinterval="--base 25000" # option 100000 for old PCs

nglx=5
runtime=7200
STARTTIME=$(date +%s)
ENDTIME=$(($runtime + $STARTTIME))

# I would not insist on the particular page, however, it provides 
# some cpu load at least and is google independant...
# plus (big PLUS for me) rather instructunal and entertaining...
page='https://vimeo.com/150574260'
browser="firefox"

user=$(id -u -n 1000)

prefix=$(date +%s)

LCNCVersion="undetermined"

swappre=$(grep Swap /proc/meminfo | grep Cached | awk '{print $2}')
swaprun=""

ParPorts="$(ls -l /proc/sys/dev/parport | grep parport | wc -l)"

# Stuff for beautyfying outputs
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color


###################################################################
#                                                                 #
# General helper functions                                        #
#                                                                 #
###################################################################

waitDot() {
  
  # waitDot( waittime echosign )
  if [ "$1" == "" ]; then break; return 0; fi
  i=0
  if [ "$2" == "" ]; then 
    while [ "$i" != "$1"  ]; 
    do    
      sleep 1; 
      ((i++)); 
    done
    return 0
  else
    while [  "$i" != "$1"  ]; 
    do    
      echo -n "$2"
      sleep 1; 
      ((i++)); 
    done
    return 0
  fi

}


###################################################################
#                                                                 #
# Check required software and environment                         #
#                                                                 #
###################################################################

CheckRequierments() {

  if [ "$EUID" -ne 0 ]
  then
    echo "This script must be run as root."
    exit 1
  fi

  for i in $packages 
  do
    if ! [ "$(dpkg -l $i* | grep ii)"  ]; then
    #if ! [ -x "$(command -v $i)" ]; then
      missing="$missing $i"
    fi
  done
  if [ "$missing" != "" ]; then
    echo "Cannot proceed, need missing packages to be installed:"
    echo "for example: apt-get install $missing"
    read -p "Want me to install it for you now? [Y/n]" answ
 
    if  [ "$answ" == "Y" ] || [ "$answ" == "y" ] || \
        [ "$answ" == "" ]; then
      echo "Standby, I am going to install the required packages."
      echo -n "you will be prompted when cpmpleted, " 
      echo "this may take a while..."
      failed="echo \"Installation failed\"; exit 2"
      apt-get -y install $missing || eval $failed
      echo "success!"
      return 0
    fi
  fi 
  # code to do the configs for apps that require it
  # lm-sensors: sensors-detect --auto
  if [ $(echo $missing | grep lm-sensors) ] ; then
    echo "running setup for lm-sensors..."
    sensors-detect --auto
  fi

  return 0

}


###################################################################
#                                                                 #
# Let the user select the options                                 #
#                                                                 #
###################################################################

UserOptions() {

  # set some reusable parameters
  btitle="Script to collect Hardware data and"
  btitle="$btitle latency data for  LinuxCNC, $version" 

  # greeter TODO: change this to a yesno for quick
  text="Do you want to run using the following default"
  text="$text parameters?\n" 
  text="$text  - User to run the latency test is \"$user\"\n" 
  text="$text  - Your PC has $ParPorts parallel ports. \n"
  if [ $ParPorts -gt 0 ] ; then
    text="$text   \n"
    text="$text     If you intend to use it for LinuxCNC, \n"
    text="$text     latency-test should be run with a basethread.\n"
    text="$text     Include a basethread with \"25000\" nS interval.\n"
    text="$text  \n"
  else
    text="$text   \n"
    text="$text    latency-test will not "
    text="$text    be run with a basethread.\n\n"
    text="$text    if you still want a basethread,\n"
    text="$text    you have to set it manually by pressing \"No\".\n\n"

    basebin="--nobase"
    baseinterval=""
  fi
  text="$text  - run for \"$runtime\" seconds\n"
  text="$text  - use \"$nglx\" glxgears simultaneously\n"
  # text="$text  starting $browser with \"$page\"\n"
  text="$text  - safe data in files with prefix \"$prefix\"\n\n"
  text="$text  Yes, accepts, No lets you alter the values in\"\"\n"
  text="$text  Escape (twice) aborts"

  dialog --title "Hello" \
         --backtitle "$btitle" \
         --yesno "$text" 16 70 
  response=$?

  case $response in
    0) clear
       latoutf=$prefix.lat
       hwoutf=$prefix.hw

       echo -e "Using default parameters:\n" ;; 
    1) \  # select individual parameters
       # Let the user alter the standard user
       text="Enter the user name the latency test should use,\n"
       text+="default is $user"
       exec 3>&1
       user=$(dialog --title "Enter User name" \
                        --backtitle "$btitle" \
                        --inputbox "$text" \
                        10 60 $user 2>&1 1>&3)
       retval=$?
       exec 3>&-
       # choice of base thread or no base thread
       dialog --title "Base thread" \
              --backtitle "$btitle" \
              --yesno "Do you want data for base thread too?" \
                7 60
       response=$?
       case $response in
        0) echo "Base thread data will be collected"
           # option for base thread at 100000
           text="Enter base thread interval,\n"
           text="$text use 100000 for old PCs"
           exec 3>&1
           baseint=25000
           baseint=$(dialog --title "Base thread" \
                            --backtitle "$btitle" \
                            --inputbox "$text" \
                              10 60 "$baseint" 2>&1 1>&3)
           baseinterval="--base $baseint"
           exec 3>&- ;;
        1) basebin="--nobase"; baseinterval=""; 
           echo "No base thread data will be collected";;
        255) echo "[ESC] key pressed, aborting."; return 255;;
       esac 
       # Let the user alter the standard runtime
       text="Enter time the latency test should run in seconds,\n"
       text+="default is $runtime seconds"
       exec 3>&1
       runtime=$(dialog --title "Base thread" \
                        --backtitle "$btitle" \
                        --inputbox "$text" \
                        10 60 $runtime 2>&1 1>&3)
       retval=$?
       exec 3>&-

       # Option for user to define prefix for outfiles
       text="Enter prefix for output files\n"
       text+="default is $prefix (current date in seconds)"
       exec 3>&1
       prefix=$(dialog --title "Prefix for output files" \
                       --backtitle "$btitle" \
                       --inputbox "$text" \
                       10 60 "$prefix" 2>&1 1>&3)
       retval=$?
       exec 3>&-
       latoutf=$prefix.lat
       hwoutf=$prefix.hw

       # Option for user to define no of glxgears to run
       text="Enter number of glxgears to run simultaneosly\n"
       text+="default is $nglx."
       exec 3>&1
       nglx=$(dialog --title "No. of glxgears" \
                     --backtitle "$btitle" \
                     --inputbox "$text" \
                     10 60 "$nglx" 2>&1 1>&3)
       retval=$?
       exec 3>&-

       # Option for user to define the website to view
       # text="Enter URL of webpage to be called\n"
       # text+="default is $page."
       # text+="It is not recommended to change this."

       # exec 3>&1
       # page=$(dialog --title "Webpage to run" \
       #                 --backtitle "$btitle" \
       #                 --inputbox "$text" \
       #                 10 60 "$page" 2>&1 1>&3)
       # retval=$?
       # exec 3>&-

       clear 
       
       echo -e "Using user defined parameters:\n" ;; 

   255) clear
       echo "aborting."
       return 255 ;;
  esac  

  echo "Result: running as user: $user, "
  echo "runtime: $runtime seconds, "
  echo "parameters: $basebin, $baseinterval, "
  echo "prefix for outfiles $prefix"
  echo -e "nglx: $nglx,\n" # page: $page\n"
  #echo "$latoutf, $hwoutf"

}


###################################################################
#                                                                 #
# Prepare output header                                           #
#                                                                 #
###################################################################

OutputHeader() {

  echo -n "LinuxCNC pc tests, version $version, "
  echo "started $(date "+%d.%m.%Y %T")"
  printf '*%.0s' {1..80} 
  echo -e "" 

}


###################################################################
#                                                                 #
# Collect general system info                                     #
#                                                                 #
###################################################################

GeneralInfo() {

  echo "General info:" 
  cat /sys/devices/virtual/dmi/id/board_vendor | tr -d "\n" 
  echo ",  $(cat /sys/devices/virtual/dmi/id/product_name)"
  echo -n "Bios version "
  echo -n $(cat /sys/devices/virtual/dmi/id/bios_version | \
            tr -d """\\n""")
  echo -n ", dated " 
  cat /sys/devices/virtual/dmi/id/bios_date
  echo -n "Chipset: "
  lspci | grep -i chipset | sed 's/^\(.*: \)*//' \
        | sed -e 's/Chipset.*$//p' | tail -n1
  if [ "$(cat /sys/block/sda/queue/rotational)" == "1" ] ; then
    echo "Harddisk is rotational"
  else
    echo "Harddisk is non-rotational, ie. SSD"
  fi
  echo "The amount of swap currently used is $swappre"
  printf '=%.0s' {1..80}
  echo ""

}


###################################################################
#                                                                 #
# Collect CPU related data                                        #
#                                                                 #
###################################################################

CPUData() {

  echo "CPU related data:"
  grep "model name" /proc/cpuinfo | sed '1!d'
  grep "cpu cores" /proc/cpuinfo | sed '1!d'
  grep "stepping" /proc/cpuinfo | sed '1!d'
  grep "cache size" /proc/cpuinfo | sed '1!d'
  printf '=%.0s' {1..80}
  echo ""

}


###################################################################
#                                                                 #
# Collect RAM related data                                        #
#                                                                 #
###################################################################

RAMData() {

  echo "RAM related data:" 
  dmidecode -t memory | grep Maximum | sed  "s/\t//g"
  dmidecode -t memory | grep Size | sed  "s/\t//g"
  printf '=%.0s' {1..80} 
  echo "" 

}


###################################################################
#                                                                 #
# Collect GPU related data                                        #
#                                                                 #
###################################################################

GPUData() {

  echo "GPU related data:"
  lshw -class display -quiet | grep product | sed  's/^[ \t]*//' 
  lshw -class display -quiet | grep configuration | \
       sed  's/^[ \t]*//' 
  printf '=%.0s' {1..80} 
  echo "" 

}


###################################################################
#                                                                 #
# Collect parport related data                                    #
#                                                                 #
###################################################################


PPData() {

  echo "Parallel port related data:"
  nPP="$(ls -l /proc/sys/dev/parport | grep parport | wc -l)"
  # ls -l /proc/sys/dev/parport | grep parport | awk '{print $9}'
  echo "number of parallel ports is $nPP"
  p=0
  ((nPP--))
  while [ $p -le $nPP ]
  do
    port="parport$p"
    dmesg | grep $port | sed -n 1p | sed 's/^.*] //'
    ((p++))
  done
  printf '=%.0s' {1..80} 
  echo "" 

}


###################################################################
#                                                                 #
# Collect OS related data                                         #
#                                                                 #
###################################################################

OSData() {

  echo "Os and desktop related data:"
  grep "PRETTY_NAME" /etc/os-release | sed '1!d' 
  echo ""
  echo "Should the info below not match with your machine,"
  echo "kindly post the output of "pstree" here, including a brief"
  echo "desciption of your desktop environment, window manager and"
  echo "display manager. Tia"
  echo "https://forum.linuxcnc.org/18-computer/39370-script-for-automated-testing-of-computer-latency"
  echo ""

  # Windowmanager
  # according to 
  # https://www.ubuntupit.com/best-20-linux-window-managers-a-comprehensive-list-for-linux-users/
  # there are these:
  # i3, Awesome, XMonad, Openbox, dwm, Gala, KWin, Fluxbox, musca, 
  # spectrwm, herbstluftwm, Enlightenment, JWM, Window Maker, 
  # IceWM, Pantheon, XFWM, Ratpoison, Compiz, Wayland, 
  # according to
  # https://www.slant.co/topics/390/~best-window-managers-for-linux
  # there are also these:
  # bspwm, KWin, SWay, Qtile, Blackbox, wmutils, Windowlab, flwm,
  # AmiWM, Muffin, 5dwm, notion, sawfish, MWM, FVWM, twm, Pekwm,
  # Wongo, 9wm, WMFS2, cwm, wm2, UWM, monsterwm, 2bwm, snapwm, 
  # cinnamon, Pop!_OS, Lightning, kde, Gnome, lxde, Budgie, CDE, 
  # LXqt, 

  # graphical displaymanagers according to:
  # https://wiki.archlinux.org/index.php/Display_Manager
  # Entrance, gdm, LightDM, LXDM, SDDM, XDM

  # Desktop environment
  # according to:
  # https://wiki.archlinux.org/index.php/Desktop_Environment#List_of_desktop_environments
  # Budgie, Cinnamon, Deepin, Enlightment, Gnome, gnome flashback, 
  # KDE Plasma, LXDE, LXQt, Mate, Sugar


  ##############################################################################
  # Desktop environment
  # which are the most used ones on debian?
  # according to :
  # these are supported or selectable during install. So I will 
  # concentrate on these for the time being:
  # GNOME, Xfce, KDE Plasma, Cinnamon, 
  # MATE, LXDE, LXQt
  # further these are avalable for debian:
  # Budgie, Enlightenment, FVWM-Crystal, 
  # GNUstep/Window Maker, Sugar Notion WM 
  ##############################################################################

  des="GNOME|Xfce|KDE|Plasma|Cinnamon|MATE|LXDE|LXQt|Budgie|Enlight|FVWM"
  des="$des|GNUstep|Window Maker|Sugar"
  dm=$(pstree | grep -Ei "$des" | sed -n 1p | cut -d'-' -f2)
  echo "Desktop environment : $dm"

  # Windowmanager

  WID="$(su $user -c "DISPLAY=:0.0 xprop -root -notype | grep _NET_SUPPORTING_WM_CHECK: | sed 's/.* # //'")"
  WM="$(su  $user -c "DISPLAY=:0.0 xprop -id $WID -notype -f _NET_WM_NAME 8t | grep '_NET_WM_NAME = '  | sed 's/.* \"//' | sed 's/\"//' " )"
  echo "Windowmanager       : $WM"

  # Display manager
  # gdm, sddm, lxdm, xdm, lightdm, slim, wdm, nodm
  dms="gdm|sddm|lxdm|xdm|lightdm|slim|wdm|nodm"
  dm=$(pstree | grep -Ei "$dms" | sed -n 1p | cut -d'-' -f2)
  echo "Displaymanager      : $dm"

  printf '=%.0s' {1..80} 
  echo "" 

}


###################################################################
#                                                                 #
# Collect Kernel related data                                     #
#                                                                 #
###################################################################

KernelData() {

  echo "Kernel related data:"
  echo "Kernel $(uname -r)" 
  grep "GRUB_CMDLINE_LINUX_DEFAULT" /etc/default/grub | sed '/^#/d' 
  echo -n "Cpu idle driver: "
           cat /sys/devices/system/cpu/cpuidle/current_driver
  printf '=%.0s' {1..80} 
  echo "" 

}


###################################################################
#                                                                 #
# Collect harmful modules data                                    #
#                                                                 #
###################################################################

ModulesData() {

  echo "Kernel modules data:"
  echo -n "Check if pcspkr is loaded: "
  if [ $(lsmod | grep spkr) =="" ]; then
    echo -e "${GREEN}No, which is good!${NC}" 
  else 
    echo -e "${RED}Yes, may cause bigger latency.${NC}"
    echo "  can be eliminated using:" 
    echo -n "  sudo echo \"install pcspkr /bin/true\""
    echo " >/etc/modprobe.d/pcspkr.conf" 
    #echo "  sudo echo \"blacklist pcspkr\" > /etc/modprobe.d/pcspkr-blacklist.conf"
  fi
  echo -n "Check if snd modules are loaded: "
  if [ "-$(lsmod | grep snd)-" == "--" ]; then
    echo -e "${GREEN}No, which is good!${NC}" 
  else 
    echo -e "${RED}Yes, may cause bigger latency.${NC}"
    echo "  can probably be disabled in bios, alternatively, you can run"
    file="block-snd-modules"
    echo "#! /bin/bash" > $file
    echo "if [ \"-\$(lsmod | grep snd)-\" != \"--\" ] ; then " >> $file
    echo "  echo \"Currently these \\\"snd\\\" modules are loaded: \$(lsmod | grep snd | awk '{print \$1}')\"" >> $file
    echo "  for i in \$(lsmod | grep snd | awk '{print \$1}' | tr  '\\n' ' ') " >> $file
    echo "  do" >> $file
    echo "    echo \"install \$i /bin/false\" >> /etc/modprobe.d/snd.conf" >> $file
    echo "  done" >> $file
    echo "  echo \" reboot now and check using \\\"lsmod | grep snd\\\"\"" >> $file
    echo "else" >> $file
    echo "  echo \" There are no \\\"snd\\\" modules loaded.\"" >> $file
    echo "fi" >> $file
    chmod 700 $file
    echo -e "   \"./$file\"\n   which has just now been created for your convenience. " 
  fi
  printf '=%.0s' {1..80} 
  echo "" 

}


###################################################################
#                                                                 #
# Collect kbd & mouse related data                                #
#                                                                 #
###################################################################

KbdMouseData() {

  echo "Keyboard & Mouse related data:" 
  nmice="$(ls /sys/class/input | grep mouse | wc -l)" 
  echo "Number of mice: $nmice" 
  m=0
  ((nmice--))
  while [ $m -le $nmice ]
  do
    echo "Mouse $m : $(cat /sys/class/input/mouse$m/device/name)" 
    ((m++))
  done
  echo "Mice  attached to USB:" 
  lsusb | grep -i mouse | sed -r 's/.*:.{4} //' 
  echo "Keyboards attached to USB:" 
  lsusb | grep -i keyboard | sed -r 's/.*:.{4} //' 
  printf '=%.0s' {1..80} 
  echo "" 

}


###################################################################
#                                                                 #
# Collect Linuccnc related data                                   #
#                                                                 #
###################################################################

LCNCData() {

  echo "LinuxCNC related data:" 
  LCNCVersion=$(apt-cache show linuxcnc-uspace | grep Version | \
                head -n1 | awk '{print $2}' | sed 's/^..//' )
  echo "LinuxCNC version is: $LCNCVersion"
  printf '=%.0s' {1..80} 
  echo "" 

}


###################################################################
#                                                                 #
# Prepare everything to run latency test                          #
#                                                                 #
###################################################################

PrepLat() {

  echo "Preparing for latency test:" 
  # left for reference
  #browser=$(xdg-mime query default x-scheme-handler/http | 
  #          \sed 's/-.*$//g')"

  echo -n "Started glxgears No."
  i=1
  while [ $i -le $nglx ]
  do
    su $user -c "DISPLAY=:0.0 glxgears > /dev/null & "
    echo -n " $i"
    ((i++))
    if [ $i -le $nglx ] ; then
      echo -n ","
    else
      echo ""
    fi 
  done

  su $user -c "DISPLAY=:0.0 $browser --new-window '$page' &"
  waitDot 10 "." 
  echo ""

  WID="$(su $user -c "DISPLAY=:0.0 xdotool search --name vimeo | tail -n1")"
  echo "Started $browser with '$page', WID $WID" 
  printf '=%.0s' {1..80} 
  echo "" 

}


###################################################################
#                                                                 #
# Run the very latency test                                       #
#                                                                 #
###################################################################

RunLat() {

  echo "STartime = $(date)"
  STARTTIME=$(date +%s)
  ENDTIME=$(($runtime + $STARTTIME))

  command="su $user -c "
  command="$command \"latency-histogram $basebin"
  command="$command $baseinterval"
  command="$command $servobin --nox 2>&1 | tee -a $latoutf &\"" 

  text="Command for latency test is:\n$command"
  echo -e "$text"  > >(tee  -a $hwoutf)
  text="\nLatency testing loop started $(date -d@$STARTTIME), "
  echo -e "$text"                              > >(tee  -a $hwoutf)
  text="should end after $(date -d@$ENDTIME)"
  echo -e "$text"                              > >(tee  -a $hwoutf)

  su $user -c "sensors | grep Core 2>&1 | tee -a $latoutf "

  eval "$command"
  echo "done command, retval = $?"

  # put browser into action and fullscreen
  waitDot "5" "." # 5 seconds be may too much to make sure the default browser
  # started and was able to load the page : TODO 
  #                          " " starts play back, "f" full screen
  WID="$(su $user -c "DISPLAY=:0.0 xdotool search --name vimeo | tail -n1")"

  su $user -c "DISPLAY=:0.0 xdotool windowactivate $WID type --delay 500 \" f\" "  

  # get all the glxgear's window ids in order to play with them
  # using 
  WIDS="$(su $user -c "DISPLAY=:0.0 xdotool search --name glxgears")"
  #echo "glxgears : $WIDS"

  for id in $WIDS; do
    su $user -c "DISPLAY=:0.0 xdotool windowsize $id 100% 100%"
  done

  swaprun=$(grep Swap /proc/meminfo | grep Cached | awk '{print $2}')

  while [ $(date +%s) -le $ENDTIME ]
  do
    secs=$(($ENDTIME - $(date +%s)))
  
    # kind of randomize windowsize
    wsize="$((($secs % 3 + 1) * 33))%"
    remain=$(printf '%dd %dh:%dm:%ds\n' $(($secs/86400)) \
           $(($secs%86400/3600)) $(($secs%3600/60)) $(($secs%60)))
    echo -e  "\e[0K\r Remaining $remain, press ^C to abort... "
  
    for id in $WIDS; do
      su $user -c "DISPLAY=:0.0 xdotool windowsize $id $wsize $wsize"
    done
    waitDot 5 "" 

    su $user -c "sensors | grep Core 2>&1 | tee -a $latoutf "

    # do we have realtime errors?
    if [ "$(grep delay $latoutf)"  ] 
    then
      echo "" > >(tee  -a $hwoutf)
      text="Useless to proceed, read the relevant part in $latoutf\n" 
      echo -e "$text " > >(tee  -a $hwoutf)
      echo -e "${RED}>>>$(grep "delay" $latoutf)<<<${NC}" > >(tee  -a $hwoutf)
      echo -n "Stand by while preparing for gracefull exit "
      waitDot 5 "."
      ENDTIME=$(($(date +%s) - 1))
    fi

  done

}


###################################################################
#                                                                 #
# Catch Ctrl-C, prepare for a graceful exit                       #
#                                                                 #
###################################################################

CtrlC() {

  text="\nManual termination requested at $(date "+%d.%m.%Y %T")"
  echo -e "$text " > >(tee  -a $hwoutf)
  text="\nStand by and let me stop gracefully...\n"
  echo -e "$text "
  ENDTIME=$(($(date +%s) - 1))

}


###################################################################
#                                                                 #
# Clean up everything                                             #
#                                                                 #
###################################################################

cleanup() {

  echo "Cleaning up"

  if ps aux | grep -q "[/l]atency"; then
    kill -SIGINT $(ps -A | awk '/latency/{print $1}')
    echo "terminated latency-test ..."
  fi
  echo -n "."
  # pause required for xdotools
  waitDot "10" "." 

  WID="$(su $user -c "DISPLAY=:0.0 xdotool search --name vimeo | tail -n1")"
  echo -e "\nClosing browser $browser with WID of $WID"

  su $user -c "DISPLAY=:0.0 xdotool windowactivate --sync $WID key --clearmodifiers  --delay 100 alt+F4"
  
  if ps aux | grep -q "[g]lxgears"; then
    killall glxgears
    echo "terminated glxgears ..."
  fi
  echo  "Done. "
  
  return 0

}


###################################################################
#                                                                 #
# Analyze temperature data                                        #
#                                                                 #
###################################################################

TemperatureData() {

  nC="$(grep """cpu cores""" /proc/cpuinfo | sed '1!d' | sed 's/^.*: //')"

  # determine core numbering
  cores=""
  c=0
  j=-1
  while [[ $c < $nC ]]
  do
    ((j++))
    if [ "-$(grep "Core $j" $latoutf | sed -n  '1p')-" != "--" ]; then
      cores="$cores $j"
      ((c++))
    fi
  done

  # get temp data
  for i in $cores 
  do
    min=1000.0
    max=0.0
    while IFS= read -r line
    do
      t=$(echo $line | grep "Core $i" | awk '{print $3}' | sed -n 's/°C.*//; s/.*[+-]//; p; q')
      if [ "X$t" != "X" ]; then
        if (( $(echo "$t < $min" | bc -l) )); then
          min=$t
        fi
        if (( $(echo "$t > $max" | bc -l) )); then
          max=$t
        fi
      fi
    done < "$latoutf"
    tcrit=$(grep "Core $i"  $latoutf | sed -n  '1p'| sed  's/^.*(//')
    echo  "Core $i: Tmin: $min°C, Tmax: $max°C, ($tcrit"
  done

}


###################################################################
###################################################################
##                                                               ##
## the main program                                              ##
##                                                               ##
###################################################################
###################################################################

CheckRequierments

UserOptions

OutputHeader          > >(tee  -a $hwoutf)

sleep 1

echo "Hardware info is logged here: $hwoutf."
echo -e "Latency data is logged here: $latoutf.\n"

GeneralInfo           > >(tee  -a $hwoutf)
CPUData               > >(tee  -a $hwoutf)
RAMData               > >(tee  -a $hwoutf)
GPUData               > >(tee  -a $hwoutf)
PPData                > >(tee  -a $hwoutf)
OSData                > >(tee  -a $hwoutf)
KernelData            > >(tee  -a $hwoutf)
ModulesData           > >(tee  -a $hwoutf)
KbdMouseData          > >(tee  -a $hwoutf)
LCNCData              > >(tee  -a $hwoutf)

# Next we start some processin background. In order to be able
# to terminate everyting gracefully, we need to get hold off
# ^C being pressed: trap it

PrepLat               > >(tee  -a $hwoutf)

trap 'CtrlC' SIGINT

RunLat

cleanup

waitDot "10" "." 

echo ""                   | tee  -a $hwoutf
printf '=%.0s' {1..80}    | tee  -a $hwoutf
text="Swap useage: prerun: $swappre, running: $swaprun"
echo -e "\n$text"         | tee  -a $hwoutf
echo ""                   | tee  -a $hwoutf
printf '=%.0s' {1..80}    | tee  -a $hwoutf
echo ""                   | tee  -a $hwoutf

text="CPU core temperatures:"
echo "$text"              | tee  -a $hwoutf
TemperatureData         > >(tee  -a $hwoutf)
printf '=%.0s' {1..80}    | tee  -a $hwoutf

text="\nlast latency data is as follows:." 
echo -e "$text"           | tee  -a $hwoutf
grep secs $latoutf \
     | tail -n2           | tee  -a $hwoutf

sed -i 's/\\033\[[;0-9;]*[JKmsu]//g' $hwoutf
sed -i 's/\x1B\[[;0-9;]*[JKmsu]//g' $hwoutf

echo -e "\nHardware info is logged here: $hwoutf."
echo -e "Full latency-test data is logged here: $latoutf.\n\n"
echo "Kindly place the contents of \"$hwoutf\" here:"
echo -n "https://forum.linuxcnc.org/18-computer/39371-results-of-"
echo "latency-test-list-of-computers-tested-for-use-with-linuxcnc"

echo -e "\nThanks, and good bye."

exit 0