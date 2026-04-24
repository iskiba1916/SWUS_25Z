# 🔐 Kubernetes Network Policy Management

> **Zarządzanie politykami sieciowymi w systemie Kubernetes**  
> Projekt zrealizowany w ramach przedmiotu **Sieci Wielousługowe (SWUS)**  
> Wydział Elektroniki i Technik Informacyjnych, Politechnika Warszawska

---

## 👥 Autorzy

Igor Skiba, Radosław Rytel-Kuc

 **Data:** 25 stycznia 2026

---

## 📋 Opis projektu

Projekt demonstruje pełny cykl zabezpieczania komunikacji wewnątrz klastra Kubernetes poprzez implementację **polityk sieciowych**. Domyślna, płaska sieć Kubernetes pozwala na swobodną komunikację każdego poda z każdym - projekt udowadnia to zagrożenie i skutecznie je eliminuje.

### Problem

```
Frontend ──────────────────────────────► Baza Danych  ⚠️ LUKA BEZPIECZEŃSTWA
Baza Danych ────────────────────────────► Backend     ⚠️ LUKA BEZPIECZEŃSTWA
```

### Rozwiązanie

```
Frontend ──► Backend ──► Baza Danych   ✅ BEZPIECZNA ŚCIEŻKA
Frontend ──► Baza Danych               ❌ ZABLOKOWANE
Baza Danych ──► Backend                ❌ ZABLOKOWANE
```

---

## 🏗️ Architektura

Klaster oparty na **3 węzłach** (VirtualBox + Ubuntu 22.04 LTS):

```
┌─────────────────────────────────────────────────┐
│                  k8s-master                     │
│              (control-plane)                    │
│         192.168.0.146  ·  v1.34.2               │
└────────────────────┬────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                        ▼
┌───────────────┐        ┌───────────────┐
│  k8s-worker1  │        │  k8s-worker2  │
│ 192.168.0.116 │        │ 192.168.0.185 │
└───────────────┘        └───────────────┘
```

### Wdrożona aplikacja trójwarstwowa

| Warstwa | Obraz | Port | Repliki |
|---|---|---|---|
| **Frontend** | `node:20-alpine` | `3000` | 2 |
| **Backend** | `nginx` | `81` | 2 |
| **Baza Danych** | `redis` | `6379` | 2 |

---

## ⚙️ Stos technologiczny

| Komponent | Wersja / Szczegóły |
|---|---|
| **Kubernetes** | v1.34.2 |
| **CNI** | Cilium (`--set ipam.mode=kubernetes`) |
| **Container Runtime** | Docker + `cri-dockerd` |
| **System operacyjny** | Ubuntu 22.04 LTS (amd64) |
| **Inicjalizacja klastra** | `kubeadm` |
| **Pod CIDR** | `10.244.1.0/16` |
| **Service CIDR** | `10.96.1.0/12` |

---

## 🚀 Etapy realizacji

### Etap 1 — Utworzenie klastra Kubernetes

1. Wyłączenie mechanizmu SWAP na wszystkich węzłach
2. Konfiguracja parametrów jądra (`sysctl`, moduł `br_netfilter`)
3. Instalacja Docker + `cri-dockerd`
4. Instalacja `kubeadm`, `kubelet`, `kubectl`
5. Inicjalizacja klastra (`kubeadm init`)
6. Dołączenie węzłów roboczych (`kubeadm join`)
7. Instalacja CNI Cilium

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.0.146 \
  --pod-network-cidr=10.244.1.0/16 \
  --service-cidr=10.96.1.0/12 \
  --cri-socket unix:///var/run/cri-dockerd.sock
```

### Etap 2 — Wdrożenie aplikacji

```bash
kubectl apply -f wdrozenie.yaml
kubectl get pods -o wide
```

Wszystkie 6 podów (2 repliki × 3 serwisy) osiągnęły status `Running` z adresami IP z puli `10.244.x.x`.

### Etap 3 — Implementacja polityk sieciowych

```bash
kubectl apply -f polityki.yaml
kubectl get networkpolicies
```

---

## 🛡️ Polityki sieciowe

Zastosowano strategię **Default Deny** — zablokowanie całego ruchu z selektywnym otwieraniem niezbędnych ścieżek.

### `db-policy` — Baza danych

```yaml
# Ingress: tylko z podów app=backend na porcie 6379
# Egress:  tylko DNS (UDP/53) do kube-dns
policyTypes:
  - Ingress
  - Egress
```

### `backend-policy` — Backend

```yaml
# Ingress: tylko z podów app=frontend na porcie 81
policyTypes:
  - Ingress
```

### `frontend-policy` — Frontend

```yaml
# Ingress: nieograniczony (dostęp dla użytkowników zewnętrznych)
policyTypes:
  - Ingress
ingress:
  - {}
```

---

## 🧪 Wyniki testów

Testy przeprowadzono przy użyciu `netcat` (`nc -zv` / `nc -zv -w 5`).

### Przed wdrożeniem polityk

| Kierunek | Port | Wynik | Ocena |
|---|---|---|---|
| Frontend → Backend | 81 | ✅ sukces | Pożądane |
| Backend → Baza Danych | 6379 | ✅ sukces | Pożądane |
| **Frontend → Baza Danych** | **6379** | **✅ sukces** | **⚠️ LUKA** |
| **Baza Danych → Backend** | **81** | **✅ sukces** | **⚠️ LUKA** |

### Po wdrożeniu polityk

| Kierunek | Port | Wynik | Ocena |
|---|---|---|---|
| Frontend → Backend | 81 | ✅ sukces | Pożądane |
| Backend → Baza Danych | 6379 | ✅ sukces | Pożądane |
| **Frontend → Baza Danych** | **6379** | **❌ timeout** | **✅ Zablokowane** |
| **Baza Danych → Backend** | **81** | **❌ timeout** | **✅ Zablokowane** |

> **Uwaga:** Wynik `Operation timed out` (zamiast `Connection refused`) świadczy o tym, że pakiety SYN są porzucane przez filtr eBPF w Cilium — bezpieczniejsze zachowanie, gdyż nie ujawnia istnienia usługi.

---

## 💡 Wnioski

Projekt potwierdza trzy kluczowe korzyści z zastosowania NetworkPolicy:

- **Minimalizacja wektora ataku** — kompromitacja frontendu nie daje dostępu do bazy danych
- **Izolacja komponentów** — baza danych nie może inicjować połączeń wychodzących (ochrona przed eksfiltrację danych)
- **Zgodność z regulacjami** — wymuszanie ścisłych reguł przepływu danych jest wymagane m.in. przez **RODO**

Bezpieczeństwo w klastrze musi być dynamiczne i ściśle powiązane z **tożsamością logiczną obciążeń**, a nie tylko z ich lokalizacją w sieci.

---

## 📚 Literatura

1. Kubernetes Documentation — [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
2. Kubernetes Documentation — [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
3. Kubernetes Documentation — [Cilium Network Policy](https://kubernetes.io/docs/tasks/administer-cluster/networkpolicy-provider/cilium-network-policy/)
4. Cilium Documentation — [Installing Cilium](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)
5. Kubernetes Documentation — [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

---

<p align="center">
  <sub>Politechnika Warszawska · Wydział Elektroniki i Technik Informacyjnych · SWUS 2025/2026</sub>
</p>
