Chalo dono files line-by-line poori tarah samjhata hoon.

## 🚀 DEPLOYMENT — Line by Line

```yaml
apiVersion: apps/v1
```
Kubernetes ko batata hai ke ye resource "apps/v1" API group ka hai. Deployment, StatefulSet, DaemonSet jaisi resources isi group mein aati hain (jabke Pod, Service "v1" group mein aate hain — isliye inka apiVersion sirf `v1` hota hai).

```yaml
kind: Deployment
```
Batata hai ke hum kaunsi resource bana rahe hain. Deployment ka kaam: pods ko manage karna — kitne chalne hain, crash ho jayen to dobara banana, update/rollback karna.

```yaml
metadata:
  name: {{ .Values.name }}
```
Deployment ka apna naam — values.yaml se aata hai (`rocket-service`). Ye Deployment object ki identity hai, kubectl mein `kubectl get deployment` karo to yehi naam dikhega.

```yaml
  namespace: {{ .Values.namespace }}
```
Ye Deployment kis namespace ke andar banega — `rocket` namespace.

```yaml
  labels:
    app: {{ .Values.name }}
```
Ye label khud Deployment resource par lagi hai (organizational tag — filtering/searching ke liye, jaise `kubectl get deploy -l app=rocket-service`). Isका asar pod par nahi hota.

```yaml
spec:
```
Ab yahan se "specification" shuru hoti hai — ke Deployment ko kya karna hai.

```yaml
  replicas: {{ .Values.replicaCount }}
```
Kitne identical pods chalne chahiye — values.yaml mein `1` hai, matlab sirf 1 copy chalegi. Agar `3` karo, Kubernetes 3 identical pods bana dega (load balancing/high-availability ke liye).

```yaml
  selector:
    matchLabels:
      app: {{ .Values.name }}
```
Deployment ko batata hai: "jin pods par label `app: rocket-service` lagi ho, wo sab **mere** pods hain, main unko manage karunga." Ye Deployment aur uske pods ke beech ka connection hai.

```yaml
  template:
```
Ye **naye pod ka blueprint** hai. Jab bhi Deployment ko pod banana ho (pehli baar, crash ke baad, scale-up mein), isi template ko copy karke pod banayega.

```yaml
    metadata:
      labels:
        app: {{ .Values.name }}
```
Naye pod par yehi label lagegi (`app: rocket-service`) — ye upar wale `selector.matchLabels` se match karti hai, isliye Deployment apne pods ko pehchan leta hai.

```yaml
    spec:
      containers:
```
Ab pod ke andar konse containers chalenge — list start ho rahi hai (yahan sirf ek container hai).

```yaml
        - name: {{ .Values.name }}
```
Container ka naam — pod ke andar (agar multiple containers hote to har ek ka apna naam hota).

```yaml
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```
Kaunsi Docker image chalani hai — values.yaml se `shahiddevops1/rocket-site:latest` banta hai. Kubernetes isi image ko Docker Hub se pull karke container chalayega.

```yaml
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```
Image kab pull karni hai — `Always` matlab har baar pod banate waqt fresh image download karo (chahe pehle se local mein image mojood ho), taake latest version mile.

```yaml
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
```
Container ke andar app kis port par sun rahi hai (nginx port `80` par listen karta hai). Ye sirf **documentation/info** hai Kubernetes ke liye — port ko actually block ya open ye line nahi karti, container apne aap us port par listen karega ya nahi ye image ke Dockerfile par depend karta hai.

---

## 🌐 SERVICE — Line by Line

```yaml
apiVersion: v1
kind: Service
```
Service "v1" core API group mein aati hai. Service ka kaam: pods tak **stable network address** dena — pods create/delete hote rehte hain (naye naam, naye IP), lekin Service ka apna fixed naam/IP rehta hai jisse traffic bheja ja sakta hai.

```yaml
metadata:
  name: {{ .Values.name }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.name }}
```
Same jaisa Deployment mein — Service ka naam, namespace, aur uski apni label.

```yaml
spec:
  type: {{ .Values.service.type }}
```
Service ka type — `ClusterIP` matlab ye Service sirf **cluster ke andar** accessible hai (bahar internet se nahi). Isi liye aapko `kubectl port-forward` karna pad raha hai bahar se access karne ke liye. (Doosre types: `NodePort` — node ke IP+fixed port se bahar access, `LoadBalancer` — cloud ka external load balancer bana deta hai.)

```yaml
  ports:
    - port: {{ .Values.service.port }}
```
Ye wo port hai jis par **Service khud** available hoti hai (cluster ke andar se). Jab koi doosra pod `rocket-service:8000` ya jo bhi port ho, us par request bhejega, to yehi port hit hota hai.

```yaml
      targetPort: http
```
Ye batata hai ke request ko **pod ke andar kis port** par forward karna hai. Yahan naam `http` diya hai jo Deployment ke `containerPort` ki `name: http` se match hota hai — Kubernetes us naam ko dekh kar samajh jata hai ke actual port kya hai.

⚠️ **Yehi wo jagah thi jahan aapka masla tha** — `service.port` values.yaml mein `8000` tha lekin container andar `80` par sun raha tha, isliye traffic forward nahi ho pa raha tha (connection refused).

```yaml
      protocol: TCP
      name: http
```
Protocol TCP hai (HTTP bhi TCP par hi chalta hai), aur is port entry ka naam `http` hai.

```yaml
  selector:
    app: {{ .Values.name }}
```
**Sabse important line.** Service khud koi traffic serve nahi karti — ye sirf ek "router" hai jo dekhta hai: "jin pods par label `app: rocket-service` lagi hai, unko main traffic bhejungi." Yehi wo mechanism hai jisse Service ko pata chalta hai ke traffic kis pod(s) tak pahunchana hai — chahe pods kitni baar bhi restart/replace hon, jab tak label same hai, Service unhe dhoond legi.

---

### Poora flow ek nazar mein:
```
Aap → Service (port 8000/80, selector: app=rocket-service)
         ↓ selector se pods dhoondti hai
Pod (label: app=rocket-service, container port 80 par sun raha hai)
```

Koi specific line abhi bhi confusing lage to bata do, us par aur deep jaunga.



apiVersion: apps/v1                          # apps/v1 API group ki resource hai
kind: Deployment                              # Deployment type ki resource bana rahe hain

metadata:                                     # Deployment ki apni identity
  name: {{ .Values.name }}                    # → rocket-service
  namespace: {{ .Values.namespace }}          # → rocket
  labels:
    app: {{ .Values.name }}                   # Deployment par tag/label

spec:                                         # ab batayenge Deployment ko kya karna hai
  replicas: {{ .Values.replicaCount }}        # → 1 (kitne pod copies chalengi)

  selector:                                   # Deployment kaunse pods manage karega
    matchLabels:
      app: {{ .Values.name }}                 # jin pods par ye label ho, wo iske hain

  template:                                   # pod banane ka blueprint
    metadata:
      labels:
        app: {{ .Values.name }}               # naye pod par yehi label lagegi
    spec:
      containers:
        - name: {{ .Values.name }}            # container ka naam
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          # → shahiddevops1/rocket-site:latest
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          # → Always (har baar fresh image pull karo)
          ports:
            - name: http                      # port ka naam (Service isse match karegi)
              containerPort: {{ .Values.service.port }}
              # → 80 (nginx isi port par sun raha hai)
              protocol: TCP
