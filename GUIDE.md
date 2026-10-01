# Publicera sajten på K3s med Gateway API

Den här guiden skapar resurserna i `yatest` och publicerar sajten via K3s inbyggda Traefik och Kubernetes Gateway API. Containern hämtas från ett publikt Docker Hub-repo, så ingen registry-secret behövs.

> Kör kommandona från katalogen där manifesten och `kubeconfig.yaml` finns. Ersätt `kubeconfig.yaml` med din egen kubeconfig om den heter något annat. Dela inte kubeconfig-filen; den ger åtkomst till klustret.

## 0. Hämta kubeconfig från K3s-servern

Kör första kommandot på K3s-servern via SSH eller direkt i dess terminal. Det kopierar K3s kubeconfig till din användares hemkatalog med rättigheter bara för dig:

```bash
cat  /etc/rancher/k3s/k3s.yaml 
```
kopiera in i kubeconfig.yaml


Kubeconfigen pekar normalt på `https://127.0.0.1:6443`, vilket bara fungerar från själva servern. Öppna den lokala `kubeconfig.yaml` i en texteditor och ändra `server:` till en adress som din dator kan nå, till exempel `https://SERVER_ADDRESS:6443`. Använd en adress som finns med i K3s API-certifikatets SAN-lista; om adressen saknas där behöver K3s konfigureras med `tls-san` för den adressen. Begränsa brandväggens port `6443` till betrodda IP-adresser — exponera inte Kubernetes API öppet mot internet.

Kontrollera sedan från din dator:

```bash
kubectl --kubeconfig=./kubeconfig.yaml get nodes
```

När kopieringen lyckats kan du ta bort den tillfälliga kopian från servern:


## 1. Kontrollera anslutningen till klustret

```bash
kubectl --kubeconfig=./kubeconfig.yaml get nodes
kubectl --kubeconfig=./kubeconfig.yaml get pods -n kube-system
```

Kontrollera att noden är `Ready` och att Traefik körs i `kube-system`. K3s installerar Traefik som standard. Denna guide förutsätter att den inte har stängts av med `--disable=traefik`. Jag har lagt till `kubeconfig.yaml` i `.gitignore` så att den inte råkar läggas till framöver. Om filen redan är spårad av Git måste den tas bort från Git-indexet, och om den redan publicerats ska kubeconfigens credentials roteras. Den gamla `skapasecret.txt` innehöll också registry-credentials i klartext: rotera lösenordet om det är/var giltigt. Att ta bort filerna nu tar inte bort dem ur Git-historiken; rensa historiken innan repot görs publikt.

## 2. Kontrollera Gateway API:s CRD:er

K3s paketerar Gateway API:s standard-CRD:er tillsammans med Traefik. Kontrollera att de finns i klustret:

```bash
kubectl --kubeconfig=./kubeconfig.yaml get crd gatewayclasses.gateway.networking.k8s.io gateways.gateway.networking.k8s.io httproutes.gateway.networking.k8s.io
```

Installera inte en separat uppsättning CRD:er ovanpå K3s-versionen; K3s/Traefiks CRD-addon hanterar dem. Om de saknas, kontrollera K3s-versionen, HelmChart `traefik-crd` och eventuella fel i kube-system:

```bash
kubectl --kubeconfig=./kubeconfig.yaml get helmcharts -n kube-system
kubectl --kubeconfig=./kubeconfig.yaml get addons -n kube-system
```

## 3. Aktivera Gateway API i K3s inbyggda Traefik

Manifestet `02-traefik-gateway-config.yaml` aktiverar Traefiks Gateway API-provider via K3s `HelmChartConfig`:

```bash
kubectl --kubeconfig=./kubeconfig.yaml apply -f 02-traefik-gateway-config.yaml
```

Vänta tills Traefik har uppdaterats och kontrollera att Traefik-podden körs samt att Traefik själv har skapat sin `GatewayClass`:

```bash
kubectl --kubeconfig=./kubeconfig.yaml get pods -n kube-system -l app.kubernetes.io/name=traefik
kubectl --kubeconfig=./kubeconfig.yaml get gatewayclass traefik
kubectl --kubeconfig=./kubeconfig.yaml logs -n kube-system deployment/traefik --tail=100
```

## 4. Skapa namespace, app, Service och route

Gateway och GatewayClass skapas av K3s Traefik Helm chart när du aktiverar provider i steg 3. Applicera appens namespace, Deployment/Service och route från katalogen med manifesten:

```bash
kubectl --kubeconfig=./kubeconfig.yaml apply -f 01-namespace.yaml
kubectl --kubeconfig=./kubeconfig.yaml apply -f 40-site.yaml
kubectl --kubeconfig=./kubeconfig.yaml apply -f 50-gateway.yaml
```

`50-gateway.yaml` skapar en `HTTPRoute` för `stefanssupersajt.jumpingcrab.com` kopplad till `traefik-gateway` i `kube-system`. HelmChartConfig ber Traefik skapa Gateway och GatewayClass, samt tillåta routes från alla namespaces. Standardlistenern `web` på intern port `8000` går via K3s ServiceLB till port `80` utifrån.

## 5. Kontrollera att resurserna är redo

```bash
kubectl --kubeconfig=./kubeconfig.yaml get pods,svc -n yatest
kubectl --kubeconfig=./kubeconfig.yaml get gatewayclass,gateway,httproute -A
kubectl --kubeconfig=./kubeconfig.yaml describe gateway traefik-gateway -n kube-system
kubectl --kubeconfig=./kubeconfig.yaml describe httproute sites -n yatest
```

Vänta tills Gateway lyser `Programmed=True` och HTTPRoute visar att den är accepterad och ansluten till Gateway. Om Gateway API-resurserna saknas, kontrollera CRD-installationen och Traefiks loggar.

När Gateway och HTTPRoute är redo, och du har kontrollerat att routen fungerar, ta bort gamla resurser som inte längre används.


## 6. Peka DNS och öppna brandväggen

Ändra i 50-gateway.yaml (stefanssupersajt.jumpingcrab.com) till itsX.systementor.se

Kör ändringar
```bash
kubectl --kubeconfig=./kubeconfig.yaml apply -f 50-gateway.yaml
```


Öppna inkommande TCP-port 80 (och senare 443 om du sätter upp HTTPS) i serverns och eventuell molnleverantörs brandvägg. DNS-namnet måste peka till den här servern.

K3s ServiceLB exponerar normalt Traefik på port 80. Kontrollera den faktiska tjänsten och adressen:

```bash
kubectl --kubeconfig=./kubeconfig.yaml get svc -n kube-system traefik
kubectl --kubeconfig=./kubeconfig.yaml get gateway traefik-gateway -n kube-system -o yaml
```

Om trafik inte når tjänsten, kontrollera att ingen annan tjänst använder port 80 och att serverns publika IP/NAT och brandvägg är korrekt konfigurerade.

## 7. Testa sajten

Från en klient som kan nå servern:

```bash
curl -i http://stefanssupersajt.jumpingcrab.com/
```

Den här konfigurationen är HTTP-only. För HTTPS krävs ett giltigt TLS-certifikat och en TLS-konfigurerad Gateway-listener; det skapas inte i den här guiden.

## Uppdatera sajten senare

```bash
kubectl --kubeconfig=./kubeconfig.yaml rollout restart -n yatest deployment/yagolangsite
kubectl --kubeconfig=./kubeconfig.yaml rollout status -n yatest deployment/yagolangsite
```

## Felsökning

```bash
kubectl --kubeconfig=./kubeconfig.yaml get events -n yatest --sort-by=.lastTimestamp
kubectl --kubeconfig=./kubeconfig.yaml logs -n kube-system deployment/traefik --tail=200
kubectl --kubeconfig=./kubeconfig.yaml describe pods -n yatest
```

`ImagePullBackOff` kan tyda på att imagen inte är publikt tillgänglig eller att noden inte kan nå Docker Hub. `Pending` pods kan tyda på resurs- eller nodproblem. DNS/anslutningsfel utifrån beror ofta på DNS, port 80 eller brandvägg/NAT.
