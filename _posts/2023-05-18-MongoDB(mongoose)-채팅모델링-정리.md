---
layout: post
title: "[MongoDB] $lookup과 $project Stage을 통한 역할 분리하기"
date: 2023-05-18
categories:
  - "🎸"
tags:
  - MongoDB
  - mongoose
  - Express
image:
  path: https://webimages.mongodb.com/_com_assets/cms/kuzt9r42or1fxvlq2-Meta_Generic.png
  thumbnail: https://webimages.mongodb.com/_com_assets/cms/kuzt9r42or1fxvlq2-Meta_Generic.png
author: NOWIL
---

이전 프로젝트에서 코치와 사용자간의 1대1 채팅을 구성하였습니다. 코치의 역할과 사용자의 역할이 분명하게 분리되어 있습니다. 따라서 저희는 코치와 사용자의 컬렉션을 분리하여 모델링을 하였습니다.

그래서 채팅 발송자를 필드에 추가하려면 한 컬렉션의 필드가 여러 컬렉션을 references로 가리켜야 합니다. 또한 채팅을 보낸 유저의 role이 코치냐 사용자냐에 상관없이 sender로 똑같아야 합니다. 이 문제를 해결하기 위한 방법을 정리하였습니다.

# Mongoose ‘refPath’

위 문제점을 해결하기 위해 mongoose에서 `refPath` 를 제공합니다.

[Mongoose v6.2.10: Query Population](https://mongoosejs.com/docs/populate.html#dynamic-ref)

# refPath를 사용하지 않는 이유

[$lookup gets null result from refpath in array](https://stackoverflow.com/questions/69984367/lookup-gets-null-result-from-refpath-in-array)

- refpath를 사용할 경우 mongodb의 \$lookup aggregation을 사용할 수 없음.
- 무조건 populate() 메서드를 사용해야 함.
- populate() 메서드의 문제점 : 성능 문제, 컬렉션별 $project stage 활용의 어려움.

# 대안 : `null` 필드와 \$lookup 활용

[Mongoose가 'undefined'를 처리하는 방식에 대해](https://blog.ull.im/engineering/2019/03/22/mongooses-undefined-handling.html)

> **[mongoose](https://mongoosejs.com/)를 사용하면서 특정 필드의 값이 채워지지 않은채로 입력값이 들어오는 경우 `null` 필드가 만들어질 수 있다**

# \$lookup과 \$project Stage 활용

\$lookup의 주의점

> To each input document, the \$lookup stage **adds a new array field** whose elements are the matching documents from the "joined" collection.
> (각 입력 문서에 \$lookup 단계는 요소가 "결합된" 컬렉션의 일치하는 문서인 **새 배열 필드를 추가**합니다.)

- 참고 : mongodb 공식문서 [https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/)

예시)

```jsx
db.orders.aggregate([
  {
    $lookup: {
      from: "inventory",
      localField: "item",
      foreignField: "sku",
      as: "inventory_docs",
    },
  },
]);
```

```json
{
   "_id" : 1,
   "item" : "almonds",
   "price" : 12,
   "quantity" : 2,
   "inventory_docs" : [
      { "_id" : 1, "sku" : "almonds", "description" : "product 1", "instock" : 120 }
   ]
}
{
   "_id" : 2,
   "item" : "pecans",
   "price" : 20,
   "quantity" : 1,
   "inventory_docs" : [
      { "_id" : 4, "sku" : "pecans", "description" : "product 4", "instock" : 70 }
   ]
}
{
   "_id" : 3,
   "inventory_docs" : [
      { "_id" : 5, "sku" : null, "description" : "Incomplete" },
      { "_id" : 6 }
   ]
}
```

\$lookup stage로 사용자와 코치의 컬렉션을 각각 `sender_user` ,`sender_coach` 필드에 결합하였습니다. 그리고 각각의 필드는 배열로 결합되기 때문에 $project stage에서 원활하게 사용하기 위해 \$unwind stage를 사용합니다. 그리고 \$isNull stage를 사용하기 위해 \$unwind stage의 `preserveNullAndEmptyArrays` 옵션을 `true`로 설정합니다.

> Deconstructs an array field from the input documents to output a document for each element.
> (입력 문서에서 배열 필드를 분해하여 각 요소에 대한 문서를 출력합니다.)

- 참고 : mongodb 공식문서
  [https://www.mongodb.com/docs/manual/reference/operator/aggregation/unwind/](https://www.mongodb.com/docs/manual/reference/operator/aggregation/unwind/)

그리고 \$unwind stage가 반드시 \$lookup stage 뒤에 정의되어야 합니다.

> In the `db.collection.aggregate()` method and `db.aggregate()` method, pipeline stages appear in an array. **Documents pass through the stages in sequence.**
> (db.collection.aggregate() 메서드 및 db.aggregate() 메서드에서 파이프라인 단계는 배열에 나타납니다. 문서는 순서대로 단계를 거칩니다.)

- 참고 : mongodb 공식문서
  [https://www.mongodb.com/docs/manual/reference/operator/aggregation/unwind/](https://www.mongodb.com/docs/manual/reference/operator/aggregation/unwind/)
