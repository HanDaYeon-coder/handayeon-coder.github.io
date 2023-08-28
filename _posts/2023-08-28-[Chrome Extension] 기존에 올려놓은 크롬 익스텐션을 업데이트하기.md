---
title: "[Chrome Extension] 기존에 올려놓은 크롬 익스텐션을 업데이트하기"
date: 2023-08-28
categories: [Chrome Extension]
---
### PART 1. 잘못된 크롬 확장 프로그램 업데이트 방식

기존에 올려 놓은 크롬 확장 프로그램을 버그를 수정한 새로운 코드로 업데이트 시켜야 했습니다.

따라서, Chrome Web Store Developer Dashboard에 들어가서 크롬 확장 프로그램을 업로드했던 방식과 동일하게 우측 상단에 `+ 새 항목 버튼`을 눌러서 새로운 코드로 변경된 크롬 확장 프로그램의 폴더를 업로드했습니다.

이때, 크롬 확장 프로그램의 버전을 이전(version : 1.0.1)보다 높게 하여 version 1.1.0으로 설정하여 새로운 파일의 확장 프로그램을 업로드시켜주었습니다.

```json
{
  "manifest_version": 3,
  "name": "DADAMDA",
  "description": "Add images, bookmarks, text highlights to your dadamda script book.",
  "version": "1.1.0",
  "background": {
    "service_worker": "/background/background.js"
  },
  "action": {
    "default_title": "bookmark",
    "default_icon": {
      "16": "/assets/image/dadamda-logo16.png",
      "48": "/assets/image/dadamda-logo48.png",
      "128": "/assets/image/dadamda-logo128.png"
     }
   },
   "icons": {
      "16": "/assets/image/dadamda-logo16.png",
      "48": "/assets/image/dadamda-logo48.png",
      "128": "/assets/image/dadamda-logo128.png"
   },
  "host_permissions": ["https://api-dev.dadamda.me/*"],
  "permissions": [
    "tabs",
    "storage",
    "activeTab",
    "scripting",
    "contextMenus"
  ],
  "web_accessible_resources": [
    {
      "resources": [ "/assets/image/dadamda-logo128.png", "/assets/image/icon/shortcutIcon.svg" ],
      "matches": [ "http://*/*", "https://*/*" ]
    },
    {
      "resources": [ "/assets/login/login.html"],
      "matches": [ "http://*/*", "https://*/*" ]
    }
  ]
}
```

🚨 그러나, 위의 방식처럼 하면 기존의 크롬 익스텐션에서 새로운 코드로 업데이트 시키는 방식이 아니라 기존 크롬 익스텐션은 업데이트 되지 않고 새로운 크롬 익스텐션을 게시하게 되었습니다. 

따라서 아래의 그림과 같이 동일한 chrome 웹 스토어에 확장 프로그램이 2개가 나왔습니다. 

그렇게 되면 기존에 다운 받은 사용자들이 버그가 수정된 버전의 확장 프로그램으로 업데이트를 할 수 없습니다.

![Alt text](/assets/img/2023-08-28-02/image-10.png)

![Alt text](/assets/img/2023-08-28-02/image-11.png)

---

### PART 2. Chrome developer 공식 문서 참고하기

![Alt text](/assets/img/2023-08-28-02/image-2.png)

출처 : <a href="https://developer.chrome.com/docs/webstore/update/"> https://developer.chrome.com/docs/webstore/update/ </a>

크롬 개발자 공식 문서를 참고해보면, `기존에 존재하는 크롬 웹 스토어 의 아이템을 업데이트하기 위해서는 변경된 파일들과 변경되지 않은 파일들을 포함하여 새로운 압축 파일을 업로드해야 한다고 했습니다.` 

따라서 이 글을 읽고 변경된 파일들과 변경되지 않은 파일들을 두 개 다 포함해서 압축 폴더를 만들어서 업로드했습니다.

그러나, 이러한 경우 동일한 이름의 manifest.json 파일이 2개가 중복되기 때문에 업로드할 수 없다는 에러가 발생하였습니다.

따라서, 직접 Chrome Web Store Developer Dashboard에 여러 시도를 해보면서 해결해냈습니다.

다음 파트에서 바로 해결해낸 방법에 대해서 소개해드리겠습니다.

---

### PART 3. 기존의 크롬 익스텐션 업데이트하기

직접 Chrome Web Store Developer Dashboard를 살펴보고, 알아낸 업데이트 방법에 대해서 소개해드리겠습니다.

#### 1. Chrome Web Store Developer Dashboard에 들어가서, 업데이트할 크롬 익스텐션을 선택해줍니다.

- 아래의 사진 중에서 맨 아래의 확장 프로그램인 버전 1.0.0을 선택하여 주겠습니다.
![Alt text](/assets/img/2023-08-28-02/image-10.png)

#### 2. 왼쪽의 카테고리 중 빌드 > 패키지를 선택해줍니다.
![Alt text](/assets/img/2023-08-28-02/image-3.png)

#### 3. 우측 상단의 새 패키지 업로드 버튼을 눌러서, 새로운 코드로 구성된 압축 파일을 업로드합니다.

![Alt text](/assets/img/2023-08-28-02/image-4.png)

![Alt text](/assets/img/2023-08-28-02/image-8.png)

- 업로드하면, 위의 사진과 같이 초안 버전은 1.1.0으로 되어 있고, 게시됨 버전은 1.0.0으로 되어 있습니다.
- 따라서, 1.0.0 버전에서 1.1.0 버전으로 업데이트할 수 있다는 의미를 나타냅니다.
- 🚨 이때, 주의할 점은 반드시 `게시됨 버전보다 업데이트할 버전이 높아야 합니다.` 

#### 4. 업데이트한 내용이 반영되었다면, `제출하여 검토받기 버튼`을 누릅니다.

![Alt text](/assets/img/2023-08-28-02/image-6.png)
![Alt text](/assets/img/2023-08-28-02/image-7.png)

![Alt text](/assets/img/2023-08-28-02/image-5.png)

- 위의 사진과 같이 최종 업데이트 날짜가 오늘 날짜로 되어 있음을 알 수 있습니다.
이때, 바로 업데이트가 되지 않고, 검토 대기중으로 변경됩니다! 
- 따라서 평일 기준 1~2일 후에 게시됨으로 변경되면 기존의 확장 프로그램을 새로운 버전으로 업데이트할 수 있습니다.

#### 5. 게시된 버전으로 설치된 확장 프로그램 업데이트 시키기
- 기존의 확장 프로그램이 새로운 버전으로 업데이트 되어 게시가 되면, 기존의 설치된 확장 프로그램을 새로운 버전으로 업데이트 시킬 수 있습니다.
- `chrome://extensions/` 페이지에서 설치된 확장 프로그램을 확인할 수 있습니다.
- 이때, 우측 상단의 개발자 모드를 켜고, 업데이트를 누르면 버전이 새로운 버전으로 변경되는 모습을 확인할 수 있습니다.
![Alt text](/assets/img/2023-08-28-02/image-9.png)

---

Chrome extension을 직접 개발하면서 유지 보수를 해나가면서, 공식 문서에서는 해결하기 이번 처럼 어려운 문제가 여러 번 발생했습니다. 

그리고 이전에 영어 버전으로만 많이 나오고, 한국어로 된 자료들과 문서를 찾기 어려웠습니다. 

이번에도 업데이트하는 방법도 사용자가 업데이트 시키는 방법만 나오고, 개발자가 업데이트하는 방식은 나오지 않았습니다.

따라서, 제가 직접 해결해낸 방법을 글로 상세하게 작성해서 조금이라도 크롬 확장 프로그램 개발하는 데에 도움이 되었으면 좋겠습니다~!