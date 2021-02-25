# Lstats - Mixmaster Remailer Dashboard

This script can be used by a remailer sysops to monitor their mixmaster server and remailer performance.  
Copy the script code below into a file called Lstats.sh and make it executable (sudo chmod 755 Lstats.sh).  
Then execute it with a cron as explained in at the top of the script code.  
The output can be accessed by: yourDN/Lstats.html  
  
  
<p align="left">
  <img src="/images/lstats.png" width="1218" height="1500">
</p>
  
```
#!/bin/bash
#
# Lstats v2.4
#
# Script to build remailer server statistics Lstats.html
#
#-------------------------------------------------------------------------#
#              --- Server statistics web page builder ---                 #
#                                                                         #
# The following script can be used to monitor a remailer servers.         #
# Several statistics concerning the server and mixmaster are displayed.   #
# The script must be executed every 1 minutes to get a more accurate      #
# pool count. The script must also be executed at 0000 hours to reset     #
# the pool count display to zero. Can name Lstats.sh                      #
#                                                                         #
# Retrieve stastics for server web page - cron job                        #
# */1 * * * * /path/to/Lstats.sh &> /dev/null                             #
# To highlight up to 5 remailer lines in the Remailer Statistics, execute #
# */1 * * * * /path/to/Lstats.sh name1 name2 ... name5 &> /dev/null       #
#                                                                         #
# Cert expdt: (Must point to server's certificate in certpath=)           #
# MTD bandwidth: (Must install & run: vnstatd -n)                         #
#                                                                         #
#-------------------------------------------------------------------------#

#top
export PATH=$PATH:/usr/sbin
export PATH=$PATH:/sbin
export PATH=$PATH:/bin

webpgnm="/Lstats.html"
mixmastername="mixmaster"
mixpath="/var/mixmaster"       # no trailing /
webpgpath="/var/www/html"
sshlog="/var/log/auth.log"
certpath="/etc/ssl/private/letsencrypt-domain.pem"
tempdisp="yes"
expdwarn=30
filePath=${0%/*}  # current file path
stathighlight="#ff1493"
varupt=$(uptime)
remailerid1="$1"
remailerid2="$2"
remailerid3="$3"
remailerid4="$4"
remailerid5="$5"
dname=""
tempvar=""
tempnum=0
fontsz="2"
hfontsz="2"
bgclr=#EBEADF
fontcolor=#000040
fontcl=FFFFFF
machinewidth=430
roguewidth=200
freewidth=230
netstatswidth=230
mixmasterwidth=200
iptableswidth=125
miscstats=200
mixfiles=160
remailerstatistics=230
titlecolor=#0072AA
MachineRogueTableWidth=1112
MixMiscRemailerTableWidth=870


##'----------------------------'
## BEGIN Stats source selection
##'----------------------------'
# Place the stat source URL followed by a semicolon followed by any stat title in the statarray.
# Example: "stat source URL;stat title"

statarray=(
"http://www.mixmin.net/echolot/mlist.txt;mixmin4096"
"sec3.net/echolot/mlist.txt;sec3"
"pinger.borked.net/mlist.txt;borked"
"https://apricot.fruiti.org/echolot/mlist.txt;apricot"
)
##'--------------------------'
## END Stats source selection
##'--------------------------'


##'------------------------'
## BEGIN Surround function
##'------------------------'
function Surround {
   line1=$1
   keyword=$2
   pre=$3
   post=$4

   if  [[ $keyword == "" ]]; then
       result="$pre$line1$post"
       output=$result
       else
       result="$pre$keyword$post"
       output=$result
   fi

   echo "$output" >> $filePath/templ.txt2
   cat $filePath/templ.txt2   #//   <------ TEST ONLY   DELETE DELETE DELETE DELETE DELETE DELETE DELETE DELETE DELETE DELETE DELETE DELETE DELETE DELETE DELETE
}
##'----------------------'
## END Surround function
##'----------------------'


##'------------------------'
## BEGIN poolcount function
##'------------------------'
function poolcount(){      #count pool for day
        if [ ! -s $filePath/savepool.txt ];  then  # create file first time
            echo "0" > $filePath/savetodaypoolcnt.txt
            echo "0" >> $filePath/savetodaypoolcnt.txt
            echo "" > $filePath/savepool.txt
            else
            if [ $(date +"%H:%M") = "00:00" ]; then  # reset at midnight
               savetodaypoolcnt=$(head -n 1 $filePath/savetodaypoolcnt.txt)   # save previous days count
               echo "0" > $filePath/savetodaypoolcnt.txt                      # zero new days count
               echo "$savetodaypoolcnt" >> $filePath/savetodaypoolcnt.txt     # save previous days count
               savetodaypoolcnt=0                                             # zero out todays bucket
               echo "$(ls $mixpath/pool/)" > $filePath/savepool.txt           # save current pool at BOD for comparison
               else
               savetodaypoolcnt=$(head -n 1 $filePath/savetodaypoolcnt.txt)
               savepriorpoolcnt=$(sed -n 2p $filePath/savetodaypoolcnt.txt)
               echo "$(ls $mixpath/pool/)" > $filePath/temppool.txt              # get current pool in seq list

               while read line ; do
                  if grep $line $filePath/savepool.txt ; then  # is this message in savepool?
                     continue                                                 # yes
                     else
                    ((savetodaypoolcnt++))                                   # found a new message
                    echo $line >> $filePath/savepool.txt                     # save the message file name for future ref
                  fi
               done<$filePath/temppool.txt                                       # read file line by line

               echo $savetodaypoolcnt > $filePath/savetodaypoolcnt.txt           # save the pool cnt for next chk
               echo $savepriorpoolcnt >> $filePath/savetodaypoolcnt.txt          # save the pool cnt for next chk

               rm $filePath/temppool.txt
            fi
        fi
}
##'------------------------'
##  END poolcount function
##'------------------------'


cat /dev/null > $webpgpath/$webpgnm  # clear html file

echo "<html><head><title>Server Stats</title></head><body bgcolor=\"$bgclr\" TEXT=\"$fontcolor\" LANG=\"en-US\" DIR=\"LTR\">" > $webpgpath/$webpgnm

##'-------------------'
## BEGIN Top date line
##'-------------------'
echo "<font face=\"Verdana\" size=\"$fontsz\" color=\"$fontcolor\"><b>&nbsp;" >> $webpgpath/$webpgnm
MLvar=$(date | awk '{print $0" "$2" "$3" "$6" "$4}' | awk '{print "("$1") "$7" "$8", "$9" &nbsp;&nbsp; "$10}')
MLvar="${MLvar%:*} $(date | awk '{print $5}') &nbsp;&nbsp; ${0##*/}"
echo "&nbsp;$MLvar" >> $webpgpath/$webpgnm
echo "</font></b><br>" >> $webpgpath/$webpgnm
##'-------------------'
##  END Top date line
##'-------------------'


##'----------------------------------------------'
##'----------------------------------------------'
## BEGIN Machine & Netstats/Free horzontal tables
##'----------------------------------------------'
##'----------------------------------------------'
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm


##'-------------------'
## BEGIN Machine table
##'-------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<b><font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Machine</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## linux info
echo "$(lsb_release -d) $(uname -m)<br>" | sed -e 's/Description://g' | tr -d "\t" >> $webpgpath/$webpgnm
echo "$(uname -mrs)" >> $filePath/templ.txt
sed '/No LSB/d' $filePath/templ.txt
sed -e 's/$/<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm

# ip address
echo "IP server: $(hostname -I)<br>" >> $webpgpath/$webpgnm

# domain name
dname=$(dig -x $(hostname -I) +short | awk -F '.' '{print $1"."$2}')
echo "Domain name: $dname<br>" >> $webpgpath/$webpgnm

# MTU
echo "$(ifconfig | sed '1!d' | awk '{print "MTU: " $4}')<br>" >> $webpgpath/$webpgnm

## ipv4 TTL & ipv6 HOP count
echo "ipv4 TTL: $(sysctl net.ipv4.ip_default_ttl | awk '{ print $3 }')<br>" >> $webpgpath/$webpgnm
echo "ipv6 HOP: $(sysctl net.ipv6.conf.all.hop_limit | awk '{ print $3 }')<br>" >> $webpgpath/$webpgnm

## openssl
echo "$(openssl version)" | awk '{print $1" "$2" ("$3" "$4" "$5")<br>"}' >> $webpgpath/$webpgnm

## certificate expdt
#notAfter=Feb  8 12:19:20 2021 GMT
SSLvar=$(openssl x509 -enddate -noout -in $certpath)  # /etc/ssl/private/letsencrypt-domain.pem)
SSLvar=$(cut -d'=' -f2 <<<$SSLvar)
SSLvar=$(awk '{print $1"-"$2"-"$4" "$3" "$5}' <<<$SSLvar)
SSLvar1=$(awk '{print $1}' <<<$SSLvar)
SSLvar2=$(let DIFF=(`date +%s -d $SSLvar1` - `date +%s`)/86400 && echo $DIFF)
if [[ $SSLvar2 -le 3 ]];then
   echo "<font color=FF0000><b>Cert expdt: $SSLvar<br></b></font>" >> $webpgpath/$webpgnm
   else
   echo "Cert expdt: $SSLvar<br>" >> $webpgpath/$webpgnm
fi

## last boot time
lsvar1=$((who -b) | awk '{print $3}')  # 2021-01-06
lsvar2=$(date -d $lsvar1 +%b-%d-%Y)    # cvt date to Jan-06-2021
lsvar3=$((who -b) | awk '{print $4}')  # 12:43
echo "Last system reboot: $lsvar2 $lsvar3" >> $webpgpath/$webpgnm  # Last system boot: 2020-11-14 15:17

## up time
varupt=`echo ${varupt//up/Up}`
varupt=`echo ${varupt//load/Load}`
awk -F '[ \t\n\v\r]' '{print "<br>"$2" "$3" "$4" "$5" "$8" "$9" "$10" "$11" "$12" "$13" "$14" "$15}' <<< $varupt >> $webpgpath/$webpgnm

###Bandwidth
if pidof -x "vnstatd" >/dev/null; then
   var1=$(date |  awk '{print $2" "}')  #
   var2=$(date +"'%y")
   var3=$var1$var2
   var4=$(grep -e "$var3" <<< $(vnstat))
   bline=$(awk '{print "MTD bandwidth: " $1" "$2" - "$9" "$10}' <<<$var4)
   echo "<br>$bline" >> $webpgpath/$webpgnm
else
   echo "<br>MTD bandwidth: vnstat not running!" >> $webpgpath/$webpgnm
fi

## last login
(last -a root) > $filePath/templ.txt
sed -n -e 1p $filePath/templ.txt > $filePath/templ2.txt
(cat $filePath/templ2.txt | colrm 1 22 | colrm 35 100) > $filePath/templ.txt
lines=$(head -n 1 $filePath/templ.txt)
echo "<br>Last login: $lines" >> $webpgpath/$webpgnm
echo "</font></b></td></tr></table><br>" >> $webpgpath/$webpgnm    # end build Machine
##'-------------------'
##  END Machine table
##'-------------------'


##'-------------------------------------'
## BEGIN dummy table below Machine table
##'-------------------------------------'
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm
echo "</td><td>" >> $webpgpath/$webpgnm
echo "</td></tr></table><br>" >> $webpgpath/$webpgnm
##'-------------------------------------'
##  END dummy table below Machine table
##'-------------------------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: vertical divider between Machine and Netstat/Free
##'------------------------------------'


##'--------------------'
## BEGIN Netstats table
##'--------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Netstats</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm
netstat -vatnp > $filePath/templ.txt

sed -i "/tcp6/d" $filePath/templ.txt            # remove tcp6 lines
sed -i "/python2.7/d" $filePath/templ.txt  # remove bitmessage python2.7 messages
sed -i "/nginx\: worker/d" $filePath/templ.txt  # remove nginx: worker messages
sed -i "/TIME_WAIT/d" $filePath/templ.txt  # remove TIME_WAIT messages
sed -i 's/ /\&nbsp;/g' $filePath/templ.txt
sed -i 's/ //g' $filePath/templ.txt
sed -e 's/$/<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'--------------------'
##  END Netstats table
##'--------------------'


##'----------------'
## BEGIN Free table
##'----------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Free</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

free > $filePath/templ.txt
sed -i 's/ /\&nbsp;/g' $filePath/templ.txt
sed -i 's/ //g' $filePath/templ.txt
sed -e 's/$/\&nbsp;\&nbsp;<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm  # append <br>
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'----------------'
##  END Free table
##'----------------'

echo "</td></tr></table>" >> $webpgpath/$webpgnm
##'----------------------------------------------'
##'----------------------------------------------'
##  END Machine & Netstats/Free horzontal tables
##'----------------------------------------------'
##'----------------------------------------------'


##'-----------------------------------------------------'
##'-----------------------------------------------------'
## BEGIN Misc Stats/Mixmaster/Mixmaster horzontal tables
##'-----------------------------------------------------'
##'-----------------------------------------------------'
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm


##'-------------------------------'
## BEGIN 1st Mixmaster stats table
##'-------------------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Mixmaster</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## mailq
echo "mailq count: " > $filePath/templ.txt
vattest=$(mailq)
if [[ $vattest = "Mail queue is empty" ]]; then
   cat /dev/null > $filePath/notification.txt
   echo "0" >> $filePath/templ.txt
   varzero="y"
   else
   varzero="n"
   mailq | grep -c "^[A-Z0-9]" >> $filePath/templ.txt
   varmq=$(mailq | grep -c "^[A-Z0-9]")

   if [ $(expr $(date +"%H") % 4) -eq 0 -a $(date +"%M") = "00" -a $varmq -gt 200 ]; then   # mail every 4 hours if mailq > 9
      mutt -s "Error from mixharbor.xyz -  Mailq backup - $varmq" x0007@cotse.net <<<"From /etc/Servstats/Lstats.sh: Mailq backup - $varmq"
   fi
fi

sed '1~2 {N;N;s/\n/ /g}' $filePath/templ.txt > $filePath/templ2.txt  # concat lines 1 and 2
if [[ $varzero = "n" ]]; then
   vartemp=$(<$filePath/templ2.txt)
   echo "<font color=FF0000><b>"$vartemp"</b></font>" > $filePath/templ2.txt  # color 'mailq count:' red
fi

cat $filePath/templ2.txt >> $webpgpath/$webpgnm
echo "<br>" >> $webpgpath/$webpgnm
## mailq end

## pool count
poolcount
echo "pool count:&nbsp;" > $filePath/templ.txt
find $mixpath/pool -type f | wc -l >> $filePath/templ.txt
sed '1~2 {N;N;s/\n/ /g}' $filePath/templ.txt > $filePath/templ2.txt  # concat lines 1 and 2
cat $filePath/templ2.txt >> $webpgpath/$webpgnm
echo "<br>" >> $webpgpath/$webpgnm
## pool count end

## pool today and yesterday
if [ ! -s $filePath/savetodaypoolcnt.txt ]; then
   ptd=0
   pyd=0
   echo "0" > $filePath/savetodaypoolcnt.txt
   echo "0" >> $filePath/savetodaypoolcnt.txt
fi

if [ -e $filePath/poolcount.sh ] && [[ $(cat $filePath/savetodaypoolcnt.txt | wc -l) -gt 0 ]]; then  # running poolcount.sh?
   ptd=$(head -n 1 $filePath/savetodaypoolcnt.txt)  # get 1st line = total thus far today
   pyd=$(sed -n 2p $filePath/savetodaypoolcnt.txt)  # get 2nd line = prior
   echo "pool total: &nbsp;$ptd<br>" >> $webpgpath/$webpgnm
   echo "pool prior: &nbsp;$pyd<br>" >> $webpgpath/$webpgnm
fi
## pool today and yesterday end

## error count
if [[ $(grep -c 'Error:' $mixpath/error.log) -ne 0 ]]; then
   echo "<font color=FF0000><b>error count: $(grep -c 'Error:' $mixpath/error.log)<br></b></font>" >> $webpgpath/$webpgnm
   else
   echo "error count: 0<br>" >> $webpgpath/$webpgnm
   fi

## allpinger/tls dates
var1=$(grep "# Updated:" $mixpath/allpingers.txt)
var2=$(awk -F '[ \t\n\v\r.]' '{print $4" "$5" "$6}' <<< $var1)
var3="all:&nbsp; $(date --date "$var2" +%b" "%d" "%Y)<br>"
echo $var3 >> $webpgpath/$webpgnm

echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm   # end build Misc
##'-------------------------------'
##  END 1st Mixmaster stats table
##'-------------------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: vertical divider between 1st Mixmaster & 2nd Mixmaster
##'------------------------------------'


##'-------------------------------'
## BEGIN 2nd Mixmaster stats table
##'-------------------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Mixmaster</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## top stats
top -b -n1 > $filePath/templ.txt
var1=$(grep 'PID USER' $filePath/templ.txt)  # get header
var2=$(grep mixmaster $filePath/templ.txt)   # get mixmaster line
echo "$var1" > $filePath/templ.txt
echo "$var2" >> $filePath/templ.txt
sed -i 's/ /\&nbsp;/g' $filePath/templ.txt
sed -e 's/$/<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm  # append <br>
echo "<br>" >> $webpgpath/$webpgnm

## start date/time
echo "$(grep "mixmaster" <<< "$(ps -eo lstart,cmd)")" | awk '{print "Started: "$1" "$2" "$3" "$4}' >> $webpgpath/$webpgnm

## md5
echo "<br>MD5: $(md5sum /usr/bin/mixmaster)" > $filePath/templ.txt
cat $filePath/templ.txt >> $webpgpath/$webpgnm

## pub key
var1=$(head -n 1 $mixpath/key.txt)   # get pubkey header
awk '{print "<br>Pubkey: " $3}' <<< "$var1" >> $webpgpath/$webpgnm

## key(s)
var1="<br>Seckey: "
grep "Created:" $mixpath/secring.mix > $filePath/temp1.txt  # extract Created: date
grep "Expires:" $mixpath/secring.mix > $filePath/temp12.txt  # extract Expires: date

cat $filePath/temp12.txt > $filePath/temp16.txt
sed -i 's/Expires: //g' $filePath/temp16.txt  # del Expires:

paste $filePath/temp1.txt $filePath/temp12.txt > $filePath/temp13.txt  # stack dates side by side
sed -i 's/\t/ /g' $filePath/temp13.txt              # replace tab btwn dates with space
sed -i 's/Created: //g' $filePath/temp13.txt
sed -i 's/Expires: //g' $filePath/temp13.txt

var7=$(awk 'c&&!--c;/Expires:/{c=1'} $mixpath/secring.mix)
echo "$var7" > $filePath/temp14.txt
var2=0

while read line1; do
   ((var2++))
   var3=$(awk -F '[ \t\n\v\r.]' '{print $1" "$2}' <<< $line1)  # get 2014-05-08 2014-11-04
   var9=$(awk -v var8=$var2 'NR==var8' $filePath/temp16.txt)  # pull stacked dates from temp12.txt
   var10=$(date +"%Y-%m-%d")
   days=$(( ($(date --date=$var9 +%s) - $(date --date=$var10 +%s) )/(60*60*24) ))   # calc days between
   var5=$days
   var7=$(sed -n $var2'p' $filePath/temp14.txt)
   var4="$var1 $var3 ($var5) ($var7)"                    #<- days remaining calc
   echo $var4 >> $webpgpath/$webpgnm
done<$filePath/temp13.txt

## key count
var5="<br>Mix key count: Pub "$(grep -c 'Begin Mix Key' $mixpath/key.txt)"&nbsp;&nbsp;Sec "$(grep -c 'Begin Mix Key' $mixpath/secring.mix)"<br>"
echo $var5 >> $webpgpath/$webpgnm
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'-------------------------------'
##  END 2nd Mixmaster stats table
##'-------------------------------'


echo "</td></tr></table>" >> $webpgpath/$webpgnm
##'-----------------------------------------------------'
##'-----------------------------------------------------'
##  END Misc Stats/Mixmaster/Mixmaster horzontal tables
##'-----------------------------------------------------'
##'-----------------------------------------------------'


##'-------------------------------'
##'-------------------------------'
## BEGIN mixmaster error log table
##'-------------------------------'
##'-------------------------------'
if [[ $(grep -c " Error: " $mixpath/error.log) -gt 0 ]] ; then
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" size=\"$fontsz\" color=\"$fontcl\" color=FF0000><b>Mix Errors</b></font></td></tr>
   <tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm
   echo "(This error list will be removed only by clearing the error.log)" > $filePath/templ.txt
   grep " Error: " $mixpath/error.log >> $filePath/templ.txt
   sed -i 's/$/<br>/' $filePath/templ.txt   # add <br> to end of every rec
   cat $filePath/templ.txt >> $webpgpath/$webpgnm
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
fi
##'-------------------------------'
##'-------------------------------'
##  END mixmaster error log table
##'-------------------------------'
##'-------------------------------'


##'--------------------------------------'
##'--------------------------------------'
## BEGIN remailers stats horzontal tables
##'--------------------------------------'
##'--------------------------------------'
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm


##'--------------------------'
## BEGIN remailer stats table
##'--------------------------'
varaLS=0

for i in "${statarray[@]}"; do
   varLS=$i
   ((varaLS++))

   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Remailer Statistics (${varLS##*;})</b></font></td></tr>
   <tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

      if [[ $varaLS -eq 1 ]]; then  #  only pause at 1st stat download
         sleep 5                  #  pause on 1st stat collect for pingers to finish updating their stats
      fi

      wget  --no-check-certificate --timeout=15  -t 1 ${varLS%%;*} -O $filePath/varmlist.txt
      echo $(date) > $filePath/statdate.txt

   savdate=$(< $filePath/statdate.txt)
   grep "%" $filePath/varmlist.txt | colrm 16 28 > $filePath/astats.txt
   sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
   sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
   sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
   sed -i "1i&nbsp;$savdate" $filePath/astats.txt
   cat $filePath/astats.txt > $filePath/templ.txt

  while read line1; do
      if [[ $line1 =~ "CST" ]]; then  # test ans save remailer date line
         varM=$line1
         continue
      fi

      if [[ $line1 =~ $remailerid1 && ! $remailerid1 == "" ]] || \
         [[ $line1 =~ $remailerid2 && ! $remailerid2 == "" ]] || \
         [[ $line1 =~ $remailerid3 && ! $remailerid3 == "" ]] || \
         [[ $line1 =~ $remailerid4 && ! $remailerid4 == "" ]] || \
         [[ $line1 =~ $remailerid5 && ! $remailerid5 == "" ]]; then
         Surround $line1 "" "<font size=\"$fontsz\" color=^ff1493^><u>" "</u></font>"        # surround whole line (keyword="")
      else
         echo "$line1" >> $filePath/templ.txt2
      fi

   done< $filePath/templ.txt

   sed -i "1i$varM" $filePath/templ.txt2  # restore remailer date line to 1st rec in file
   sed -e 's/\^/\"/g' $filePath/templ.txt2 > $filePath/astats.txt
   sed -i 's/$/<br>/' $filePath/astats.txt   # append <br>

   rm $filePath/templ.txt
   rm $filePath/templ.txt2

   cat $filePath/astats.txt >> $webpgpath/$webpgnm
   echo "</font></td></tr></table>" >> $webpgpath/$webpgnm

   if [[ $varaLS -ne ${#statarray[*]} ]]; then
      echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: vertical divider between 1st and 2nd remailer stats tables
   fi

done
##'--------------------------'
##  END remailer stats table
##'--------------------------'

   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'--------------------------------------'
##'--------------------------------------'
##  END remailers stats horzontal tables
##'--------------------------------------'
##'--------------------------------------'

##'-------------------'
##'-------------------'
## BEGIN display mailq
##'-------------------'
##'-------------------'
vattest=$(mailq)
if [[ ! $vattest = "Mail queue is empty" ]]; then
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Mailq</b></font></td></tr>
   <tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

   echo "$vattest" | sort | uniq -c | sort -nk1 | awk '{$1=$1}1' | sed '/^.\{10,50\}$/!d' | grep -v "Request" > $filePath/templ.txt
   sed -e 's/$/<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm
   echo "<br>" >> $webpgpath/$webpgnm

   echo "$vattest" > $filePath/templ.txt
   sed -e 's/$/<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
fi
##'-------------------'
##'-------------------'
##  END display mailq
##'-------------------'

##'------------------------------------------'
##'------------------------------------------'
## BEGIN pool & multiple web page hits tables
##'------------------------------------------'
##'------------------------------------------'
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm  # BEGIN


##'----------------'
## BEGIN pool table
##'----------------'
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" color=\"$fontcl\" size=\"$fontsz\"><b>Pool "-" $(find $mixpath/pool -type f | wc -l) "-" $(date +"%r")</font></td></tr>
   <tr><td><font face=\"Courier New\" size=\"$fontsz\" color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## put count of each rec type here
   cat /dev/null > $filePath/templ.txt
   for i in $mixpath/pool/* ; do data=$(stat $i --format="%y") && echo "${data%.*} $i $(sed -n -e 2p $i)" >> $filePath/templ.txt ; done
   sed -i 's/\/home\/mix\/Mix\/pool\///g' $filePath/templ.txt
   sed -i 's/\/var\/mixmaster\/pool\///g'  $filePath/templ.txt
   sed -e 's/$/<br>/' $filePath/templ.txt | sort >> $webpgpath/$webpgnm  # append <br> to each line in templ.txt
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'----------------'
##  END pool table
##'----------------'


echo "</td></tr></table>" >> $webpgpath/$webpgnm  # END
##'------------------------------------------'
##'------------------------------------------'
##  END pool & multiple web page hits tables
##'------------------------------------------'
##'------------------------------------------'


echo "</body></html>" >> $webpgpath/$webpgnm

rm $filePath/temp.txt
rm $filePath/temp1.txt
rm $filePath/temp2.txt
rm $filePath/templ.txt
rm $filePath/templ2.txt
rm $filePath/temp12.txt
rm $filePath/temp13.txt
rm $filePath/temp14.txt
rm $filePath/temp16.txt
rm $filePath/temp19.txt
rm $filePath/temp20.txt
rm $filePath/temp21.txt

exit 0


```
