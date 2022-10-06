# Data Driven Smart Society Thingsboard Configuration

Dit is een handleiding voor het installeren van Thingsboard op Ubuntu. Op het moment wordt er gebruik gemaakt van een Ubuntu virtual machine op het Azure platform van Microsoft. Op deze repository staan geen databases of andere gebruikersdata van Thingsboard.

## Welke services worden er geïnstalleerd

Er wordt gebruik gemaakt van het open source en gratis IoT platform Thingsboard. Er bestaat ook een closed source versie Thingsboard PE (Premium Edition), en Thingsboard Cloud Hosted. Deze versies kosten een maandelijks bedrag en bieden extra functionaliteit die niet nodig is voor dit project.

Thingsboard heeft ook een MQTT server waarnaar data gestuurd kan worden met de metingen van iedere sensor.

Normaliter zou de ingebouwde Postgresql server van Thingsboard gebruikt worden om data op te slaan. Echter was het een eis dat de database van buitenaf te benaderen is. Dit is niet mogelijk met de Docker image van Thingsboard. Daarom is er een aparte postgresql service geïnstalleerd.

Om de database te benaderen wordt er ook gebruik gemaakt van adminer, een tool waarmee met verschillende type SQL databases verbonden kan worden.

Er wordt ook een mailserver (Poste) geïnstalleerd zodat het IoT bijvoorbeeld wachtwoord reset mail kan versturen naar de eigenaar van het account.

Om zowel de verschillende services beschikbaar te stellen via HTTPS wordt er ook een webserver geïnstalleerd, hiervoor is een domeinnaam vereist. De gekozen webserver hiervoor is NGINX. De Docker image SWAG van linuxserver wordt gebruikt omdat deze versie standaard alle webpaginas beveiligd door TLS certificaten bij Let's Encrypt aan te vragen. De opgehaalde certificaten kunnen gebruikt worden gemakkelijk gebruikt worden in de configuratie van NGINX.

## Systeem vereisten

- 2 vCores
- 4GB werkgeheugen
- 20GB opslag (SSD)

## Voorvereisten

Voor dat de docker images gebruikt kunnen worden moeten er een aantal dingen voorbereid worden op het systeem. De benodigde stappen worden uitgelegd in dit hoofdstuk.

### Docker installeren

Installeer docker op het systeem en voeg de hoofdgebruiker toe aan de docker groep. Dit is uitgelegd op de [installatie pagina van Docker voor Ubuntu](https://docs.docker.com/engine/install/ubuntu/), en de [post installation steps pagina](https://docs.docker.com/engine/install/linux-postinstall/). Voor de compleetheid zijn de commando's hieronder ook nog neergezet.

```sh
# Set up the repository
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# Install Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
# Manage Docker as a non-root user
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

Dit zou ook de Docker Compose plug-in moeten installeren. Anders staat het installatie process daarvoor [hier beschreven](https://docs.docker.com/compose/install/compose-plugin/#installing-compose-on-linux-systems).

### Poorten openen in Azure

Voor het platform is het nodig om een aantal poorten te openen zodat deze van buitenaf bereikbaar zijn.

- 1883 (mqtt)
- 8883 (mqtt ssl)
- 7070 (thingsboard / coap)
- 5683-5688/udp (thingsboard)
- 80 (redirect naar https)
- 443 (https)
- 25 (smpt)
- 110 (pop3)
- 143 (imap)
- 465 (smtp ssl)
- 587 (smtp ssl)
- 993 (pop3 ssl)
- 995 (imap ssl)
- 4190 (mail sieve)

### Domeinnaam koppelen

Om een verbinding te maken met de Thingsboard omgeving via het world wide web is er een domeinnaam nodig. Alleen zo kan er een beveiligde verbinding tot stand gebracht worden (tenzei je voor een betaalde oplossing kiest die wel IP adressen kan beveiligen maar het kopen van een domeinnaam is goedkoper dan zo een oplossing). Bedenk dit dus voor het aanmaken van jou omgeving (het propageren over het hele web kan namelijk wel tot 24 uur duren). Deze verwijzingen zijn nodig om de installatie zonder errors te laten verlopen.

Denk hierbij aan het domeinnaam wat jij hebt en wilt gaan gebruiken hiervoor, Hierbij een voorbeeld:

We hebben de azure server "127.0.0.1" en het domeinnaam "sensebox.nl" en willen deze gaan instellen om te gebruiken voor de verbinding naar thingsboard ( dit domeinnaam en ip adres is natuurlijk fictief kies hier uiteraard je eigen domeinnaam en ip adres ).

We willen de thingsboard omgeving benaderen door naar "sensebox.nl" te gaan daarvoor zal een DNS record aangemaakt moeten worden die naar de server wijst.

sensebox.nl -> 127.0.0.1

Thingsboard maakt gebruik van een mailserver om zo informatie door te sturen aan geregistreerde gebruikers. Deze mailserver word standaard bijgeleverd met deze installatie maar het kan ook zo zijn dat je een eigen hebt. Voor deze uitleg gaan we er van uit dat je deze niet hebt, maak dan ook een record aan genaamt "mail.sensebox.nl"

mail.sensebox.nl -> 127.0.0.1

Voor de database manager die we ook via het web willen benaderen maken we ook een subdomein aan deze noemen we "adminer.sensebox.nl"

adminer.sensebox.nl -> 127.0.0.1

### De repository clonen

In deze repository staan de meeste bestanden die je nodig hebt om een **nieuwe** installatie van Thingsboard te installeren. Het onderstaande commando cloned het project naar *~/thingsboard*.

```sh
git clone https://github.com/senseboxti/thingsboard-config ~/thingsboard
cd ~/thingsboard
```

## Installatie

Voor het installeren van alle services zijn een aantal korte stappen nodig.

### Vereiste mappen aanmaken

Als het goed is zit je nu al in de *thingsboard* map, waar je de repository naartoe hebt gecloned. Om Thingsboard goed te kunnen starten moeten we een aantal mappen aanmaken waar configuratie bestanden in komen te staan.

NOTE: *Als een van deze mappen al bestaat is het van belang om deze te legen het overzetten van de configuraties word namelijk verder in het proces uitgelegd*.

```sh
cd ~/thingsboard
mkdir -p thingsboard/data thingsboard/logs thingsboard/certs postgresql
sudo chown -R 799:799 thingsboard
```

## Aanpassingen maken aan de configuratie

### MQTT certificaat

Om MQTT apparaten beveiligd te verbinden met de ingebouwde MQTT server van Thingsboard moeten we certificaten specificeren die Thingsboard kan gebruiken voor de verbinding. In de configuratie hebben we aangegeven dat deze opgeslagen zullen zijn */certs/server.pem* en */certs/server_key.pem*. De map *thingsboard/certs* is gemapped naar */certs* in de container, de public key (server.pem) en private key (server_key.pem) moet dus in de *thingsboard/certs* map staan.

Voor de test server is er al een key aangemaakt die ook is ingeprogrammeerd in de SenseBox, daarom is het wellicht handig om deze voor de installatie over te nemen anders zul je alle apparaten die op het huidige moment verbinden met de test server opnieuw moeten voorzien van een key om zo weer veilig te kunnen verbinden via MQTT.

Het ophalen van de keys kun je doen door in te loggen op de test server en vervolgens te navigeren naar de certs folder, Hierin staat een public key en een private key. Beide kan je met `cat server.pem & cat server_key.pem` uitlezen. Kopieer deze en plaats deze vervolgens in de zelfde mappen structuur op de nieuwe server door te navigeren naar de folder en dan het commando `sudo nano server.pem` & `sudo nano server_key.pem` Let wel op dat je de juiste key op de juiste plek plaatst.

Het is ook mogelijk om nieuwe keys te genereren zoals beschreven in de [documentatie van Thingsboard](https://thingsboard.io/docs/user-guide/mqtt-over-ssl/). Zorg er voor dat de key bestanden in de certs map komen te staan met de juiste naam.

In beide gevallen moet de eigenaar van de bestanden bijgewerkt worden zodat de thingsboard gebruiker in de docker container ze kan aanpassen.

```sh
cd ~/thingsboard
sudo chown -R 799:799 thingsboard/certs
```

### Thingsboard | Server configuratie

Veel instellingen van Thingsboard kunnen op server level aangepast worden. Voeg deze instellingen toe aan de environment variabelen van de thingsboard service in *docker-compose.yml*. Een lijst met alle variabelen die gezet kunnen worden zijn beschikbaar in de [documentatie van Thingsboard](https://thingsboard.io/docs/user-guide/install/config/).

### webserver | Bijwerken welke domeinnamen er beveiligd zijn

Dit is ingesteld via de environment variabelen van de nginx service. Op de test server maken we gebruik van het subdomain *test.*, dus zet alleen hier **ONLY_SUBDOMAINS** op *true*. Het is belangrijk dat de DNS records van de (sub)domeinen ook naar de server wijzen waarop je de TLS certificaten aanvraagt (dit wordt dus automatisch gedaan door de *swag docker image*). Voor de testserver hebben we momenteel de volgende subdomeinen:

- test.
- mail.
- adminer.test.

Voor de productie server zou dit veranderen naar:

- mail.
- adminer.

Je ziet hier één subdomein minder omdat we het hoofddomein zullen gebruiken om Thingsboard te benaderen.
Wanneer er een productie server ingesteld wordt, pas dan ook het mail subdomein van de test server aan naar *mail.test* aan. Of verwijder de mailserver volledig van de testserver en stel beide Thingsboard servers in om de mail server van de productie te gebruiken.

### webserver | Domeinnaam proxy/content aanpassen

De enige configuratie bestanden die aangepast zouden moeten worden staan in *nginx/nginx/active-confs*. De drie services zijn al beschikbaar gemaakt op _ (thingsboard), mail.* en adminer.*. Dit betekent dat er dus een domeinnaam vereist is om de verschillende addressen te bereiken. Echter zou het technisch gezien ook mogelijk zijn om enkel je hosts file lokaal aan te passen. Maar dan zullen de pagina's niet beveiligd kunnen worden met TLS.

### Postgresql | Listen address aanpassen

Dit is vastgelegd in het bestand *postgresql/postgresql.conf*, pas listen `listen_addresses` aan naar de gewenste waarde.

### Postgresql | Login methodes voor hosts aanpassen

Voor verschillende IP addressen (hosts) kunnen login methodes aangewezen worden. Zo is het privé docker netwerk (172.17.0.0/24) waar de containers op aangesloten zijn een vertrouwd netwerk (*trust*). Als we externe verbindingen willen toestaan kunnen we dit beter beveiligen. De configuratie hiervan staat in het *postgresql/pg_hba.conf* bestand. Meer informatie over deze configuratie staat beschreven in de [documentatie van Postgresql](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html).


## Starten van Thingsboard

### De eerste keer

Voorafgaand aan het starten van thingsboard is het aan te bevelen om de standaard wachtwoorden van de postgresql service aan te passen. Dit kan binnen het bestand *docker-compose.yml*

In het bestand staan 2 environment variabelen (*- SPRING_DATASOURCE_PASSWORD=postgres* & *- POSTGRES_PASSWORD=postgres*). Pas deze beide aan naar een nieuw wachtwoord dat jij bepaald. Denk hierbij eraan dat in beide gevallen het wachtwoord hetzelfde moet zijn. Je kan dus niet 1 variabele hebben met het wachtwoord "abc" en een ander met "cba".

De eerste keer opstarten zal nog niet losgekoppeld zijn van de terminal. Dit zullen we doen met het volgende commando
```sh
cd ~/thingsboard
docker-compose up
```

Zodra deze is opgestart zal er gewacht moeten worden totdat de installatie voltooid is (dit kan tot 5 minuten duren afhankelijk van de server). Zodra de server klaar is met het installeren moeten er nog een aantal configuraties omgezet worden. Sluit de docker af door op `ctrl + c` te drukken en wacht totdat je weer in de normale terminal zit.

Vervolgens gaan we de configuraties omzetten.

```sh
cd ~/thingsboard
cp ./postgresqlcfg/postgresql.conf ./postgresql
cp ./postgresqlcfg/pg_hba.conf ./postgresql
```

Nu zijn we klaar en kan de omgeving normaal gestart worden.

### Na installatie

Thingsboard draait vanuit een geïsoleerde omgeving dankzij Docker. Dit maakt het ook makkelijk en snel om updates uit te voeren.

Voor het starten van Thingsboard is niet meer nodig dan één command.

```sh
cd ~/thingsboard
docker-compose up -d
```

Om de container in te komen kan je het volgende commando gebruiken
```sh
cd ~/thingsboard
docker exec -it thingsboard-thingsboard1 bash
```

## Updates installeren

Als je updates wilt toe passen kan dat met de onderstaande commando's.

```sh
cd ~/thingsboard
docker-compose pull
docker-compose up -d
```

Hierna zullen alle services geüpdatet zijn naar de nieuwst beschikbare versie. Dit process duurt slechts enkele minuten en resulteert in bijna geen down-time.

## Note

Het kan zijn dat voor sommige containers bij de eerste keer opstarten de gemapte docker map mount leeg moet zijn. Hernoem dan de betreffende map, maak een nieuwe map aan met de zelfde naam en kopieer de configuratie bestanden terug in de map nadat de container volledig opgestart is. De logs van een container kan met `docker logs <container_name>`. Gebruik -f voor follow en --tail kan gebruikt worden om het aan regels dat je wilt zien.

© Data Driven Smart Society - door [Luc Appelman](https://github.com/controlol)
