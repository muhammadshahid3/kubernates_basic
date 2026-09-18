# Rocket Service — Kubernetes Helm Chart Documentation

Ye document `rocket-service` application ke liye use hone wale **Deployment** aur **Service** Kubernetes manifests ko explain karta hai. Dono files Helm templates hain, matlab name, namespace, image, aur ports jaisi values `values.yaml` se inject hoti hain.

---

## Table of Contents

- [Overview](#overview)
- [Deployment Manifest](#deployment-manifest)
- [Service Manifest](#service-manifest)
- [Request Flow](#request-flow)
- [Known Issue & Fix](#known-issue--fix)

---

## Overview

| Resource   | API Group | Purpose                                                        |
|------------|-----------|------------------------------------------------------------------|
| Deployment | `apps/v1` | Pods ka lifecycle manage karta hai — creation, scaling, restarts, rollouts |
| Service    | `v1`      | Pods tak stable network endpoint deta hai taake traffic route ho sake |

---

## Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
```
Ye batata hai ke resource **Deployment** hai, jo `apps/v1` API group mein aata hai (jabke `Pod` aur `Service` core `v1` group mein aate hain). Deployment ka kaam hai pods manage karna: kitne chalne hain, crash hone par dobara banana, aur update/rollback handle karna.

```yaml
metadata:
  name: {{ .Values.name }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.name }}
```
| Field       | Resolved Value    | Description                                              |
|-------------|--------------------|------------------------------------------------------------|
| `name`      | `rocket-service`   | Deployment ki identity (`kubectl get deployment` se dikhegi) |
| `namespace` | `rocket`           | Wo namespace jahan ye Deployment banta hai                |
| `labels`    | `app: rocket-service` | Deployment object par lagi organizational tag (filtering ke liye, jaise `kubectl get deploy -l app=rocket-service`). Iska pods par koi asar nahi hota. |

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
```
Batata hai ke kitne identical pods chalne chahiye. Filhal `1` set hai. Isko badha kar (jaise `3`) karne se Kubernetes 3 identical pods chala dega — load balancing aur high availability ke liye.

```yaml
  selector:
    matchLabels:
      app: {{ .Values.name }}
```
Deployment ko batata hai ke kaunse pods uske hain: jis bhi pod par label `app: rocket-service` lagi ho, wo Deployment ke control mein aayega. Ye Deployment aur uske pods ke darmiyan ka connection hai.

```yaml
  template:
    metadata:
      labels:
        app: {{ .Values.name }}
```
Ye **naye pod ka blueprint** hai. Jab bhi Deployment ko pod banana ho (pehli baar, crash ke baad, scale-up mein), isi template ko copy karke pod banayega. Yahan lagayi gayi label (`app: rocket-service`) upar wale `selector.matchLabels` se match karti hai — isi wajah se Deployment apne pods ko pehchan leta hai.

```yaml
    spec:
      containers:
        - name: {{ .Values.name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```
| Field              | Resolved Value                          | Description                                            |
|--------------------|-------------------------------------------|------------------------------------------------------|
| `name`             | `rocket-service`                        | Pod ke andar container ka naam                        |
| `image`            | `shahiddevops1/rocket-site:latest`      | Wo Docker image jo pull karke chalayi jayegi           |
| `imagePullPolicy`  | `Always`                                | Har baar pod banate waqt fresh image pull karo, taake latest version mile |

```yaml
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
```
Batata hai ke app container ke andar kis port par sun rahi hai (nginx port `80` par listen karta hai). Ye sirf **documentation/info** ke liye hai — port ko actually open ya block ye line nahi karti. Container us port par listen karega ya nahi, ye image ke Dockerfile par depend karta hai.

---

## Service Manifest

```yaml
apiVersion: v1
kind: Service
```
Service core `v1` API group mein aati hai. Service ka kaam hai pods ko **stable network address** dena. Pods create/delete hote rehte hain (naye naam, naye IP milte hain), lekin Service ka apna fixed naam/IP hamesha wahi rehta hai jisse traffic bheja ja sakta hai.

```yaml
metadata:
  name: {{ .Values.name }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.name }}
```
Same pattern jaisa Deployment mein tha — Service ka naam, namespace, aur uski apni organizational label.

```yaml
spec:
  type: {{ .Values.service.type }}
```
Batata hai ke Service kaise expose hogi. `ClusterIP` ka matlab hai ye sirf **cluster ke andar hi accessible** hai — bahar se nahi. Isi wajah se bahar se access ke liye `kubectl port-forward` karna padta hai.

| Service Type   | Accessibility                                              |
|-----------------|-------------------------------------------------------------|
| `ClusterIP`     | Sirf cluster ke andar se access (default)                  |
| `NodePort`      | Node ke IP + fixed port ke zariye bahar se access           |
| `LoadBalancer`  | Cloud ka external load balancer provision karta hai         |

```yaml
  ports:
    - port: {{ .Values.service.port }}
```
Ye wo port hai jis par **Service khud** cluster ke andar available hoti hai. Koi bhi pod `rocket-service:<port>` par request bhejega to yehi port hit hoga.

```yaml
      targetPort: http
```
Batata hai ke request ko **pod ke andar kis port** par forward karna hai. Yahan naam `http` diya gaya hai jo Deployment ke `containerPort` ki `name: http` se match karta hai — Kubernetes is naam ko resolve karke actual port number nikal leta hai.

> ⚠️ **Asal masle ki wajah:** `values.yaml` mein `service.port` ko `8000` set kiya gaya tha, lekin container asal mein port `80` par listen kar raha tha. Isi mismatch ki wajah se traffic forward nahi ho pa raha tha aur connection-refused error aa raha tha.

```yaml
      protocol: TCP
      name: http
```
Protocol `TCP` hai (HTTP bhi TCP par hi chalta hai), aur is port entry ka naam `http` hai.

```yaml
  selector:
    app: {{ .Values.name }}
```
**Service ki sabse important line.** Service khud koi traffic serve nahi karti — ye ek router ki tarah kaam karti hai. Ye kehti hai: "jis bhi pod par label `app: rocket-service` lagi hai, usko traffic bhejo." Yehi mechanism hai jisse Service ko pata chalta hai ke traffic kis pod(s) tak pahunchana hai — chahe pods kitni bhi baar restart/replace hon, jab tak label same hai, Service unhe dhoond legi.

---

## Request Flow

```
Client
  │
  ▼
Service (port 8000/80 → selector: app=rocket-service)
  │  selector se matching pods dhoondti hai
  ▼
Pod (label: app=rocket-service, container port 80 par sun raha hai)
```

---

## Known Issue & Fix

**Symptom:** Service access karte waqt connection refused aata tha.

**Wajah:** `service.port` (`8000`) container ke actual listening port (`80`) se match nahi kar raha tha, aur/ya `targetPort` sahi tarah resolve nahi ho raha tha.

**Fix:** Ensure karo ke Service ka `targetPort`, Deployment mein diye gaye `containerPort` (ya uske `name`) se match kare, taake traffic Service se pod ke asal listening port tak sahi tarah forward ho sake.

---

## Full Annotated Deployment Reference

```yaml
apiVersion: apps/v1                          # apps/v1 API group ki resource
kind: Deployment                              # Deployment resource bana rahe hain

metadata:                                     # Deployment ki apni identity
  name: {{ .Values.name }}                    # → rocket-service
  namespace: {{ .Values.namespace }}          # → rocket
  labels:
    app: {{ .Values.name }}                   # Deployment par tag/label

spec:                                         # Deployment ko kya karna hai
  replicas: {{ .Values.replicaCount }}        # → 1 (kitne pod copies chalengi)

  selector:                                   # Deployment kaunse pods manage karega
    matchLabels:
      app: {{ .Values.name }}                 # Jin pods par ye label ho, wo iske hain

  template:                                   # Pod banane ka blueprint
    metadata:
      labels:
        app: {{ .Values.name }}               # Naye pod par yehi label lagegi
    spec:
      containers:
        - name: {{ .Values.name }}            # Container ka naam
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          # → shahiddevops1/rocket-site:latest
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          # → Always (har baar fresh image pull karo)
          ports:
            - name: http                      # Port ka naam (Service isse match karegi)
              containerPort: {{ .Values.service.port }}
              # → 80 (nginx isi port par sun raha hai)
              protocol: TCP
```
