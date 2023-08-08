# MongoDB

내가 정리한 [MongoDB](https://www.mongodb.com/docs/) [Compass](https://www.mongodb.com/docs/compass/current/) 메뉴얼

```javascript
//  한번에 여러 문서업데이트
db.missions.updateMany(
      {  },
      { $set: { "new_field" : ISODate('2023-07-01 00:00:00') } }
);
```
