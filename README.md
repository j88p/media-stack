# Media Stack Helm Chart

`disclaimer`: guiden är inte komplett, det kommer vara saker som stökar. Jag har med all säkerhet missat att beskriva vissa delar. Notera gärna det och skriv en issue så kan jag förtydliga efter behov. Målet är att charten ska bli så nära plug and play som möjligt.

Denna chart har som mål att automatisera installationen och delar av konfigurationen för en  privat media-server i ett `kubernetes` baserad på `Plex`, `Sonarr`, `Radarr`, `Prowlarr`, `Overseerr` samt `qBittorrent` inkapsulerat av `Gluetun VPN`. I denna första release har samtliga tjänster (utom plex) en egen vpn sidecar i podden. Det är enklare att få säkert nätverk på plats då, men på sikt hade det vart bra om vi kan få en och samma gluetun podd sköta all nätverk i ett namespace, men det kräver en del `networkPolicy` handpåläggning i klustret. Fördelen med det är att då krävs endast 1 VPN anslutning, istället för 5. Private Internet Access (VPN) erbjuder 10 anslutningar i en vanlig prenumeration. Så det äter upp en del av det poolen, vet inte hur det ser ut med andra VPN tjänster.

Just nu stödjer den bara Private Internet Access via `OpenVPN`. Det går dock bra att lägga till flera providers så säg vilka ni vill ha med, men kolla först så de stöds av OpenVPN. Målet är att kunna sätta specifika providers som flaggor vid helm installationen.

Man kommer behöva tweaka en del saker för att få det att funka. Bland annat behöver man i sin lokala maskin lägga till de Hostnamn som skapas upp i respektive `Ingress`.

- `sudo nano /etc/hosts` eller
- `code /etc/hosts`

**Lägg till i filen**:

```bash
192.168.1.128 qbit.lan ovr.lan son.lan rad.lan pro.lan plex.lan
```

se till att hostnamnen resolvar mot _din_ servers lokala ip adress som exempelvis: `192.168.1.128`. Då kommer du kunna nå din tjänster över det lokala nätet på de adresserna, till exempel `plex.lan` eller `son.lan`.

Jag rekommenderar att man kör med samma mountPaths till directories om man kan, då blir det enkelt att ta emot uppdateringar av charten. Vi kan lägga till flera mountPaths med om man vill ha mer eller bättre kategorisering av Plex innehållet.

Den är inte perfekt, men man får upp alla tjänster, det tar en stund för dom, ungefär två minuter att spinna upp helt, det beror på att det tar en stund att koppla upp varje vpn anslutning, så varje deployment har en `initialDelaySeconds` på mellan 90 och 120 sekunder för att inte probes ska faila innan anslutningen gjorts.

Man kommer behöva göra en tweakande själv. Prowlarr behöver kopplas till Sonarr och Radarr, Sonarr och Radarr behöver kopplas till QBittorrent. Overseerr behöver i sin tur kopplas till Plex, Sonarr och Radarr. Jag har försökt förkonfigurera så mycket som möjligt så att dessa kopplingar görs klusternätveker, så förhoppningsvis behöver man bara stoppa in de API nycklar som skapas upp i Sonarr, Radarr och Prowlarr under _Settings -> General -> API key_.

För att komma åt `qBittorrent`, är det viktigt att man vid **första uppstart** går in i loggarna `kubectl logs qbittorrent-xxxxxx-xxxxx -n plex -c qbittorrent`. Där ser du ett autogenerat förstagångs lösenord, använd det för att logga in. Byt sedan i menyn under _Options -> WebUi -> Authentication_. Detta använder du för att kolla sonnar och radarr mot qbittorrent.

## Att göra

### Skapa Namespace Plex

`kubectl create ns plex`

### Skapa en PIA VPN k8s secret

```sh
kubectl create secret generic pia-credentials \
   --from-literal=username='<your_pia_username>' \
   --from-literal=password='<your_pia_password>' \
   -n <namespace>
```

Ersätt `<your_pia_username>`, `<your_pia_password>` och `<namespace>` med dina uppgifter.

### Obligatoriska sökvägar och kataloger

Skapa dessa mappar och se till att dina media filer ligger där. Alternativt justera dessa paths efter eget behov, men det finns inte lösning för att inte skriva över dom om det kommer en uppdatering.

- `/opt/docker_volumes/plex/config`
- `/opt/docker_volumes/plex/transcode`
- `/opt/docker_volumes/overseerr`
- `/opt/docker_volumes/prowlarr`
- `/opt/docker_volumes/qbittorrent/config`
- `/mnt/plexpool/movies`
- `/mnt/plexpool/tvshows`
- `/mnt/plexpool/barnfilmer`
- `/mnt/plexpool/barnserier`
- `/mnt/plexpool/newDownloads`

Justeras i `values.yaml`. Glöm inte att det behöver finnas läs och skriv rättigheter till dessa mappar. Överväg att ge alla rätt att läsa och skriva till dessa om du inte hellre vill krångla med specifika rättigheter.

Vi kan som sagt även lägga till fler mountpaths för att skapa bättre kategorisering i plex UI. Exvis, Documentaries, Horror osv.

### Använd fsmerge för plexpool

För att underlätta hantering av mediebibliotek rekommenderas `FSMerge` eller motsvarande drive pooling verktyg. Det skapar virtueliserad lagring där alla externa hårddiskar abstraheras bort och ersätts av: `mnt/plexpool`. Mounta varje disk via `/etc/fstab` för att automatisera montering vid uppstart och omstart. Om man inte mountar på UUID kommer servern blanda ihop mounts vid omstart.

1. Installera fsmerge:

   ```sh
   pip install fsmerge
   ```

2. Skapa en konfigurationsfil (t.ex. `/etc/fsmerge/plexpool.conf`) som listar dina källkataloger.
3. Kör fsmerge:

   ```sh
   fsmerge /etc/fsmerge/plexpool.conf /mnt/plexpool
   ```

### Montera diskar via UUID i /etc/fstab

Hämta diskarnas UUID. Här är det bra om du redan har formatterat till EXT4 format. Kör helst inte med adaptrar och NTFS om du inte måste. Kom ihåg att du rensar allt på disken om du formeterar om den.

```sh
lsblk -o NAME,UUID,MOUNTPOINT
blkid
```

Redigera `/etc/fstab` för att montera via UUID:

```
# Plex1
UUID=c6078a53-52b1-4711-9511-1cec7bfa11e8 /media/plex7 ext4 defaults,nofail 0 2

# PLEX2
UUID=c58af61f-4851-4dae-bbab-e13ead158fa7 /media/plex8 ext4 defaults,nofail 0 2

# Plex3
#UUID=<your-disk-uuid> /media/plex9 ext4 defaults,nofail 0 2

# FSMERGE
/media/plex1:/media/plex2:/media/plex3 /mnt/plexpool fuse.mergerfs defaults,allow_other,use_ino,category.create=mfs,nonempty 0 0

...etc...
```

Detta gör att dina diskar kommer mountas pålitligt vid varje uppstart.

Skapa monteringspunkter och montera:

```sh
sudo mkdir -p /mnt/plexpool/movies
```

```sh
sudo mkdir -p /media/plex1 (2, 3) osv...
```

Glöm inte att de måste finnas läs och skrivrättighter. `chmod +x ...`

För att applicera förändringar:

- `sudo mount -a`

För att se monteringarna:

- `df -h`

**Du kan behöva tweaka detta och stöka lite grann men det är värt mödan, ta hjälp av AI för att få allt på plats** 🤴🏼

Så här kan du formattera diskar till EXT4

```sh
sudo mkfs.ext4 /dev/sdX
```

## Helm

1. **Klona media-stack repot:**

   ```sh
   git clone https://github.com/j88p/media-stack.git
   cd media-stack/helm/media-stack-v1.0
   ```

2. **Redigera `values.yaml`:**
   - Ange dina media-/konfigurationssökvägar under `plex`, `sonarr`, `radarr` m.fl.
   - Ange VPN-uppgifter och region för Gluetun-sidecars.
   - Ange Ingress-värdnamn för varje app (t.ex. `son.lan`, `rad.lan`, `qbit.lan`, `plex.lan`).
   - Lägg vid behov till poster i din `/etc/hosts` som pekar dessa namn till IP-adressen för din Ingress-kontroller (t.ex. `192.168.1.128`).

3. **Installera helmchart:**

   ```sh
   helm install media-stack .
   ```

   Eller uppgradera:

   ```sh
   helm upgrade --install media-stack .
   ```

4. **Åtkomst till dina appar:**
   - Plex: via Ingress (`http://plex.lan`) som routar till tjänsten `plex-service` på port 32400
   - Övriga appar: via Ingress (`http://son.lan`, `http://rad.lan`, `http://qbit.lan`, `http://pro.lan`, `http://ovr.lan`)

## Avinstallera

```sh
helm uninstall media-stack
```
