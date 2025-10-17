# Media Stack Helm Chart

`disclaimer`: guiden är inte komplett, det kommer vara saker som stökar. Jag har med all säkerhet missat att beskriva vissa delar. Notera gärna det och skriv en issue så kan jag förtydliga vid behov. Målet är att charten ska bli så nära plug‑and‑play som möjligt.

Denna chart har som mål att automatisera installationen och delar av konfigurationen för en privat mediaserver i ett Kubernetes‑kluster baserat på tjänster som `Plex`, `Sonarr`, `Radarr`, `Prowlarr`, `Overseerr` samt `qBittorrent`, inkapsulerade av `Gluetun VPN`. I denna första release har samtliga tjänster (utom Plex) en egen VPN‑sidecar i podden. Det är enklare att få säkert nätverk på plats så, men på sikt vore det bra om en och samma Gluetun‑podd kan sköta hela nätverket i ett namespace. Det kräver dock viss `networkPolicy`‑handpåläggning i klustret. Fördelen är att det då bara krävs en VPN‑anslutning i stället för fem. Private Internet Access (VPN) erbjuder tio anslutningar i en vanlig prenumeration, så det äter upp en del av den poolen; jag vet inte hur det ser ut hos andra VPN‑tjänster.

Just nu stödjer den bara Private Internet Access via `OpenVPN`. Det går dock bra att lägga till flera providers så säg vilka ni vill ha med, men kolla först så de stöds av OpenVPN. Målet är att kunna sätta specifika providers som flaggor vid helm installationen.

Man kommer behöva tweaka en del saker för att få det att funka. Bland annat behöver man på sin lokala maskin lägga till de hostnamn som skapas upp i respektive `Ingress`.

- `sudo nano /etc/hosts` eller
- `code /etc/hosts`

**Lägg till i filen**:

```bash
192.168.1.128 qbit.lan ovr.lan son.lan rad.lan pro.lan plex.lan
```

Se till att hostnamnen pekar mot _din_ servers lokala IP‑adress, till exempel `192.168.1.128`. Då kommer du kunna nå dina tjänster över det lokala nätet på de adresserna, till exempel `plex.lan` eller `son.lan`.

Jag rekommenderar att man kör med samma mountPaths till kataloger om man kan; då blir det enklare att ta emot uppdateringar av charten. Vi kan också lägga till fler mountPaths om man vill ha mer eller bättre kategorisering av Plex‑innehållet.

Den är inte perfekt, men alla tjänster startar. Det tar ungefär två minuter att spinna upp helt. Det beror på att varje VPN‑anslutning måste kopplas upp, så varje deployment har en `initialDelaySeconds` på mellan 90 och 120 sekunder för att proberna inte ska misslyckas innan anslutningen gjorts.

Viss manuell konfiguration krävs fortfarande. Prowlarr behöver kopplas till Sonarr och Radarr, Sonarr och Radarr behöver kopplas till qBittorrent. Overseerr behöver i sin tur kopplas till Plex, Sonarr och Radarr. Jag har försökt förkonfigurera så mycket som möjligt så att dessa kopplingar görs inom klusternätverket, så förhoppningsvis behöver man bara lägga in de API‑nycklar som skapas upp i Sonarr, Radarr och Prowlarr under _Settings -> General -> API key_.

För att komma åt `qBittorrent` är det viktigt att man vid **första uppstart** går in i loggarna: `kubectl logs qbittorrent-xxxxxx-xxxxx -n plex -c qbittorrent`. Där ser du ett autogenererat förstagångslösenord – använd det för att logga in. Byt sedan i menyn under _Options -> WebUI -> Authentication_. Detta använder du för att konfigurera Sonarr och Radarr mot qBittorrent.

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

Skapa dessa mappar och se till att dina mediafiler ligger där. Alternativt justera dessa sökvägar efter eget behov, men det finns ingen lösning för att undvika att de skrivs över vid uppdateringar.

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

Vi kan som sagt även lägga till fler mountPaths för att skapa bättre kategorisering i Plex‑UI. Exempelvis Documentaries, Horror osv.

### Använd fsmerge för plexpool

För att underlätta hantering av mediebibliotek rekommenderas `fsmerge` eller motsvarande drive‑pooling‑verktyg. Det skapar virtualiserad lagring där alla externa hårddiskar abstraheras bort och ersätts av: `/mnt/plexpool`. Montera varje disk via `/etc/fstab` för att automatisera montering vid uppstart och omstart. Om man inte monterar på UUID kommer servern blanda ihop monteringarna vid omstart.

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

Hämta diskarnas UUID. Här är det bra om du redan har formaterat till ext4‑format. Kör helst inte med adaptrar och NTFS om du inte måste. Kom ihåg att du rensar allt på disken om du formaterar om den.

```sh
lsblk -o NAME,UUID,MOUNTPOINT
blkid
```

Redigera `/etc/fstab` för att montera via UUID:

```ini
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

Detta gör att dina diskar kommer monteras pålitligt vid varje uppstart.

Skapa monteringspunkter och montera:

```sh
sudo mkdir -p /mnt/plexpool/movies
```

```sh
sudo mkdir -p /media/plex1 (2, 3) osv...
```

Glöm inte att det måste finnas läs‑ och skrivrättigheter. `chmod +x ...`

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
   - (Valfritt) Aktivera TLS för Ingress med `global.tls.enabled: true`. Se avsnittet "TLS" nedan.

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

## TLS

Du kan aktivera TLS för dina Ingress-resurser genom att sätta:

```yaml
global:
   tls:
      enabled: true
```

När TLS är aktiverat:

- Varje app med `ingressHost` får en `spec.tls`-sektion med samma host.
- `secretName` för TLS antas vara `<app>-tls` om inget annat anges.
- Du kan åsidosätta per app med `tlsSecretName` i `values.yaml`, t.ex.:

```yaml
apps:
   - name: sonarr
      ingressHost: son.lan
      tlsSecretName: sonarr-custom-tls
```

### Använda cert-manager

Om du använder cert-manager kan du låta certifikaten skapas automatiskt genom att ange en issuer:

```yaml
global:
   tls:
      enabled: true
      issuerKind: cluster-issuer   # eller "issuer" om du använder en namespaced Issuer
      issuerName: letsencrypt-prod # namnet på din (Cluster)Issuer
```

Detta lägger till en annotation på varje Ingress:

```yaml
cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

cert-manager skapar då TLS-sekreten automatiskt med samma namn som i `secretName` (default `<app>-tls`, eller `tlsSecretName` per app). Du behöver alltså inte skapa secreten manuellt när cert-manager används.

### Exempel: `certificate.yaml`

Nedan finns två exempel på hur du kan skapa certifikat med cert-manager.

1) Självsignerad rot + Issuer i namespace `plex` (passar interna .lan-domäner)

Skapa en självsignerad CA och använd den för att utfärda ett delat certifikat för alla dina interna hosts. Allt sker i namespace `plex` och funkar helt lokalt.

```yaml
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
   name: selfsigned-issuer
   namespace: plex
spec:
   selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
   name: media-ca
   namespace: plex
spec:
   isCA: true
   commonName: media-ca
   secretName: media-ca-root
   issuerRef:
      name: selfsigned-issuer
      kind: Issuer
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
   name: media-ca-issuer
   namespace: plex
spec:
   ca:
      secretName: media-ca-root
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
   name: media-stack-certificate
   namespace: plex
spec:
   secretName: media-stack-tls
   issuerRef:
      name: media-ca-issuer
      kind: Issuer
   commonName: media.lan
   dnsNames:
      - qbit.lan
      - son.lan
      - rad.lan
      - pro.lan
      - ovr.lan
      - plex.lan
```

Använd detta tillsammans med:

```yaml
global:
   tls:
      enabled: true
      issuerKind: issuer
      issuerName: media-ca-issuer
```

Vill du återanvända samma certifikat för alla appar, sätt per app:

```yaml
apps:
   - name: sonarr
      ingressHost: son.lan
      tlsSecretName: media-stack-tls
   - name: radarr
      ingressHost: rad.lan
      tlsSecretName: media-stack-tls
   # ...osv
```

1) Let’s Encrypt (publik domän)

Om du har en publik domän och vill använda Let’s Encrypt, skapa (Cluster)Issuer separat (exempelvis `letsencrypt-prod`) enligt cert-manager-dokumentationen och referera den i `values.yaml`:

```yaml
global:
   tls:
      enabled: true
      issuerKind: cluster-issuer
      issuerName: letsencrypt-prod
```

Skapa sedan ett Certificate (valfritt) om du vill dela en och samma secret, annars kommer cert-manager skapa en secret per Ingress automatiskt:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
   name: media-stack-certificate
   namespace: plex
spec:
   secretName: media-stack-tls
   issuerRef:
      name: letsencrypt-prod
      kind: ClusterIssuer
   dnsNames:
      - son.example.com
      - rad.example.com
      - qbit.example.com
      - pro.example.com
      - ovr.example.com
      - plex.example.com
```

Obs: Let’s Encrypt fungerar inte för `.lan` eller icke-publika domäner med HTTP-01. Använd DNS-01 eller självsignerad/egen CA för interna domäner.

Applicera filen:

```sh
kubectl apply -f certificate.yaml
```

## Avinstallera

```sh
helm uninstall media-stack
```
