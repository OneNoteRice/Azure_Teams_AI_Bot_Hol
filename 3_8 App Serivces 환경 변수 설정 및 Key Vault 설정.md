- [x] `🟢 최종 App servcies 환경 변수`

```
AZURE_OPENAI_DEPLOYMENT_NAME : 질문에 답변을 생성해 주는 대화용 AI 모델 (GPT-4o)

AZURE_OPENAI_EMBEDDING_DEPLOYMENT : 문장을 AI 검색용 숫자(임베딩)로 바꿔주는 전용 모델 (text-embedding-3-small)

AZURE_OPENAI_ENDPOINT : Azure OpenAI 서비스에 접속하기 위한 인터넷 주소(URL)입니다.

AZURE_SEARCH_ENDPOINT : Azure AI Search 서비스에 접속하기 위한 인터넷 주소(URL)입니다.

AZURE_SEARCH_INDEX : 변환된 문서 데이터가 실제 저장되고 검색되는 AI Search 내부 저장소(인덱스) 이름입니다.

DOCKER_REGISTRY_SERVER_PASSWORD : Github 도커 이미지 저장소(ACR 등)에서 봇 프로그램을 가져오기 위한 접속 비밀번호입니다.

DOCKER_REGISTRY_SERVER_URL : Github 봇 프로그램이 이미지 형태로 저장되어 있는 도커 레지스트리 서버 주소입니다.

DOCKER_REGISTRY_SERVER_USERNAME : Github 도커 레지스트리 서버에 접속하기 위한 사용자 아이디입니다.

MicrosoftAppId : Bot 서비스 등록 시 Microsoft에서 발급받은 봇 고유 식별 번호(앱 ID)입니다.

MicrosoftAppPassword : Bot 서비스 인증에 사용하는 보안 비밀번호(Client Secret)입니다.

MicrosoftAppTenantId : Bot 서비스 소속된 회사/조직의 Azure Entra ID(구 Active Directory) 테넌트 고유 ID입니다.

STORAGE_CONNECTION_STRING : 리포트 파일(PDF, PPT)을 저장하는 Azure Blob 스토리지 창고에 접근할 수 있는 마스터 열쇠(연결 문자열)입니다.

WEBSITES_ENABLE_APP_SERVICE_STORAGE : App Service 내부의 자체 파일 저장 공간을 사용할지 여부를 설정합니다

WEBSITES_PORT : App Service가 외부 통신 요청을 받아들이기 위해 열어두는 봇 프로그램의 포트 번호 (8000)
```

<img width="1307" height="998" alt="image" src="https://github.com/user-attachments/assets/d37b3fd2-dd51-4306-adc1-0fe4a12c5b40" />


- [x] `🟢 아래 환경 변수의 이름과 값들을 복사`

```
STORAGE_CONNECTION_STRING: 스토리지 전체 권한을 가진 마스터 키

MicrosoftAppPassword: Teams 봇 로그인 비밀번호

DOCKER_REGISTRY_SERVER_PASSWORD: GitHub/Docker 레지스트리 접근 토큰
```

- [x] `🟢 키 자격 증명 모음으로 이동 후 액세스 제어 및 추가 - 역할 추가 진행`

<img width="1384" height="837" alt="image" src="https://github.com/user-attachments/assets/591020a9-e222-43da-a936-2161e088413b" />


- [x] `🟢 역할 - Key Vault 관리자 선택`

<img width="1515" height="764" alt="image" src="https://github.com/user-attachments/assets/913d242f-fc38-48d9-adf7-ae9857b19406" />


- [x] `🟢 구성원 - 현재 Portal에서 리소스 생성 및 관리하는 계정 선택 후 할당 추가 진행`

<img width="1515" height="744" alt="image" src="https://github.com/user-attachments/assets/9c8a5bee-c25a-4aa1-8bd2-96049e1ee401" />
<img width="1184" height="527" alt="image" src="https://github.com/user-attachments/assets/f7d4eb40-9208-45fb-a6ff-8576e120fe56" />


- [x] `🟢 비밀 선택 후 생성`

<img width="1515" height="600" alt="image" src="https://github.com/user-attachments/assets/1e115b3b-45bd-4fbb-870d-1a0ecf4f7594" />


- [x] `🟢 이름 및 값 입력 후 만들기 진행 `

```
총 3개 진행
_ 의 경우 - 로 변경해서 진행

STORAGE_CONNECTION_STRING: 스토리지 전체 권한을 가진 마스터 키

MicrosoftAppPassword: Bot 서비스 로그인 비밀번호

DOCKER_REGISTRY_SERVER_PASSWORD: GitHub/Docker 레지스트리 접근 토큰
```

<img width="1162" height="576" alt="image" src="https://github.com/user-attachments/assets/aa5ffd89-ed6c-4b8c-a55c-3253bc123a54" />
<img width="1154" height="600" alt="image" src="https://github.com/user-attachments/assets/10207211-94b6-4525-a8b3-4f5960e80a40" />


- [x] `🟢 생성된 비밀 1개씩 선택`


<img width="1320" height="347" alt="image" src="https://github.com/user-attachments/assets/0e984266-b24a-452d-82f1-c043e8ce8944" />


- [x] `🟢 비밀 식별자 복사 각각 총 3개 `

<img width="1085" height="703" alt="image" src="https://github.com/user-attachments/assets/8f271256-341a-4495-894c-4f1a98956411" />

  
- [x] `🟢 App services에서 각각의 환경 변수의 값을 변경 후 적용 `

<img width="2200" height="986" alt="image" src="https://github.com/user-attachments/assets/1fbf8bff-07f7-4791-b7c2-d168e8e9421b" />






