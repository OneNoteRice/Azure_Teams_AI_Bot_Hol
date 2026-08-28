- [x] `🟢 스토리지 계정 검색 후 컨테이너 선택`

<img width="2151" height="742" alt="image" src="https://github.com/user-attachments/assets/cf3f1733-f01a-4768-80e4-06cf263b63eb" />


- [x] `🟢 컨테이너 추가 이름 입력(자유)`


<img width="1424" height="644" alt="image" src="https://github.com/user-attachments/assets/8588cb58-0cc1-4414-bd8f-1866a7877263" />


- [x] `🟢 PDF 파일 업로드 (가상의 업무 경비 PDF 파일)`
 
<img width="1424" height="623" alt="image" src="https://github.com/user-attachments/assets/66c48b5f-6fc7-4c04-85ee-6d7ae6fb0403" />


- [x] `🟢 PDF 파일 업로드 확인 완료`

<img width="1424" height="621" alt="image" src="https://github.com/user-attachments/assets/e16e3d4a-eff0-4c01-a07a-87c8a7e227b4" />


- [x] `🟢 AI Search 검색 후 인덱스 추가 (JSON) 선택`
- [x] `🟢  AI Search의 경우 무료로 사용 시 OpenAI - 네트워크 설정 - 선택한 네트워크 및 프라이빗 엔드포인트에서 신뢰할 수 있는 서비스 목록의 Azure 서비스가 이 인지 서비스 계정에 액세스하도록 허용합니다를 선택해도 적용 불가`
- [x] `🟢 AI Search 가격 계층을 Basic으로 하는 경우 약 $75 `
- [x] `🟢 Basic으로 하는 경우 OpenAI와 프라이빗 공유 사용 가능으로 설정이 간단해지지만 한달 이후까지 사용하려면 무료버전으로 설정 필요`
- [x] `🟢 아래 JSON 코드에서 name 부분 변경 필요🟢`
```json
{
  "name": "사용할 인덱스 이름",
  "fields": [
    {
      "name": "id",
      "type": "Edm.String",
      "key": true,
      "searchable": false,
      "filterable": false,
      "retrievable": true
    },
    {
      "name": "snippet",
      "type": "Edm.String",
      "searchable": true,
      "filterable": false,
      "retrievable": true
    },
    {
      "name": "blob_url",
      "type": "Edm.String",
      "searchable": false,
      "filterable": false,
      "retrievable": true
    },
    {
      "name": "snippet_vector",
      "type": "Collection(Edm.Single)",
      "searchable": true,
      "retrievable": true,
      "dimensions": 1536,
      "vectorSearchProfile": "my-vector-profile"
    }
  ],
  "vectorSearch": {
    "algorithms": [
      {
        "name": "my-hnsw-config",
        "kind": "hnsw"
      }
    ],
    "profiles": [
      {
        "name": "my-vector-profile",
        "algorithm": "my-hnsw-config"
      }
    ]
  }
}
```

<img width="966" height="658" alt="image" src="https://github.com/user-attachments/assets/8034d808-cc76-40ba-b85c-95af35bec932" />

