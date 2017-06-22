---
layout: page-fullwidth
title: Vollständig
published: true
author: mwerner
comments: true
meta_description: "Bash kann bekanntlich nicht nur Filenamen komplettieren. Man kann auch eigene Erweiterungen schreiben"
date: 2013-11-19 06:11:58
tags:
    - bash
categories:
    - computer
permalink: /2013/11/19/vollstaendig
---
Die Bourne-Again-Shell, bekannt unter dem Kurznamen `bash`, hat viele nette Eigenschaften.
  
Eine ist die Vervollständigung von Pfad-Namen. Dazu tippt man bekanntlich den Anfang ein und drückt die Tabulator-Taste. Nicht ganz so bekannt ist, dass die Vervollständigungsfunktion auch für andere Elemente als Filenamen funktioniert.

Mit der Verbreiteten Erweiterung `bash_completion` wird z.B. bei Programmen wie `ssh` oder `telnet`, die als Argument einen Rechnernamen haben, dieser Name entsprechend ergänzt. Dazu müssen die möglichen Rechner in der Datei `/etc/hosts` gelistet sein. Oder es werden bei Programmen wie `su` die Nutzernamen aus `/etc/passwd` ergänzt.

Es ist auch nicht schwierig, eigene Erweiterungen zu schreiben. So habe ich in meinen Directory-Baum ein Verzeichnis, die ich häufig als Ausgangspunkt zur Dateisuche nutze, als eine Art alternatives Home-Verzeichnis (konkret ist dies ein Link auf ein aktuelles Projektverzeichnis). Dafür habe ich in meiner `.bash_profile` die Zeile
{% highlight bash linenos %}
function cdc () { cd ~/Doc/current/$*; }
{% endhighlight %}
die als ein spezieller Change-Directory-Befehl mit relativen Bezug zu dem current-Directory arbeitet.
  
Durch folgenden (nicht ganz optimierten) Code funktioniert die Bash-Vervollständigung auch für `cdc`:
{% highlight bash linenos %}
# Completation
function cmpl_path(){
local cur pre ctest

pre=$1
cur=$pre${COMP_WORDS[COMP_CWORD]:-*}
ctest=`echo $cur*/ | cut -f 1 -d' ' `

if [ -z "${COMP_WORDS[COMP_CWORD]}" ]; then
  COMPREPLY=( `ls -d -R $cur* | sed -e "s#${pre}##g" ` );
elif [ -d $ctest ]; then
  COMPREPLY=( `ls -d -R $cur*/ 2&gt; /dev/null | sed -e "s#${pre}##g" ` );
else
  COMPREPLY='';
echo -n '\a' 
fi
}

function cmpl_current() {
cmpl_path "/Users/mwerner/Doc/current/"
}

complete -o nospace -F cmpl_current cdc
{% endhighlight %}
Die Bedeutung der Variablen `COMP_WORDS` und `COMPREPLY` kann in der Bash-Manualseite nachgeschlagen werden.
  
Wie man sieht, können leicht weitere Funktionen für andere Spezial-cd-Befehle definiert werden.
