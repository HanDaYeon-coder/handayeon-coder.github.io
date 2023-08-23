---
title: "Route53를 사용한 사이트를 Google Search Console에 등록하기"
date: 2023-08-23
categories: [Route53, Google Search Console]
---

### PART 1. Google Search Console을 등록하는 이유
![Alt text](/assets/img/2023-08-23/image.png)

먼저, Google OAuth2를 프로덕션으로 푸시하기 위해서는 위의 사진 4가지를 모두 제공해야 했습니다.

따라서, 4번인 `Google Search Console에서 확인된 모든 도메인`이라는 조건을 만족하기 위해서는 Google Search Console에 현재 프로젝트 진행중인 dadamda.me 사이트를 등록해야 했습니다.

그러면 `Google Search Console`은 무엇을 하는 곳일까요??

말 그대로 **구글 검색**과 관련된 것들을 제공해줍니다.

새롭게 사이트 도메인 주소를 구매해서 구글 검색창에 검색해봐도 자신의 사이트와 관련된 정보가 나오지 않습니다. 

따라서 자신의 도메인 주소와 관련된 정보들을 구글 검색시에 나오도록 Google Search Console에 등록해줘야 합니다.

그래야 사용자들이 검색을 통해서 자신의 사이트에 들어올 수 있습니다.

이때, 누구나 사이트 도메인 주소를 Google Search Console에 등록할 수 있다면, 원하지 않는 정보와 관련되게 등록 및 검색에서 제거하거나, Google Search Console를 통해서 검색한 정보 통계자료 등을 확인할 수 있기 때문에 해당 사이트 도메인 주소가 자신의 소유권임을 확인해줘야 합니다.

---

### PART 2. Google Search Console에 등록하기

#### 1. 먼저, Google Search Console 사이트에 들어가서, 원하는 사이트의 속성을 추가합니다.

- <a href="https://search.google.com/search-console/about">https://search.google.com/search-console/about</a>

    ![Alt text](/assets/img/2023-08-23/image-1.png)

    ![Alt text](/assets/img/2023-08-23/image-2.png)

    속성 유형을 선택할 때에는 크게 **도메인**, **URL  접두어**로 나누어집니다.
    
    이때 새롭게 사이트 도메인 주소를 구매했다면, 도메인을 통해서 인증하는 게 좋습니다. 
    
    모든 하위 도메인들과 여러 프로토콜을 포함해서 등록해 주기 때문입니다.
    
    도메인을 사용하여 속성을 추가하면, example.com 속성이 추가되고 아래의 속성도 자동으로 추가됩니다.

    - http://example.com/dresses/1234
    - https://example.com/dresses/1234
    - http://www.example.com/dresses/1234
    - http://support.m.example.com/dresses/1234
    
    그러나, URL 접두사 속성을 사용하면, 제한된 속성을 생성할 수 있어서 여러 속성을 추가해줘야 합니다.

    따라서 저는 **도메인 속성**을 추가해주도록 하겠습니다.

    출처 : <a href="https://support.google.com/webmasters/answer/34592"> https://support.google.com/webmasters/answer/34592 </a>

<br/>

#### 2. DNS 레코드를 통해, 도메인 소유권 확인하기
    - 도메인 이름 제공 업체에 TXT 레코드를 dadamda.me의 DNS 설정에 복사해서 넣어줍니다.
    
    ![Alt text](/assets/img/2023-08-23/image-3.png)

    - 이때, 저는 도메인 이름 제공업체를 도메인 주소를 구매한 사이트로 생각하고 '가비아'(https://www.gabia.com)라는 도메인 구매 사이트에서 도메인 관리 페이지에 들어가고 DNS 설정을 했습니다. 
    - TXT, @(호스트), TXT 레코드(값/위치), 3600(TTL)을 등록해주었습니다.
    
    ![Alt text](</assets/img/2023-08-23/image-10.png>)
    
    ![Alt text](/assets/img/2023-08-23/image-5.png)

    그러나, Google Search Console에서 `소유권 확인 실패`라는 문구가 떴습니다. 

    시간이 소요될 수 있다는 문구를 보고, 하루 후에도 확인해봤는데 동일한 오류 문구가 나왔습니다.

    ![Alt text](</assets/img/2023-08-23/image-11.png>)

---

### PART 3. Route53에 Google Search Console TXT 레코드 등록하기

실패 문구에서 '도메인 이름 공급업체'에 주목해서, 도메인 이름 공급업체가 가비아가 아닐 수도 있다는 생각을 했습니다.

이때, dadamda.me의 네임서버가 aws로 되어 있어서 가비아가 아닌 **aws에서 도메인 관련된 Route53**에 등록해야 함을 알게 되었습니다.


참고 사이트 : <a href="https://delay100.tistory.com/m/32"> https://delay100.tistory.com/m/32 </a>

#### 1. aws > Route 53 > Hosted zones 페이지로 이동합니다.

![Alt text](/assets/img/2023-08-23/image-6.png)

#### 2. 등록하고 싶은 host(dadamda.me)를 선택하고, Create record를 클릭합니다.

![Alt text](/assets/img/2023-08-23/image-8.png)

#### 3. Record type은 TXT, Value는 Google Search Console에서 복사한 값을 넣어주고 Create records를 선택합니다. 
- TTL은 해당 값이 반영되는 시간입니다.
(초기값이 300으로 설정되어 있으며, 따로 변경하지는 않았습니다.)

![Alt text](/assets/img/2023-08-23/image-9.png)

#### 4. 다시 Google Search Console로 돌아와서 확인을 해주면 `소유권이 확인됨`이라는 문구를 확인할 수 있습니다.

![Alt text](</assets/img/2023-08-23/image-12.png>)
