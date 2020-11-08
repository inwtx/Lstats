# Lstats

This script can be used by remailer sysops to monitor their server and remailer performance.  
Copy the script code below into a file called Lstats.sh and make it executable (sudo chmod 755 Lstats.sh).  
Then execute it with a cron as explained in at the top of the script code.  
  
  
```
#!/bin/bash
#
# Script to build server statistics Lstats.html
#
#------------------------------------------------------------------------#
#         --- Mixmaster Server statistics web page builder ---           #
#                                                                        #
# The following script can be used to monitor a remailer servers.        #
# Several statistics concerning the server and mixmaster are displayed.  #
# The script is executed as: servstats.sh <remailer-name>                #
# The lstats.sh can be executed MUST be executed every 1 minute and      #
# must executed at 0000 hours to reset the pool count display to zero.   #
# The script must not be execute at the same time with former Lstats     #
# scripts or the results will be unpredictable.                          #
#                                                                        #
# Retrieve stastics for server web page - cron job                       #
# */1 * * * * /path/to/lstats.sh <remailer short name>                   #
#                                                                        #
#------------------------------------------------------------------------#

#top
export PATH=$PATH:/sbin
export PATH=$PATH:/bin

### 'BEGIN These parameters may need to be modified.' ###
webpgnm="/Lstats.html"
mixmastername="mixmaster"
mixpath="/var/mixmaster"       # no trailing /
webpgpath="/var/www/html"      # no trailing /
sshlog="/var/log/auth.log"
erroremail=""
### 'END These parameters may need to be modified.' ###


tempdisp="yes"
dostats="n"
expdwarn=30
filePath=${0%/*}  # current file path
stathighlight="#ff1493"
varupt=$(uptime)
remailerid="$1"
mmid=$remailerid
serverid="$remailerid - "
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


function statcolor(){      # stat coloring function
    var="<font size=$fontsz color="c94c4c"><b>" && awk -vvar2=$(cut -d'.' -f1 <<<" ") -vf1="$var" 'index($0, var2){print f1;print;next}1' \
    $filePath/astats.txt > $filePath/templ.txt  # non 100.00% color
    var="</b></font>" && awk -vvar2=$(cut -d'.' -f1 <<<"100.00%") -vf1="$var" 'index($0, var2){print;print f1;next}1' \
    $filePath/templ.txt > $filePath/astats.txt  # non 100% color

    var="<font size=$fontsz color="000080"><b>" && awk -vvar2=$(cut -d'.' -f1 <<<"100.00%") -vf1="$var" 'index($0, var2){print f1;print;next}1' \
    $filePath/astats.txt > $filePath/templ.txt  # 100.00% color
    var="</b></font>" && awk -vvar2=$(cut -d'.' -f1 <<<"100.00%") -vf1="$var" 'index($0, var2){print;print f1;next}1' \
    $filePath/templ.txt > $filePath/astats.txt

    var="<font size=$fontsz color="$stathighlight"><b><u>" && awk -vvar2=$(cut -d'.' -f1 <<<$remailerid) -vf1="$var" 'index($0, var2){print f1;print;next}1' \
    $filePath/astats.txt > $filePath/templ.txt  # remailer shortname color
    var="</u></b></font>" && awk -vvar2=$(cut -d'.' -f1 <<<$remailerid) -vf1="$var" 'index($0, var2){print;print f1;next}1' \
    $filePath/templ.txt > $filePath/astats.txt

    var="<font size=$fontsz color="000000"><b>" && awk -vvar2=$(cut -d'.' -f1 <<<"CDT") -vf1="$var" 'index($0, var2){print f1;print;next}1' \
    $filePath/astats.txt > $filePath/templ.txt # title color
    var="</b></font>" && awk -vvar2=$(cut -d'.' -f1 <<<"CDT") -vf1="$var" 'index($0, var2){print;print f1;next}1' \
    $filePath/templ.txt > $filePath/astats.txt
} # end statcolor


cat /dev/null > $webpgpath/$webpgnm  # clear html file


if [[ $1 == "" ]]; then
   echo "<html><head><title>Server Stats</title></head><body bgcolor=\"$bgclr\" TEXT=\"$fontcolor\" LANG=\"en-US\" DIR=\"LTR\">" > $webpgpath/$webpgnm
   echo "<font face=\"Verdana\" size=3 color=\"$fontcolor\"><b>&nbsp;" >> $webpgpath/$webpgnm
   echo "Error: Lstat.sh script must have your remailer short name appended when starting.<br>" >> $webpgpath/$webpgnm
   echo " &nbsp;&nbsp;Example:  Lstat.sh &ltremailer short name&gt" >> $webpgpath/$webpgnm
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
   echo "</body></html>" >> $webpgpath/$webpgnm
   exit 1
fi


echo "<html><head><title>Server Stats</title></head><body bgcolor=\"$bgclr\" TEXT=\"$fontcolor\" LANG=\"en-US\" DIR=\"LTR\">" > $webpgpath/$webpgnm

#date begin
echo "<font face=\"Verdana\" size=$fontsz color=\"$fontcolor\"><b>&nbsp;" >> $webpgpath/$webpgnm
echo $serverid $(date | cut -c 1-10 && date | cut -c 25-28 && echo "-" && date | cut -c 12-23) - ${0##*/} >> $webpgpath/$webpgnm
echo "</font></b><br>" >> $webpgpath/$webpgnm
#date begin end

##echo "<table width=\"$MachineRogueTableWidth\"><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm  # BEGIN: divider table for Machine and Rogue
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm  # BEGIN: divider table for Machine and Rogue

# machine begin
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<b><font face=\"Verdana\" size=$fontsz><b>Machine - Netherlands</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

## linux info
echo "$(lsb_release -d) $(uname -m)<br>" | sed -e 's/Description://g' | tr -d "\t" >> $webpgpath/$webpgnm

# ip address
echo "IP server: $(hostname -I)<br>" >> $webpgpath/$webpgnm

# domain name
echo "Domain name: $(dig -x $(hostname -I) +short | awk -F '.' '{print $1"."$2}')<br>" >> $webpgpath/$webpgnm

# MTU
echo "$(ifconfig | sed '1!d' | awk '{print "MTU: " $4}')<br>" >> $webpgpath/$webpgnm

## openssl
echo "$(openssl version)" | awk '{print $1" "$2" ("$3" "$4" "$5")<br>"}' >> $webpgpath/$webpgnm

## last boot time
varbt=`echo $(who -b)`
echo ${varbt/system boot/Last system boot:} >> $webpgpath/$webpgnm

## up time
varupt=`echo ${varupt//up/Up}`
varupt=`echo ${varupt//load/Load}`
awk -F '[ \t\n\v\r]' '{print "<br>"$2" "$3" "$4" "$5" "$8" "$9" "$10" "$11" "$12" "$13" "$14" "$15}' <<< $varupt >> $webpgpath/$webpgnm


###Bandwidth
  ###Month to date bandwidth
if pidof -x "vnstatd" >/dev/null; then
   var1=$(date |  awk '{print $3" "}')  #
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
# machine end

echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm

echo "</td><td>" >> $webpgpath/$webpgnm

echo "</td></tr></table><br>" >> $webpgpath/$webpgnm

echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE: divider table for Machine and Rogue

# netstat 2
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>Netstats</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm
netstat -vatnp > $filePath/templ.txt
sed -i "/python2.7/d" $filePath/templ.txt  # remove bitmessage python2.7 messages
sed -i "/nginx\: worker/d" $filePath/templ.txt  # remove nginx: worker messages
sed -i "/TIME_WAIT/d" $filePath/templ.txt  # remove TIME_WAIT messages
sed -i 's/ /\&nbsp;/g' $filePath/templ.txt
sed -i 's/ //g' $filePath/templ.txt
sed -e 's/$/<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
# netstat 2 end

# free begin
echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>Free</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

free > $filePath/templ.txt
sed -i 's/ /\&nbsp;/g' $filePath/templ.txt
sed -i 's/ //g' $filePath/templ.txt
sed -e 's/$/\&nbsp;\&nbsp;<br>/' $filePath/templ.txt >> $webpgpath/$webpgnm  # append <br>
echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
# free end

echo "</td></tr></table>" >> $webpgpath/$webpgnm  # END: divider table for Machine and Rogue

#Mix Files, Misc Stats, Remailer Stats
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm  # BEGIN divider table for Mix Files, Misc Stats, and Remailer stats

# misc stats

echo "</td><td>" >> $webpgpath/$webpgnm

# mix begin
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

   if [[ ! $(which mutt) == "" ]]; then
      if [ $(expr $(date +"%H") % 4) -eq 0 -a $(date +"%M") = "00" -a $varmq -gt 200 ] && [[ ! $erroremail == "" ]]; then   # mail every 4 hours if mailq > 9
         mutt -s "Error from server -  Mailq backup - $varmq" $erroremail <<<"From Lstats.sh: Mailq backup - $varmq"
      fi
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

##$filePath/poolcount.sh
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
           echo "$(ls $mixpath/pool/)" > $filePath/savepool.txt      # save current pool at BOD for comparison
           exit 0
        fi

        savetodaypoolcnt=$(head -n 1 $filePath/savetodaypoolcnt.txt)
        savepriorpoolcnt=$(sed -n 2p $filePath/savetodaypoolcnt.txt)
        echo "$(ls $mixpath/pool/)" > $filePath/temppool.txt         # get current pool in seq list

        while read line ; do
           if grep $line $filePath/savepool.txt ; then  # is this message in savepool?
                 continue                                               # yes
                 else
                 ((savetodaypoolcnt++))                                      # found a new message
                 echo $line >> $filePath/savepool.txt                        # save the message file name for future ref
           fi
        done<$filePath/temppool.txt                                      # read file line by line

        echo $savetodaypoolcnt > $filePath/savetodaypoolcnt.txt                   # save the pool cnt for next chk
        echo $savepriorpoolcnt >> $filePath/savetodaypoolcnt.txt                  # save the pool cnt for next chk

        rm $filePath/temppool.txt

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

## mbox count
if [[ $(wc -l /etc/mbox | awk '{print $1}') -gt 0 ]]; then  # line cnt > 0
   echo "<font color=FF0000><b>mbox count:&nbsp; $(wc -l /etc/mbox | awk '{print $1}')<br></b></font>" >> $webpgpath/$webpgnm
   else
   echo "mbox count:&nbsp; 0<br>" >> $webpgpath/$webpgnm
   fi

## error count
if [[ $(grep -c 'Error:' $mixpath/error.log) -ne 0 ]]; then
   echo "<font color=FF0000><b>error count: $(grep -c 'Error:' $mixpath/error.log - (Remove this error message by clearing error.log.))<br></b></font>" >> $webpgpath/$webpgnm
   else
   echo "error count: 0<br>" >> $webpgpath/$webpgnm
   fi

## allpinger/tls dates
var1=$(grep "# Updated:" $mixpath/allpingers.txt)
var2=$(awk -F '[ \t\n\v\r.]' '{print $4" "$5" "$6}' <<< $var1)
var3="all:&nbsp; $(date --date "$var2" +%b" "%d" "%Y)<br>"
echo $var3 >> $webpgpath/$webpgnm

echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm   # end build Misc
# mix end

echo "</td><td>" >> $webpgpath/$webpgnm

# mixmaster begin
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
#mixmaster end

echo "</td><td>" >> $webpgpath/$webpgnm

echo "</td></tr></table>" >> $webpgpath/$webpgnm  # end double wide divider table for Mix Files and Misc Stats
# Misc Stats, Mix Files, Remailer Stats END

# error log msgs begin
if [[ $(grep -c " Error: " $mixpath/error.log) -gt 0 ]] ; then
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" size=$fontsz color=FF0000><b>Mix Errors</b></font></td></tr>
   <tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm
   grep " Error: " $mixpath/error.log > $filePath/templ.txt
   sed -i 's/$/<br>/' $filePath/templ.txt   # add <br> to end of every rec
   cat $filePath/templ.txt >> $webpgpath/$webpgnm
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
fi
# error log msg end

# all stats display begin
echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm

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

statcolor

rm $filePath/templ.txt

cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm

echo "</td><td>" >> $webpgpath/$webpgnm


if true; then
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

statcolor

rm $filePath/templ.txt

cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm

echo "</td><td>" >> $webpgpath/$webpgnm
fi


if true; then

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
   wget  --no-check-certificate --timeout=15  -t 1 pinger.borked.net/mlist.txt -O $filePath/borked.txt
   echo $(date) > $filePath/statdate.txt
fi
#t5
savdate=$(< $filePath/statdate.txt)
grep "%"  $filePath/borked.txt | colrm 16 28 > $filePath/astats.txt
sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
sed -i "1i&nbsp;$savdate" $filePath/astats.txt
sed -i 's/$/<br>/' $filePath/astats.txt

statcolor

rm $filePath/templ.txt

cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm

echo "</td><td>" >> $webpgpath/$webpgnm

fi

if false; then

echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
<font face=\"Verdana\" size=$fontsz><b>Remailer Statistics (zip2)</b></font></td></tr>
<tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

if [[ $(date +"%M") = "00" ]] || \
   [[ $(date +"%M") = "10" ]] || \
   [[ $(date +"%M") = "20" ]] || \
   [[ $(date +"%M") = "30" ]] || \
   [[ $(date +"%M") = "40" ]] || \
   [[ $(date +"%M") = "50" ]] || \
   [[ ! -s $filePath/astats.txt ]] || [[ $dostats = "y" ]]; then
   wget  --no-check-certificate --timeout=15  -t 1 zip2.in/echolot/mlist.txt -O $filePath/zip2.txt
   echo $(date) > $filePath/statdate.txt
fi

savdate=$(< $filePath/statdate.txt)
grep "%"  $filePath/zip2.txt | colrm 16 28 > $filePath/astats.txt
sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
sed -i "1i&nbsp;$savdate" $filePath/astats.txt
sed -i 's/$/<br>/' $filePath/astats.txt

statcolor

rm $filePath/templ.txt

cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm

echo "</td><td>" >> $webpgpath/$webpgnm

fi

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

   wget  --no-check-certificate --timeout=15  -t 1 apricot.fruiti.org/echolot/mlist.txt -O $filePath/apricot.txt
   echo $(date) > $filePath/statdate.txt
fi

savdate=$(< $filePath/statdate.txt)
grep "%"  $filePath/apricot.txt | colrm 16 28 > $filePath/astats.txt
sed -i 's/ /\&nbsp;/g' $filePath/astats.txt
sed -i 's/^/\&nbsp;/' $filePath/astats.txt       # prepend a blank
sed -i 's/$/\&nbsp;/' $filePath/astats.txt       # append a blank
sed -i "1i&nbsp;$savdate" $filePath/astats.txt
sed -i 's/$/<br>/' $filePath/astats.txt

statcolor

rm $filePath/templ.txt

cat $filePath/astats.txt >> $webpgpath/$webpgnm
echo "</font></td></tr></table>" >> $webpgpath/$webpgnm

# all remailer stats end
#t3
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm

# all remailer stats display end

echo "<table><tr valign=\"top\"><td>" >> $webpgpath/$webpgnm  # BEGIN

# pool list begin
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" size=$fontsz><b>Pool "-" $(find $mixpath/pool -type f | wc -l) "-" $(date +"%r")</font></td></tr>
   <tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm
   cat /dev/null > $filePath/templ.txt
   for i in $mixpath/pool/* ; do data=$(stat $i --format="%y") && echo "${data%.*} $i $(sed -n -e 2p $i)" >> $filePath/templ.txt ; done
   sed -i 's/\/home\/mix\/Mix\/pool\///g' $filePath/templ.txt
   sed -i 's/\/var\/mixmaster\/pool\///g'  $filePath/templ.txt
   sed -e 's/$/<br>/' $filePath/templ.txt | sort >> $webpgpath/$webpgnm  # append <br> to each line in templ.txt
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm
# pool list end

echo "</td><td>" >> $webpgpath/$webpgnm  # MIDDLE

# display multiple web page hits begin
cat /dev/null > $filePath/temp20.txt
cat /dev/null > $filePath/temp21.txt
cat /dev/null > $filePath/AccessLogHits.txt

cat /var/log/nginx/access.log > $filePath/temp19.txt
cat /var/log/nginx/access.log.1 >> $filePath/temp19.txt
tail -1000000 $filePath/temp19.txt | awk '{print $1}' | sort | uniq -c |sort -n > $filePath/temp20.txt

while read line1; do
      if [[ $(awk '{print $1}') -gt 40 ]] <<< $line1; then
         if [[ ! $line1 =~ "95.85.40.163" ]] && [[ ! $line1 =~ "127.0.0.1" ]] && [[ ! $line1 =~ "50.26.242.136" ]] && [[ ! $line1 =~ "23.237.26.103" ]]; then
            printf '%-5s %-17s\n' $(awk '{print $1" "$2}' <<< $line1) >> $filePath/temp21.txt
         fi
      fi
done< $filePath/temp20.txt # read file line by line

while read line1; do
      varip=$(awk "NR==1{print;exit}" <<< $line1 | whois $(awk '{print $2}') | grep country | awk '{print $2}' | sed '$!N; /^\(.*\)\n\1$/!P; D')  # pull out country code
      line2=$line1" "$varip
      varx1=$(printf '%-5s %-16s %-3s\n' $(awk '{print $1" "$2" "$3}' <<< $line2))
      varx2=${varx1// /\&nbsp\;}
      echo $varx2 >> $filePath/AccessLogHits.txt
done< $filePath/temp21.txt

if [ -e $filePath/AccessLogHits.txt ] && [[ $(cat $filePath/AccessLogHits.txt | wc -l) -gt 0 ]]; then
   echo "<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\"><tr><td bgcolor=\"$titlecolor\">
   <font face=\"Verdana\" size=$fontsz><b>Web access.log hits &gt 20</b></font></td></tr>
   <tr><td><font face=\"Courier New\" size=$fontsz color=\"$fontcolor\"><b>" >> $webpgpath/$webpgnm

   sed -e 's/$/<br>/' $filePath/AccessLogHits.txt >> $webpgpath/$webpgnm
   echo "</font></td></tr></table><br>" >> $webpgpath/$webpgnm

   rm $filePath/temp19.txt
   rm $filePath/temp20.txt
   rm $filePath/temp21.txt
fi
# display multiple web page hits end

echo "</td></tr></table>" >> $webpgpath/$webpgnm  # END

# display mailq begin
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
# display mailq end

echo "</body></html>" >> $webpgpath/$webpgnm

#if false; then
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
#fi

exit 0

# Lstats.sh
```
