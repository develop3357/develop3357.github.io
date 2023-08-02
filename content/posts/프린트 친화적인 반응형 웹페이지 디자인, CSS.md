---
title: "프린트 친화적인 반응형 웹페이지 디자인, CSS"
date: 2023-08-02T15:43:27+09:00
draft: false
---

아무래도 개발자인만큼 워드프로세서로 하나의 문서를 만드는 것보다는 데이터 따로, 템플릿 따로해서 결합하는 형태로 만드는 것을 선호한다. 또 그렇게 만들어둔 이력서 내지는 포트폴리오를 그대로 출력하거나 PDF로 제출하려고 하는 상황이 있을 수 있겠다.

이번에 이력서를 웹페이지로 작성하고, 이걸 PDF로 바로 출력해서 사용하려다가 마주한 몇 가지 문제를 해결한 방법을 서술한다.

## 웹페이지를 출력 했을 때 어색한 이유

주요 문제는 두 가지 정도 꼽을 수 있다.

1. 비율 문제: 글자가 너무 크거나 작음
2. 부자연스러운 페이지 분할

두 문제를 해결하면 위화감 없이 웹페이지 단순 출력이 아니라 출판물 같은 느낌을 낼 수 있다.

## 해결 방법

### 비율 문제 해결

```css
@page {
  size: A4;
  margin: 10mm;
}

@media print {
  .container {
    width: 1024px;
  }
}
```

경험적으로 위처럼 설정하는 것이 좋았다.

출력 페이지 마진을 CSS로 직접 관리할 수도 있지만, 아래에서 다룰 `break-inside` 속성을 사용해 페이지를 분할 할 경우 엘레멘트마다 일일히 여백을 관리하기에는 번거롭고 복잡해지기 때문이다.

`@media print`로 루트의 폭(width)을 설정할 때, 1024px 같은 데스크탑 스크린 기준 사이즈와 맞추는게 좋았다. 이유는 PDF로 출력할 경우 해상도가 높으면 더 깔끔하고, 출력만을 위한 별도의 사이즈를 따로 관리하지 않아도 되기 때문이다.

#### 참고사항

> `size`는 실제 출력 시 옵션에서 변경할 수도 있기 때문에 절대적인 의미를 갖지는 않는다.

> **A4 용지 기준 픽셀 크기**
>
> - 72 dpi (web) = 595 X 842 pixels
> - 300 dpi (print) = 2480 X 3508 pixels
> - 600 dpi (high quality print) = 4960 X 7016 pixels
>
> https://stackoverflow.com/questions/3341485/how-to-make-a-html-page-in-a4-paper-size-pages

### 페이지 분할 문제 해결

```css
.section {
  break-inside: avoid-page; /* this! */
}

.foo {
  break-before: page;
}

.bar {
  break-after: page;
}
```

`break-before`와 `break-after`는 엘레멘트 앞뒤에서 무조건 분할하도록 하는 속성이다. 가변 길이 문서에서 가장 유용한 것은 `break-inside: avoid-page;`이다. 엘레멘트가 너무 길어서 도중에 페이지 분할이 일어날 경우, 그렇게 두지 않고 페이지를 분할해서 다음 페이지 맨 위로 보낸다. 웹페이지의 주요 단락을 구성하는 엘레멘트마다 해당 속성을 설정해주면 출력물의 페이지 분할을 고려하지 않아도 별다른 문제가 생기지 않는다.

#### 참고사항

> `page-break-before`, vs `break-before`? → 전자가 후자로 교체되었다.

> **Safari** 브라우저는 break-inside 속성을 **지원하지 않는다**. 출력은 되도록 다른 브라우저를 사용하도록 하자. (https://developer.mozilla.org/en-US/docs/Web/CSS/break-inside)

### 출력할 영역만 스타일링

```css
@media print {
  /* 프린트되지 않도록 하는 클래스 */
  .no-print {
    display: none;
  }
}
```

`@media screen` 사용하듯이, `@media print`를 사용해 출력에만 영향을 주는 스타일을 `@media print` 아래로 설정한다.

### 관심 영역만 출력하기: 상위 요소를 배제

```css
@media print {
  .container {
    width: 1024px;
    margin: 0 auto;
  }

  /* (예시) container보다 상위 엘레멘트는 출력에 영향을 주지 않도록 속성을 재설정해준다. */
  html,
  body {
    width: auto;
    height: auto;
    margin: 0;
    padding: 0;
    background-color: transparent;
    border: none;
    box-shadow: none;
  }
}
```

`.container` 내부만 출력한다고 했을 때, 좌우 `margin`을 auto로 준다. 그러면 외부 엘레멘트가 존재해도 문서가 가운데로 오게 된다. 외부 엘레멘트에도 크기(width, height)나 여백(margin, padding) 등이 설정됐을 경우, 프린트 할 때는 출력되지 않도록 재설정해준다.

당연한 말이지만 만약 상위 요소에 따로 지정된 스타일이 없다면 위처럼 재설정하지 않아도 된다.

## 결론

출판물과는 다르게 웹페이지는 세로 길이가 가변적이기 때문에 각각의 엘레멘트의 길이에 따라 동적으로 페이지를 분할해주는 `break-inside` 속성을 유용하게 사용할 수 있다. 컨텐츠가 확정된 경우 페이지 분할을 더 엄격하게 관리하려면 `break-before`/`break-after` 속성을 사용하면 된다.

페이지 비율을 설정하기 위해 `@media print` 내에서 width를 설정해주고, 또 관심 영역만 출력되도록 상위 엘레멘트들의 CSS를 설정해주면 프린트 친화적인 반응형 웹페이지를 디자인했다고 할 수 있겠다.
