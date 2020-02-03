  # Microservices in Kubernetes

## **Katacoda lessons**

- [Docker Deploy static HTML page](https://www.katacoda.com/courses/docker/create-nginx-static-web-server)
- [Kubernetes Setup Cluster](https://katacoda.com/courses/kubernetes/launch-single-node-cluster)
- [Kubernetes Deployment Files](https://www.katacoda.com/courses/kubernetes/creating-kubernetes-yaml-definitions)

## **Country Provider Service - CPS**

Gruppenname: **hator**

## **Ziel**

Nach dem Deployment soll der Country Service unter [http://ae8333ccd462511eaba7d0af1187e991-581492011.eu-central-1.elb.amazonaws.com:8081/hator/](http://ae8333ccd462511eaba7d0af1187e991-581492011.eu-central-1.elb.amazonaws.com:8081/hator/) erreicht werden können. Die dargestellten Deployment-Files können direkt aus der HTML Seite kopiert werden. Zusätzlich sind Deployment-Files als Textdateien vorhanden. Sie liegen im Gruppenverzeichnis unter `deployment-files` und können auch von dort kopiert werden. Sie sind nach Service sortiert und jeweils durch --- innerhalb der Datei getrennt.

## Deployment in Kubernetes

Kubernetes-Dashboard aufrufen [http://ae8333ccd462511eaba7d0af1187e991-581492011.eu-central-1.elb.amazonaws.com:8081/](http://ae8333ccd462511eaba7d0af1187e991-581492011.eu-central-1.elb.amazonaws.com:8081/)

Login mit `User: exxcellent PW: dashboard_14!`

Im Kubernetes Dashboard werden jetzt alle Informationen zu Deployments, Namespaces, Pods, Services, Volumes usw. angezeigt. In der oberen rechten Ecke ist ein `create` Button zu sehen. Dort gibt es  mehrere Möglichkeiten, mithilfe von YAML-formatierten Dateien neue Ressourcen anzulegen. Entweder kann eine YAML Datei hochgeladen werden oder sie wird textbasiert eingefügt. Wir nutzen diese textbasierte Möglichkeit, um unsere Services innerhalb des Clusters zu deployen.

Um den Status der jeweiligen Services zu sehen, bietet sich auf der linken Seite der Reiter `Overview` an. Dort sind alle Informationen zu Pods, Services usw. aufgelistet.

### **Namespace erstellen**

Jede Gruppe kann sich ihren eigenen Namespace erstellen, welcher innerhalb des Clusters eine isolierte Umgebung darstellt. In einem Namespace können Ports, Servicenamen usw. wiederverwendet werden. Diese Option ist z.B. in unserem Workshop sehr wichtig, da jede Gruppe dieselben Services deployed. Nach dem Upload  muss der Namespace selektiert werden. Das dazugehörige Dropdown befindet sich auf der linken Seite unter Namespace. Alle weiteren Ressourcen werden für den jeweiligen Namespace angezeigt, welcher in dem Dropdown ausgewählt ist. Sollte der Namespace nicht direkt im Dropdown erscheinen, kurz warten und das Fenster nochmal aktualisieren.

Ein Namespace wird mit folgender YAML Datei über den `Create` Button angelegt. Um eindeutig zu sein, wird der Gruppenname als Name des Namespaces verwendet. Mit dem `Upload` Button wird die Eingabe bestätigt und der Namespace erstellt.

```
apiVersion: v1
kind: Namespace
metadata:
  name: hator
```

## Deployment

In den folgenden Abschnitten werden drei Backend- und ein Frontend-Microservice deployed (Country-App-Frontend, Country-Service, Language-Serivce, Currency-Service). Dies ist jeweils über den Create-Button durchzuführen. Im Anschluss jedes Deployments sollte der Status des Pods überprüft werden.

### **Country App (Frontend) Deployment**

#### Pod

Ein Kubernetes POD ist die kleinste atomare Einheit in Kubernetes und enthält mehrere Container. Die Container sind jeweils über ihre geöffneten Ports erreichbar. Im Deployment-File des Pods sind die Images hinterlegt, welche beim Erstellen geladen werden sollen. Ebenfalls können im Deployment-File des Pods verschiedene Environment-Variablen hinterlegt werden, welche dann im Docker Container verfügbar sind. Z.B. sind dies häufig die URLs der Microservices, welche von diesem Pod angesprochen werden. Die Variablen werden in unserem Fall aus einer Configmap geladen, welche separat deployed werden muss.

In den Deployment-Files dieses Services sind die Files noch durch mehrere Codekommentare versehen, welche die jeweiligen YAML-Deklarationen weiter erklären.


```
apiVersion: v1
kind: ConfigMap
metadata:
  # Name der ConfigMap
  name: country-app-svc-configmap
  namespace: hator
data:
  # Environment Variable
  COUNTRY_SERVICE_URL: "http://ae8333ccd462511eaba7d0af1187e991-581492011.eu-central-1.elb.amazonaws.com:8081/hator-cs"
  CURRENCY_SERVICE_URL: "http://ae8333ccd462511eaba7d0af1187e991-581492011.eu-central-1.elb.amazonaws.com:8081/hator-cu"
  LANGUAGE_SERVICE_URL: "http://ae8333ccd462511eaba7d0af1187e991-581492011.eu-central-1.elb.amazonaws.com:8081/hator-ls"
```
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  # Name des Pods
  name: country-app-svc-deployment
  # Name des Namespaces
  namespace: hator
  # Label zur Identifikation des Pods via Key-Value Paar
  labels:
    service: country-app_api_service
spec:
  # Anzahl wie oft der Pod repliziert wird
  replicas: 1
  selector:
    matchLabels:
      name: country-app-api-service-selector
  template:
    metadata:
      labels:
        name: country-app-api-service-selector
    spec:
      containers:
        - name: country-app-api
          # image welches für den Container geladen wird
          image: exxcellent/cps-country-app-service:hator
          # wann soll das Image gepullt werden
          imagePullPolicy: Always
          ports:
          # geöffneter Port des Containers
          - containerPort: 80
          securityContext:
          # environment variablen aus Configmap laden
          envFrom:
          - configMapRef:
              name: country-app-svc-configmap
```

#### Service

Der Lebensdauer eines Kubernetes-Pods kann nicht vorhergesagt werden, da sie ständig neu erstellt, gelöscht oder skaliert werden können. Der Kubernetes-Service dient deshalb als eine weitere Abstraktionsebene über dem POD. Ein Service nimmt die Anfragen an geöffnetes Ports und leitet diese via Round-Robin Verfahren an die Ports der darunterliegende Pods weiter. Um eine bleibene Namensauflösung zu ermöglichen, ist ein Service beim Kubernetes DNS  hinterlegt. Er kann dann wie folgt erreicht werden: `servicename.namespace:port`.  

```
apiVersion: v1
kind: Service
metadata:
  name: country-app-svc-service
  namespace: hator
spec:
  type: ClusterIP
  ports:
    - port: 8484
      targetPort: 80
      name: api
      protocol: TCP
  selector:
    name: country-app-api-service-selector
```

#### Ingress

Ingress ist ein layer-7 basierendes Routingkonzept, welches pfadbasiert arbeitet. Um dies umsetzen zu können, ist im Cluster ein Traefik-Ingress Controller deployed. Die URL auf das Cluster führt immer zum Traefik-Controller. Dieser entscheidet dann anhand des Pfades, zu welcher Apllikation weitergeleitet wird. Ein weiterer Vorteil ist, dass die Konfiguration der Ingress Routen zur Laufzeit passiert, wordurch Routen jederzeit hinzugefügt, verändert oder entfernt werden können.

Die Option PathPrefixStrip legt fest, ob der unter path angegebene Pfad mit an den Service übergeben wird oder abgeschnitten. Alles nachfolgende nach dem angegeben Pfad wird immer weitergegeben.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  # Name der Ingress-Route
  name: country-app-svc-ingress
  # Namespace der Ingress-Route
  namespace: hator
  annotations:
    kubernetes.io/ingress.class: traefik
    # Pfad weiterleiten oder nicht
    #traefik.ingress.kubernetes.io/rule-type: PathPrefixStrip
spec:
  rules:
  - host: 
    http:
      paths:
      # Pfad welcher zum angegebenen Service weiterleitet
      - path: /hator/
        backend:
          serviceName: country-app-svc-service
          servicePort: 8484
```

Die Ingress Route kann unter [http://ae8333ccd462511eaba7d0af1187e991-581492011.eu-central-1.elb.amazonaws.com:8081/traefik-dashboard/dashboard/](http://ae8333ccd462511eaba7d0af1187e991-581492011.eu-central-1.elb.amazonaws.com:8081/traefik-dashboard/dashboard/) kontrolliert werden. Dort sind auf der linken Seite die jeweiligen Routen zu sehen. Wichtig ist, dass auf der rechten Seite ein Pod zugehörig ist. Zu sehen ist dies an einer angegeben IP-Adresse des Pods.

### **Country Service Deployment**

#### Pod

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: country-svc-configmap
  namespace: hator
data:
  CS_VERSION: ""
```

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: country-svc-deployment-latest
  namespace: hator
  labels:
    service: cs_api_service
spec:
  replicas: 1
  selector:
    matchLabels:
      name: cs-api-service-selector
  template:
    metadata:
      labels:
        name: cs-api-service-selector
        app: cs-api-backend
    spec:
      containers:
        - name: cs-api
          image: exxcellent/cps-country-service:latest
          imagePullPolicy: Always
          ports:
          - containerPort: 8080
          securityContext:
            runAsNonRoot: true
            runAsUser: 82
          envFrom:
          - configMapRef:
              name: country-svc-configmap
```

#### Service

```
apiVersion: v1
kind: Service
metadata:
  name: country-svc-service
  namespace: hator
  labels:
spec:
  type: ClusterIP
  ports:
    - port: 8080
      name: api
      protocol: TCP
  selector:
    name: cs-api-service-selector
```

#### Ingress

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: country-svc-ingress
  namespace: hator
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/rule-type: PathPrefixStrip
spec:
  rules:
  - host: 
    http:
      paths:
      - path: /hator-cs/
        backend:
          serviceName: country-svc-service
          servicePort: 8080
```

### **Language Service Deployment**

Der Language-Service wird als komplette YAML-Datei deployed. Hiefür sind die Deployment-Deklarationen für Services, Ingress, Deployments, usw. in ein gemeinsames Deployment-File geschrieben. Innerhalb der Datei werden sie jeweils mit drei `---` als Trennzeichen unterteilt. Im Gruppenverzeichnis unter `deployment-files` ist das Deployment-File des Language-Services zu finden. Der komplette Inhalt der Datei kann kopiert und ebenfalls über `Create` hinzugefügt werden. Nun werden alle benötigten Konfigurationen für den Service erstellt und bereitgestellt.

### **Currency Service Deployment**

Der Currency-Service wird ebenfalls mithilfe einer kompletten YAML-Datei deployed. Für dieses Deployment wird allerdings nicht das textbasierte Deployment verwendet, sondern die Datei via `Create` hochgeladen. Hierzu auf den `Create` Button klicken und `CREATE FROM FILE` auswählen. Als Zieldatei wird im Gruppenverzeichnis unter `deployment-files` das Currency-Deployment-File verwendet. 

## Möglichkeiten im Fehlerfall

Im Dashboard zuerst den Namespace der Gruppe auswählen und auf der linken Seite zum Reiter `Pods` wechseln. Dort kann der Status der Pods eingesehen werden. Sehr hilfreich sind zusätzlich die Events und die Logs des Pods, welche am oberen Rand unter `Logs` eingesehen werden können.