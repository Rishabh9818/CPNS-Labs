# 11. Vaja: Upravljanje omrežja z orodjem Netfilter

## Navodila

0. Uporabite omrežje in navidezne računalnike iz prejšnjih vaj. 
1. Omogočite dostop drugega navideznega računalnika do spleta preko prvega navideznega računalnika.
2. Na prvem navideznem računalniku preprečite drugemu navideznemu računalniku dostop do IP naslova 8.8.8.8 .
3. Na prvem navideznem računalniku preprečite dostop drugega navideznega računalnika do prvega navideznega računalnika preko varne lupine (SSH).

## Dodatne informacije

[Netfilter](https://en.wikipedia.org/wiki/Netfilter) je ogrodje, ki je del Linux jedra in nam omogoča upravljanje z omrežjem, na primer: filtriranje paketkov, preslikovanje IP naslovov in vrat za namen usmerjanja ter preprečevanja dostopa do delov omrežja.

[iptables](https://en.wikipedia.org/wiki/Iptables) predstavljajo zastarelo orodje za filtriranje paketov na osnovi Netfilter sistema kavljev (angle. hook).

[nftables](https://en.wikipedia.org/wiki/Nftables) novo orodje, ki nadomesti `iptables` ter omogoča dodatne funkcionalnosti, kot je na primer preiskovanje paketov.

## Podrobna navodila

### 1. Naloga

Če smo do sedaj uporabljali `iptables`, potem izbrišemo vsa pravila.

    iptables -t nat -L
    iptables -t nat -F

Trenutne nastavitve `nftables` lahko preverimo z ukazom `nft`. [Navodila](https://www.netfilter.org/projects/nftables/manpage.html) za uporabo `nft`.

    nft list ruleset

Orodje `nftables` uporablja tabele (`tables`) in verige (`chains`) za hierarhično hranjenje pravil (`rules`). Vse tabele, verige in pravila imajo svoje identifikatorje (`handles`), ki jih uporabljamo za brisanje. Na primer:

    nft -a list table mytable
    nft delete rule mytable mychain handle 10

Na prvem navideznem računalniku omogočimo delovanje prehoda, tako da ustvarimo novo tabelo `table`, novo verigo `chain` in novo pravilo `rule` za preslikovanje IP naslovov paketov.

    nft add table mytable
    nft 'add chain mytable postroutingchain { type nat hook postrouting priority -100; }'
    nft add rule mytable postroutingchain masquerade

Prav tako moramo omogočiti usmerjanje na prvem navideznem računalnik v datoteki `/etc/sysctl.conf`.

    nano /etc/sysctl.conf

    net.ipv4.ip_forward=1

Da se sprememba parametrov jedra Linux-a upošteva, uporabimo ukaz `sysctl`.

    sysctl -p

### 2. Naloga

Na prvem navideznem računalniku dodamo pravilo za filtriranje vseh paketov s ponornim IP naslovom `8.8.8.8`, ki izvirajo iz drugega navideznega računalnika.

    nft 'add chain mytable forwardchain { type filter hook forward priority -100; }'
    nft add rule mytable forwardchain ip daddr . ip saddr { 8.8.8.8 . SECOND_VM_IP } drop

### 3. Naloga

Na prvem navideznem računalniku dodamo pravilo za filtriranje vseh paketov s protokolom SSH, ki izvirajo iz drugega navideznega računalnika.

    nft 'add chain mytable inputchain { type filter hook input priority -100; }'
    nft add rule mytable inputchain tcp dport . ip saddr { 22 . SECOND_VM_IP } drop

