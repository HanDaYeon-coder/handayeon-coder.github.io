---
title: "[Chrome Extension] jest와 jest-webextension-mock 사용해서 테스트 코드 작성하는 법"
date: 2023-08-20
categories: [Chrome Extension, jest, jest-webextension-mock, test-code]
---

### PART 1. chrome test 코드 작성시, 에러 발생

먼저, <a href="https://github.com/SWM-team-forever/dadamda-chrome-extension/blob/develop/background/background.js">
다담다 크롬 익스텐션의 background.js 파일</a>의 함수들에 대해서 테스트 코드를 작성해보겠습니다. 

```javascript
// dadamda-chrome-extension/blob/develop/background/background.js
function handleScrapOrHighlightResponse(url, tab, data) {
  chrome.storage.local.get('signedIn').then(async (result) => {
    if (result.signedIn) {
      contentScriptJS(tab.id, "content/content.js");
      contentScriptCSS(tab.id, "content/content.css");

      await chrome.storage.local.get('accessToken').then((result) => {
        newToken = result.accessToken;
      });

      let response = await postAPI(url, data);

      if (response === "Success") {
        contentScriptJS(tab.id, "content/successContent.js");
      } else if (response === "BR002") {
        contentScriptJS(tab.id, "content/duplicatedScrap.js");
      } else if (response === "NF002" || response === "BR001") {
        googleLogin();
      } else {
        contentScriptJS(tab.id, "content/errorContent.js");
      }
    } else {
      googleLogin();
    }
  })
}
```

여러 함수들 중에서 `handleScrapOrHighlightResponse` 함수에 대한 테스트 코드를 작성해보겠습니다.

그러나, 다음과 같은 테스트 코드 작성시, chrome 모듈을 찾기 못하는 에러가 발생하였습니다.

<img width="563" alt="Pasted Graphic" src="https://github.com/HanDaYeon-coder/handayeon-coder.github.io/assets/75533232/c21593fa-ae0d-4962-8b97-f73519c06697">

에러가 발생하는 이유는 `chrome.local`같이 chrome은 크롬에서 제공해주는 API이기 때문입니다. 

따라서 테스트 코드 작성시에는 외부 API 호출과 같은 외부 의존성이 있는 Chrome API를 mock하여 의존성을 제거하기로 하였습니다.

---

### PART 2. chrome mockup npm library 설정
따라서, chrome 모듈을 mockup 해줄 수 있는 npm 라이브러리에 대해서 조사했습니다.

![image](https://github.com/HanDaYeon-coder/handayeon-coder.github.io/assets/75533232/4168c107-7311-4b5b-82fe-2cc5ed200ce2)

출처 : https://npmtrends.com/jest-chrome-vs-jest-webextension-mock-vs-sinon-chrome

위의 3개의 라이브러리 중에서 업데이트가 가장 최신이고, 다운로드 수가 높은 `jest-webextension-mock`와 `jest-chrome` 중 한 개로 선택하려고 했습니다.

첫번째로, `jest-chrome`는 타입 스크립트를 기반으로 작성되어 있어서, 현재 개발하고 있는 다담다 크롬 익스텐션은 타입 스크립트를 사용하고 있지 않아서, 예상치 못한 타입 및 설정에 대한 오류가 발생할 것이라고 생각했습니다.
또한, typescript를 다루어 본 적이 없어서, 테스트 코드 작성시에도 오랜 시간이 걸릴 것이라고 예상했습니다.

따라서, `jest-webextension-mock`를 사용하여, chrome mockup을 진행하기로 하였습니다.

1) jest-webextension-mock 설치하기
```
npm i --save-dev jest-webextension-mock
```

2) setup
- package.json에 아래의 문구를 추가하여,jest 사용시 jest-webextension-mock를 사용할 수 있도록 하였습니다.
  
```
"jest": {
  "setupFiles": [
    "jest-webextension-mock"
  ]
}
```

- 출처 : https://www.npmjs.com/package/jest-webextension-mock

---

### PART 3. handleScrapOrHighlightResponse 함수에 대한 테스트 코드 작성

간단하게 handleScrapOrHighlightResponse함수에서 chrome.storage.local.get가 정상적으로 호출되는지 확인해보는 테스트 코드를 작성했습니다.

```javascript
import { handleScrapOrHighlightResponse } from "./background/background";

const googleLogin = jest.fn();
const contentScriptJS = jest.fn();
const contentScriptCSS = jest.fn();
const postAPI = jest.fn();


describe('handleScrapOrHighlightResponse test', () => {
  it('handleScrapOrHighlightResponse', () => {
    const url = 'https://www.google.com';
    const tab = { id: 123 };
    const data = "data";

    handleScrapOrHighlightResponse(url, tab, data);
    expect(chrome.storage.local.get).toHaveBeenCalled();
  });
});
```
그러나, 아래의 사진과 같이 jest는 import module을 사용할 수 없다는 에러가 나왔습니다.

![image](https://github.com/HanDaYeon-coder/handayeon-coder.github.io/assets/75533232/ea0f1b34-139a-4c86-83a4-f022a14633ca)


따라서, 이번에 테스트 코드를 작성하는 목표가 chrome api가 mock이 정상적으로 되었는지 확인하기 위함이기 때문에,
import module을 사용하지 않고, 직접 함수를 추가하여 테스트를 진행하였습니다.

```javascript
// import { handleScrapOrHighlightResponse } from "./background/background";

const googleLogin = jest.fn();
const contentScriptJS = jest.fn();
const contentScriptCSS = jest.fn();
const postAPI = jest.fn();

function handleScrapOrHighlightResponse(url, tab, data) {
  chrome.storage.local.get('signedIn').then(async (result) => {
    if (result.signedIn) {
      contentScriptJS(tab.id, "content/content.js");
      contentScriptCSS(tab.id, "content/content.css");

      await chrome.storage.local.get('accessToken').then((result) => {
        newToken = result.accessToken;
      });

      let response = await postAPI(url, data);

      if (response === "Success") {
        contentScriptJS(tab.id, "content/successContent.js");
      } else if (response === "BR002") {
        contentScriptJS(tab.id, "content/duplicatedScrap.js");
      } else if (response === "NF002" || response === "BR001") {
        googleLogin();
      } else {
        contentScriptJS(tab.id, "content/errorContent.js");
      }
    } else {
      googleLogin();
    }
  })
}

describe('handleScrapOrHighlightResponse test', () => {
  it('handleScrapOrHighlightResponse', () => {
    const url = 'https://www.google.com';
    const tab = { id: 123 };
    const data = "data";

    handleScrapOrHighlightResponse(url, tab, data);
    expect(chrome.storage.local.get).toHaveBeenCalled();
  });
});
```

![image](https://github.com/HanDaYeon-coder/handayeon-coder.github.io/assets/75533232/de71ca59-98f6-457a-b83a-309e83e17204)

위의 사진처럼, 정상적으로 chrome api가 mock되어서 test 코드를 작성할 수 있게 되었습니다.

다음에는 import module 관련 문제를 해결하여 해당 함수를 작성하지 않아도 되도록 테스트 코드를 작성해보겠습니다!

