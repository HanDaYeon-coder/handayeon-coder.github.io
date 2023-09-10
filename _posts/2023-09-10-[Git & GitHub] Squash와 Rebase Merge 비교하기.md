---
title: "[Git & GitHub] GitHub의 Merge, Squash and merge, Rebase and merge에서 Merge 방식 선택하기"
date: 2023-09-10
categories: [Git, GitHub]
---

# 0️⃣ 서론

이전에 '*jacoco를 사용하여 test coverage report 적용하기*'라는 글의 제목으로 test coverage report 작성시 발생한 문제점 해결책으로 Pull Request를 보내는 구조를 변경한 적이 있습니다.

<a href="https://handayeon-coder.github.io/posts/jacoco를-사용하여-test-coverage-report-적용하기/"> https://handayeon-coder.github.io/posts/jacoco를-사용하여-test-coverage-report-적용하기/ </a>

<a href="https://velog.io/@da_na/TestCode-jacoco를-사용하여-test-coverage-report-적용하기"> https://velog.io/@da_na/TestCode-jacoco를-사용하여-test-coverage-report-적용하기 </a>

아래의 사진으로 이전 구조와 변경된 구조를 나타냈습니다.

![Alt text](/assets/img/2023-09-10/image.png)

즉, 1번의 Pull Request와 Merge 구조에서 2번의 Pull Request와 Merge 구조로 변경되었음을 알 수 있습니다.

따라서, 이러한 구조로 인해서 기존의 Git-Flow 구조에서 깔끔한 Develop Branch에서 Featrue Branch로 뻗어나가는 구조에서 Develop Branch에서 2개로 분기되는 구조로 Branch를 파악하기 어려운 구조가 되었습니다.

아래의 사진은 Upstream/develop 브랜치를 기준으로 본 Git  구조입니다.

- 이전의 Git-Flow의 Branch 구조

![Alt text](/assets/img/2023-09-10/image-5.png)

- 바뀐 후의 Git-Flow의 Branch 구조

![Alt text](/assets/img/2023-09-10/image-4.png)

아래의 사진으로  이러한 구조가 나오게 된 이유를 파악해보면, 2번의 Pull Request와 Merge 구조와 Merge의 방식 때문이었습니다.

![Alt text](/assets/img/2023-09-10/image-1.png)

- 1️⃣ Pull Request and Merge 사진
![Alt text](/assets/img/2023-09-10/image-2.png)

- 2️⃣ Pull Request and Merge 사진
![Alt text](/assets/img/2023-09-10/image-3.png)

Merge 방식은 pull Request를 Merge할 때 기본적으로 설정되어 있는 '**Create a merge commit**'이라는 것을 선택했습니다.

![Alt text](/assets/img/2023-09-10/image-6.png)

그러면 바뀐 후의 Git-Flow의 Branch 구조를 이전처럼 깔끔하고 파악하기 쉬운 Git-Flow Branch으로 변경하기 위해서는 어떻게 할까??

고민하다가, Merge 방식을 **Squash and merge**, **Rebase and merge**로 변경해보면 어떨까??라는 생각이 들어서, 해당 Squash and Merge와 Rebase and merge이 어떻게 다른지 파악해보고자 하였습니다.


---

#  1️⃣ 본론
## 🖍️ Merge 방식 비교하기
### 1. Create a merge commit
![Alt text](/assets/img/2023-09-10/image-7.png)

하단의 있는 브랜치를 Main 브랜치(파란색), 위의 있는 브랜치(주황색)를 feature 브랜치라고 하겠습니다.

featrue 브랜치의 변경 이력 전체(a, b, c)를 main 브랜치에 합칩니다.

이떄, m이라는 노드를 통해서 a + b + c가 main 브랜치에 추가됩니다.

### 2. Squash and merge
![Alt text](/assets/img/2023-09-10/image-8.png)

featrue 브랜치의 a + b + c 커밋들을 합쳐서 새로운 commit, abc를 만들어지고 main 브랜치에 추가됩니다.

feature 브랜치의 commit history를 합쳐서 깔끔하게 만들기 위해 사용합니다.

### 3. Rebase and merge
![Alt text](/assets/img/2023-09-10/image-9.png)
모든 commit들이 합쳐지지 않고 각각 master 브랜치에 추가됩니다.

각 commit은 모두 하나의 parent를 가지게 됩니다.

참고 자료 : https://im-developer.tistory.com/182

---

## 🔄 Merge 방식 변경하기
선택한 Merge 방식은 **Rebase and merge**로 하기로 하였습니다.

이전과 같은 Bracnh 구조를 하기 위해서는 모든 commit들을 합치지 않고 각각 브랜치를 추가한 구조기 때문에 Rebase and merge 구조가 적합하다고 생각했습니다.

- 이전 Branch 구조

![Alt text](/assets/img/2023-09-10/image-5.png)

- Rebase and merge를 사용했을 때의 Branch 구조

![Alt text](/assets/img/2023-09-10/image-10.png)

아래와 같이 2번의 Pull Request and Merge가 있었지만, 이전처럼 Branch 구조를 가짐을 알게 되었습니다.

![Alt text](/assets/img/2023-09-10/image-11.png)
![Alt text](/assets/img/2023-09-10/image-12.png)

---

# 3️⃣ 결론 

이번 계기를 통해서 더 자세하게 GitHub의 Merge 구조를 더 정확하게 파악해볼 수 있었습니다.

그리고 직접 적용해봄으로써 현재 더 나은 프로젝트에 필요한 방법에 대해서 생각해볼 수 있었습니다.

하지만, Git-flow를 따른다고 했을 때 유용한 구조는 아래와 같다고 합니다.

> develop - feature 브렌치간 머지 : Squash and Merge가 유용합니다. feature의 복잡하고 지저분한 커밋 히스토리를 모두 묶어 완전 새로운 커밋으로 develop 브렌치에 추가하여, develop 브렌치에서 독자적으로 관리할 수 있기 때문입니다. 일반적으로 머지 후에 feature 브렌치를 삭제해버리는 점을 떠올려 보면, feature 브렌치의 커밋 히스토리를 모두 develop 브렌치에 직접 연관 지어 남길 필요가 없습니다.

> master - develop 브렌치간 머지 : Rebase and Merge가 유용합니다. develop의 내용을 master에 추가할 때에는 별도의 새로운 커밋을 생성할 이유가 없기 때문입니다.

> hotfix - develop, hotfix - master 브렌치간 머지 : Merge 또는 Squash and Merge 모두 유용합니다. 때에 따라 골라 사용하면 좋을 것 같습니다. hotfix 브렌치 작업의 각 커밋 히스토리가 모두 남아야 하는 경우 Merge, 필요 없는 경우 Squash and Merge를 사용하면 됩니다.

출처 : https://meetup.nhncloud.com/posts/122

현재 저희 팀에서는 위의 방식으로 합의되지 않아서, 당장 위의 구조로 변경할 수는 없지만 앞으로 프로젝트에서는 위의 머지 방식을 적용하여 commit과 branch를 관리하여 Git-flow 방식을 더 효율적으로 사용해보고 싶다는 다짐을 하게 되었습니다.