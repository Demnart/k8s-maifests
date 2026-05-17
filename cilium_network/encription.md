# Шифрование трафика

Cilium поддерживае прозрачное шифрование трафика между подами с помощью WireGuard или IPsec. Шифрование работает на уровни сети - в приложение не нужно ничего менять.

## WireGuard 

WireGuard - современный протокол шифрования, встроенный в ядро Linux(начиная с версии 5.6). Cilium использует WireGuard для создания зашифрованных туннелей между нодами кластера.

### Преимущества WireGuard

  - **Простота** - один параметр Helm для включения.  
  - **Производительность** - реализация в ядре, минимальные накладные расходы.  
  - **Автоматическая ротация ключей** - каждая нода генерирует пару ключей, обмен происходит автоматически через Cilium.  

#### Проверка поддержки:  

WireGuard должен быть доступен как модуль ядра на всех нодах: 

```sh
lsmod | grep WireGuard
```
```sh
wireguard             114688  0
curve25519_x86_64      36864  1 wireguard
libchacha20poly1305    16384  1 wireguard
libcurve25519_generic    49152  2 curve25519_x86_64,wireguard
ip6_udp_tunnel         16384  3 geneve,wireguard,vxlan
udp_tunnel             32768  3 geneve,wireguard,vxlan

```  
Если модуль не загружен:

```sh
sudo modprobe wireguard
```

### Вклюлчение WireGuard

```sh
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --namespace kube-system \
  --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=wireguard
```

После обновления Helm необходимо перезапустить Cilium агенты:

```sh
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout status daemonset/cilium -n kube-system --timeout=120s
```
```sh
daemon set "cilium" successfully rolled out
```

### Проверка

```sh
kubectl exec -n kube-system ds/cilium -- cilium-dbg encrypt status
```
```sh
Encryption: Wireguard                 
Interface: cilium_wg0
        Public key: teEB1NlFmMyk6cWz34iATkRLfvyYKZLkX4O1zYpGVTY=
        Number of peers: 2
```

  - **Interface: cilium_wg0** - WireGuard интерфейс создан.  
  - **Number of peers: 2** - установлены туннели с 2 другими нодами(в кластере 3 ноды).  

Детальная информация:

```sh
kubectl exec -n kube-system ds/cilium -- cilium-dbg status --verbose 2>/dev/null | grep Encryption
```
```sh
Encryption:       Wireguard       [NodeEncryption: Disabled, cilium_wg0 (Pubkey: teEB1NlFmMyk6cWz34iATkRLfvyYKZLkX4O1zYpGVTY=, Port: 51871, Peers: 2)]
```

### Что шифруется

По умолчанию WireGuard шифрует **трафик между подами на разных нодах**. Трафик между подами на одной ноде не шифруется(он не выходит за пределы ноды).  

```txt
Pod A (r4)                                    Pod B (r5)
    │                                              ▲
    ▼                                              │
cilium_wg0 ──── WireGuard tunnel (UDP:51871) ──── cilium_wg0
    │          зашифрованный трафик                 │
    ▼                                              ▲
   eth0 ────────── физическая сеть ─────────────  eth0
```

Для шифрования трафика между нодами (включая трафик kubelet, API server и т.д.)
можно включить `encryption.nodeEncryption`:

```sh
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --namespace kube-system \
  --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=wireguard \
  --set encryption.nodeEncryption=true
```

### Отключение шифрования

```sh
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --namespace kube-system \
  --reuse-values \
  --set encryption.enabled=false
```

```sh
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout status daemonset/cilium -n kube-system --timeout=180s
```

Проверим, что шифрование отключено:

```sh
kubectl exec -n kube-system ds/cilium -- cilium-dbg encrypt status
```

```txt
Encryption: Disabled
```

## IPsec

Cilium также поддерживает IPsec как альтернативу WireGuard. IPsec может быть
предпочтительнее в средах с требованиями FIPS-совместимости.

Включение:

```sh
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
  --namespace kube-system \
  --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=ipsec
```

> **Примечание:** WireGuard рекомендуется как более простой и производительный вариант.
> IPsec имеет смысл только при специфических требованиях к криптографии.

---