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
