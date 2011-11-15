---
layout: post
title: "Hubot telepítése a felhőbe"
date: 2011-11-12 20:57
comments: true
author: Egyed Gábor (1ed)
categories: node.js hubot cloud
keywords: node.js, hubot, cloud, smartmachine
description: Hogyan telepítsünk Hubotot SmartMachine-re
published: true
---


A [github][] néhány hete mindenki számára elérhetővé tette a [Hubot][] névre hallgató chat
botot. A Hubot nem más mint egy [node.js][] szerver, ami különböző adapterek segítségével tud a
külvilággal kommunikálni. Például egy chat szobához csatlakozva várja a többiek parancsait,
kérdéseit, amikre válaszol. Keres nekünk a youtube-on, különböző valuta árfolyamok között
átváltást végez, kinyitja az ajtót, zenét játszik, SMS-t küld.

<!--more-->

A projekt igen nagy népszerűségnek örvend. A kezdetekben is meglévő [Campfire][] és IRC adapterek
mellett sorra jelentek meg az újak, így ma már 12 adapter közül lehet választani, köztük pl.
twitter, twillo, flowdock, gtalk. A gtalk adapternek köszönhetően pedig elég egy gmail account
ahhoz, hogy saját Hubotunk legyen, akivel a GTalk chaten keresztül beszélhetünk. Persze egy
hely is kell, ahol a bot fut és fogadja a parancsainkat. A github erre a [Heroku][] nevű, ingyenesen
is elérhető cloud szolgáltatást ajánja. Azonban hiába ingyenes, az extra szolgáltatások,
mint pl. Redis szerver (ami szükséges a Hubot futtatásához), használatához szükség van
egy bankkártya számra. Ha ezt nem szeretnénk megadni, vagy nincs megfelelő bankkártyánk,
szerencsére van egy másik, szintén ingyenes megoldás is. A [Joyent][] node.js cloud szolgáltatása.
Most ebben a felhőben fogunk életre kelteni egy Hubotot.


## 1. Node SmartMachine létrehozása

Először szükség lesz egy szerverre a felhőben, ehhez regisztráljunk be [http://no.de][node]
oldalon. A regisztrációkor meg kell adni egy RSA kulcsot, ha még nincs, akkor az `ssh-keygen -t
rsa -C "saját@email.com"` parancsal generálhatunk egy kulcspárt. Ennek a publikus részért kell
megadnunk a regszitráció során (alapértelmezetten a `~/.ssh/id_rsa.pub` fájlban található).
Ha túl vagyunk a regisztráción az "Order a machine" gombra kattintva hozhatunk létre egy új
szervert. Ahhoz, hogy az új szerverünkhöz csatlakozni tudjunk, a kapott beállításokat el kell
helyeznünk a `~/.ssh/config` fájlban, pl.:

```
Host account.no.de
  Port 60799
  User node
  ForwardAgent yes
```

Az `ssh account.no.de` paranccsal tudunk csatlakozni a szerverünkhöz. A belépéshez az RSA
kulcshoz tartozó jelszónkat kell megadni.


## 2. A Redis telepítése

A Hubotnak szüksége lesz egy Redis szerverre. A következő parancsokkal telepíthetünk egyet a
szerveren:

```
pkgin update; pkgin install redis
svccfg import /opt/local/share/smf/manifest/redis.xml
svcadm enable redis
```


## 3. Hubot beállítása

Valahogy azt is tudatni kell a robotunkkal, hogy hogyan kommunikálhat a külvilággal, vagyis ki
kell választani a megfelelő adaptert és konfigurálni kell azt. Az adaptereket különböző
környezeti változók segítségevel lehet beállítani, amiket a `/home/node/node-service/profile`
fájlban kell elhelyezni, ha még nincs ilyen akkor hozzunk létre:

``` bash
cd ~; mkdir node-service
vim node-service/profile
```

Majd a megnyíló szerkesztőbe az `i` megnyomása után írjuk be a gtalk adapterhez szükséges
változókat, majd ESC és `:wq`, pl.:

``` bash
HUBOT_GTALK_USERNAME="felhasználó@gmail.com"                                # milyen userrrel lépjen be a bot
HUBOT_GTALK_PASSWORD="password"                                             # mi a jelszava
HUBOT_GTALK_WHITELIST_DOMAINS="foo.com,bar.com,domain.com"                  # domainek, ahonnan beléphetnek (opcionális)
HUBOT_GTALK_WHITELIST_USERS="másik_felhasználó@gmail.com,mégegy@gmail.com"  # kik azok a userek akikkel beszélhet (opcionális)
```

Az utolsó két beállítás segítségével lehet meghatározni, hogy a robot kivel beszélhet.
Használatuk nem kötelező, ha nem adjuk meg akkor a Hubot mindenkivel szóbaáll.


## 4. Hubot életre keltése

Töltsük le a legújabb stabil verziót, majd csomagoljuk ki:

```
wget https://github.com/downloads/github/hubot/hubot-1.1.11.tar.gz
tar xvf hubot-1.1.11.tar.gz
```

Ahhoz, hogy a távoli szerver automatikusan elindítsa a robotot, adjuk hozzá a `package.json`
fájlhoz a `, "scripts": {"start": "./bin/hubot --adapter gtalk" }` sort, tehát a fájlunk most
valahogy így kell, hogy kinézzen:

``` javascript package.json
{
  "name":        "hosted-hubot",
  "version":     "0.1.0",
  "author":      "GitHub Inc.",
  "keywords":    "github hubot campfire bot",
  "description": "A simple helpful Robot for your Company",
  "licenses":     [{
    "type":       "MIT",
    "url":        "http://github.com/github/hubot/raw/master/LICENSE"
  }],

  "repository" : {
    "type" : "git",
    "url" : "http://github.com/github/hubot.git"
  },

  "dependencies": {
    "hubot": "1.1.11",
    "hubot-scripts": "1.1.7",
    "optparse": "1.0.3"
  },

  "scripts": {"start": "./bin/hubot --adapter gtalk" }
}
```

Szükség lesz még egy másik konfigurációs fájlra, ami megmondja, hogy melyik node.js veziót
szeretnénk használni:

``` bash
echo '{"version": "v0.4.11"}' > config.json
```

Végül pedig git segítségével fel kell töltenünk a szerverre:

``` bash
git init
git commit -a -m "initial commit"
git remote add account.no.de account.no.de:repo
git push account.no.de master
```

Ha a sok szöveg végén azt látjuk, hogy `win!`, akkor a robutunk elvileg elindult. A szerverre
belépve és a `node-service-log` parancsot beírva ellenőrizhetjük is, hogy minden rendben volt-e,
ha az utolsó sorban azt látjuk, hogy `Hubot is online, talk.google.com!` akkor a Hubotunk életre
k]lt és várja az utasításainkat.

A `Hubot help` (a `Hubot` rész nem kötelező) parancs kiírja, hogy mi az amit megért. Például
a `youtube me hubot` a `hubot` kifejezésre keres a youtube-on és a találatok közül visszad
egyet véletlenszerűen, vagy `image me hubot` egy képet ad vissza. A `TIME` kiírja a szerveridőt
`calculate me 4+38` visszadja, hogy `42`, a `convert me 1000 HUF to EUR` pedig átváltja a forintot
eurora, stb. Ezek a parancsok általában valamilyen publikus szolgáltatlást (leginkább google)
használnak és inkább szórakoztatóak mint hasznosak. A Hubot nagyszerűsége igazából abban
rejlik, hogy nagyon könnyen bővíthető (tanítható). A megoldható feladatoknak szinte csak
a fantáziánk szab határt, pl. megtaníthatjuk a robotunkat, hogy nyissa ki az iroda ajtaját
(ha van valamilyen beléptető rendszerünk), játszon zenét vagy mondjon be valamit az irodai
hangosbemondón, szóljon ha valaki frissített egy éles oldalt, vagy ha valami baj van az egyik
általunk fejlesztett oldallal, a MAC cím alapján megmondja ki van az irodában, vagy SMS-t
küldjön a kollégáknak. Mindezt különboző scriptek segítségével lehet megtanítani
vele. A [hubot-scripts][] oldalon találhatunk néhány ilyen publikus scriptet, amiket a
`hubot-scripts.json` fájlhoz hozzáadva lehet használatba venni.

[github]:         http://github.com                        "Github"
[Hubot]:          http://hubot.github.com                  "Hubot"
[node.js]:        http://node.js                           "node"js"
[Campfire]:       http://campfirenow.com                   "Campfire"
[Heroku]:         http://www.heroku.com                    "Heroku"
[Joyent]:         http://www.joyent.com                    "Joyent"
[node]:           http://no.de                             "No.de"
[hubot-scripts]:  https://github.com/github/hubot-scripts  "Hubot scripts"
