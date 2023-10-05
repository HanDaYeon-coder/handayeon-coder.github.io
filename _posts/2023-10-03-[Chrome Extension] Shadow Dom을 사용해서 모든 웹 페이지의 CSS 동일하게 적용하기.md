---
title: "[Chrome Extension] Shadow Dom을 사용해서 모든 웹 페이지의 injected CSS 동일하게 적용하는 방법"
date: 2023-10-03
categories: [Chrome Extension, Shadow Dom]
---

# 0️⃣ 서론

이전에 저희 프로젝트의 크롬 확장 프로그램을 개발한 글을 작성한 적이 있습니다!

https://handayeon-coder.github.io/posts/크롬-익스텐션-다담다-확장-프로그램-개발-회고/

저희의 다담다 크롬 확장 프로그램을 누르면, 해당 웹 페이지에 HTML 와 CSS에 같이 들어가게 됩니다.

즉, 아래의 크롬 확장 프로그램의 구조를 살펴보면 contentscript.js를 통해서 웹 페이지에 DOM에 접근할 수 있게 되고, DOM에 원하는 HTML과 CSS을 삽입하여 사용자가 보고 있는 웹 페이지에서 확인할 수 있게 됩니다.

![Alt text](/assets/img/2023-10-03/image-1.png)

사진 출처 : https://developer.chrome.com/docs/extensions/mv2/architecture-overview/

![Alt text](/assets/img/2023-10-03/image-2.png)

사진 출처 : https://developer.chrome.com/docs/extensions/mv3/devtools/

저희 서비스는 스크랩하고 싶은 웹 페이지에서 크롬 확장 프로그램을 누르게 된다면, 아래의 사진과 같이 스크랩이 저장되고 있는 화면과 성공, 실패 여부를 알려주는 화면을 확장 프로그램을 누른 웹 페이지에 보여주게 됩니다.

![Alt text](/assets/img/2023-10-03/image.png)

그러나, 아래의 사진과 같이 원래 의도한 CSS가 제대로 적용되지 않는 모습을 확인할 수 있습니다.

![Alt text](/assets/img/2023-10-03/image-3.png)

![Alt text](/assets/img/2023-10-03/image-4.png)

이러한 원인이 무엇인지, 그리고 이를 해결하기 위해서 어떻게 해야할지를 이야기해보겠습니다.

---

# 1️⃣ 본론

## 1. 현재 코드 살펴보기
아래의 코드는 background.js로 크롬 익스텐션에서 특정한 웹 사이트 tabId를 통해서 해당 페이지에 JS와 CSS를 넣어주게 되는 로직입니다.

### background.js 파일 일부분

```js
function contentScriptJS(tabId, file) {
  chrome.scripting.executeScript({
    target: { tabId: tabId },
    files: [`${file}`]
  });
}

function contentScriptCSS(tabId, file) {
  chrome.scripting.insertCSS({
    target: { tabId: tabId },
    files: [`${file}`]
  })
}

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

### content.css

```css
@import url(https://cdn.jsdelivr.net/gh/moonspam/NanumSquare@2.0/nanumsquare.css);

#dadamda-popup{
  box-sizing: border-box;
  font-family: 'NanumSquare', sans-serif;
  z-index: 100001212 !important;
  position: fixed !important;
  left: 80%;
  margin: 10px 0 20px 0;
  background: #fff;
  padding: 25px;
  border-radius: 15px;
  max-width: 380px;
  width: 100%;
  box-shadow: 0 10px 15px rgba(0,0,0,0.1);
  top: 80px;
  opacity: 1;
  pointer-events: auto;
  transform:translate(-50%, -50%) scale(1);
}

#dadamda-icon {
  width: 13%;
}

#dadamda-popup :is(header){
  display: flex;
  align-items: center;
  justify-content: space-between;
}

#dadamda-popup header{
  padding-bottom: 15px;
}

#dadamda-title{
  font-size: 21px;
  font-weight: 700;
  color: #475467;
}

#dadamda-popup #dadamda-content{
  margin: 20px 0;
}

#dadamda-text1 {
  font-size: 16px;
  font-weight: 300;
  color: #98A2B3;
}

#dadamda-loader {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

#dadamda-progress-bar {
  position: relative;
  width: 100%;
  height: 5px;
  border-radius: 100px;
  background-color: #F8FAFC;
  overflow: hidden;
}

#dadamda-progress-bar-gauge {
  position: absolute;
  height: 5px;
  border-radius: 15px;
  background-color: #B2CCFF;
  animation-name: loading-bar;
  animation-duration: 15s;
  animation-iteration-count: infinite;
  animation-timing-function: ease-out;
}

@keyframes loading-bar {
  0% {
    width: 0;
    opacity: 1;
  }
  80% {
    width: 100%;
    opacity: 1;
  }
  100% {
    width: 100%;
    opacity: 0;
  }
}
```
더 자세한 코드는 https://github.com/SWM-team-forever/dadamda-chrome-extension 에서 확인할 수 있습니다.

아래의 사진과 같이 똑같은 content.js와 successContent.js를 웹 페이지에 삽입하더라도 웹 페이지 별로 적용되는 CSS가 다르기 때문에, 제가 크롬 확장 프로그램을 통해서 넣은 CSS와 HTMl이 작용되지 않습니다.

![Alt text](/assets/img/2023-10-03/image-7.png)

![Alt text](/assets/img/2023-10-03/image-10.png)


![Alt text](/assets/img/2023-10-03/image-8.png)

![Alt text](/assets/img/2023-10-03/image-9.png)

즉, 해당 웹 페이지의 CSS의 영향을 받지 않도록 해야 합니다.

---

## 2. 해결 방법 검토하기

해당 웹 페이지에 적용된 기존 CSS의 영향을 받지 않도록 하기 위해서 해결 방법을 검토했습니다.

즉, 기존의 CSS와 새롭게 적용할 CSS 스타일 충돌을 방지해야 했습니다.

따라서 아래의 3가지의 방법 중 한 가지를 사용하기로 하였습니다.

1. 문제가 되는 부분을 !important로 처리하기
2. 기존 CSS 스타일 초기화하기
3. Shadow DOM 사용하기

### 2-1. 문제가 되는 부분을 !important로 처리하기

문제가 되는 부분을 !important하는 것은 어떤 웹 페이지의 CSS와 충돌날지를 모르기 때문에 제가 작성한 크롬 익스텐션의 CSS 모든 부분을 !important를 적용시켜야 했습니다.

그리고 나중에 다른 스타일을 적용하는 경우 우선순위를 정하기 어렵다는 생각을 했습니다.

따라서 모든 부분의 공통적으로 적용되어야 하는 부분인 위치 정보와 관련된 #dadamda-popup의 z-index와 position에서만 !important를 적용시켰습니다.

### 2-2. 기존 CSS 스타일 초기화하기

CSS 스타일 초기화는 브라우저마다 기본으로 제공하는 스타일을 기본값으로 설정해주는 세팅입니다.

따라서 크로스 브라우징 문제를 해결하기 위해서 자주 사용되는 방법입니다.

https://stackoverflow.com/questions/10608924/how-can-i-efficiently-overwrite-css-with-a-content-script

![Alt text](/assets/img/2023-10-03/image-11.png)

```css
html, body, div, span, applet, object, iframe,
h1, h2, h3, h4, h5, h6, p, blockquote, pre,
a, abbr, acronym, address, big, cite, code,
del, dfn, em, img, ins, kbd, q, s, samp,
small, strike, strong, sub, sup, tt, var,
b, u, i, center,
dl, dt, dd, ol, ul, li,
fieldset, form, label, legend,
table, caption, tbody, tfoot, thead, tr, th, td,
article, aside, canvas, details, embed, 
figure, figcaption, footer, header, hgroup, 
menu, nav, output, ruby, section, summary,
time, mark, audio, video {
    margin: 0;
    padding: 0;
    border: 0;
    font-size: 100%;
    font: inherit;
    vertical-align: baseline;
}
/* HTML5 display-role reset for older browsers */
article, aside, details, figcaption, figure, 
footer, header, hgroup, menu, nav, section {
    display: block;
}
```
그러나, 동일하게 injected stylesheet보다 우선순위가 높은 CSS가 적용되기 때문에 CSS 스타일이 초기화되지 않고 그대로 CSS가 이상하게 적용되는 모습을 볼 수 있습니다.

더 나아가서, CSS 스타일을 초기화하면 기존의 CSS 스타일이 초기화되어 웹 페이지의 모든 CSS 스타일에 영향을 줄 수도 있어서, 해당 방법은 사용하지 않는 것이 좋다고 판단하였습니다!

![Alt text](/assets/img/2023-10-03/image-12.png)

위의 사진에서 중간에 회색의 선을 보면, 의도한 CSS가 적용되지 않음을 알 수 있습니다.

출처 : https://inpa.tistory.com/entry/CSS-📚-RESET-스타일-초기화-🔨

### 2-3. Shadow DOM 사용하기

크롬 익스텐션을 사용한 CSS를 초기화하는 방법만 찾아보다가, isolate라는 단어를 보게 되었습니다!!

따라서, 이처럼 CSS를 기존의 CSS에도 영향을 주지 않도 나만의 HTML에 CSS를 적용시키는 독립(격리)시켜야겠다고 생각했습니다.

아래의 참고 자료를 통해서 shadow DOM을 사용해야겠다고 생각했습니다.

https://stackoverflow.com/questions/12783217/how-to-really-isolate-stylesheets-in-the-google-chrome-extension

### Shadow DOM이란?
![Alt text](/assets/img/2023-10-03/image-13.png)

- 웹 컴포넌트는 재사용할 수 있는 커스텀 HTML element를 생성하고, 해당 요소를 캡슐화하는 기술입니다.
- 캡슐화를 통해 마크업, 스타일, 동작을 외부로부터 격리하여, 웹페이지의 다른 구성 요소의 간섭을 방지할 수 있게 도와줍니다.

따라서 저의 크롬 익스텐션을 통해서 넣은 CSS와 HTML element를 Shadow Tree를 통해서 분리시켜서 외부로부터 격리하고 캡슐화를 하면 외부의 CSS의 간섭을 방지할 수 있다고 판단하였습니다.

---

## 3. Shadow DOM 적용하기

![Alt text](/assets/img/2023-10-03/image-16.png)

먼저, shadow DOM을 붙일 Shadow host(Shadow DOM이 붙어 있는 일반적인 DOM 노드)를 선언해주고 일반적인 DOM에 해당 Shadow host를 child로 추가해줍니다.

```js
// 여기에서 shadowDiv는 shadow host입니다.
var shadowDiv = document.createElement("div"); 
shadowDiv.setAttribute("id", "dadamda-shadow");

document.body.appendChild(shadowDiv);
```

shadowRoot를 생성해줍니다. 
```js
var shadowRoot = shadowDiv.attachShadow({ mode: 'open' });
```

그리고 해당 shadowRoot 안에 독립될 element들(shadow tree)을 넣어줍니다.

저희는 CSS와 dadamda-popup 창을 독립시킬 예정이라서 shadowStyle이라는 CSS 스타일 코드를 만들어서 shadowRoot의 child로 넣어줍니다.

```js
var shadowStyle = document.createElement('style');
shadowStyle.textContent = ``; // 실제 위의 content.css를 넣어주면 됩니다. 
shadowRoot.appendChild(shadowStyle);
```

그리고 dadamda-popup의 div를 생성하여 shadowRoot의 child로 추가해주었습니다.

```js
var popupDiv = document.createElement("div");
popupDiv.setAttribute("id", "dadamda-popup");
shadowRoot.appendChild(popupDiv);
```
따라서 아래와 같이 정상적으로 원하는 CSS만 적용되는 모습을 확인할 수 있었습니다.

![Alt text](/assets/img/2023-10-03/image-14.png)

![Alt text](/assets/img/2023-10-03/image-15.png)

참고 자료 
- https://tech.inflab.com/202208-shadow-root/
- https://leeproblog.tistory.com/185

---

# 2️⃣ 결론

- 이전까지는 웹 페이지마다 다르게 적용되어서, 어디에서 오류가 날지를 예측하지 못한 채로 크롬 확장 프로그램을 사용하면서 불안감을 느꼈습니다. 그러나, 이번에 드디어 모든 웹 페이지에서 동일하게 적용되어 크롬 확장 프로그램을 안정적으로 운영할 수 있게 되어서 정말 뿌듯한 시간이었습니다.
- shadow DOM이라는 새로운 용어와 DOM의 기술을 살펴보면서 새로운 웹 기술에 대해서 알 수 있게 되는 시간이었습니다.
- 이번에 CSS 스타일을 분리하자!!라는 생각으로 구글에 검색했는데, 원하는 내용을 얻기까지 많은 시간이 걸렸고, isolate라는 단어만 검색하면 바로 원하는 답을 얻을 수 있었다는 것을 알고 나서, 검색의 중요성을 다시 한 번 알게 되었습니다. 앞으로는 알고 싶은 내용을 검색하기 전에 어떠한 단어를 선정하여 검색할 지를 신중하게 선택해야겠습니다!