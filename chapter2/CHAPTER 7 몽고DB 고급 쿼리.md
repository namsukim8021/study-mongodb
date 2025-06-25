# 📘 Mastering MongoDB - Chapter 7
## 몽고DB 고급 쿼리 기법 (Advanced Query Techniques)


**브랜드와 브랜드 국제화 정보 조인 예시**
```
// --- 브랜드와 브랜드 국제화 정보 조인 예시 ---
db.brand.aggregate([
   {
        $lookup: {
            from: "brand_i18n",
            localField: "_id",
            foreignField: "_id",
            as: "brand_global_info"
        }
   }
],
{
    allowDiskUse: true
}
)
```