# Approach Notes

## Backend

### BE-1 — Exception on checkout

**อาการ**

`POST /v1/th/orders` ที่มีสินค้าอยู่ในตะกร้า คืน 500 ทุกครั้ง traceback ชี้ที่:

```
File "BE/app/services/order.py", line 69, in create_from_cart
    meal = self.db.meals[first_line["mealId"]]
KeyError: 'mealId'
```

**สาเหตุ**

`OrderLine` ใน `BE/app/models/schemas.py` ประกาศ field เป็น snake_case และไม่ได้ตั้ง `alias_generator` ไว้:

```python
class OrderLine(BaseModel):
    meal_id: str      # ← ชื่อ field จริง
    meal_name: str
    ...
```

`model_dump()` จึงคืน dict ที่ key ชื่อ `meal_id` การอ่าน `first_line["mealId"]` เลยไม่เจอ key แล้วโยน `KeyError` ทุกรอบที่ checkout

**สิ่งที่แก้**

ลบทั้ง block ออก (3 บรรทัด + comment กำกับ) แทนที่จะแก้ key ให้ถูก เพราะตรวจแล้วว่าเป็น dead code:

### BE-2 — Logical pricing bug

**อาการ**

ใส่ `meal_1` จำนวน 2 แล้วได้ subtotal 360 ที่ถูกคือ 158

| | ราคา/ชิ้น | × 2 |
|---|---|---|
| ที่ระบบคิด | `original_price` ฿180 | ฿360 |
| ที่ควรคิด | `discounted_price` ฿79 | ฿158 |

**สาเหตุ**

`CartService.get_cart` ใน `BE/app/services/cart.py` หยิบราคาก่อนลดมาคิด:

```python
unit_price = meal.original_price     # ← ราคาก่อนลด
line_total = unit_price * quantity
subtotal += line_total
```

**สิ่งที่แก้**

```python
unit_price = meal.discounted_price
```

แก้บรรทัดเดียวจบ เพราะ `line_total` กับ `subtotal` คิดต่อจาก `unit_price` อยู่แล้ว ส่วน `OrderService.create_from_cart` ก็ copy ทั้งสามค่ามาจาก cart ตรงๆ ไม่ได้คิดราคาเอง — ราคาฝั่ง order จึงถูกตามไปเอง

### BE-3 — Logical inventory bug on cancel

**อาการ**

สั่ง `meal_2` จำนวน 3 แล้ว cancel ออเดอร์ stock กลับมาแค่ 3 ที่ถูกคือ 5 เท่าเดิม

| ขั้นตอน | stock ที่ควรเป็น | ที่เกิดขึ้นจริง |
|---|---|---|
| หลัง reset | 5 | 5 |
| ใส่ตะกร้า 3 → place order | 2 | 2 |
| cancel order | 5 | **3** |

**สาเหตุ**

`OrderService.cancel` ใน `BE/app/services/order.py` วนคืน stock ทีละ order line แต่ hardcode จำนวนเป็น 1:

```python
for line in order.lines:
    self.stock.apply(
        meal_id=line.meal_id,
        quantity=1,                       # ← คืนแค่ 1 ชิ้นต่อ line
        event_type=StockEventType.INCREMENT,
```

order 1 line ที่มี quantity 3 จึงคืนแค่ 1 หน่วย หายไป 2 หน่วย เพราะจำนวนที่คืนต้องเท่าที่ตอน checkout ตัดออกไป คือ 3

**สิ่งที่แก้**

เปลี่ยนจำนวนที่คืน จากเลข 1 ที่ hardcode ไว้ เป็นจำนวนจริงของ order line นั้น:

```diff
- quantity=1
+ quantity=line.quantity
```

order line มี 3 ชิ้น ก็คืน 3 ชิ้น เท่ากับที่ตัดออกไปตอนใส่ตะกร้า stock จึงกลับเป็น 5

### BE-4 — Logical inventory bug on cart quantity increase

**อาการ**

`meal_2` ใส่ตะกร้า 1 ชิ้น แล้วกดเพิ่มเป็น 3 stock ขึ้นเป็น 6 ที่ถูกคือ 2

| ขั้นตอน | stock ที่ควรเป็น | ที่เกิดขึ้นจริง |
|---|---|---|
| หลัง reset | 5 | 5 |
| ใส่ตะกร้า 1 | 4 | 4 |
| เพิ่มเป็น 3 | 2 | **6** |

**สาเหตุ**

`CartService.update_item` ใน `BE/app/services/cart.py` ตอนเพิ่มจำนวน สั่งบวก stock แทนที่จะตัด:

```python
if delta > 0:                                 # ลูกค้าเพิ่มจำนวน
    self.stock.apply(
        meal_id=meal_id,
        quantity=delta,
        event_type=StockEventType.INCREMENT,  # ← บวกของคืนเข้า stock
        note="reserve on cart increase",
```

เพิ่มจำนวนในตะกร้าคือจองของเพิ่ม stock ต้องลด แต่โค้ดบวกคืน ตัวเลขเลยวิ่งผิดทาง จาก 4 ควรลงไป 2 กลับขึ้นไป 6

**สิ่งที่แก้**

เปลี่ยน event type ตอนเพิ่มจำนวน ให้ตัด stock ตามที่ `note` ของมันเขียนไว้ว่า reserve:

```diff
- event_type=StockEventType.INCREMENT
+ event_type=StockEventType.DECREMENT
```

เพิ่มในตะกร้า 2 ชิ้น ก็ตัด stock 2 ชิ้น เหลือ 2 ส่วนตอนลดจำนวนกับ `clear()` ยังใช้ `INCREMENT` เหมือนเดิม เพราะสองอันนั้นคืนของจริง


## Frontend

### FE-1 — Wrong “You pay” price

**อาการ**

หน้า meal list โชว์ราคาขีดฆ่ากับราคาที่ต้องจ่ายเป็นเลขเดียวกัน ทั้งที่ `meal_1` ลดจาก ฿180 เหลือ ฿79

| | ที่ควรโชว์ | ที่โชว์จริง |
|---|---|---|
| ราคาขีดฆ่า | ฿180 | ฿180 |
| You pay | ฿79 | **฿180** |

**สาเหตุ**

`MealsPage` ใน `FE/src/pages/MealsPage.tsx` ส่ง `original_price` เข้าไปทั้งสองช่อง:

```tsx
<span className="strike">{formatBaht(meal.original_price)}</span>
<strong className="pay">{formatBaht(meal.original_price)}</strong>   {/* ← ควรเป็นราคาลด */}
```

ช่อง `.strike` ใช้ราคาก่อนลดถูกแล้ว แต่ช่อง `.pay` copy มาจากบรรทัดบน เลยได้ราคาก่อนลดตามไปด้วย

**สิ่งที่แก้**

```diff
- <strong className="pay">{formatBaht(meal.original_price)}</strong>
+ <strong className="pay">{formatBaht(meal.discounted_price)}</strong>
```

API ส่ง `discounted_price` มาให้อยู่แล้ว แก้แค่ฝั่ง frontend ไม่ต้องแตะ backend
