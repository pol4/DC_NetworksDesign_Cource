# Построение Underlay сети (eBGP + BFD) в архитектуре Leaf-Spine

Этот репозиторий содержит конфигурационные файлы и документацию для построения Underlay-сети с использованием протокола eBGP в архитектуре Leaf-Spine.

## 1. Топология сети

Архитектура состоит из 2-х Spine коммутаторов и 3-х Leaf коммутаторов.
- К `leaf1` и `leaf2` подключено по одному серверу.
- К `leaf3` подключено два сервера.

## 2. План IP-адресации

### 2.1. Underlay Network (Инфраструктурная сеть)

- **Loopback-интерфейсы (`/32`):** Блок `192.168.0.0/24`. Используются для идентификации роутеров (Router ID) и в качестве next-hop в BGP.
- **Point-to-Point линки (`/31`):** Блок `10.0.0.0/24`. Используются для физического соединения Leaf и Spine коммутаторов.

| Устройство | Интерфейс | IP-адрес/Маска | Назначение |
|------------|-------------|------------------|--------------------------------|
| **Spines** |             |                  |                                |
| spine1     | Loopback0   | `192.168.0.1/32` | Router-ID, Management IP       |
|            | Ethernet1   | `10.0.0.0/31`    | Линк к leaf1                   |
|            | Ethernet2   | `10.0.0.2/31`    | Линк к leaf2                   |
|            | Ethernet3   | `10.0.0.4/31`    | Линк к leaf3                   |
| spine2     | Loopback0   | `192.168.0.2/32` | Router-ID, Management IP       |
|            | Ethernet1   | `10.0.0.6/31`    | Линк к leaf1                   |
|            | Ethernet2   | `10.0.0.8/31`    | Линк к leaf2                   |
|            | Ethernet3   | `10.0.0.10/31`   | Линк к leaf3                   |
| **Leaves** |             |                  |                                |
| leaf1      | Loopback0   | `192.168.0.101/32`| Router-ID, Management IP       |
|            | Ethernet1   | `10.0.0.1/31`    | Линк к spine1                  |
|            | Ethernet2   | `10.0.0.7/31`    | Линк к spine2                  |
| leaf2      | Loopback0   | `192.168.0.102/32`| Router-ID, Management IP       |
|            | Ethernet1   | `10.0.0.3/31`    | Линк к spine1                  |
|            | Ethernet2   | `10.0.0.9/31`    | Линк к spine2                  |
| leaf3      | Loopback0   | `192.168.0.103/32`| Router-ID, Management IP       |
|            | Ethernet1   | `10.0.0.5/31`    | Линк к spine1                  |
|            | Ethernet2   | `10.0.0.11/31`   | Линк к spine2                  |

### 2.2. План BGP AS

| Устройство | Номер AS | Тип |
|------------|----------|-----|
| spine1     | 65001    | eBGP|
| spine2     | 65001    | eBGP|
| leaf1      | 65011    | eBGP|
| leaf2      | 65012    | eBGP|
| leaf3      | 65013    | eBGP|

### 2.3. Overlay/Access Network (Сети для серверов)

| Устройство | VLAN ID | Интерфейс VLAN | Подсеть (Gateway)        | Назначение             |
|------------|---------|----------------|--------------------------|------------------------|
| leaf1      | 10      | Vlan10         | `172.16.10.254/24`       | Сеть для server1       |
| leaf2      | 20      | Vlan20         | `172.16.20.254/24`       | Сеть для server2       |
| leaf3      | 30      | Vlan30         | `172.16.30.254/24`       | Сеть для server3, server4 |

## 3. Логика работы eBGP

Протокол eBGP используется для построения Underlay-сети, обеспечивая полную IP-связность между всеми коммутаторами (Leaf и Spine).

- **`router bgp <AS>`**: Запускает процесс BGP с указанным номером автономной системы (AS).
- **`router-id`**: В качестве ID роутера используется IP-адрес с интерфейса `Loopback0`.
- **`neighbor <ip> remote-as <AS>`**: Устанавливает eBGP-соседство с другим коммутатором, находящимся в другой AS.
- **`address-family ipv4`**: Конфигурация для обмена IPv4 маршрутами.
- **`network <prefix>`**: Анонсирует указанную сеть (префикс) в BGP. В нашей конфигурации каждый коммутатор анонсирует свой Loopback и подключенные к нему сети серверов.
- **`neighbor <ip> activate`**: Активирует обмен маршрутами с указанным соседом в рамках текущей адресной семьи.
- **`neighbor <ip> bfd`**: Включает BFD для данного BGP-соседа, что ускоряет обнаружение сбоев.

## 4. Файлы конфигурации

- `spine1.conf`
- `spine2.conf`
- `leaf1.conf`
- `leaf2.conf`
- `leaf3.conf`

## 5. Порядок развертывания и проверки

1.  Загрузить соответствующую конфигурацию на каждое устройство.
2.  **Проверка BGP:**
    - На каждом коммутаторе выполнить команды `show ip bgp summary`, `show ip bgp detail` и `show ip bgp neighbors`, чтобы убедиться, что BGP-сессии установлены (состояние `Estab`) и маршруты получены.
    - На любом коммутаторе выполнить `show ip route bgp`, чтобы проверить, что в таблице маршрутизации присутствуют маршруты ко всем `Loopback` интерфейсам и `VLAN` интерфейсам других коммутаторов.

    ### Пример вывода `show ip bgp summary` (на spine1):
    ```
    spine1#show ip bgp summary
    BGP summary information for VRF default
    Router identifier 192.168.0.1, local AS number 65001
    Neighbor Status Codes: m - Under maintenance
      Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc PfxAdv
      Link-to-leaf1            10.0.0.1 4 65011             26        23    0    0 00:13:48 Estab   2      2      5
      Link-to-leaf2            10.0.0.3 4 65012             26        23    0    0 00:13:48 Estab   2      2      5
      Link-to-leaf3            10.0.0.5 4 65013             24        23    0    0 00:13:48 Estab   2      2      5
    ```

    ### Пример вывода `show ip route bgp` (на spine1):
    ```
    spine1#show ip route bgp

    VRF: default
    Source Codes:
           C - connected, S - static, K - kernel,
           O - OSPF, O IA - OSPF inter area, O E1 - OSPF external type 1,
           O E2 - OSPF external type 2, O N1 - OSPF NSSA external type 1,
           O N2 - OSPF NSSA external type2, O3 - OSPFv3,
           O3 IA - OSPFv3 inter area, O3 E1 - OSPFv3 external type 1,
           O3 E2 - OSPFv3 external type 2,
           O3 N1 - OSPFv3 NSSA external type 1,
           O3 N2 - OSPFv3 NSSA external type2, B - Other BGP Routes,
           B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
           I L2 - IS-IS level 2, A B - BGP Aggregate,
           A O - OSPF Summary, NG - Nexthop Group Static Route,
           V - VXLAN Control Service, M - Martian,
           DH - DHCP client installed default route,
           DP - Dynamic Policy Route, L - VRF Leaked,
           G  - gRIBI, RC - Route Cache Route,
           CL - CBF Leaked Route

     B E      172.16.10.0/24 [200/0]
               via 10.0.0.1, Ethernet1
     B E      172.16.20.0/24 [200/0]
               via 10.0.0.3, Ethernet2
     B E      172.16.30.0/24 [200/0]
               via 10.0.0.5, Ethernet3
     B E      192.168.0.101/32 [200/0]
               via 10.0.0.1, Ethernet1
     B E      192.168.0.102/32 [200/0]
               via 10.0.0.3, Ethernet2
     B E      192.168.0.103/32 [200/0]
               via 10.0.0.5, Ethernet3
    ```

3.  **Проверка BFD:**
    - На каждом коммутаторе выполнить команду `show bfd neighbors`, чтобы убедиться, что BFD-сессии установлены (`Status: Up`).
    - Также статус BFD для каждого BGP-соседа можно увидеть в выводе `show ip bgp neighbors` (строка `BFD is enabled and state is Up`).

    ### Пример вывода `show ip bgp neighbors` (фрагмент для одного соседа на spine1):
    ```
    BGP neighbor is 10.0.0.1, remote AS 65011, external link
     Description: Link-to-leaf1
      ...
      BGP state is Established, up for 00:20:41
      ...
      BFD is enabled and state is Up
      ...
    ```

4.  **Проверка связности:**
    - С каждого коммутатора выполнить `ping` до `Loopback` IP-адресов всех остальных коммутаторов в сети.
    - С сервера `server1` (`172.16.10.1`) выполнить `ping` до своего шлюза `172.16.10.254`.
    - Проверить связность между серверами, например, с `PC1` (который может быть подключен к `leaf1` или быть отдельным тестовым хостом) до серверов за `leaf2` и `leaf3`.

    ### Пример проверки связности с `PC1`:
    ```
    PC1> ping 172.16.30.2

    84 bytes from 172.16.30.2 icmp_seq=1 ttl=61 time=5.743 ms
    84 bytes from 172.16.30.2 icmp_seq=2 ttl=61 time=3.984 ms
    84 bytes from 172.16.30.2 icmp_seq=3 ttl=61 time=3.762 ms
    84 bytes from 172.16.30.2 icmp_seq=4 ttl=61 time=4.191 ms
    84 bytes from 172.16.30.2 icmp_seq=5 ttl=61 time=3.352 ms

    PC1> ping 172.16.30.1

    84 bytes from 172.16.30.1 icmp_seq=1 ttl=61 time=3.790 ms
    84 bytes from 172.16.30.1 icmp_seq=2 ttl=61 time=3.735 ms
    84 bytes from 172.16.30.1 icmp_seq=3 ttl=61 time=3.908 ms
    84 bytes from 172.16.30.1 icmp_seq=4 ttl=61 time=3.864 ms
    84 bytes from 172.16.30.1 icmp_seq=5 ttl=61 time=3.638 ms

    PC1> ping 172.16.20.1

    84 bytes from 172.16.20.1 icmp_seq=1 ttl=61 time=4.799 ms
    84 bytes from 172.16.20.1 icmp_seq=2 ttl=61 time=4.523 ms
    84 bytes from 172.16.20.1 icmp_seq=3 ttl=61 time=3.799 ms
    84 bytes from 172.16.20.1 icmp_seq=4 ttl=61 time=3.801 ms
    84 bytes from 172.16.20.1 icmp_seq=5 ttl=61 time=4.120 ms
    ```

## 6. Проверка ECMP (Equal-Cost Multi-Path)

ECMP в BGP позволяет распределять трафик по нескольким путям с одинаковой стоимостью. В нашей топологии у каждого Leaf-коммутатора есть два равноценных пути до любой удаленной подсети через `spine1` и `spine2`.

Наличие ECMP можно проверить командой `show ip route` на любом из Leaf-коммутаторов.

### Пример вывода для `leaf1`:

```
leaf1#show ip route

VRF: default
Codes: C - connected, S - static, R - RIP, B - BGP, O - OSPF,
       IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type 2, I - IS-IS,
       L1 - IS-IS level-1, L2 - IS-IS level-2,
       * - candidate default, > - best route, % - ECMP head

Gateway of last resort is not set

 C        10.0.0.0/31 is directly connected, Ethernet1
 C        10.0.0.6/31 is directly connected, Ethernet2
 C        172.16.10.0/24 is directly connected, Vlan10
 C        192.168.0.101/32 is directly connected, Loopback0
 B >      192.168.0.1/32 [20/0]
           via 10.0.0.0, Ethernet1
 B >      192.168.0.2/32 [20/0]
           via 10.0.0.6, Ethernet2
 B %      172.16.20.0/24 [20/0]
   >       via 10.0.0.0, Ethernet1
   >       via 10.0.0.6, Ethernet2
 B %      172.16.30.0/24 [20/0]
   >       via 10.0.0.0, Ethernet1
   >       via 10.0.0.6, Ethernet2
 B %      192.168.0.102/32 [20/0]
   >       via 10.0.0.0, Ethernet1
   >       via 10.0.0.6, Ethernet2
 B %      192.168.0.103/32 [20/0]
   >       via 10.0.0.0, Ethernet1
   >       via 10.0.0.6, Ethernet2
```

### Анализ вывода:

Ключевые моменты, подтверждающие работу ECMP:
-   **Маршрут к `172.16.20.0/24` (сеть за `leaf2`):**
    Символ `%` перед маршрутом и два `next-hop` (`via 10.0.0.0` и `via 10.0.0.6`) указывают на то, что это ECMP-маршрут. Трафик к этой сети будет балансироваться между `spine1` и `spine2`.
-   Аналогичная картина наблюдается для всех маршрутов, полученных от других Leaf-коммутаторов. Это доказывает, что `leaf1` будет использовать оба своих аплинка для отправки трафика, эффективно распределяя нагрузку.
```