---
title: "DynamoDB 훑어보기"
date: 2023-07-17T12:13:50+09:00
draft: true
---

시간적 자원이 충분한 경우 공식 문서를 읽어보는게 좋겠지만, 그 전에 빠르게 훑어볼 수 있는 내용을 간략히 작성해봤다.

## DynamoDB

> Amazon DynamoDB는 모든 규모에서 고성능 애플리케이션을 실행하도록 설계된 완전관리형의 서버리스 키-값 NoSQL 데이터베이스입니다. DynamoDB는 기본 제공 보안, 지속적인 백업, 자동화된 다중 리전 복제, 인 메모리 캐시 및 데이터 가져오기/내보내기 도구를 제공합니다.

https://aws.amazon.com/ko/dynamodb/

### Documentations

- https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/
- https://github.com/aws/aws-sdk-js-v3
- https://docs.aws.amazon.com/ko_kr/amazondynamodb/latest/developerguide/Introduction.html

## 훑어보기

### NoSQL이며 오픈소스 SQL 호환 쿼리 언어인 PartiQL을 지원

```sql
INSERT INTO "Music" value {'Artist' : 'Acme Band','SongTitle' : 'PartiQL Rocks'}
```

https://partiql.org

직접 PartiQL 문법을 사용할 수 있지만, SDK가 있으니 편리하게 이용하면 된다.

### 데이터 형식, Marshaling, Unmarshaling

DynamoDB를 읽고 쓸 때 데이터 형식은 아래와 같다. 프로그램에서 일반적으로 사용하는 데이터 객체의 형식과 상이하기 때문에 변환하는 과정이 필요하다.

```javascript
{
  TableName: "string",
  Item: {
    "type": {
      "S": "fruit"
    },
    "name": {
      "S": "사과"
    },
    "production_year": {
      "N": 2023
    }
  }
}
// (키): (값)의 형태가 아니라
// (키): {
//    (타입): (값)
// } 의 형태이다.
```

아래는 별도의 변환없이 데이터를 추가하는 코드 예시이다.

```javascript
import { DynamoDBClient, PutItemCommand } from "@aws-sdk/client-dynamodb"; // ES Modules import
// const { DynamoDBClient, PutItemCommand } = require("@aws-sdk/client-dynamodb"); // CommonJS import
const client = new DynamoDBClient(config);
const input = {
  // PutItemInput
  TableName: "STRING_VALUE", // required
  Item: {
    // PutItemInputAttributeMap // required
    "<keys>": {
      // AttributeValue Union: only one key present
      S: "STRING_VALUE",
      N: "STRING_VALUE",
      B: "BLOB_VALUE",
      SS: [
        // StringSetAttributeValue
        "STRING_VALUE",
      ],
      NS: [
        // NumberSetAttributeValue
        "STRING_VALUE",
      ],
      BS: [
        // BinarySetAttributeValue
        "BLOB_VALUE",
      ],
      M: {
        // MapAttributeValue
        "<keys>": {
          //  Union: only one key present
          S: "STRING_VALUE",
          N: "STRING_VALUE",
          B: "BLOB_VALUE",
          SS: ["STRING_VALUE"],
          NS: ["STRING_VALUE"],
          BS: ["BLOB_VALUE"],
          M: {
            "<keys>": "<AttributeValue>",
          },
          L: [
            // ListAttributeValue
            "<AttributeValue>",
          ],
          NULL: true || false,
          BOOL: true || false,
        },
      },
      L: ["<AttributeValue>"],
      NULL: true || false,
      BOOL: true || false,
    },
  },
  Expected: {
    // ExpectedAttributeMap
    "<keys>": {
      // ExpectedAttributeValue
      Value: "<AttributeValue>",
      Exists: true || false,
      ComparisonOperator:
        "EQ" ||
        "NE" ||
        "IN" ||
        "LE" ||
        "LT" ||
        "GE" ||
        "GT" ||
        "BETWEEN" ||
        "NOT_NULL" ||
        "NULL" ||
        "CONTAINS" ||
        "NOT_CONTAINS" ||
        "BEGINS_WITH",
      AttributeValueList: [
        // AttributeValueList
        "<AttributeValue>",
      ],
    },
  },
  ReturnValues:
    "NONE" || "ALL_OLD" || "UPDATED_OLD" || "ALL_NEW" || "UPDATED_NEW",
  ReturnConsumedCapacity: "INDEXES" || "TOTAL" || "NONE",
  ReturnItemCollectionMetrics: "SIZE" || "NONE",
  ConditionalOperator: "AND" || "OR",
  ConditionExpression: "STRING_VALUE",
  ExpressionAttributeNames: {
    // ExpressionAttributeNameMap
    "<keys>": "STRING_VALUE",
  },
  ExpressionAttributeValues: {
    // ExpressionAttributeValueMap
    "<keys>": "<AttributeValue>",
  },
  ReturnValuesOnConditionCheckFailure: "ALL_OLD" || "NONE",
};
const command = new PutItemCommand(input);
const response = await client.send(command);
```

위처럼 데이터 형식을 직접 다루지 않고 편리하게 아래 방법으로 변환해서 사용할 수 있다.

1. @aws-sdk/util-dynamodb의 marshal/unmarshal 함수를 사용해 직접 변환 https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-aws-sdk-util-dynamodb/
2. @aws-sdk/lib-dynamodb의 DynamoDBDocumentClient 사용 https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-aws-sdk-lib-dynamodb/

전자는 직접 marshal(), unmarshal() 함수를 직접 호출해서 변환할 수 있다. 후자는 단순히 DynamoDBDocumentClient를 사용해, 입출력 사이에 marshaling/unmarshaling을 자동으로 한다. 후자가 편리하므로 대부분의 경우 바람직하다. 아래는 후자의 예시이다.

```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { PutCommand, DynamoDBDocumentClient } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

export const main = async () => {
  const command = new PutCommand({
    TableName: "HappyAnimals",
    Item: {
      CommonName: "Shiba Inu",
    },
  });

  const response = await docClient.send(command);
  console.log(response);
  return response;
};
```

### Index, Partition Key, Order Key

DynamoDB에서 데이터에 접근할 때 사용하는 인덱스로 파티션 키, 정렬 키가 있다.

일반적으로 DynamoDB에서 테이블의 수를 최소한으로 설계할 것을 권장한다. 파티션 키는 동일 테이블 내에서 물리적으로 공간을 분리하는 것을 의미한다. PK 값이 다르면 물리적으로 다른 공간에 존재한다. 테이블을 설계할 때 성능 최적화 목적으로 활용할 수 있다. 빈번히 쓰이는 데이터들을 동일한 PK 아래로 놓지 않고 분산해주면, 하나의 읽기 유닛에 작업이 집중되어 병목이 일어나는 것을 방지할 수 있고 여러 유닛이 작업을 분담하여 병렬 처리의 이점을 취할 수 있다.

정렬 키는 파티션 키와 더불어 테이블에서 데이터를 검색하는데 사용할 수 있으며, 검색 결과를 정렬하는데도 사용할 수 있다. DynamoDB에서는 파티션 키, 정렬 키 이외의 키 값으로 검색하는 것은 불가능하다. 검색의 제약 조건은 파티션 키와 정렬 키로만 설정 가능하며, DynamoDB단에서는 해당하는 데이터 전체를 불러오므로 추가적인 필터링은 프로그램단에서 수행해야 한다.

#### 보조 인덱스

오로지 파티션 키, 정렬 키로만 검색할 수 있다면 문제가 될 것이다. 불필요하게 많은 입출력과 연산이 발생하게 될 것이다. 데이터를 더 효율적으로 불러오기 위해서 보조 인덱스를 설정할 수 있다. 그러면 해당 보조 인덱스를 제약 조건으로 데이터를 탐색할 수 있다.

- 글로벌 보조 인덱스
- 로컬 보조 인덱스
- https://docs.aws.amazon.com/ko_kr/amazondynamodb/latest/developerguide/SecondaryIndexes.html
- https://docs.aws.amazon.com/ko_kr/amazondynamodb/latest/developerguide/GSI.html

#### 인덱스 모범 사례

- 파티션 키 설계: https://docs.aws.amazon.com/ko_kr/amazondynamodb/latest/developerguide/bp-partition-key-design.html
- 정렬 키 설계: https://docs.aws.amazon.com/ko_kr/amazondynamodb/latest/developerguide/bp-sort-keys.html

### 테스트 환경 만들기

아래에서 제공하는 로컬 버전을 사용할 수 있다.

https://docs.aws.amazon.com/ko_kr/amazondynamodb/latest/developerguide/DynamoDBLocal.html
