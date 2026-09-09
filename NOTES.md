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

`OrderLine` (`app/models/schemas.py:94`) ประกาศ field เป็น snake_case และไม่ได้ตั้ง `alias_generator` ไว้ ดังนั้น `model_dump()` จึงคืน dict ที่มี key ชื่อ `meal_id` ไม่ใช่ `mealId` — การอ่าน `first_line["mealId"]` เลยโยน `KeyError` ทุกรอบที่ checkout

**สิ่งที่แก้**

ลบทั้ง block ออก (3 บรรทัด + comment กำกับ) แทนที่จะแก้ key ให้ถูก เพราะตรวจแล้วว่าเป็น dead code:



