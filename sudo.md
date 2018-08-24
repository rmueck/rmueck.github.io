#Einführung von zeitlich begrenzten *sudo* Berechtigungen für AIX und Linux



##Einleitung

Im täglichen Betrieb von Linux und AIX Systemen werden häufiger Requests nach temporären Hochberechtigungen (root) gestellt. 

Auch werden häufig `sudo`Berechtigungen im Kontext von Applikationen benötigt, die aber nur zeitlich begrenzt vergeben werden sollen.

Dies kann verschiedene Gründe haben; in der Regel werden diese Berechtigungen zur Installation oder Konfiguration von Software benötigt.

Da diese Hochberechtigungen natürlich ein erhöhtes Risiko darstellen, sollte ein Verfahren entwickelt werden, das die Berechtigungen nach Ablauf einer **definierten** Zeitspanne wieder entzieht.

Grundlage für die Erteilung jeder `sudo`Berechtigung ist ein Service-Request im ServiceNow. Hierdurch ist sichergestellt, dass jeder Request dokumentiert ist und einer Prüfung unterzogen wurde. Darüber hinaus ist sichergestellt, dass jeder Request mit einer eindeutigen Request-Nummer (**RITM**) verknüpft ist.

In diesem Dokument soll ein Verfahren für diese Anforderungen dargestellt/entwickelt  werden.



##Status Quo


Zum jetzigen Zeitpunkt werden die notwendigen Arbeiten manuell durchgeführt:

- Eintragung in `/etc/sudoers.d/99-custom` mit Hinweis auf das RITM, das die Grundlage darstellt.
- Entfernen des Eintrags nach Ablauf einer definierten Zeitspanne.

>  Gerade dieser Task stellt ein nicht unerhebliches Risiko dar, da z.Z hierfür kein hinreichender Prozess existiert und so unter Umständen Hochberechtigungen über das vorgesehene Zeitfenster hinaus existieren.



##Zielscenario

Im Zielzustand soll eine neue `sudoers`Datei eingeführt werden, in der ausschließlich zeitlich begrenzte Einträge enthalten sind. Diese Datei soll den Namen `100-tempsudoers`erhalten. Die datei `99-custom`dient weiterhin für Einträge die nicht durch ein `Puppet`Modul gemanaged werden aber trotzdem permanenten Charakter besitzen.

> Es sollte aber langfristig angestrebt werden vorhandene Einträge in dieser Datei unter die Kontrolle von `Puppet`zu bringen!



Es ist anzunehmen, dass Einträge in `99-custom` immer auch einen Bezug zu einer Applikation oder eines Dienstes besitzen und somit eine Kontrolle über entsprechende, von `Puppet` gemanagte `sudo` Module sinnvoll sind.



**Im Zielzustand sollen folgende Schritte Automatisiert bzw. Teilautomatisiert werden:**

- Eintragung/Löschen von `sudo`Berechtigungen auf Benutzer,- oder Gruppenebene mit Ablaufdatum und RITM Nummer. Dieser Teil ist bereits automatisiert für den Fall `User ALL=(ALL) ALL`
- Manuelle Eintragung von `sudo`Berechtigungen mit RITM-Nummer und Ablaufdatum in `100-tempsudoers`. Dieser Teil wird für komplexere `sudo` Konstrukte eingesetzt, da hier häufig auch Dinge wie `Cmnd_Alias` , `User_Alias` oder andere Direktiven zum Einsatz kommen.
- Automatisiertes Löschen aller Einträge in `100-tempsudoers` die das Ablaufdatum erreicht haben.
- Automatisiertes Löschen aller Einträge die **kein** Ablaufdatum besitzen.
- Benachrichtigung der Gruppe `GIS-CIN-UNX` per Email, wenn Berechtigungen entzogen wurden.
- Zyklisches Prüfen der `100-tempsudoers` auf Einträge **ohne** Ablaufdatum und optionales Löschen dieser Einträge und gleichzeitige Email Benachrichtigung



### Schematische Darstellung Zielscenario

####Automatische Eintragung und Löschung

~~~mermaid
graph LR

start((Start)) --> SNOW
SNOW((Service<br>Now)) -->|RTIM| Admin
Admin(Admin) -->Rundeck((Rundeck))


subgraph Automatic Prozess
Rundeck -->|Start Job| LINUX
Rundeck -->|Start Job| AIX
AIX -->at
LINUX --> at(scheduling)
at --> sudo(remove entry)
sudo --> email[send notification]
end

email--> End((End))
~~~

####Manuelle Eintragung

~~~mermaid
graph LR

start((Start)) --> SNOW
SNOW((Service<br>Now)) -->|RTIM| Admin
Admin(Admin) -->System((System))


subgraph Manual Prozess
System -->|Entry| LINUX
System -->|Entry| AIX
end

LINUX--> End((End))
AIX--> End((End))
~~~

#### Löschen von Einträgen

~~~mermaid
graph LR

start((Start)) --> Rundeck((Scheduling))
subgraph Rundeck
Rundeck  -->|Check Entry| SYSTEM{SYSTEM}
SYSTEM -->|Expired| DELETE
SYSTEM -->|Invalid| DELETE
DELETE --> NOTIFY

end
NOTIFY --> END((END))
~~~

## Implementation

###Automatische Eintragung und Löschung

Die Implementation dieses Prozesses wird mittels **Rundeck** (RD) realisiert. Der Prozess erfordert hierbei folgende Parameter:

- RITM-Nummer

- UserID (V-Key) oder Gruppenname

- Zeitraum (Vordefiniert: 1-6 Monate)

- VM/LPAR FQDN



Der RD Job implementiert hierbei:

- Eintragung in `/etc/sudoers.d/100-tempsudoers` nach folgendem Schema:

  ~~~sh
<V-Key/group> ALL=(ALL) ALL # granted till <Expire Date> by <RITM-Nummer> 
  ~~~

Beispiel:

~~~
 v761204 ALL=(ALL) ALL # granted till 2018-07-17 by RITM000000000
~~~

- Anlegen eines `at` Jobs im System, der nach Ablauf des Zeitfensters den Eintrag entfernt und gleichzeitig eine Email an den UNIX Briefkasten versendet. Hierbei soll das Subject der Mail folgendem Schema folgen:

  ~~~
  root access removed for <V-Key> on <LPAR/VM> [<RITM-Nummer>]
  ~~~
  Hierdurch erfolgt die Dokumentation der Entziehung der Hochberechtigung. Diese Jobs sind *rebootfest*, so das sichergestellt ist, das die Entziehung der Berechtigung zum festgelegten Zeitpunkt erfolgt.

>Dieser Prozess ist nur für Einträge der Form `<V-Key/group> ALL=(ALL) ALL` geeignet.



### Manuelles Eintragen und automatisches Löschen

Wie bereits weiter oben angeführt, ist eine Vollautomation für komplexere Konstrukte wenig sinnvoll bzw. nur mit nicht vertretbarem Aufwand zu realisieren. Die Einträge in der `100-temsudoers`werden auf Grundlage von *RITM* aus ServiceNow manuell vorgenommen und mit einem Ablaufdatum versehen.

Ein Rundeck Job prüft zyklisch auf Einträge, deren Ablaufdatum erreicht ist und löscht diese. Gleichzeitig wird eine Benachrichtigung per Email verschickt.

#### Beispiel

~~~sh
# Cmnd alias specification
Cmnd_Alias ECHO_CMD=/bin/echo A *,/bin/echo B * # RITM198293  EXP:2018-07-15

#userx
userx ALL=(root) NOPASSWD: /sbin/cmd1,/sbin/cmd2,ECHO_CMD # RITM198293  EXP:2018-07-15
~~~

In diesem Fall würden alle Einträge mit dem Ablaufdatum 15. Juli 2018 gelöscht werden.



### Rundeck Job zur Löschung

Ein zyklisch ablaufender Rundeck Job prüft täglich ob auf Systemen abgelaufene Einträge in der `100-tempsudoers` existieren. Liegen solche Einträge vor, werden diese gelöscht und es erfolgt eine Benachrichtigung über die Löschung per Email.

Darüber hinaus prüft der Job, ob Einträge **ohne** Ablaufdatum existieren. Diese werden ebenfalls gelöscht und eine Benachrichtigung erfolgt auch in diesem Fall.



### Ausblick

Wie bereits weiter oben angeführt, sollte eine `99-custom` auf lange Sicht überflüssig werden, nachdem eine Analyse der bestehenden Einträge durchgeführt wurde und eine Kategorisierung stattgefunden hat. Aus dieser Kategorisierung sollte sich ein eine Möglichkeit entwickeln lassen diese Einträge mit Hilfe von `Puppet` in gemanagte `sudo Module`zu überführen.

Eine weitere Möglichkeit der Automation wäre mittels Steuerdateien auch komplexere `sudo` Konstrukte ohne direkten Eingriff auf dem System selber weiter zu automatisieren. Diese könnten in einem Repository auf dem Rundeck Server gepflegt werden und durch einen Rundeckjob auf die Zielsysteme verteilt werden.



#### Konsolidierung und Bereinigung der bestehenden 99-custom

Eine erste Betrachtung der derzeitigen Situation der `99-custom` hat insbesondere im Linux Umfeld gezeigt, das hier in der Vergangenheit eine Situation geschaffen wurde, die nur schwer in den Griff zu bekommen ist.







------



## Anhang

### Code (Rundeck):

#### Linux

```sh
#!/bin/sh

export RTIMID=@option.rtimid@

# Log details
echo "Granting root access to user @option.userid@ for a period of @option.time_period@ weeks"

# Write entry into 100-tempsudoers
echo "@option.userid@ ALL=(ALL) ALL # $(date -I) granted for @option.time_period@ week(s) @option.rtimid@ ; RD-Job: @job.name@ / ID: @job.execid@" >> /etc/sudoers.d/100-tempsudoers

# Schedule job
at now + @option.time_period@ minute <<< "sed -i "/$RTIMID/d" /etc/sudoers.d/100-tempsudoers ; \
mail -r rundeck@lde5003p -s 'root access removed for @option.userid@ on $(hostname) [@option.rtimid@]' ruediger.mueck@generali.com"

# Log Job ID
echo "The job id is $(cat /var/spool/at/.SEQ)" 
```



#### AIX

```bash
#!/usr/bin/bash

export RTIMID=@option.rtimid@

echo "Granting root access to user @option.userid@ for a period of @option.time_period@ weeks"

echo "@option.userid@ ALL=(ALL) ALL # $(date "+%d-%m-%Y") granted for @option.time_period@ week(s) @option.rtimid@ ; RD-Job: @job.name@ / ID: @job.execid@" >> /etc/sudoers.d/99-custom

# schedule job

# TODO: change to week after test
at now + @option.time_period@ minute <<< "perl -i.bak -ne \"/$RTIMID/ || print \"  /etc/sudoers.d/100-tempsudoers ; \
mail -r rundeck@lde5003p -s 'root access removed for @option.userid@ on $(hostname) [@option.rtimid@]' ruediger.mueck@generali.com < /etc/sudoers.d/100-tempsudoers" 

#echo "The job id is $(cat /var/spool/at/.SEQ)"
```



####100-tempsudoers (Beispiel)

~~~shell
# DO NOT ALTER THIS HEADER - IT IS MENT TO ACT FOR DOCUMENTATION PURPOSES!
#
# Do NOT forget to add a EXP:<EXPIRY-DATE> statement in the format YYYY-MONTH-DAY
# after the entry itself. You should also add the corresponding RITM within  the entry
#
# Example:
#
#       Cmnd_Alias ECHO_CMD=/bin/echo A *,/bin/echo B * # RITM198293  EXP:2018-07-15
#       userx ALL=(root) NOPASSWD: /sbin/cmd1,/sbin/cmd2,ECHO_CMD # RITM198293  EXP:2018-08-15
#
#
# Please note that "illegal" entries without an expiry date are deleted and a
# notification mail is send to the UNIX admins!
#
# A notification Email is also send to the UNIX admins when an entry is removed (expired)
#
# Please do NOT use blank line to separate entries (they will be deleted!)
# Use a # sign instead!
#
# This file is checked by a Rundeck job for expired and "illegal" entries
# which will be deleted.
#
Cmnd_Alias ECHO_CMD=/bin/echo A *,/bin/echo B * # RITM198293  EXP:2018-07-15
userx ALL=(root) NOPASSWD: /sbin/cmd1,/sbin/cmd2,ECHO_CMD # RITM198293  EXP:2018-09-15
userx ALL=(root) NOPASSWD: /sbin/cmd1,/sbin/cmd2,ECHO_CMD # RITM198293  EXP:2017-08-15
#
#
Cmnd_Alias AWSS3_CMD = /usr/local/bin/aws s3 cp *, /usr/local/aws/bin/aws s3 cp *
userb ALL=(root) NOPASSWD: AWSS3_CMD
~~~



#### sed

~~~bash
#illegal
sed   '/EXP\|^#\|^$/d' ~/100-tempsuoers > illegal
#=> SEND MAIL!

# expired 
sed  '/'"${EXPDATE}"'\|^$\|^#/!d' 100-tempsuoers |sed '/^#/d' > expired
#=> SEND MAIL!

# Fix it
sed  '/'"${EXPDATE}"'\|^$/d' 100-tempsuoers |sed  '/EXP\|^#\|^$/!d'
~~~


