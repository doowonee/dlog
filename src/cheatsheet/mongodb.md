# MongoDB

mongos는 [공식 문서](https://docs.mongodb.com/manual/core/sharded-cluster-components/)를 보다시피 app과 mongodb간에 존재하는 서버로 앱서버와 가까이 두는걸 권고한다. 그래서 보통 앱서버와 같은 서버에서 기동하는거 같음. 참고로 몽고디비는 `Document` 삽입시 `Collection`이 없으면 자동으로 만든다.

```javascript
//  한번에 여러 문서업데이트
db.missions.updateMany(
      {  },
      { $set: { "new_field" : ISODate('2023-07-01 00:00:00') } }
);

// index 스켄을 하지 않는 쿼리들을 조회 한다.
db.adminCommand({
 "currentOp": true,
 "op" : "query",
 "planSummary": "COLLSCAN"
})

// write lock 을 기다리는 쿼리를 조회 한다.
db.adminCommand({
 "currentOp": true,
 "waitingForLock" : true,
 "$or": [
    { "op" : { "$in" : [ "insert", "update", "remove" ] } },
    { "query.findandmodify": { "$exists": true } }
  ]
})

// 3초이상 실행되고있는 쿼리들을 조회 한다.
// 여기서 opid를 얻어 죽이면 넓은 범위의 삭제쿼리 죽이는거 가능하다.
db.adminCommand({
    "currentOp": true,
     "microsecs_running" : {"$gte" : 30000000}
})

// 지정된 opid에 해당하는 쿼리를 강제로 실행 중지 시킨다.
db.killOp(-607775162)

// 현재 in-progress operations 중인것들을 조회 한다
db.currentOp()
```

## index

Firestore랑 비슷하다. asc, desc 등의 순서와 문서 스캔시 여러 컬럼을 사용하려면 compound index를 별도로 만들어 주어야 한다. [문서](https://www.mongodb.com/docs/manual/indexes/)를 보면 알겠지만 단일 필드에 대한 인덱스는 순서가 상관이 없다

<https://www.mongodb.com/docs/manual/core/index-creation/#behavior> 보면 인덱스 생성은 synchronous operation으로 그냥 만들면 RW가 인덱스 생성 완료 될때 까지 대기 처리 되기 때문에 **프로덕션 환경에서 절대 사용 하면 안된다.** 그래서 `background` 옵션을 true로 하고 만들어야 하며 4.2 부터는 인덱스 구조의 효율성이 background 여부과 관계없이 동일하다. 클러스터 구성된 몽고디비의 경우 서비스를 예로들면 12억개 문서가 있는 컬렉션에 인덱스를 생성해보니 1개의 코어를 100% 사용하며 3시간 정도 소요 되고 생성이 완료 되면 슬레이브의 CPU 사용률이 마스터처럼 된다. 아마도 변경사항을 전파 하는과정에서 슬레이브도 동일한 연산을 내부적으로 하는거 같음.

```bash
# use db 한 db의 collection에 compound index를 asc 순서로 만든다
db.collection_name.createIndex( { name: 1, a: 1, p: 1 }, { background: true } )
```

## Data design

최대한 embedded document 모델을 사용하기를 권장한다. 물론 여러 문서 데이터 변경하는 multi document ACID transaction도 지원한다. 게시글과 댓글의 경우 nested로 하기 힘든 부분이 있음 댓글이 100개 이상 달릴때 또 댓글은 nested Objec라 limit도 정렬해서 가저올수도 없다는 단점이 있다. 이 구조는 유저의 배송 주소같은 상위 문서에 완전 종속되는 페이징 필요 없는 데이터를 지정하기에 안성맞춤 이라고 할수 있음. 또는 유저의 닉네임 히스토리 등도 해당할듯
