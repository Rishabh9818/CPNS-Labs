# 12. Vaja: Upravljanje omrežij z protokolom RADIUS

## Navodila

0. Uporabite omrežje in navidezne računalnike iz prejšnjih vaj. 
1. Postavite strežnik RADIUS na prvem navideznem računalniku ter preverite njegovo delovanje.
2. Namestite apache2 na prvem navideznem računalniku in ustvari neko preprosto spletno stran. Poskrbite, da bo dostop do spletne strani omejen z uporabniškim imenom in geslom. Prav tako poskrbite, da bo apache2 za overitev uporabljal RADIUS.
3. Ustvari svoje kraljestvo za overitev (RADIUS realm). Ustvarite tudi RADIUS strežnik s svojim kraljestvom še na drugem navideznem računalniku in omogočite dostop do spletne strani uporabnikom iz tega kraljestva.

## Dodatne informacije

[Remote Authentication Dial-In User Service (RADIUS)](https://en.wikipedia.org/wiki/RADIUS) je omrežni protokol, ki zagotavlja centralizirano overitev, pooblaščanje in beleženje (Authentication, Authorization and Accounting - AAA) za uporabnike, ki uporabljajo omrežne storitve.

[Overitev](https://en.wikipedia.org/wiki/Authentication) je postopek preverjanja istovetnosti identitete.

[Pooblaščanje](https://en.wikipedia.org/wiki/Authorization) je postopek dodelitev pravic uporabniku za dostop do zahtevanega vira.

[Beleženje](https://en.wikipedia.org/wiki/Audit_trail) je izvajanje dnevnik z zapisi o operacijah nad dodeljenimi viri. 

## Podrobna navodila

### 1. Naloga

Na prvem navideznem računalniku namestimo `freeradius` implementacijo RADIUS protokola preko upravljalca paketkov našega operacijskega sistema.

    apt install freeradius

V nastavitveni datoteki `clients.conf` programa `freeradius` nastavimo geslo za lokalni dostop do podatkov RADIUS strežnika.

    nano /etc/freeradius/3.0/clients.conf

    client localhost {
        secret = password1s	
    }

V nastavitveni datoteki `users` programa `freeradius` ustvarimo novega uporabnika.

    nano /etc/freeradius/3.0/users

    user1 Cleartext-Password := "user1password"
	    Reply-Message := "Welcome, %{User-Name}!"

Da se nove nastavitve in novi uporabniku upoštevajo moramo ponovno zagnati RADIUS strežnik

    service freeradius restart

Sedaj ugotovimo na katerih omrežnih vrata posluša RADIUS strežnik, tako da namestimo orodja za spremljanje omrežja `net-tools` oziroma bolj specifično `netstat`.

    apt install net-tools

    netstat -n -l -p

Ugotovimo, da RADIUS strežnik posluša na omrežnih vratih 1812 in sedaj preizkusimo delovanje RADIUS strežnika lokalno, tako da poženemo ukaz `radtest`.

    radtest user1 user1password localhost 1812 password1s

### 2. Naloga

Začnemo z namestitvijo spletnega strežnika Apache2 `apache2` in knjižice za ovirajanje z RADIUS protokolom `libapache2-mod-auth-radius` preko upravljalca paketkov našega operacijskega sistema.

    apt install apache2 libapache2-mod-auth-radius

V nastavitveni datoteki `000-default.conf` omogočimo ovirajanje z našim lokalnim RADIUS strežnikom za datoteke v mapi `/radius` ter omogočimo dostop le registriranim RADIUS uporabnikom.

    nano /etc/apache2/sites-available/000-default.conf

    AddRadiusAuth localhost:1812 password1s 5:3
    AddRadiusCookieValid 5

    <Location /radius/>
        AuthType Basic
        AuthName "RADIUS Authentication!"
        AuthBasicProvider radius
        AuthRadiusActive On
        require valid-user
    </Location>

V mapi sedaj ustvarimo novo mapo `radius` v mapi `/var/www/html` in v njej ustvarimo poljubno HTML datoteko. Da se nove nastavitve upoštevajo, moramo ponovno zagnati apache2 strežnik.

    mkdir /var/www/html/radius

    cp /var/www/html/index.html /var/www/html/radius/

    nano /var/www/html/radius/index.html

    service freeradius restart

Delovanje ovirjanja preizkusimo, tako da z brskalnikom poskusimo dostopati do naslova `http://localhost/radius`, kjer moramo biti pozvani za vnos uporabniškega imena in gesla. Ob vnosu uporabniškega imena in gesla RADIUS uporabnika, ki smo ga ustvarili v prejšnjem koraku, nam je dostop do strani omogočen, drugače pa nam je dostop zavrnjen.

![Okence za vnos uporabniškega imena in gesla.](slike/vaja12-apache1.png)

### 3. Naloga

Na drugem navideznem računalniku prav tako namestimo implementacijo `freeradius` protokola RADIUS preko upravljalca paketkov našega operacijskega sistema.

    apt install freeradius

V nastavitveni datoteki `clients.conf` programa `freeradius` nastavimo geslo za lokalni dostop do podatkov RADIUS strežnika.

    nano /etc/freeradius/3.0/clients.conf

    client localhost {
        secret = password2s	
    }

V nastavitveni datoteki `users` programa `freeradius` ustvarimo novega uporabnika.

    nano /etc/freeradius/3.0/users

    user2 Cleartext-Password := "user2password"
	    Reply-Message = "Welcome, %{User-Name}!"

Da se nove nastavitve in novi uporabniku upoštevajo moramo ponovno zagnati RADIUS strežnik

    service freeradius restart

Sedaj preizkusimo delovanje RADIUS strežnika lokalno, tako da poženemo ukaz `radtest`.

    radtest user2 user2password localhost 1812 password2s

Da bo do spletne strani na prvem navideznem računalniku lahko dostopal tudi uporabnik `user2` z RADIUS strežnika na drugem navideznem računalniku, prav tako na drugem navideznem računalniku definiramo novo kraljestvo v datoteki `proxy.conf`.

    nano /etc/freeradius/3.0/proxy.conf

    realm realm2.si {

    }

Sedaj pa še omogočimo dostop do lokalnega RADIUS strežnika s prvega navideznega računalnika, tako da ga dodamo v datoteko `clients.conf`.

    nano /etc/freeradius/3.0/clients.conf

    client server1 {
        ipaddr = IP_OF_THE_FIRST_VM
        secret = server1p
    }

Da se nove nastavitve upoštevajo moramo ponovno zagnati RADIUS strežnik.

    service freeradius restart

Sedaj se vrnemo na prvi navidezni računalnik kamor moramo dodati podatke o kraljestvu na drugem navideznem računalniku v datoteko `proxy.conf`.

    nano /etc/freeradius/3.0/proxy.conf

    home_server server2 {
        ipaddr = IP_OF_THE_SECOND_VM
        secret = server1p
    }

    home_server_pool pool_server2 {
        type = fail-over
        home_server = server2
    }

    realm realm2.si {
        auth_pool = pool_server2
        nostrip
    }

Da se nove nastavitve upoštevajo moramo ponovno zagnati RADIUS strežnik.

    service freeradius restart

Najprej preizkusimo delovanje z ukazom `radtest` in nato še preko brskalnika, kjer uporabnik user2 lahko uspešno pridobi dostop do naslova `http://localhost/radius`.

    radtest user2@realm2.si user2password localhost 1812 password1s

![Okence za vnos uporabniškega imena in gesla.](slike/vaja12-apache2.png)

Ob primeru težav s programom `freeradius`, ga ustavite in poženite v ukazni vrstici, da pridete do izpisa beležk.

    service freeradius stop

    freeradius -X