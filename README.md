# Lstats - Mixmaster Remailer Dashboard

This script can be used by a remailer sysops to monitor their mixmaster server and remailer performance.  
Copy the script code below into a file called Lstats.sh and make it executable (sudo chmod 755 Lstats.sh).  
Then execute it with a cron as explained in at the top of the script code.  
The output can be accessed by: yourDN/Lstats.html
  
  
```
#!/bin/bash
#
# Lstats v1.2
#
# Script to build remailer server statistics Lstats.html
#
#------------------------------------------------------------------------#
#              --- Server statistics web page builder ---                #
#                                                                        #
# The following script can be used to monitor a remailer servers.        #
# Several statistics concerning the server and mixmaster are displayed.  #
# The script must be executed every 1 minutes to get a more accurate     #
# pool count. The script must also be executed at 0000 hours to reset    #
# the pool count display to zero. Can name Lstats.sh                     #
#                                                                        #
# Retrieve stastics for server web page - cron job                       #
# */1 * * * * /path/to/Lstats.sh &> /dev/null                            #
#                                                                        #
# Cert expdt: (Must point to server's certificate in certpath=)          #
# MTD bandwidth: (Must install & run: vnstatd -n)                        #
#                                                                        #
#------------------------------------------------------------------------#

#top
export PATH=$PATH:/sbin
export PATH=$PATH:/bin

webpgnm="/private/Lstats.html"
mixmastername="mixmaster"
mixpath="/var/mixmaster"       # no trailing /
webpgpath="/var/www/html"      # no trailing /
sshlog="/var/log/auth.log"
certpath="/etc/ssl/private/letsencrypt-domain.pem"
tempdisp="yes"
dostats="$1"
expdwarn=30
filePath=${0%/*}  # current file path
stathighlight="#ff1493"
varupt=$(uptime)
remailerid="$1"
mmid=$remailerid
serverid="$remailerid - "
dname=""
tempvar=""
tempnum=0
fontsz="2"
hfontsz="2"
bgclr=#EBEADF
fontcolor=#00000040
titlecolor=#f0f0f0
machinewidth=430
roguewidth=200
freewidth=230
netstatswidth=230
mixmasterwidth=200
iptableswidth=125
miscstats=200
mixfiles=160
remailerstatistics=230
titlecolor=#f0f0f0
MachineRogueTableWidth=1112
MixMiscRemailerTableWidth=870


##'------------------------'
## BEGIN poolcount function
##'------------------------'
function poolcount(){      #count pool for day
        if [ ! -s $filePath/savepool.txt ];  then  # create file first time
            echo "0" > $filePath/savetodaypoolcnt.txt
            echo "0" >> $filePath/savetodaypoolcnt.txt
            echo "" > $filePath/savepool.txt
            exit 0
        fi

        if [ $(date +"%H:%M") = "00:00" ]; then  # reset at midnight
           savetodaypoolcnt=$(head -n 1 $filePath/savetodaypoolcnt.txt)   # save previous days count
           echo "0" > $filePath/savetodaypoolcnt.txt                      # zero new days count
           echo "$savetodaypoolcnt" >> $filePath/savetodaypoolcnt.txt     # save previous days count
           savetodaypoolcnt=0                                             # zero out todays bucket
           echo "$(ls $yamnpath/pool/)" > $filePath/savepool.txt           # save current pool at BOD for comparison
           exit 0
        fi

        savetodaypoolcnt=$(head -n 1 $filePath/savetodaypoolcnt.txt)
        savepriorpoolcnt=$(sed -n 2p $filePath/savetodaypoolcnt.txt)
        echo "$(ls $yamnpath/pool/)" > $filePath/temppool.txt              # get current pool in seq list

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
}
##'------------------------'
##  END poolcount function
##'------------------------'


cat /dev/null > $webpgpath/$webpgnm  # clear html file

echo "<html><head><title>Server Stats</title></head><body bgcolor=\"$bgclr\" TEXT=\"$fontcolor\" LANG=\"en-US\" DIR=\"LTR\">" > $webpgpath/$webpgnm


##'-------------------'
## BEGIN Top date line
##'-------------------'
echo "<font face=\"Verdana\" size=$fontsz color=\"$fontcolor\"><b>&nbsp;" >> $webpgpath/$webpgnm
MLvar=$(date | cut -c 1-10 && date | cut -c 25-28 && echo "-" && date | cut -c 12-23)
MLvar=${MLvar//:01 / } && MLvar=${MLvar//:02 / } && MLvar=${MLvar//:03 / }  # remove seconds position
echo "$MLvar" - ${0##*/} >> $webpgpath/$webpgnm
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
<b><font face=\"Verdana\" size=$fontsz><b>Machine</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

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
xvar1=$(who -b | awk '{print $3}' | date -d $? "+%b-%d-%Y")        # Nov-22-2020
xvar2=$(echo $(who -b) | awk '{print "Last "$1" "$2": "$3" "$4}')  # Last system boot: 2020-11-14 15:17
xvar3=$(awk '{print $1" "$2" "$3}' <<<$xvar2)
xvar4=$(awk '{print $5}' <<<$xvar2)
echo "$xvar3 $xvar1 $xvar4" >> $webpgpath/$webpgnm

## up time
varupt=`echo ${varupt//up/Up}`
varupt=`echo ${varupt//load/Load}`
awk -F '[ \t\n\v\r]' '{print "<br>"$2" "$3" "$4" "$5" "$8" "$9" "$10" "$11" "$12" "$13" "$14" "$15}' <<< $varupt >> $webpgpath/$webpgnm

###Bandwidth
if pidof -x "vnstatd" >/dev/null; then
   var1=$(date |  awk '{print $2" "}')  #
   var2=$(date +"'%g")
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
<font face=\"Verdana\" size=$fontsz><b>Netstats</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm
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
<font face=\"Verdana\" size=$fontsz><b>Free</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

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
<font face=\"Verdana\" size=$fontsz><b>Mixmaster</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

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
<font face=\"Verdana\" size=$fontsz><b>Mixmaster</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

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
   var10=$(date +"%G-%m-%d")
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
   <font face=\"Verdana\" size=$fontsz color=FF0000><b>Mix Errors</b></font></td></tr>
   <tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm
   echo "(This error list will be removed only by clearing the error.log)" > $filePath/templ.txt
   grep " Error: " $mixpath/error.log >> $filePath/templ.txt
#   grep " Error: " $mixpath/error.log > $filePath/templ.txt
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


##'------------------------------'
## BEGIN 1st remailer stats table
##'------------------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>Remailer Statistics (mixmin4096)</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

if [[ $(date +"%M") = "00" ]] || \
   [[ $(date +"%M") = "10" ]] || \
   [[ $(date +"%M") = "20" ]] || \
   [[ $(date +"%M") = "30" ]] || \
   [[ $(date +"%M") = "40" ]] || \
   [[ $(date +"%M") = "50" ]] || \
   [[ ! -s $filePath/astats.txt ]] || [[ $dostats = "y" ]]; then
   sleep 5  # pause on 1st stat collect for pingers to finish updating their stats
   wget  --no-check-certificate --timeout=15  -t 1 http://www.mixmin.net/echolot/mlist.txt -O $filePath/mixmin4096.txt
   echo $(date) > $filePath/statdate.txt
fi

savdate=$(< $filePath/statdate.txt)
grep "%"  $filePath/mixmin4096.txt | colrm 16 28 > $filePath/astats.txt
sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
sed -i "1i&nbsp;$savdate" $filePath/astats.txt
sed -i 's/$/<br>/' $filePath/astats.txt

#statcolor

rm $filePath/templ.txt

cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm
##'------------------------------'
##  END 1st remailer stats table
##'------------------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: vertical divider between 1st and 2nd remailer stats tables
##'------------------------------------'


##'------------------------------'
## BEGIN 2nd remailer stats table
##'------------------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>Remailer Statistics (sec3)</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

if [[ $(date +"%M") = "00" ]] || \
   [[ $(date +"%M") = "10" ]] || \
   [[ $(date +"%M") = "20" ]] || \
   [[ $(date +"%M") = "30" ]] || \
   [[ $(date +"%M") = "40" ]] || \
   [[ $(date +"%M") = "50" ]] || \
   [[ ! -s $filePath/astats.txt ]] || [[ $dostats = "y" ]]; then
   sleep 5  # pause on 1st stat collect for pingers to finish updating their stats
   wget  --no-check-certificate --timeout=15  -t 1 sec3.net/echolot/mlist.txt -O $filePath/sec3.txt
   echo $(date) > $filePath/statdate.txt
fi

savdate=$(< $filePath/statdate.txt)
grep "%"  $filePath/sec3.txt | colrm 16 28 > $filePath/astats.txt
sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
sed -i "1i&nbsp;$savdate" $filePath/astats.txt
sed -i 's/$/<br>/' $filePath/astats.txt

#statcolor

rm $filePath/templ.txt

cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm
##'------------------------------'
##  END 2nd remailer stats table
##'------------------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: vertical divider between 2nd and 3rd remailer stats tables
##'------------------------------------'


##'-------------------------------'
## BEGIN 3rd Mixmaster stats table
##'-------------------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>Remailer Statistics (borked)</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

if [[ $(date +"%M") = "00" ]] || \
   [[ $(date +"%M") = "10" ]] || \
   [[ $(date +"%M") = "20" ]] || \
   [[ $(date +"%M") = "30" ]] || \
   [[ $(date +"%M") = "40" ]] || \
   [[ $(date +"%M") = "50" ]] || \
   [[ ! -s $filePath/astats.txt ]] || [[ $dostats = "y" ]]; then
   sleep 5  # pause on 1st stat collect for pingers to finish updating their stats
   wget  --no-check-certificate --timeout=15  -t 1 pinger.borked.net/mlist.txt -O $filePath/borked.txt
   echo $(date) > $filePath/statdate.txt
fi

savdate=$(< $filePath/statdate.txt)
grep "%"  $filePath/borked.txt | colrm 16 28 > $filePath/astats.txt
sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
sed -i "1i&nbsp;$savdate" $filePath/astats.txt
sed -i 's/$/<br>/' $filePath/astats.txt

#statcolor

rm $filePath/templ.txt

cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm
##'------------------------------'
##  END 3rd remailer stats table
##'------------------------------'


##'------------------------------------'
echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: vertical divider between 3rd and 4th remailer stats tables
##'------------------------------------'


##'-------------------------------'
## BEGIN 4th Mixmaster stats table
##'-------------------------------'
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>Remailer Statistics (apricot)</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

if [[ $(date +"%M") = "00" ]] || \
   [[ $(date +"%M") = "10" ]] || \
   [[ $(date +"%M") = "20" ]] || \
   [[ $(date +"%M") = "30" ]] || \
   [[ $(date +"%M") = "40" ]] || \
   [[ $(date +"%M") = "50" ]] || \
   [[ ! -s $filePath/astats.txt ]] || [[ $dostats = "y" ]]; then
   sleep 5  # pause on 1st stat collect for pingers to finish updating their stats
   wget  --no-check-certificate --timeout=15  -t 1 https://apricot.fruiti.org/echolot/mlist.txt -O $filePath/apricot.txt
   echo $(date) > $filePath/statdate.txt
fi

savdate=$(< $filePath/statdate.txt)
grep "%"  $filePath/apricot.txt | colrm 16 28 > $filePath/astats.txt
sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
sed -i "1i&nbsp;$savdate" $filePath/astats.txt
sed -i 's/$/<br>/' $filePath/astats.txt

#statcolor

rm $filePath/templ.txt

cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm
##'------------------------------'
##  END 4th remailer stats table
##'------------------------------'



   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
##'--------------------------------------'
##'--------------------------------------'
##  END remailers stats horzontal tables
##'--------------------------------------'
##'--------------------------------------'


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
   <font face=\"Verdana\" size=$fontsz><b>Pool "-" $(find $mixpath/pool -type f | wc -l) "-" $(date +"%r")</font></td></tr>
   <tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

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


##'-------------------'
##'-------------------'
## BEGIN display mailq
##'-------------------'
##'-------------------'
vattest=$(mailq)
if [[ ! $vattest = "Mail queue is empty" ]]; then
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" size=$fontsz><b>Mailq</b></font></td></tr>
   <tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

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
##'-------------------'

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

# Lstats.sh
```
