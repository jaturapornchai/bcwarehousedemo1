# Prompt สร้างใหม่: BC WareHouse 3D Warehouse Simulation

ไฟล์นี้คือ prompt สำเร็จรูปให้ AI เขียนโค้ด (Claude Code / Codex) สร้างเดโม BC WareHouse ขึ้นใหม่ทั้งหมดจากศูนย์
**คัดลอกทุกอย่างใต้เส้นคั่นไปวางให้ AI**
เนื้อหาทุกตัวเลข/ชื่อ/คีย์ ถอดมาจาก `index.html` (ตัวจริง) โดยตรง ไม่ได้เดา

---

# บทบาทและเป้าหมาย

คุณคือวิศวกร front-end + 3D อาวุโส จงสร้างเว็บเดโม "BC WareHouse" แบบจำลองคลังวัสดุก่อสร้าง 3 มิติ ที่ใช้ฟิสิกส์ rigid-body จริง ให้ได้ผลเหมือนสเปกด้านล่างทุกจุด เมื่อเสร็จต้องเปิดดูแล้วรถยก AI ทำงานเองได้ต่อเนื่องโดยไม่ต้องมีคนแตะ

## ข้อบังคับเด็ดขาด
1. ไฟล์เดียว `index.html` (HTML + CSS + `<script type="module">` ในไฟล์เดียว) ห้ามมีไฟล์รูป/โมเดล/เสียงแยก ทุกพื้นผิวสร้างด้วย canvas 2D แบบ procedural (`THREE.CanvasTexture`)
2. ไลบรารีผ่าน CDN เท่านั้น เวอร์ชันตายตัว:
```html
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&family=Noto+Sans+Thai:wght@400;500;600;700&display=swap" rel="stylesheet" />
<script src="https://cdn.tailwindcss.com"></script>   <!-- Tailwind Play CDN; tailwind.config fontFamily.sans = Inter, Noto Sans Thai, system-ui, sans-serif -->
<script type="importmap">{ "imports": {
  "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
  "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/",
  "cannon-es": "https://unpkg.com/cannon-es@0.20.0/dist/cannon-es.js" } }</script>
```
   ใช้ addons: `OrbitControls`, `RoundedBoxGeometry`, `RoomEnvironment`, `mergeGeometries` (BufferGeometryUtils)
3. UI ภาษาไทยที่อ่านง่าย (ศัพท์เทคนิค/รหัสเป็นอังกฤษได้) `<html lang="th">` title `BC WareHouse — 3D Construction Material Warehouse Simulation`
4. รันจาก static server: `python -m http.server 8765 --bind 127.0.0.1` แล้วเปิด `http://127.0.0.1:8765/index.html` (ห้ามเปิดแบบ file://)
5. ข้อมูลทั้งหมดเป็นการจำลองในเบราว์เซอร์ ห้ามอ้างหรือทำท่าว่าเชื่อม ERP/WMS จริง (ข้อความ "เชื่อมต่อ WMS แล้ว" ใน log เป็นแค่ฉาก)
6. ห้ามเรียกระบบว่า Digital Twin / ดิจิทัลทวิน เพราะไม่มีข้อมูลจากคลังจริงเชื่อมอัตโนมัติ (ตามนิยาม ISO 23247-1 และ Kritzinger 2018 ระบบนี้คือ Digital Model) ให้เรียก "แบบจำลองคลังสินค้า 3 มิติ (3D Simulation)"
7. ไม่มี secret/อีเมล/IP จริงในโค้ด ไม่มี backend ไม่มี localStorage

## หน่วยและแกน
เมตร, +X ตะวันออก, +Z ใต้, Y ขึ้น. ทิศหัวรถ: yaw หมุน +X รอบ +Y → forward = (cos ψ, 0, −sin ψ); `yawOfDir(dx,dz)=atan2(−dz,dx)`. `DT = 1/60`.

---

# สเปกแยกตามส่วน

## 1. อาคารและเลย์เอาต์
- ตัวอาคาร `W = {X0:-26, X1:26, Z0:-17, Z1:17}`. ผนังหลัง z=-17 สูง 6.2 m (หน้าต่าง clerestory กว้าง 3.6 สูง 0.9 ทุก 4.4 m ที่ y=5.05, x −22..22) + แผ่นป้าย "BC WareHouse" / "North Hub · WH-01 · คลังวัสดุก่อสร้าง" ที่ x=14, y=3.2
- ผนังตะวันตก x=-26 สูง 4.0 (cutaway) มีช่องท่า 2 ช่อง ครึ่งกว้าง 1.7 สูงประตู 3.6; ด้านใต้และตะวันออกเปิด มีขอบเตี้ย 0.28 m
- ลานนอกตะวันตกต่ำกว่าพื้นคลัง 1.25 m (ลาดยางกว้าง 46×60 ที่ x=-49.5) เพื่อให้รถบรรทุกขนถ่ายที่ระดับท่า; พื้นนอกเป็นวงกลมรัศมี 150 y=-0.03
- `L = { rackX0:-8, bay:3.0, bays:4, depth:1.1, levels:[0,1.75,3.5], rackH:5.0, aisleA:-12.6, aisleB:-4.2, mainZ:2.6, westX:-11.2, eastX:7.2, dockZ:7.5, dock2Z:-8, conveyor:{x0:12,x1:20.1,z:-12.5,h:0.85}, dropPad:{x:16.5,z:9.2} }`
- **Dock 01 ขาออก** (z=7.5, ไฟเขียว/แดงตามประตู, ป้าย `01` / `DOCK · ขาออก`) มีประตูม้วน (kinematic body, เปิดตอนเริ่ม) + dock leveler แผ่นเหล็กลาย diamond ที่ x=-26.3. **Dock 02 ขาเข้า** (z=-8, ป้าย `02` / `DOCK · ขาเข้า`, ไฟแดง, ประตูปิดตลอด)
- แสง: Hemisphere(0xffffff,0xaab6c8,0.85), Directional "sun" 0xfff4e5 ความเข้ม 2.7 ที่ (24,46,20) เงา PCFSoft 2048², shadow camera ±38/±32, near 5 far 120, bias −0.0003, normalBias 0.035; fill 0xdce8ff 0.55 ที่ (−30,24,34); RoomEnvironment เป็น environment; ACESFilmic exposure 1.02; fog (0xe2e8f0, 110, 220); pixelRatio ≤ 1.75
- Bollard (static): มุมหลังของ rack ทั้ง 4 แถวฝั่งซ้าย-ขวา, 4 ตัวที่ปากท่า (x=-25.35, z=dockZ±2.15 และ dock2Z±2.15), และ (conveyor.x0−1.6, z+1.2), (conveyor.x1+1.9, z+1.2), (9.6,−9.2), (9.6,−6.8); ทรงกระบอกลายเหลือง-ดำ สูง 1.1

## 2. พื้นและโซน (วาดบน canvas 64 px/m เดียวทั้งพื้น)
- พื้น epoxy เทาอ่อน gradient + ก้อนเปื้อน + noise + เส้นกริดทุก 6 m
- โซนกรอบเหลือง: `INBOUND / รับสินค้าเข้า` (−24.4,−15.9)→(−13.6,−0.9); `PICKING AREA / พื้นที่หยิบสินค้า` (−9.4,5.3)→(3.8,11.8); `OUTBOUND STAGING / พักสินค้าขาออก` (8.8,−15.9)→(23.8,0.3); `LONG GOODS / เหล็กเส้น / ท่อ PVC` (−24.4,9.9)→(−13.6,15.9)
- เลนรถ (เส้นขาวประ + ลูกศรทิศทาง): main z=2.6 (x −25.6..24, กว้าง 4), west x=−11.2 และ east x=7.2 (z −15.6..0.6), ท่าเทียบ z=7.5 (x −25.6..−9.2), และ x=−11.2 ช่วง z 4.6..5.5
- เส้นกลางทางเดิน AISLE A (z=−12.6) / AISLE B (z=−4.2) เหลืองประ + ข้อความ
- ทางม้าลายข้ามเลน main ที่ x=7.8 (กว้าง 1.2) + ป้าย `WALK`; ลายเหลือง-ดำ (hazard hatch) ที่ปากท่า 2 ช่อง; แท่น `DROP TEST / ทดสอบตกกระแทก` 3×3 m ที่ (16.5, 9.2) ขอบ hatch
- ช่องวางพื้น (วาดเส้นประขาว + ป้ายรหัส): ดู §3. ช่องจอดรถ `P-01..P-04` (กรอบน้ำเงิน, ข้อความ `FORKLIFT PARKING`) และช่องชาร์จ 2 ช่อง (กรอบเขียว `⚡ CHARGING`)

## 3. Rack และ Slot registry
- Rack 4 แถว: `A1 {front:-15.6, dir:-1}`, `A2 {front:-9.6, dir:1}`, `B1 {front:-7.2, dir:-1}`, `B2 {front:-1.2, dir:1}`; x จาก −8 ยาว 4 bay × 3.0 m = −8..4; ลึก 1.1; เสาน้ำเงิน (upright 0.09×5.0 m), คานส้ม ที่ระดับ 1.75 และ 3.5 (สองคานหน้า-หลัง), ค้ำทแยง, ฟุตเหล็ก, การ์ดเหลืองปลายแถว, ป้ายปลายแถว `A1`.. + แถบรหัสตำแหน่งที่หน้าคาน (`A2-03  ▮▮▯▮`)
- **Slot ID**: `{rack}-{NN}` ; `NN = pad2(level*8 + bay*2 + p + 1)` (level 0..2, bay 0..3, p 0..1) → 24 ช่อง/แถว = 96 ช่อง; ตำแหน่ง x = กลาง bay ± 0.62, z = front + dir*0.55, ความสูงวาง h = levels[level]; `aisleZ` = aisleA (แถว A) / aisleB (แถว B)
- ช่องพื้น: `IN-01..05` (x=−16.6, z = −14.2,−11.4,−8.6,−5.8,−3.0, yaw π); `ST-01..06` (z=−2.8, x = 10,12.2,14.4,16.6,18.8,21.0, yaw π/2); `PK-01..04` (z=7.2, x = −7,−4.2,−1.4,1.4, yaw −π/2); `TR-01..05` (บนกระบะรถ, z=7.5, x = −32.5,−31.25,−30,−28.75,−27.5, yaw π)
- ช่องจอด `P-01..P-04` x = 8.6,10.4,12.2,14.0 ที่ z=9.8 yaw −π/2 (ไม่อยู่ใน slot registry); ที่ชาร์จ 2 จุด (23.2, 5.5) และ (23.2, 8.0) พร้อมตู้ชาร์จ/สายที่ x≈25.7
- slot object: `{id, kind:'rack'|'inbound'|'staging'|'picking'|'truck', x, z, yaw, h, reservedBy, cargo}`

## 4. สินค้า (CARGO_TYPES)
พาเลท + ของ = **rigid body ก้อนเดียว** (compound collider; CoM จริง; pallet 1.2×1.0 m มวล 25 kg ไม้ลายกระดาน หรือพลาสติกน้ำเงิน ~22%)
| type | label | mass (kg, ของล้วน) | half-extents load |
|---|---|---|---|
| cement | ปูนซีเมนต์ 50 กก. × 24 | 1200 | .59,.39,.44 (ถุง 6 ชั้น × 4) |
| aac | อิฐมวลเบา AAC | 850 | .575,.475,.475 |
| brick | อิฐแดง (มอญ) | 850 | .55,.36,.46 |
| mixed | Mixed SKU · สี/ฮาร์ดแวร์ | 420 | .58,.46,.48 (กล่อง + ถังสี 18 L พร้อมฝาสุ่มสี) |
| empty | พาเลทเปล่า | 0 | — |
- มวลรวม = 25 + มวลของ; ID พาเลท `PL-4000+n` (นับเพิ่มทีละ 1); runner ของพาเลทเป็น shape group `RUNNER` เพื่อให้ปลายงาผ่านระหว่างซี่ได้
- Long goods: `makeBundle('rebar')` เหล็กเส้นข้ออ้อย DB28 · 6 m มวล 1000+30, `makeBundle('pvc')` ท่อ PVC 4" · 4 m มวล 192+30 บน skid ไม้ (ไม่เก็บเข้า slot; วางที่ (−19,11.3), (−19,12.6) rebar และ (−17.3,14.3), (−21.9,14.3) pvc)
- ของเล็กบนสายพาน: กล่อง "Hardware box" 15–35 kg, ถังสี "Paint bucket 18 L" 17–21 kg (38% ถัง)
- Collision groups (bit): STATIC 1, CHASSIS 2, FORK 4, CARGO 8, WORKER 16, ROLLER 32, RUNNER 64
- ค่า contact: world friction .4, restitution .05, stiffness 1e8, relaxation 3; materials floor/steel/wood/cement/fork/roller/carton/plastic/rubber/masonry พร้อมตารางคู่ (เช่น fork–cement 0.8, roller–carton 0.75, cement–wood 0.75)
- sleep: speedLimit 0.12, timeLimit 0.7; solver iterations 12, tolerance 1e-4, SAPBroadphase, gravity −9.81

## 5. สายพานลำเลียง
- roller conveyor x 12→20.1, z=−12.5, สูง 0.85; roller ทรงกระบอก r=0.05 ทุก 0.2 m เป็น kinematic body ให้ความเร็วผิวผ่าน `angularVelocity.z = −speed/0.05` (collider ไม่หมุนจริง; ตั้ง quaternion identity ทุก postStep); รางข้าง 2 ข้าง; เกตหยุดปลายสาย (kinematic, ยกขึ้นเมื่อเต็ม)
- ความเร็วเริ่ม 0.60 m/s, สไลเดอร์ 0–1.5 ก้าว 0.05
- spawn ของที่ x0+0.35 ทุก ~2.2 s (เร็วขึ้นตามสไลเดอร์ `spawnT -= dt*clamp(speed/0.6,0.15,3)`), เหลือไม่เกิน 26 ชิ้น, ต้องไม่มีของค้างจุดปล่อย
- ปลายสายมีพาเลทพลาสติกรับ `CV-OUT`; เมื่อมีของบนพาเลท ≥10 ชิ้น → เกตขึ้น + "พันฟิล์ม" 5 s → ลบของ, `conv.packed++`, `KPI.ready++`, log `แพ็กเสร็จ CV-OUT #n → Ready Pallets +1`; ของตกพื้นเกิน 6 s ถูกลบ
- โต๊ะแพ็ก + จอ "BC WareHouse · Pack Station 01" ที่ (x0−1.2, z+1.25) และพนักงาน packer

## 6. รถยก 6 คัน (Forklift)
- `FL-01..FL-06` (`FL-{pad2(idx+1)}`) : **4 คันทำงาน (FL-01..04)** + **2 คันชาร์จ (FL-05, FL-06)** จอดที่แท่นชาร์จ สถานะ `Charging` ไม่มีคนขับ ไม่รับงาน แบตเพิ่ม 0.08%/s. คันทำงานแบตเริ่ม 64–97%, ลดตามระยะขับ; จุดเกิด FL-01 (−9.0,−12.6,yaw 0), FL-02 (13.0,3.6,0), FL-03 (−3.0,1.6,π), FL-04 (−10.2,−6.0,π/2); home = ช่องจอด P-01..P-04 ตามลำดับ
- Chassis: **`CANNON.RaycastVehicle`** (right=Z, forward=X, up=Y), มวล 3000 kg, CoM (−1.25, 0.62, 0) (ถ่วงหลัง), ล้อ 4: หน้า r=0.36 (track ±0.5) ขับ, หลัง r=0.29 (track ±0.46, wheelbase 1.9) **เลี้ยวล้อหลัง** (`STEER_SIGN=-1`); suspension stiffness 55, rest 0.18, travel 0.12, damping relax 2.6 / compress 4.6, frictionSlip 2.2, rollInfluence 0.04. maxSteer 1.0 rad, slew 2.2 rad/s. engine force ที่ล้อหน้า 2 ล้อ, brake ทั้ง 4
- collider chassis 3 กล่อง (เฟรม+ฝากระโปรง, counterweight, overhead guard)
- **Mast/forks**: carriage แยกเป็น body ที่ 2 (มวล 500, tine 2 ซี่ ยาว 1.1 ห่าง ±0.225 + backrest) ผูกด้วย `LockConstraint` (maxForce 2.5e5) ที่ปรับ pivot/แกนทุก step เพื่อ lift และ tilt. mast pivot (0.42, 0.45), heel x=0.56, `forkY0=0.02`, **maxLift 3.95 m**; free-lift ถึง 1.37 m แล้วเสาท่อนกลาง/ท่อนใน telescopic ยืด (mid = ext*0.5, inner = ext)
- ความเร็วไฮดรอลิก: ยก ≤ (0.5 − 0.2·lm) m/s, ลง ≤ 0.55 m/s (lm = loadMass/2500 clamp 0–1); tilt ±0.075 rad/s; tilt ช่วง −3°..+5° (บวก = เอียงหลัง; `TILT_BACK=4°`)
- ขับ: `drive(vT, steer)` controller ความเร็วเป้าหมาย: accel = 1.15−0.45·lm, decel = 1.7−0.75·lm (m/s²), gain เบรก 3.4 / เร่ง 1.9; หยุดนิ่ง brake 70
- จับของ: ใน AI ใช้ `LockConstraint` carriage↔cargo (maxForce 3e5) ตอน pick; โหมดขับเอง **ไม่ล็อก** ใช้แรงเสียดทานล้วน
- ภาพ: ตัวรถ RoundedBox ขาว/น้ำเงิน, เสาโลหะ, คนขับ (เสื้อกั๊กเหลือง หมวกขาว), ไฟเตือนส้มกะพริบเมื่อเคลื่อนที่/มีงาน, ป้ายชื่อ `FL-0n` บนตัวรถ, วงสว่างน้ำเงินหน้ารถบนพื้น, ป้ายลอยหน้าจอ (`.fl-label`) แสดงจุดสีสถานะ + ชื่อ + รหัสงาน (หรือสถานะ) 
- Telemetry ต่อเฟรม: speed, pitch, stab = ((xF − xEff)/(xF − xR))/0.52 clamp 0–1 (CoM รวม chassis+carriage+โหลด, xEff ขยับตามความเร่ง), tipped เมื่อ `upY < 0.82` หรือ `|pitch| > 0.3` (log สีแดง)
- Sensor: ยิง ray 3 เส้นขนาน (z 0 / ±0.56) ข้างหน้า/หลัง ระยะ `1.3 + v²/2.6 + |v|·0.55` mask CHASSIS|FORK|WORKER|CARGO; ข้ามของที่ถืออยู่/ของใน rack keep-out

## 7. การนำทาง (Navigation)
- Waypoint graph: เลน main 12 node `M0..M11` ที่ z=2.6 x = −11.2,−7,−4.2,−1.4,1.4,7.2,10,13,16,19,22,24.8; `MW` (−20.5, 2.6)→M0; `WN`(−11.2,−15.2), `WA`(−11.2,−12.6), `WB`(−11.2,−4.2), `WD`(−11.2,7.5), `DX`(−18.5,7.5), `DK`(−24,7.5); `EA`(7.2,−12.6), `EB`(7.2,−4.2). edges: M ต่อกันเป็นสาย, WN-WA-WB-M0-WD-DX-DK(lane 0), EA-EB-M5, rack aisle `WA-EA` และ `WB-EB` lane 0 น้ำหนัก 1.8 (เลนเดียว)
- Path search: A* (Dijkstra + heuristic ระยะตรง) จากจุดใกล้สุดบน edge ถึงเป้า; ปรับโทษ U-turn ที่ edge แรก +30; ห้ามย้อนกลับกลางทาง
- **Keep-right lanes**: ทุกเส้นถูก offset ไปทางขวา 1.0 m (lane=1) โดย mitre ที่จุดต่อ; ทางโค้ง cap ความเร็ว 1.15 (เลี้ยว >1.0 rad) / 1.9 (>0.4 rad)
- ติดตามเส้นทาง: pure pursuit ที่ล้อหน้า (look-ahead 0.85–2.6 m), หยุดสั้นกว่าเป้าตาม lag, cap ความเร็วตามความโค้ง `sqrt(0.8/κ)` (ถือของ 0.5)
- การเข้า slot: lane → โค้ง 90° รัศมี `ARC_R=1.65` → ตรงเข้า (rack: ตรงเข้าความเร็ว ≤0.42 "creep", ต้องมี lead-in ตรง ≥3.2 m ไม่งั้นถอยหรือเข้าทางปลายอีกด้าน); ออก: ถอยตรงแล้วถอยโค้ง 90° (ลองทั้ง 2 ทิศ × หลายระยะถอย ตรวจชน rack/ผนัง/bollard/รถอื่น เลือกคะแนนดีสุด); ออกจากช่อง `TR-*` ถอยออกลานแล้วถอยโค้ง 180°
- Mutex zones ป้องกันทางแคบ: `A` Aisle A (x −9.4..5.4, z −15.9..−9.3), `B` Aisle B (z −7.5..−0.9), `DOCK` (x −36..−17, z 5..10). ขอสิทธิ์ล่วงหน้า ≤7 m, ถ้าเจ้าของคนอื่นให้หยุดรอก่อนเข้า 4.5 m, ปล่อยเมื่อตัวรถพ้น

## 8. WMS Dispatcher
- ออเดอร์ `SO-{45821+}` (`total 5`, loaded, picked, skus 12–31, units 60–149, ทีม Team Blue/Orange/Green × "3 staff", ลูกค้าสุ่มจาก 5 ราย เช่น Siam BuildPro Co., Ltd.). เริ่มต้น `SO-45821` loaded 3 (บนรถ TR-01..03 = cement,aac,brick) + PICK-7843 จาก **A2-03** (cement) ไป ST-02 ให้ FL-01; `SO-45822` มีพาเลท 3 ใบรอที่ ST-01 (brick), ST-04 (cement), ST-06 (mixed)
- งาน 3 ชนิด ID `PICK-7843+`, `LOAD-3108+`, `PUT-5214+` (นับเพิ่มต่อชนิด) พร้อม phase: `Queued → En route → Lifting → Transporting → Placing → Done`
  - **PICK**: rack → ช่องพัก `ST-xx`; แหล่งสุ่มจากช่องที่มีของ (ไม่ใช่ empty, ไม่ถูกจอง) ถ่วง 75% ไปชั้นล่าง
  - **LOAD**: `ST-xx` → `TR-xx` (`TR-{pad2(loaded+1)}`) เฉพาะเมื่อรถบรรทุก docked
  - **PUT**: `IN-xx` → rack ว่าง (ปูนเก็บเฉพาะชั้น 0–1)
- สร้าง PICK ครั้งละให้มี pending ≤3 ใบ; ออเดอร์ใหม่เมื่อออเดอร์ท้ายสุด pick ครบและมีไม่เกิน 3 ออเดอร์
- **ลำดับความสำคัญ**: `LOAD > PICK ของออเดอร์ปัจจุบัน > PUT` (แล้วจึง PICK ออเดอร์อื่น), เท่ากันดู n (ลำดับสร้าง)
- มอบหมายให้ **รถว่างที่ใกล้ต้นทางที่สุด**; ปลายทาง PICK = **ช่อง ST ว่างที่ใกล้ต้นทางที่สุด** (สำรองช่องไว้ให้ PICK ของออเดอร์ปัจจุบันที่ยังค้าง); **`maxPicks = 2`** PICK active พร้อมกัน, ปุ่ม **Expedite** ตั้งเป็น **3** และ log `เร่งด่วน SO-… · เพิ่มรถหยิบสินค้าพร้อมกันเป็น 3 คัน`
- `dispatchTick` ทุก 0.5 s; งานที่พาเลทถูกย้าย → `failTask` + คืนการจอง; log ทุกเหตุการณ์สำคัญ
- Inbound: ทุก rand(24–36) s วางพาเลทใหม่ (cement ×2, aac, brick, mixed) ที่ช่อง IN ว่างที่ไม่มีรถอยู่ใกล้ (<3.4 m) ถ้า IN มีของ <3, rack ว่าง ≥5, pool ยังเหลือ; `KPI.pending++` ทุก rand(22–40) s

## 9. Autopilot (state machine ต่อรถ)
`step`: `null` (ว่าง) → `toSrc` → `pick` → `exit` → `toDst` → `place` → `settle` → `null`; อื่นๆ: `toPark`, `backoff`, `yield`
- ความเร็ว AI: ว่าง 3.2, ถือของ 2.4, ในทางเดิน rack 1.6 m/s; ถือของยกสูง >0.55 m → 0.55; ยกสูง >1.2 → 0.32; ออกจาก slot 1.0; หลีก/ถอย 0.8
- **ทางม้าลาย** (x 7.8 ± 2.4, z 2.6 ± 2.4): จำกัด **≤1.5 m/s**
- **Ray braking**: `v_ok = sqrt(2·1.0·max(0, dist − 0.8))` (คน 1.5 m); ถ้า <0.25 ถือว่า "blocked"
- **Yield หลัง 2.5 s**: blocked ต่อเนื่อง >2.5 s กับรถอีกคัน → `tryYield` (ถอยตรง ≤1.6 m ตาม heading), ยกเลิก yield เมื่อ blocked >4 s หรือ yield >6 s แล้วรอ 2 s ก่อนทำงานต่อ; ปรับ: คนที่ว่าง (ไม่ถือของ) ยอมก่อน, เท่ากันให้ idx สูงกว่าถอย
- มี swing guard / exit-sweep guard / `exitConflict` (ถอยออกจาก slot ติดกันทีละคัน) / `breakCycle` (วงรอ ≥3 คัน)
- **Recovery**: ติดไม่ขยับ >25 s หรือรอทาง >45 s หรือพลิก >6 s → `recoverForklift`: ยกเลิกงาน, คืน mutex, reset ท่ากลับ home, log `กู้คืนอัตโนมัติ (…) → กลับจุดจอด P-0n`
- ว่างเกิน 3.5 s ไม่จอด → `toPark` (สถานะ `Returning`) ไปจอดที่ช่อง P ของตัวเอง; ปรับยกงา: TRAVEL 0.14, CARRY 0.24, ระดับ pick = h+0.02 → ยกขึ้น h+0.18 + เอียงหลัง 4°, place = ลดเอียง → h+0.05 → ปล่อย

## 10. รายการสถานะ (STATUS)
`Idle` ว่าง · `Picking` หยิบสินค้า · `Loading` โหลดขึ้นรถ · `Put-away` จัดเก็บ · `Returning` กลับจุดจอด · `Charging` ชาร์จไฟ · `Manual` ขับเอง · `Tipped` พลิกคว่ำ (แสดงแทนเมื่อ tipped). สีจุด: #94a3b8, #2563eb, #4f46e5, #f59e0b, #64748b, #10b981, #f97316, #e11d48 ตามลำดับ

## 11. รถบรรทุก Dock 01
- รถ 6 ล้อกล่อง ป้ายทะเบียน **`TRK-6W 70-1234`** (กระบะ 6.8×2.5×2.45, ข้างลาย "BC WareHouse Logistics ขนส่งวัสดุก่อสร้าง · TRK-6W 70-1234"), kinematic body, จอดที่ x=−30.0, วางได้ **5 พาเลท** (`TR-01..05`, ออเดอร์ละ 5)
- วงจร: `docked` → เมื่อ `loaded ≥ total` และ DOCK mutex ว่าง ครบ 2.5 s → `departing` (เร่ง −0.9 m/s² สูงสุด 7 m/s, ปิดประตูเมื่อ x<−34.5, `KPI.pending−1`, log `รถ … ออกเดินทาง · SO-… ครบ 5 พาเลท`) → ที่ x<−62 ลบพาเลทบนรถ, shift ออเดอร์ → `away` 6 s → `arriving` ป้ายทะเบียนสุ่ม `TRK-6W {70–78}-{1000–9998}` ถอยเข้า (เร่ง 1.2, สูงสุด 6 m/s, เปิดประตูเมื่อ x>−50) → `docked` (log `เทียบท่าแล้ว · พร้อมโหลด …`)
- ล้อหมุนตามความเร็ว; พาเลทบนกระบะเคลื่อนไปกับรถด้วยแรงเสียดทาน

## 12. พนักงาน (4 คน, capsule rig + kinematic sensor body group WORKER mask CHASSIS)
`packer` (10.8,−11.9 ที่โต๊ะแพ็ก ยื่นมือเมื่อมีของเข้า), `inspector` QC (เดินตรวจ ST ที่ z=−4.35 x=10..21 ถือแท็บเล็ต หยุด 2.2–3.6 s), `walker` (เดิน (9.6,−0.9)→(7.8,0.2)→(7.8,5.0)→(7.2,9.6) ข้ามทางม้าลาย รอจนถนนว่าง ≤14 s แล้ว log `พนักงานข้ามทางม้าลาย — รถชะลอ/หยุดให้`), `picker` (x −7..1.4 ที่ z=8.65 ก้มหยิบ หยุด 2.8–4.2 s). Walk cycle ด้วยแกว่งขา/แขน

## 13. KPI และ Event log
- `KPI = {pending:128, pendingDelta:12, ready:86, readyOf:120, total:6}` — Pending Orders 128 (+12%) ; Ready Pallets 86 (+24%, "out of 120 today"); Active Forklifts = จำนวนที่ active และไม่ Charging (เริ่ม 4 "of 6 available") พร้อมแถบ 6 ช่อง: ชาร์จ `bg-emerald-300`, มีงาน/ขับเอง `bg-blue-600`, ว่าง `bg-blue-200`; Preparation Rate = ready/readyOf วงแหวน SVG (r=25, dasharray 157.08, offset = 157.08(1−rate)), target 80%
- `ready++` เมื่อ PICK เสร็จหรือแพ็ก CV-OUT เสร็จ; `pending` +1 ทุก 22–40 s, −1 เมื่อรถออก
- สีแถบแบต: >50% emerald-500, >25% amber-500, ไม่งั้น rose-500. Stability (HUD): >0.6 emerald-400, >0.3 amber-400, ไม่งั้น rose-500
- Event log: ใหม่อยู่บนสุด, **เก็บ 40 รายการ**, แต่ละรายการ: จุดสี (blue/slate/amber/green/red) · ชื่อต้นทาง · เวลา HH:MM · ข้อความ; escape HTML ทุกข้อความ
- นาฬิกา "Live" ใช้เวลาเครื่อง HH:MM; ETA = now + ((total−loaded)·7 + (total−prepared)·6 + 15, ขั้นต่ำ 2) นาที

## 14. กล้อง
PerspectiveCamera fov 30, near 0.5 far 420, เริ่ม (36,44,50) มอง (−1.5,0,−1). `OrbitControls`: damping 0.07, `minPolarAngle 0.28`, **`maxPolarAngle 1.25`**, **`minDistance 10`**, **`maxDistance 140`**, `screenSpacePanning=false`. `flyTo(x,y,z,dist=24)` นาน 1.1 s ease-in-out cubic (รักษาทิศมอง); `flyHome()` 1.2 s; โหมดตาม: เมื่อขับเองและเปิด "กล้องติดตาม" ให้ target ตามรถ (smoothing `1−exp(−4dt)`)

## 15. ขับเอง (Manual) และ HUD
- สลับ `Autonomous AI Fleet` / `Manual Drive`; เลือกรถจาก dropdown "ขับรถคัน" (เฉพาะ 4 คันที่ active) + สวิตช์ "กล้องติดตาม" (เริ่มเปิด); เลือกแล้ว AI ปล่อยงาน (`aiRelease` ยกเลิกงาน ถือของด้วยแรงเสียดทาน) และสลับกลับ AI → `aiResume` (ลดงา, ถอยถ้ายังมีของบนงา)
- คีย์ (ใช้ `e.code`): **W/↑ เดินหน้า, S/↓ ถอย, A/← เลี้ยวซ้าย, D/→ เลี้ยวขวา, Q ยกงาขึ้น, E ลดงาลง, Z เอียงเสาไปหลัง, C เอียงเสาไปหน้า, Space เบรก**. ความเร็ว: เดินหน้า 3.4 (มีของบนงา 2.2), ถอย −1.8; steer 0.9 slew 1.8; Q/E ปรับเป้า ±0.3 m ต่อเฟรมที่กด, Z/C ±0.05 rad; ไม่รับคีย์เมื่อโฟกัสที่ input/select/textarea; `preventDefault` ปุ่มขับตอนอยู่โหมดขับเอง; blur ล้างคีย์
- HUD (glass-dark ล่างกลาง กว้าง 560): ชื่อรถ, KM/H, FORK m, MAST TILT °, LOAD kg, PITCH °, แถบ `Stability (CoM)` % และป้าย `BRAKE` เมื่อกด Space/เบรกแรง
- Slider "Forklift Payload Mass" ปรับมวลพาเลทบนงาของรถที่ขับอยู่แบบสด (`setCargoMass`)

## 16. Drop Test, Reset, Collider debug
- **Drop Test · ปูน**: ปล่อยพาเลทปูน (มวลจาก slider) จากสูง **3.2–3.8 m** (3.2 + rand 0–0.6) ที่แท่น (16.5,9.2) ± สุ่ม พร้อมหมุนเล็กน้อย; id `DROP-01..`; เก็บได้ **สูงสุด 6** (ใบเก่าสุดถูกลบแบบ fade); ถ้าพูลเต็ม log เตือน
- **Reset Scene**: ลบ drop ทั้งหมด, ลบพาเลทที่หลุด/นอน, กู้รถที่พลิก/ติด, `flyHome()`, log `รีเซ็ตฉาก · ล้างพาเลททดสอบ…`
- **Collider Debug View**: toggle วาด LineSegments (≤90000 จุด) ของ collider ทุก body (static #94a3b8, kinematic #a855f7, รถ #f97316, หลับ #64748b, ตื่น #22c55e), suspension rays #22d3ee, contact normals #ef4444; ซ่อน body ที่ x<−70

## 17. ค้นหา
ช่อง search กลางแถบบน placeholder `Search pallet, SKU, order… (ค้นหา เช่น PICK-7843, A2-03)`; กด Enter ค้นหา: รถ (`FL-02` หรือ `2`), งาน (`PICK-7843`), ออเดอร์ (`SO-45821` หรือ `45821`), ตำแหน่ง (`A2-03`), พาเลท (`PL-4001`) → `flyTo` 18 m + log `พบ …`; ไม่พบ → log `ไม่พบ "…" — ลองรหัสงาน (PICK-7844), ตำแหน่ง (A2-03), รถ (FL-02) หรือออเดอร์ (SO-45821)`

## 18. UI ทั้งหมด (Tailwind + glass)
สไตล์ `.glass` = rgba(255,255,255,.74) blur 18px saturate 170%; พื้นหลัง radial เทา-ฟ้า; ปุ่ม `.btn` ลอยเมื่อ hover; toggle/range แบบ custom สีน้ำเงิน #2563eb; ฟอนต์ตัวเลข tabular
- **Loader** เต็มจอ โลโก้ shimmer + `BC WareHouse 3D Simulation` + แถบความคืบหน้า ข้อความไทยตามขั้น: `กำลังโหลดฟอนต์และพื้นผิว…`(8) → `กำลังสร้างอาคารคลังสินค้าและท่าเทียบรถ…`(28) → `กำลังสร้างสินค้าวัสดุก่อสร้างและสายพาน…`(48) → `กำลังวางแผนเส้นทาง AI และจัดรถโฟล์คลิฟท์…`(70) → `กำลังอุ่นเครื่องฟิสิกส์…`(92) → `พร้อมใช้งาน`(100); ถ้า error แสดง `เกิดข้อผิดพลาด: …`
- **Top bar** (left/right 16, top 12, สูง 56): โลโก้ + `BC WareHouse` / `WMS · 3D SIMULATION`; ช่องค้นหา (ป้ายคีย์ลัดมุมขวา **`/`**); ปุ่ม `Main Warehouse (North Hub - WH-01)`; จุดเขียวกะพริบ `Live HH:MM`; `NN FPS · NN Hz physics` (แสดงเมื่อจอ ≥ xl); กระดิ่งมีจุดส้ม; อวาตาร์ `DK` `Daniel Kim` / `Warehouse Manager`
- **KPI cards** (left 16, top 76, 4 ใบ กว้าง 212): `Pending Orders` (+12%, "คำสั่งซื้อรอดำเนินการ · vs yesterday"), `Ready Pallets` (+24%, "พาเลทพร้อมส่ง · out of 120 today"), `Active Forklifts` (แถบ 6 ช่อง), `Preparation Rate` (วงแหวน, "อัตราจัดเตรียม · target 80%")
- **Physics Control · ควบคุมฟิสิกส์** (left 16, top 196, กว้าง 268, พับได้): segmented `Autonomous AI Fleet | Manual Drive`; แถว "ขับรถคัน"; `Collider Debug View · กล่องชน/CoM/ray`; `Conveyor Speed` (0.60 m/s); `Forklift Payload Mass` (100–2500 ก้าว 25 เริ่ม 1225 แสดง `1,225 kg`, คำอธิบาย "มวลพาเลทที่ Drop Test และพาเลทบนงาของรถที่กำลังขับ"); ปุ่ม `Drop Test · ปูน` (ส้ม) / `Reset Scene`; มิเตอร์ 4 ช่อง `FPS / BODIES / CONTACTS / STEP ms` (STEP แสดง 1 ทศนิยม + " ms")
- **Sales Order inspector** (right 16, top 76, กว้าง 340): `SALES ORDER` / `Order #SO-45821` + ป้ายสถานะ `Preparing | Loading | Dispatched`; Customer, Destination `ไซต์งาน Bang Na · BKK-07`, Priority `High Priority`, ETA `HH:MM (≈ N min)` (หลังออก `ออกจากคลังแล้ว`); แถบ `Pallets prepared` `N / 5 pallets (P%)`; `Picking zone` (slot ต้นทาง PICK ถัดไป / `Dock 01` / `—`), `Items` `{skus} SKUs · {units} units`, ข้อความรายการสินค้า `ปูนซีเมนต์ 50 กก. · อิฐมวลเบา · อิฐแดง · ท่อ PVC · เหล็กข้ออ้อย · สีทาอาคาร`, `Team` (`Team Blue · 3 staff` + จุดสี); ปุ่ม `Locate` (บินไปพาเลทของงานโฟกัส หรือ home) / `Route` (สลับแสดงเส้นทางทุกคัน↔เฉพาะงานโฟกัส) / `Expedite`
- **Fleet Telemetry · สถานะรถยก** (right 16, top 498, ลงล่าง 16, กว้าง 340): `click = focus`; แถวรถ 6 แถว (เลขสีส้ม, ชื่อ, ป้ายสถานะไทย, แถบแบต + %, บรรทัดงาน `PICK-… · A2-03 → ST-04` หรือ `ควบคุมด้วยคีย์บอร์ด` / `จอดที่ P-0n` / `กำลังกลับจุดจอด` / `แท่นชาร์จ`) คลิก = `flyTo` รถ (และเลือกขับถ้าอยู่โหมดขับเอง); ใต้ลงมา `EVENT LOG · เหตุการณ์` เลื่อนได้ซ่อน scrollbar
- **Bottom panel** (left 16, right 372, bottom 16, สูง 164): `Outbound Preparation เตรียมส่งออก` timeline 3 ขั้น `1 Preparing — 2 Loading — 3 Dispatched` + แถบความคืบหน้า + `Truck TRK-6W 70-1234 @ Dock 01 · loaded N/5` (ขณะออก `… ออกจาก Dock 01 · SO-… ครบ 5 พาเลท ✓`, ขณะรอ `รถคันถัดไปกำลังเข้าเทียบ Dock 01…`); การ์ดน้ำเงิน `CURRENT TASK · งานปัจจุบัน` (phase, id ใหญ่, `from → to · FL-0n`, 4 ช่อง `Fork height` m / `Speed` km/h / `Load` kg / `Stability` %, แถบคืบหน้า); เมื่อไม่มีงาน `Idle` / `รอคำสั่งงาน`; `UPCOMING TASKS · งานถัดไป` แสดง 3 งาน pending (LOAD ก่อน) หรือ `ไม่มีงานรอคิว`, ป้าย `รอคิว`
- **3D callout**: กรอบมุม 4 มุมสีน้ำเงินรอบพาเลทของงานโฟกัส + ป้าย `PICK-7843 | A2-03` และข้อความรอง = ชื่อสินค้าตัดที่ " · " + มวลรวม เช่น `ปูนซีเมนต์ 50 กก. × 24 1,225 kg` (ซ่อนเมื่อขับรถคันนั้นเองหรืออยู่นอกจอ)
- เส้นทางบนพื้น: chevron `#2563eb` (โฟกัส) / `#60a5fa` (คันอื่น) / `#f59e0b` (ถอย) วิ่งตามทุก path สูงสุด 700 ตัว, ห่าง 0.62 m
- แถบ `Pallets prepared`/`outbound` ใช้ `setText/setHTML/setCls/setW` แบบแคชค่าเดิม ไม่เขียน DOM ซ้ำ; UI รีเฟรชทุก 0.25 s, ป้ายลอยทุกเฟรม, FPS วัดทุก 0.5 s

## 19. ฟิสิกส์ 60 Hz และ Instancing
- `world.step(DT, dt, 3)` (fixed 1/60, maxSubSteps 3, dt clamp 0.001–0.1); preStep: รถทุกคัน + conveyor; postStep: conveyor + `scanForkContacts`; ก่อนเฟรมแรกวอร์ม 30 step
- ภาพสินค้าทั้งหมดเป็น `THREE.InstancedMesh` ผ่านคลาส `Pool` (1 draw call ต่อชนิดชิ้นส่วน; ความจุสูงสุด: wDeck 170, wRun 510, pDeck 60, pRun 180, bag 1300, aac 90, brick 90, strap 400, carton 900, bucket 320, lid 320, rod 200, pipe 40, pipeIn 40, skid 30, tie 30) และคลาส `Visual` ผูก body → ชิ้นส่วน (interpolated position/quaternion, spawn grow-in scale 0.001→1 lerp 0.18, หลับแล้วไม่ sync); ลบพาเลทแบบหดหายใน 0.5 s; `poolRoom()` กันสร้างเกินความจุ
- STEP ms = เวลา `world.step` เฉลี่ย

---

# โครงสร้างโค้ดใน `<script type="module">` (26 หมวด เรียงตามนี้ ใช้ชื่อหัวข้อเดียวกัน)
```
0. UTILITIES & CONSTANTS            13. NAVIGATION — waypoint graph + Dijkstra/A*, right-hand lane offsets, pure-pursuit follower, aisle mutex zones
1. RENDERER / SCENE / CAMERA / LIGHTS    14. WMS DISPATCHER — orders, tasks (PICK / LOAD / PUT), slot reservations
2. PHYSICS WORLD (cannon-es, 60 Hz fixed step)   15. FORKLIFT AUTOPILOT — approach / pick / retreat / place state machine, sensor braking, zone mutex waits, deadlock yield, stuck & tip recovery
3. INSTANCED VISUAL SYSTEM          16. DOCK 01 TRUCK CYCLE — depart when the order is complete, return empty
4. PROCEDURAL CANVAS TEXTURES       17. WORKERS — capsule rigs + kinematic sensor bodies
5. LAYOUT                           18. INITIAL STATE — stock, staged/loaded order pallets, fleet, parking & chargers
6. FLOOR + BUILDING SHELL           19. KPI MODEL + EVENT LOG
7. SIX-WHEEL BOX TRUCK              20. ROUTE RIBBON + PHYSICS DEBUG
8. SAFETY BOLLARDS                  21. CAMERA — fly-to, follow (manual drive)
9. SELECTIVE PALLET RACKS (A1, A2, B1, B2) + SLOT REGISTRY   22. MANUAL DRIVE
10. CONSTRUCTION MATERIAL CARGO FACTORY   23. DROP TEST + RESET
11. ROLLER CONVEYOR                 24. UI BINDING — top bar KPIs, inspector, outbound timeline, task card, fleet, labels, callout
12. COUNTERBALANCE FORKLIFT — RaycastVehicle chassis + hydraulic fork carriage   25. MAIN LOOP + BOOT
```
ลำดับ boot: โหลดฟอนต์ (รอสูงสุด 2.5 s) → `buildTextures, buildMaterials` → `buildShell, buildTruck, buildBollards, buildRacks` → `buildCargoPools, buildConveyor, buildForkliftKit, paintSlots` → `buildRouteViz, buildDebug` → `populateWarehouse` (nav, keep-out, workers, chargers, stock, orders, fleet, งานแรก) → `buildUI` → warm-up 30 step → ซ่อน loader → `requestAnimationFrame`.
ชื่อ DOM id ที่ต้องมี (ใช้ตรวจ): `scene, loader, search, clock, fpsTop, physTop, kpiPending, kpiReady, kpiActive, kpiRate, physPanel, modeAI, modeManual, manualSel, tglFollow, tglDebug, sldConveyor, valConveyor, sldPayload, valPayload, btnDrop, btnReset, stFps, stBodies, stContacts, stStep, ordTitle, ordStatus, btnLocate, btnRoute, btnExpedite, fleetList, eventLog, ctId, ctPhase, upList, hud, hudName, hudStab, hudBrake, callout, bracket, flLabels`.

---

# ลำดับการสร้าง (Milestones) — ทำทีละข้อ ตรวจให้ผ่านก่อนไปต่อ

วิธีตรวจทุกข้อ: เริ่ม `python -m http.server 8765 --bind 127.0.0.1`, เปิด `http://127.0.0.1:8765/index.html` ด้วย Chrome หรือ Playwright, เปิด console. **ต้องไม่มี error/uncaught exception** และ Request ของ CDN (three, cannon-es, tailwind, fonts) ต้อง 200.

- **M1 โครง + เรนเดอร์**: importmap, loader, renderer, กล้อง/แสง, พื้น+ผนัง+ท่า, UI shell ทุกแผงเป็น static. ตรวจ: loader หายไป, เห็นอาคาร, `maxPolarAngle` ทำให้ลากกล้องต่ำกว่าพื้นไม่ได้, ซูมไม่ต่ำกว่า 10 / ไม่เกิน 140.
- **M2 ฟิสิกส์ + instancing**: world, materials, Pool/Visual, texture, rack 4 แถว + slot registry, Drop Test. ตรวจ: `BODIES` > 0 และ `STEP ms` แสดง; กด Drop Test แล้วพาเลทตกถึงพื้นและหยุด (หลับ); กด 7 ครั้งเหลือพาเลท drop ไม่เกิน 6.
- **M3 สินค้า + สายพาน**: cargo factory, stock เริ่มต้น, conveyor + เกต + CV-OUT. ตรวจ: กล่อง/ถังไหลบนสายพานไปหยุดที่เกต; เลื่อน slider เป็น 0 ของหยุด; ครบ 10 ชิ้น → log "พาเลทรับของเต็ม 10 ชิ้น".
- **M4 รถยก**: RaycastVehicle + carriage + visuals + Manual Drive + HUD. ตรวจ: สลับ Manual, W เดินหน้า, Q ยกงาขึ้น (FORK m เพิ่ม, สูงสุด 3.95), E ลง, Space เบรก; ขับสอดงาใต้พาเลทแล้วยกได้ (LOAD kg ขึ้น).
- **M5 นำทาง + WMS + Autopilot**: graph, routes, mutex, dispatcher, state machine. ตรวจ: ปล่อยทิ้ง 5 นาที log ต้องมี `รับงาน PICK-…`, `ยกพาเลท`, `${id} เสร็จ`; รถวิ่งชิดขวา; ไม่มีรถค้างถาวร.
- **M6 รถบรรทุก + ออเดอร์**: truck cycle, ประตู, LOAD/TR-xx, KPI. ตรวจ: `loaded N/5` เพิ่มจนครบ → รถออก → 6 s → รถคันใหม่ plate ใหม่ถอยเข้า.
- **M7 พนักงาน + ป้าย + route ribbon + debug + search**: ตรวจ: พนักงานข้ามทางม้าลายแล้วรถชะลอ; toggle Collider Debug แล้วเห็นเส้น; ค้นหา `A2-03`/`FL-02`/`SO-45821` บินกล้องไป.
- **M8 ขัดเกลา**: ข้อความไทยทุกแผง, อัตรา FPS, Reset Scene, Expedite, หลบ livelock (ดูหมวด "สิ่งที่ต้องทำให้ถูกตั้งแต่แรก"). ตรวจ: ผ่าน Acceptance checklist ทั้งหมด.

---

# Acceptance checklist (ตรวจได้ในเบราว์เซอร์)
1. เปิดหน้าแล้ว loader หายภายในไม่กี่วินาที, console ไม่มี error, แสดงชื่อ `BC WareHouse` และ `WMS · 3D SIMULATION`
2. UI ไทยอ่านออกครบ; ฟอนต์ Inter + Noto Sans Thai โหลด
3. มี Rack 4 แถว A1 A2 B1 B2 ป้ายปลายแถวและรหัสหน้าคาน เช่น `A2-03`; ค้นหา `A2-03` บินกล้องไปที่ช่อง
4. ป้าย Dock `01 · DOCK · ขาออก` และ `02 · DOCK · ขาเข้า` เห็นที่ผนังตะวันตก
5. Active Forklifts เริ่ม `4` "of 6 available"; Fleet list มี `FL-01..FL-06`, FL-05/06 สถานะ `ชาร์จไฟ`
6. FL-01 เริ่มมีงาน `PICK-7843` จาก `A2-03`, การ์ด Current Task แสดง `PICK-7843`; callout `PICK-7843 | A2-03` ครอบพาเลทปูน และมวลรวมที่แสดงเป็น 1,225 kg (ปูน 1200 + พาเลท 25)
7. ปล่อยนิ่ง: งาน PICK/PUT/LOAD เกิดและเสร็จต่อเนื่อง; Event log เพิ่มรายการ และไม่เกิน 40 รายการ
8. `Pallets prepared` และ timeline เดิน `Preparing → Loading → Dispatched`; รถ `TRK-6W 70-1234` ออกเมื่อครบ 5 พาเลท แล้วคันใหม่เข้าเทียบ
9. Conveyor Speed slider เปลี่ยนค่า `m/s` ที่หน้าจอและความเร็วสายพานจริง
10. Payload slider 100–2500 แสดง `kg` ถูกต้อง; Drop Test ปล่อยพาเลทจาก 3.2–3.8 m (log ระบุความสูง) และคงไม่เกิน 6 ใบ
11. Manual Drive: ปุ่มและ HUD ตามหมวด §15 (Q ขึ้น / E ลง) ทำงานตรงกับป้ายบน HUD
12. ช่องค้นหา: กด `/` โฟกัสช่องค้นหา, ป้ายมุมช่องต้องเป็น `/` ไม่ใช่ `⌘K`; Enter ค้นหาแล้ว flyTo
13. Collider Debug toggle แสดง/ซ่อนเส้นกล่องชน
14. Expedite: log "เพิ่มรถหยิบสินค้าพร้อมกันเป็น 3 คัน"; Route toggle เปลี่ยนจำนวน chevron
15. Reset Scene บินกล้องกลับ home และล้างพาเลททดสอบ
16. FPS / Hz physics แสดงที่แถบบนและแผง Physics (Hz ≈ 60)
17. ไม่มีไฟล์รูปหรือไฟล์อื่นนอกจาก `index.html`; ไม่มี request ไปที่โดเมนนอก CDN ที่กำหนด

---

# สิ่งที่ต้องทำให้ถูกตั้งแต่แรก
เวอร์ชันปัจจุบันมีบั๊กที่รู้แล้ว 3 ข้อ ให้ทำให้ถูกในงานนี้เลย:

1. **ป้ายคีย์ HUD ต้องตรงกับพฤติกรรมจริง**: โค้ดเดิมเขียน `Q / E งาลง/ขึ้น` แต่การทำงานจริงคือ **Q = ยกงาขึ้น, E = ลดงาลง** (`lift = (Q?1:0) − (E?1:0)`; ค่าบวก = ยกขึ้น). ป้าย HUD ต้องเขียนเป็น `Q ยกงาขึ้น / E ลดงาลง` (และ log ตอนเลือกรถ `Q/E ยก-ลดงา`) ส่วน `Z/C`: **Z = เอียงเสาไปด้านหลัง (สูงสุด +5°), C = เอียงไปด้านหน้า (ต่ำสุด −3°)**. ตรวจ: กด Q แล้วค่า FORK m เพิ่ม, กด E แล้วลด.
2. **ป้ายคีย์ลัดช่องค้นหาต้องตรงกับคีย์จริง**: โค้ดเดิมแสดง `⌘K` แต่คีย์ที่ผูกจริงคือ **`/`** (`e.key === '/'` แล้วโฟกัสช่องค้นหา, ไม่ทำงานถ้าโฟกัสอยู่ที่ช่องอยู่แล้ว). ป้ายต้องแสดง `/` เท่านั้น. (ไม่ต้องทำ ⌘K เพิ่ม)
3. **Livelock รถ put-away เลนตะวันตก (x = −11.2) ถอยหลีกกันไปมาไม่จบ**: เมื่อรถ 2 คันขวางกันในเลนตะวันตก ต้องมีกติกาหลีกที่ **กำหนดแน่นอน (deterministic)** เพื่อไม่ให้ทั้งสองคันถอยหลีกพร้อมกันหรือสลับบทบาทไปมา:
   - ตัดสินผู้หลีกจากลำดับตายตัวชุดเดียวกันทุกครั้งเท่านั้น: รถที่ไม่ได้ถือของหลีกก่อนรถที่ถือของ; ถ้าเท่ากัน รถที่ `idx` มากกว่าหลีก. ห้ามใช้ค่าสุ่ม/เวลา
   - ขณะคันหนึ่งอยู่ใน step `yield` (หรือรอ `holdZone` ให้อีกคัน) อีกคันห้ามเริ่ม yield ให้คู่เดียวกัน; ผู้หลีกต้องไม่กลับมาเข้าเส้นทางจนกว่าอีกคันจะผ่านพ้นพื้นที่ขวางแล้ว
   - ถ้าผู้ที่ควรหลีกถอยไม่ได้ (ห้องถอย <0.4 m) ให้อีกคันเป็นผู้หลีกแทน (เป็นกติกาที่โค้ดเดิมมี) ไม่ให้ทั้งคู่ค้าง
   - ไม่ต้องเพิ่มกลไกอื่นนอกเหนือจากนี้ (เช่น ระบบจองเลนใหม่) ; กลไก recovery เดิม (ติด >25 s / รอ >45 s) เป็นแค่ด่านสุดท้าย ไม่ใช่วิธีแก้ปกติ
   - ตรวจ: ปล่อยให้ระบบรันนานๆ (หลายออเดอร์) ใน log ต้องไม่เห็น `หลีกทางให้ …` ระหว่างคู่เดิมซ้ำสลับกันต่อเนื่อง และไม่มีรถเข้า `กู้คืนอัตโนมัติ` จากเหตุนี้

---

# นอกขอบเขต (เป็นแนวคิดเท่านั้น ไม่ต้องทำ)
- เชื่อมระบบ ERP/WMS จริง, API จริง, ฐานข้อมูลจริง
- RTLS (ระบบระบุตำแหน่งแบบเรียลไทม์) / RFID / บาร์โค้ดจริง
- แจ้งเตือน LINE หรือช่องทางส่งข้อความภายนอก
- บัญชีผู้ใช้/สิทธิ์ (อวาตาร์ `Daniel Kim` และกระดิ่งเป็นภาพตกแต่ง), การบันทึกข้อมูลข้ามเซสชัน

# ข้อความแบรนด์ที่ใช้ในแอป (คัดลอกตามนี้ ห้ามแต่งเพิ่ม)
แอปนี้ **ไม่มี footer** ห้ามเพิ่ม; ข้อความแบรนด์ที่มีจริง:
- `<title>`: `BC WareHouse — 3D Construction Material Warehouse Simulation`
- meta description: `BC WareHouse: แบบจำลองคลังวัสดุก่อสร้าง 3 มิติ พร้อมฟิสิกส์ rigid-body จริง (Three.js + cannon-es) รถยก AI, สายพานลำเลียง และแดชบอร์ดแบบ glassmorphism`
- Loader: `BC WareHouse 3D Simulation`; แถบบน: `BC WareHouse` / `WMS · 3D SIMULATION` / `Main Warehouse (North Hub - WH-01)`
- ผนังหลังในฉาก: `BC WareHouse` / `North Hub · WH-01 · คลังวัสดุก่อสร้าง`; ข้างรถบรรทุก: `BC WareHouse Logistics` / `ขนส่งวัสดุก่อสร้าง · TRK-6W 70-1234`; จอแพ็ก: `BC WareHouse · Pack Station 01`
