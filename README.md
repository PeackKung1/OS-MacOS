# ระบบไฟล์ macOS: จาก HFS+ สู่ APFS

การเปลี่ยนจาก **HFS+ (Mac OS Extended)** มาเป็น **APFS (Apple File System)** สะท้อนการเปลี่ยนแนวคิดการจัดการพื้นที่และความปลอดภัยของระบบไฟล์ จากระบบที่เน้นดิสก์แบบเดิม ไปสู่ระบบที่ใช้การคัดลอกเมื่อเขียน การแชร์พื้นที่ และสแนปช็อตเป็นพื้นฐาน

## 1. วิวัฒนาการและภาพรวม

**HFS+** เปิดตัวในปี 1998 และเป็นระบบไฟล์หลักของ Mac มายาวนาน มีระบบ journaling เพื่อช่วยปกป้องความสอดคล้องของ metadata หลังระบบขัดข้อง

**APFS** ถูกออกแบบใหม่โดยคำนึงถึง Flash และ SSD เป็นหลัก Apple เริ่มใช้เป็นระบบไฟล์เริ่มต้นกับ macOS High Sierra **10.13** ในปี 2017 — ไม่ใช่ 10.3 — และยังใช้กับ macOS และแพลตฟอร์ม Apple รุ่นใหม่ APFS สามารถใช้กับ HDD ได้เช่นกัน แม้จะเน้น Flash/SSD ([Apple: About Apple File System](https://developer.apple.com/documentation/foundation/about-apple-file-system?changes=_2), [Apple Support: Disk Utility formats](https://support.apple.com/en-gb/guide/disk-utility/dsku19ed921c/22.7/mac/26))

## 2. Volume และ Container

```text
ดิสก์ / พาร์ติชัน
└── APFS Container
    ├── Volume A
    ├── Volume B
    └── Volume C
```

Container เป็นพื้นที่จัดเก็บที่หลาย APFS volumes ใช้ร่วมกัน แต่ละ volume ใช้พื้นที่ตามต้องการ จึงไม่ต้องกำหนดขนาดตายตัวทั้งหมดตั้งแต่แรก สามารถกำหนด quota หรือ reserve ให้ volume ได้ด้วย

ควรแยก **container** ออกจาก **partition**: container อยู่ภายในพื้นที่พาร์ติชัน และการแชร์พื้นที่เกิดขึ้นระหว่าง volumes ใน container เดียวกันเท่านั้น ไม่ใช่การแชร์ข้ามพาร์ติชันหรือ container ([Apple Support: Disk Utility formats](https://support.apple.com/en-gb/guide/disk-utility/dsku19ed921c/22.7/mac/26))

## 3. Copy-on-Write, Cloning และ Snapshots

### Copy-on-Write (CoW)

หลักการ **Copy-on-Write** คือไม่เขียนทับข้อมูลเดิมทันที แต่เขียนข้อมูลที่เปลี่ยนไปยังตำแหน่งใหม่ แล้วปรับ metadata ให้ชี้ไปยังข้อมูลชุดใหม่ วิธีนี้ช่วยลดความเสี่ยงที่โครงสร้างข้อมูลจะค้างอยู่ในสภาพเขียนไม่ครบเมื่อไฟดับหรือระบบขัดข้อง

APFS ใช้ CoW กับ metadata และใช้ร่วมกับ cloning และ snapshots แต่ไม่ควรสรุปว่า **ทุกการเขียนไฟล์ของแอปเป็นธุรกรรม atomic โดยอัตโนมัติ** ความ atomicity ขึ้นอยู่กับ primitive และขอบเขตการดำเนินการ

### Cloning

APFS สามารถสร้าง **clone** ของไฟล์หรือไดเรกทอรี โดยต้นฉบับและ clone อ้างอิงบล็อกข้อมูลที่เหมือนกันร่วมกัน เมื่อแก้ไขไฟล์ ระบบจะเขียนบล็อกที่เปลี่ยนแปลงแยกต่างหาก ส่วนบล็อกเดิมที่ยังเหมือนกันจะแชร์ต่อไป

การทำสำเนาจึงมักเร็วมากและใช้พื้นที่เพิ่มน้อยในช่วงแรก แต่ไม่ควรถือว่าเป็น **O(1) ที่รับประกันทุกกรณี** หรือไม่มีต้นทุนพื้นที่ตลอดไป เมื่อข้อมูลเปลี่ยน บล็อกใหม่ต้องใช้พื้นที่เพิ่ม และบางกรณีระบบอาจต้องทำสำเนาจริง

### Snapshots

**Snapshot** บันทึกสถานะของ volume ณ เวลาหนึ่ง และให้มุมมองแบบอ่านอย่างเดียวต่อสถานะนั้น ข้อมูลที่ยังไม่เปลี่ยนสามารถแชร์กับสถานะปัจจุบัน ทำให้สร้าง snapshot ได้รวดเร็ว

Snapshot ใช้เป็นส่วนหนึ่งของการย้อนกลับสถานะระบบหรือการทำงานของ Time Machine ได้ แต่ **ไม่ใช่ backup ที่สมบูรณ์ในตัวเอง** เพราะยังอยู่บนพื้นที่จัดเก็บเดิม หากดิสก์เสีย snapshot อาจหายไปด้วย

## 4. Metadata ความทนทาน และความปลอดภัย

### Fast Directory Sizing

**Fast Directory Sizing** ช่วยติดตามขนาดรวมของเนื้อหาในไดเรกทอรี ทำให้ประเมินขนาดได้เร็วโดยไม่ต้องไล่อ่านรายการไฟล์ทั้งหมดใหม่ทุกครั้ง เหมาะกับโฟลเดอร์ที่มีการเปลี่ยนแปลงไม่ถี่ ไม่ได้หมายความว่าทุกการถามขนาดของทุกโฟลเดอร์จะได้คำตอบทันที ([Apple Developer Archive: Features](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/APFS_Guide/Features/Features.html))

### Atomic Safe-Save

APFS มี primitive สำหรับ **Atomic Safe-Save** ที่ทำให้การ rename บางรูปแบบ โดยเฉพาะกับ bundles และ directories เกิดขึ้นเป็นธุรกรรมเดียว: จากมุมมองผู้ใช้ การดำเนินการเสร็จสมบูรณ์หรือไม่เกิดขึ้น

นี่ไม่เท่ากับการรับประกันว่า “การบันทึกไฟล์ทุกครั้งเป็น atomic” เพราะการเขียนเนื้อหาภายในไฟล์กับการเปลี่ยนชื่อเป็นคนละการดำเนินการ แอปต้องใช้วิธีบันทึกที่เหมาะสม

### Encryption

APFS รองรับรูปแบบการเข้ารหัสหลายแบบ รวมถึงแบบหลายคีย์ที่แยกคีย์สำหรับข้อมูลไฟล์กับ metadata ที่ละเอียดอ่อน ความสามารถที่ใช้ได้จริงขึ้นอยู่กับฮาร์ดแวร์และระบบปฏิบัติการ ([Apple Developer Archive: Features](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/APFS_Guide/Features/Features.html))

### Crash Protection

HFS+ ใช้ journaling เพื่อช่วยกู้คืนความสอดคล้องของ metadata หลังระบบขัดข้อง ส่วน APFS ใช้ **copy-on-write กับ metadata** เพื่อหลีกเลี่ยงการแก้โครงสร้างเดิมทับโดยตรง ช่วยเพิ่มความทนทานของระบบไฟล์ แต่ไม่ได้รับประกันว่าแอปจะไม่มีข้อมูลสูญหายหากแอปไม่ได้บันทึกหรือจัดการสถานะอย่างเหมาะสม

## 5. Volume ของระบบใน macOS รุ่นใหม่

macOS แยกเนื้อหาระบบออกจากข้อมูลที่เปลี่ยนแปลงได้ ใน APFS startup container อาจมีหลาย volumes เช่น Preboot, VM, Recovery, System และ Data

- **System Volume** เก็บไฟล์ระบบและแอปที่มากับ macOS
- **Data Volume** เก็บข้อมูลผู้ใช้และเนื้อหาที่เปลี่ยนแปลงได้ เช่น เอกสาร แอปที่ติดตั้งเพิ่ม และข้อมูลในไดเรกทอรีผู้ใช้

ตั้งแต่ **macOS Catalina 10.15** Apple แยก System Volume เป็น volume เฉพาะที่อ่านอย่างเดียวโดยปริยาย ส่วน **Signed System Volume (SSV)** ซึ่งเพิ่มการตรวจสอบความถูกต้องด้วยลายเซ็นเข้ารหัส เริ่มใน **macOS 11** สองแนวคิดนี้เกี่ยวข้องกัน แต่ไม่ได้เริ่มพร้อมกัน

บน macOS 11 ขึ้นไป ระบบบูตจาก snapshot ของ System Volume ที่ผ่านการตรวจสอบ seal ด้วย หากเนื้อหาระบบไม่ตรงกับค่าที่ Apple ลงนามไว้ กลไกความปลอดภัยจะตรวจพบ ([Apple Platform Security: Signed system volume security](https://support.apple.com/en-mt/guide/security/secd698747c9/web), [Apple Platform Security: Role of Apple File System](https://support.apple.com/guide/security-pdf/role-of-apple-file-system-seca6147599e/web))

## 6. ตารางเปรียบเทียบ

| คุณสมบัติ | HFS+ (Mac OS Extended) | APFS |
|---|---|---|
| ยุคและเป้าหมายหลัก | เริ่มใช้ปี 1998; มาจากยุค HDD | ค่าเริ่มต้นตั้งแต่ macOS High Sierra 10.13; เน้น Flash/SSD |
| จำนวน allocation blocks | สูงสุด 2^32 บล็อก | สูงสุด 2^63 บล็อก |
| File ID | 32-bit | 64-bit |
| ความละเอียด timestamp | 1 วินาที | 1 นาโนวินาที |
| การปกป้อง metadata เมื่อระบบขัดข้อง | Journaling | Copy-on-write |
| การทำสำเนาแบบ clone | ไม่มี native clone แบบ APFS | แชร์บล็อกที่เหมือนกันจนกว่าจะมีการแก้ไข |
| Snapshot | ไม่มี native snapshot แบบ APFS | รองรับ |
| การแชร์พื้นที่ระหว่าง volumes | ไม่มี APFS-style space sharing | volumes ใน container เดียวกันแชร์พื้นที่ว่างได้ |
| การเข้ารหัสในระบบไฟล์ | ไม่มีรูปแบบ native แบบเดียวกับ APFS | รองรับหลายรูปแบบ |
| Fast directory sizing | ไม่มี | รองรับ |
| Atomic safe-save primitive | ไม่มี primitive แบบ APFS | รองรับการดำเนินการบางชนิด |

ตัวเลขเรื่อง allocation blocks, File ID และ timestamp อ้างอิงจากตารางเปรียบเทียบของ Apple ([Apple Developer Archive: Volume Format Comparison](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/APFS_Guide/VolumeFormatComparison/VolumeFormatComparison.html))


## แหล่งข้อมูลจาก Apple

- [About Apple File System — Apple Developer](https://developer.apple.com/documentation/foundation/about-apple-file-system?changes=_2)
- [APFS Features — Apple Developer Archive](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/APFS_Guide/Features/Features.html)
- [Volume Format Comparison — Apple Developer Archive](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/APFS_Guide/VolumeFormatComparison/VolumeFormatComparison.html)
- [Signed system volume security — Apple Platform Security](https://support.apple.com/en-mt/guide/security/secd698747c9/web)
