# AI 教學
這份筆記濃縮了我們這九個階段的所有精華，完全依照實務機房的建置順序編排。你可以把它當作未來 CCNA 考前複習，或是日後管理 Proxmox 實體網段時的「作弊小抄」。

---

## 🟢 階段一：交換器底層與高可用性 (Layer 2)

**核心觀念：**

* **VLAN (虛擬區域網路)：** 在同一台實體 Switch 上，切出不同的邏輯廣播網域，達到實體隔離的效果。
* **EtherChannel (LACP)：** 將多條實體線路綁定成一條邏輯線路，增加頻寬並提供「斷線不斷網」的備援能力。
* **Trunk (骨幹通道)：** 允許「不同 VLAN 的封包」在同一條實體線路上傳輸，靠的是 IEEE 802.1Q 標籤。

### 1. SW-Edge (邊緣交換器) 設定

這台設備負責連接終端設備（如 Vaultwarden 伺服器、Monitor-PC），並透過備援線路連回核心。

```text
enable
configure terminal

! 1. 建立 VLAN
vlan 10
 name Secure_Services
vlan 20
 name NOC_Monitor
exit

! 2. 設定存取埠 (Access Port) 給終端設備
interface gig0/3
 switchport mode access
 switchport access vlan 10
interface gig0/4
 switchport mode access
 switchport access vlan 20

! 3. 綁定 EtherChannel (LACP) 往核心交換器
interface range gig0/1-2
 channel-group 1 mode active
exit

! 4. 將綁定好的邏輯通道設定為 Trunk
interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
exit

```

### 2. SW-Core (核心交換器) 設定

負責匯聚邊緣交換器的流量，並將所有 VLAN 的封包打包送往路由器。

```text
enable
configure terminal

! 1. 建立 VLAN (核心層也必須要有 VLAN 的認知)
vlan 10
vlan 20
exit

! 2. 綁定 EtherChannel 接應 SW-Edge
interface range gig0/1-2
 channel-group 1 mode active
exit
interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
exit

! 3. 設定連往 Router 的介面 (實作中我們使用了 FastEthernet 0/3)
interface fastEthernet 0/3
 switchport mode trunk
 switchport trunk allowed vlan 10,20
exit

```

---

## 🔵 階段二：單臂路由 (Layer 3 Router-on-a-Stick)

**核心觀念：**
路由器預設網孔是關閉的。我們在一個實體網孔上，切出多個「虛擬子介面 (Sub-interfaces)」，每個子介面負責拆解特定 VLAN 的 802.1Q 標籤，並擔任該網段的預設閘道 (Default Gateway)。

### R-Core (路由器) 基礎路由設定

```text
enable
configure terminal

! 1. 喚醒實體網孔
interface gig0/0
 no shutdown
exit

! 2. 建立 VLAN 10 虛擬子介面與閘道 IP
interface gig0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.254 255.255.255.0
exit

! 3. 建立 VLAN 20 虛擬子介面與閘道 IP
interface gig0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.254 255.255.255.0
exit

```

---

## 🟡 階段三：自動發放 IP (DHCP Server)

**核心觀念：**
路由器兼任 DHCP 伺服器，讓終端設備即插即用。**關鍵考點：必須先「排除 (Exclude)」重要設備的 IP，再去建立「發放池 (Pool)」。**

### R-Core (路由器) DHCP 設定

```text
! 1. 排除保留 IP (伺服器與閘道)
ip dhcp excluded-address 192.168.10.1 192.168.10.15
ip dhcp excluded-address 192.168.10.254
ip dhcp excluded-address 192.168.20.1 192.168.20.15
ip dhcp excluded-address 192.168.20.254

! 2. 建立 VLAN 10 的發放池
ip dhcp pool VLAN10_POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.254
 dns-server 8.8.8.8
exit

! 3. 建立 VLAN 20 的發放池
ip dhcp pool VLAN20_POOL
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.254
 dns-server 8.8.8.8
exit

```

---

## 🔴 階段四：存取控制清單 (ACL 防火牆)

**核心觀念：**

* **Top-Down Processing：** 規則由上往下比對，一旦吻合就不再往下看。
* **Implicit Deny：** 清單最末端隱藏著「拒絕所有」，因此務必加上 `permit ip any any`。
* **方向性 (in/out)：** 站在路由器的角度思考。封包從外網段「進入」路由器時，就是 `in`。

### R-Core (路由器) 擴充型 ACL 設定

目標：禁止 VLAN 20 網段的任何人，連線至 Vaultwarden 伺服器 (192.168.10.10)，但放行其他所有網路流量。

```text
! 1. 建立名為 BLOCK_VLAN20 的擴充型 ACL
ip access-list extended BLOCK_VLAN20

! 2. 設定阻擋規則 (deny)
! 語法：deny [協定] [來源網段] [來源遮罩反轉] host [目標IP]
deny ip 192.168.20.0 0.0.0.255 host 192.168.10.10

! 3. 設定放行規則 (permit) - 這是保命符
permit ip any any
exit

! 4. 將 ACL 規則套用在 VLAN 20 的虛擬子介面上 (進入方向)
interface gig0/0.20
 ip access-group BLOCK_VLAN20 in
exit

```

---

## 💡 網管除錯必備 (Cheat Sheet)

在特權模式 (`#`) 下執行的神級指令，這些是你這次抓蟲最常用到的武器：

| 指令 | 實務用途說明 |
| --- | --- |
| `write memory` 或 `wr` | **【極度重要】** 將 RAM 裡的設定永久寫入 NVRAM，防止重開機後設定消失。 |
| `show ip interface brief` | 顯示所有網卡狀態。除錯看 `Status` (管理員是否開啟) 與 `Protocol` (實體線路是否正常)。 |
| `show interfaces trunk` | 檢查 Switch 的 Trunk 有沒有成功建立，以及允許哪些 VLAN 標籤通過。 |
| `show etherchannel summary` | 檢查 LACP 綁定狀態。正常運作應該顯示 `(SU)`；若顯示 `(s)` 代表懸掛故障。 |
| `show access-lists` | 檢查防火牆規則有沒有寫錯，還能看到每條規則「成功攔截了幾個封包」的統計數字。 |


---

## 🟢 階段五：NAT 網路位址轉換 (PAT / Overload)

**核心觀念：**

* **為什麼需要 NAT？** 網際網路 (外網) 路由器不接受、也不幫忙轉發 `192.168.x.x` 這類的私有 IP。
* **PAT (Port Address Translation)：** 允許多台內網電腦，共用 Router 的「一個對外 IP」上網。Router 會透過不同的 Port (通訊埠流水號) 來辨識封包該還給內網的誰，指令中稱之為 `overload`。
* **標準型 ACL (Standard ACL)：** 編號 1~99。只負責圈選「來源 IP」，不看協定，所以語法中不需要（也不能）打 `ip` 這個字。

### R-Core (路由器) NAT 設定

目標：讓 VLAN 10 與 VLAN 20 的私有 IP，透過 Router 對外的 `Gig0/1` 網孔 (8.8.8.254) 偽裝出國。

```text
enable
configure terminal

! 1. 界定「內部」與「外部」網孔
interface gig0/1
 ip nat outside
exit
interface gig0/0.10
 ip nat inside
exit
interface gig0/0.20
 ip nat inside
exit

! 2. 建立標準型 ACL (編號 1)，圈出允許上網的內網網段
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.20.0 0.0.0.255

! 3. 啟動 NAT 轉換 (將名單 1 的 IP，轉換成 gig0/1 的 IP，並開啟多對一超載)
ip nat inside source list 1 interface gig0/1 overload
exit

! 💡 網管除錯必備 (在特權模式 # 下執行)
! show ip nat translations  (查看 Router 正在進行的 IP 與 Port 翻譯紀錄簿)

```

---

### 🗺️ 準備進入 OSPF 新任務

把筆記收好後，請幫我把目前的 Packet Tracer 專案**存檔**。這個檔案已經具備了完整的企業網管底層架構，是你親手打下來的江山！

接著，請開一張**全新的空白畫布**，我們來搭建 OSPF 動態路由的舞台：

1. **放置路由器：** 從下方設備欄拉出 **3 台 Router**（建議選 `2911` 或 `4331` 型號），並將它們命名為 **R1**、**R2**、**R3**。
2. **連成三角形：** 使用黑色實線 (如果網孔是 GigabitEthernet 的話) 或交叉線，將這三台 Router 互相連接，形成一個「三角形」的拓樸：
* R1 接 R2
* R2 接 R3
* R3 接 R1



完成這個簡單的三角形佈線後跟我說。接下來，我會帶你把這三台路由器的 IP 設好，然後敲下 OSPF 指令，見證它們一瞬間「自動交換全圖視野」的魔術！


---

## 🟣  階段6：OSPF 動態路由與除錯

### 🛠️ 步驟一：底層 IP 基礎建設與除錯

路由器天生是「近視眼」，預設只認得插在自己身上的網段。設定動態路由前，實體與虛擬介面的 IP 必須 100% 正確。

#### 1. 介面設定基本功

* **實體網孔 (例如 Gig0/0/0)：** 兩台 Router 之間的跨海大橋，兩端必須在同一個網段。
* **虛擬網孔 (Loopback 0)：** 永遠不會斷線的虛擬網卡，用來模擬背後的村莊，更是 OSPF 選擇「Router ID」的最高指導原則。

#### 2. ⚠️ 網管除錯鐵則：覆蓋 IP 會報錯

如果你不小心把 IP 設錯網孔（例如把 R2 的 `10.23.0.2` 設到了 `Gig0/0/0`），直接敲新 IP 會跳出 `% IP address overlaps` 的錯誤。
**正確解法：先拔舊門牌，再掛新門牌。**

```text
enable
configure terminal
interface gig0/0/0
 no ip address                 ! 拔除舊的錯誤 IP
 ip address 10.12.0.2 255.255.255.0  ! 設定正確 IP
 no shutdown                   ! 確保網孔開啟
exit

```

---

### 📡 步驟二：三台 Router 的 OSPF 完整宣告設定

這一步是為了讓路由器開啟廣播系統，自動交換地圖。
**核心語法：`network [網段IP] [反向遮罩] area [區域號碼]**`

* **Process ID (`router ospf 1`)：** 這是本地端的軟體執行緒編號，每台都可以一樣，只是為了網管方便。
* **反向遮罩 (`0.0.0.255`)：** `0` 代表這幾個位置的數字必須嚴格比對，`255` 代表不管。
* **Area 0 (骨幹區域)：** 這是 OSPF 的交友暗號，宣告的網段必須在同一個 Area 才能互相交換地圖。

#### 【R1 完整設定檔】

```text
enable
configure terminal
router ospf 1
 network 1.1.1.0 0.0.0.255 area 0    ! 宣告自己背後的村莊 (Loopback 0)
 network 10.12.0.0 0.0.0.255 area 0  ! 宣告通往 R2 的橋樑
 network 10.13.0.0 0.0.0.255 area 0  ! 宣告通往 R3 的橋樑
exit

```

#### 【R2 完整設定檔】

```text
enable
configure terminal
router ospf 1
 network 2.2.2.0 0.0.0.255 area 0    ! 宣告自己背後的村莊 (Loopback 0)
 network 10.12.0.0 0.0.0.255 area 0  ! 宣告通往 R1 的橋樑
 network 10.23.0.0 0.0.0.255 area 0  ! 宣告通往 R3 的橋樑
exit

```

#### 【R3 完整設定檔】

```text
enable
configure terminal
router ospf 1
 network 3.3.3.0 0.0.0.255 area 0    ! 宣告自己背後的村莊 (Loopback 0)
 network 10.13.0.0 0.0.0.255 area 0  ! 宣告通往 R1 的橋樑
 network 10.23.0.0 0.0.0.255 area 0  ! 宣告通往 R2 的橋樑
exit

```

---

### 🧠 步驟三：OSPF 運作底層邏輯 (交朋友三部曲)

當指令敲完後，這三台機器在底層會經歷非常嚴謹的交友流程：

1. **決定 Router ID (網路身分證)：** OSPF 程式會自動去抓路由器身上「數字最大的 Loopback IP」當作全網唯一名字（所以 R1 叫 `1.1.1.1`，R2 叫 `2.2.2.2`）。
2. **打招呼 (Hello)：** 往外大喊「我是 1.1.1.1，我是 Area 0 的人，有人在嗎？」
3. **交換目錄 (Exchange)：** 雙方不會直接塞完整的地圖，而是先交換「DBD 目錄」，看看對方手上有什麼自己不知道的路線。
4. **索取與同步 (Loading to FULL)：** 發現目錄上有自己缺的路線（例如 R1 發現自己沒有 2.2.2.2），才會向對方要求詳細資料。
* **成功指標：** 畫面上彈出 `%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1 on GigabitEthernet0/0/0 from LOADING to FULL, Loading Done`。



---

### 🔍 步驟四：網管驗證與除錯四大神指令 (Verification)

請一律在**特權模式 (`Router#`)** 下執行。這四個指令就是網管工程師的透視鏡：

#### 1. 檢查底層實體連線

```text
show ip interface brief

```

* **看什麼：** 確認所有用到的網孔 IP 是否正確，且狀態必須是 `up / up`（實體燈號亮 / 協定正常）。

#### 2. 檢查 OSPF 自身設定與身分證

```text
show ip protocols

```

* **看什麼：** 確認 `Router ID` 是否正確抓到了 Loopback IP，以及 `Routing for Networks` 底下有沒有漏宣告的網段。

#### 3. 檢查 OSPF 鄰居交友狀態

```text
show ip ospf neighbor

```

* **看什麼：** `Neighbor ID` 必須看到對方的 Loopback IP，且 `State` 欄位必須顯示為 **`FULL`**，代表地圖 100% 同步完成。

#### 4. 驗收最終全網地圖 (Dijkstra 演算法計算結果)

```text
show ip route

```

* **看什麼 (報表解碼)：**
* **`C` (Connected)：** 自己身上直連的網段。
* **`O` (OSPF)：** 透過 OSPF 從鄰居那裡學來的新路線（例如 R1 學到了 `2.2.2.2` 與 `3.3.3.3`）。
* **`[110/2]` 的意義：** `110` 是 OSPF 的預設信任度 (AD值)；`2` 是前往該地的成本 (Cost)。
* **ECMP (等價多重路徑)：** 如果在路由表看到同一個目的地（如 `10.12.0.0/24`）底下跟著兩條 `via`，代表 OSPF 發現左邊和右邊的距離一樣近，自動開啟了負載平衡（流量一人走一邊）。



---

### 💾 步驟五：下班前的最終防線

所有測試與 Ping 都成功後，一定要把 RAM（暫存記憶體）裡的設定寫入 NVRAM（硬碟），否則重開機會全部消失：

```text
write memory

```

*(出現 `[OK]` 即代表存檔完畢)*

---

這份滿配版的筆記，把指令、參數意義、底層邏輯跟報表判讀全部結合在一起了！

了解，這就為你單獨抽出「第七階段」的完整精華，請直接複製這段：


---

## 🟠 階段七：災難演練與流量工程 (OSPF 收斂與 Cost 操控)

**核心觀念：**
動態路由最大的商業價值在於**「高可用性 (HA) 與自動備援」**。
* **收斂 (Convergence)：** 實體線路中斷時，OSPF 能在極短時間內重新計算 Dijkstra 演算法，將流量切換到備用道路。
* **流量工程 (Traffic Engineering)：** 可透過人為修改網孔的 Cost 值，欺騙路由器的大腦，強制引導流量避開壅塞路段。



### 🛠️ 步驟一：確認基準線 (Baseline)

在搞破壞之前，先確認網路的正常狀態作為對照組。
請在 **R1** 的特權模式下，專門檢視前往 R2 (`2.2.2.2`) 的路線：
```text
show ip route 2.2.2.2

```

* **看什麼：**
* `Known via "ospf 1"`：確認是 OSPF 學來的。
* `metric 2`：目前的總成本是 2。
* `via 10.12.0.2 on GigabitEthernet0/0/0`：走直達 R2 的最短路徑。



---

### 🪓 步驟二：拔線測試 (模擬實體線路中斷)

模擬 R1 與 R2 之間的光纖被挖斷。進入 R1 連接 R2 的網孔，將其強制關閉。

```text
enable
configure terminal
interface gig0/0/0
 shutdown          ! 強制關閉網孔，模擬斷線

```

* **系統日誌哀嚎：** 畫面會立刻彈出 `%OSPF-5-ADJCHG: ... from FULL to DOWN, Neighbor Down`，代表 OSPF 發現鄰居斷線，緊急重新計算全網地圖。

---

### 🔍 步驟三：見證自動收斂 (備援路線生效)

網路斷線後，退回特權模式再次查看 R1 的大腦：

```text
end
show ip route 2.2.2.2

```

* **看什麼 (報表變化)：**
* `metric 3`：成本從 2 變成 **3** (因為繞遠路，多跨一台設備)。
* `via 10.13.0.3`：下一跳自動變成 R3！R1 在一秒內決定把封包交給 R3 幫忙轉交。


* **連線驗證：** 執行 `ping 2.2.2.2`，封包依然暢通，網路成功自癒。

---

### 🎛️ 步驟四：進階流量工程 (手動竄改 Cost)

實務上，有些線路雖然直達但頻寬極低（例如跨國備援專線），我們「平常不想走，斷線才准走」。這時可手動調高 OSPF Cost 來引導流量。

```text
enable
configure terminal

! 1. 恢復連線 (等待畫面跳出 LOADING to FULL)
interface gig0/0/0
 no shutdown

! 2. 手動竄改該網孔的成本為極高值 (例如 100)
 ip ospf cost 100
exit

```

---

### ⚖️ 步驟五：驗證流量引導結果

雖然直達門已經打開，但因為大腦已被動過手腳，再來看看 R1 的選路：

```text
end
show ip route 2.2.2.2

```

* **欺騙成功：** 即便直達線路正常運作，R1 依然會選擇 `via 10.13.0.3` (丟給 R3)。因為走直達的 Cost 變成 **101** (網孔成本 100 + Loopback 成本 1)，而繞路走 R3 的 Cost 只有 **3**。路由器永遠選擇最省力的路，成功達成流量引導！




## 🛡️ 階段八：資安防堵與路由最佳化 (Passive Interface)

**核心觀念：**
* **被動介面 (Passive Interface)：** 當 OSPF 宣告了連接「終端設備（如 PC、伺服器或虛擬 Loopback）」的網段時，必須關閉該網孔的 OSPF 交友功能。這能防止駭客偽裝成路由器竄改路由表，同時節省不必要的廣播頻寬浪費。
* **主機路由 (Host Route /32)：** 在設定 Loopback 虛擬網卡時，企業實務會使用 `/32` (`255.255.255.255`) 作為遮罩。這明確告訴 OSPF「這裡只有單一管理 IP，不是一個有 256 個 IP 的村莊」，藉此節省 IP 資源並讓路由表保持乾淨易讀。

> **💡 白話文觀念 (麻瓜理論)：**
> **Passive-Interface 的狀態就是：** R1 明白那個網孔背後「全是麻瓜 (一般電腦)」，不會有其他的 Router。所以 R1 保留了網段的連線能力（依然把這條路徑告訴 R2、R3），但切斷了那個網孔的 OSPF 交友功能。

---

### 🆚 Passive Interface 運作差異

| 網孔狀態 | 發送 OSPF Hello | 接收 OSPF Hello | 網段宣告 | 資安風險 |
| :--- | :--- | :--- | :--- | :--- |
| **未設定 (預設)** | ✅ 每 10 秒對該網孔廣播 | ✅ 允許建立鄰居關係 | ✅ 正常宣告 | ⚠️ **極高** (易遭駭客入侵) |
| **設定 Passive** | ❌ 絕對安靜 (變啞巴) | ❌ 無視丟棄 (變聾子) | ✅ 正常宣告 | 🛡️ **安全** (單向宣告不交友) |

---

### 🛠️ 步驟一：設定被動介面 (封鎖虛擬村莊)

我們以 **R1** 為例，將面向 1 號虛擬村莊 (`Loopback 0`) 的網孔設為被動介面，切斷潛在的資安破口。

```text
enable
configure terminal
router ospf 1
 passive-interface loopback 0    ! 將 Loopback 0 設為被動介面
exit

```

*(💡 網管實務提醒：未來在機房中，只要路由器的實體網孔是接往「員工交換器」或「伺服器群」，都必須進入 OSPF 補上這行指令！)*

---

### 🔍 步驟二：驗證防護與宣告狀態

**1. 驗證防護是否生效 (R1 變安靜了沒？)**
請在 R1 特權模式下執行：

```text
show ip protocols

```

* **看什麼：** 在報表中尋找 `Passive Interface(s):` 欄位。只要看到 `Loopback0` 出現在下方，就代表該網孔的 OSPF 廣播已被成功靜音。

**2. 驗證地圖沒有縮水 (R2、R3 還看得到嗎？)**
請移動到 **R2** 或 **R3** 的特權模式，檢查它們的地圖：

```text
show ip route

```

* **看什麼：** 你依然能在路由表中看到 `O 1.1.1.1/32` 這條路線。證明 R1 不交朋友，但依然乖乖把該網段分享給了合法的 Router 鄰居！



{%hackmd kgfBRUWCRESEaXHxMS1yfA %}