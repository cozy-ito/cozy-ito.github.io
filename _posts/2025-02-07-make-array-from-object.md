---
title: 리터럴 객체로 배열을 만들어 보겠습니다.
description: 객체로 배열을 만들어 본다니? 이건 못 참지.
author: 이토
date: 2025-02-07 13:32:00 +0900
categories: [개발, 자바스크립트]
tags: [자바스크립트, 객체, 유사배열, 이터러블, 이터레이터, 배열 최적화 전략]
pin: false
mermaid: true
writing_time: 287
image:
  path: assets/img/common/2025-02-07-make-array-from-object/make-array-thumbnail.png
  lqip: assets/img/common/2025-02-07-make-array-from-object/make-array-thumbnail.png
  alt: Object to Array Created by ChatGPT
---

## 서론

누군가 `fetch`를 이용해 비동기 요청을 보내는 코드를 보여주었습니다. <br>
`URL` 객체를 생성해 URL 파라미터를 추가하는 로직이 포함되어 있었는데요! <br>
그 코드는 아래와 같습니다.

```js
async function getColorSurveys(params = {}) {
  const url = new URL('https://example.com/api/');
  Object.keys(params).forEach((key) =>
    url.searchParams.append(key, params[key])
  );
  const res = await fetch(url);
  const data = await res.json();
  return data;
}
```

코드를 보고 `URLSearchParams` 객체에 대해 궁금해져 MDN 문서를 보게 되었고, 한 구절을 보게 됩니다. <br>
> `URLSearchParams`를 구현하는 객체는 `for...of` 반복문으로 직접 키/값 쌍을 순회할 수 있습니다. <br>

평소에는 `for...of` 반복문이 수행되면 대충 '유사배열인가?'라는 생각을 하고 넘어갔었습니다만, <br>
좀 더 자세히 알고 싶어 자세히 파헤쳐보기로 합니다! <br>
(조사하고 보니 유사배열에 적용되는 얘기가 아니었습니다. 😅)

이것저것 궁금증을 가지다 보니, **유사배열과 이터러블**, 더 나아가 **이터레이터**는 무엇인지, <br>
**`for...of` 반복문을 실행하기 위한 조건**은 어떤 것들이 있는지, <br>
배열도 객체인데 아예 **객체로 배열을 만들 수 있을지**까지 알아보게 됩니다.

이 글은 그 과정에 대한 내용입니다.

## 유사배열과 이터러블

유사배열과 이터러블은 제가 서로 혼동하고 있던 개념입니다. <br>
배열이 아닌데 `for...of` 반복문이 실행되는 객체를 보고 '유사배열이다.' 라는 잘못된 개념을 가지고 있었는데요. <br>
먼저 각각의 개념에 대해 살펴보겠습니다.

### 유사배열 (Array-like Object)

유사배열은 아래와 같은 특징을 가집니다.

1. `length` 속성을 가진다.
2. 인덱스를 통해 요소에 접근할 수 있다.
3. 진짜 배열은 아니므로, `map`, `forEach` 등의 배열 메서드를 사용할 수 없다.
4. 이터러블을 구현하지 않으면 `for...of`문을 사용할 수 없다.

위와 같은 특징을 적용한 객체를 아래와 같이 구현해 볼 수 있겠습니다!

```js
const arrayLike = {
  0: "apple",
  1: "banana",
  2: "cherry",
  length: 3
};

console.log(arrayLike[0]); // "apple"
console.log(arrayLike.length); // 3

// ❌ 배열이 아니므로 map() 사용 불가
console.log(typeof arrayLike.map); // undefined

// ❌ for...of 사용 불가 (이터러블이 아님)
for (const item of arrayLike) {
  console.log(item); // TypeError: arrayLike is not iterable
}
```

자바스크립트에서는 자체적으로 이러한 유사배열 객체를 활용하는 경우가 있는데요! <br>
함수 선언문의 내부에서 접근 가능한 `arguments` 속성, <br>
`document.getElements...` API에서 반환되는 `HTMLCollection` 객체 등이 있습니다.

### 이터러블 객체 (Iterable Object)

이터러블 객체는 아래와 같은 특징을 가집니다.

1. `for...of` 문을 사용할 수 있다.
2. 스프레드 연산자 사용이 가능하다.
3. `Array.from(obj)`을 사용하여 배열로 변환이 가능하다.
4. 반드시 `length` 속성이 필요하지 않다.
5. 인덱스를 통한 요소의 직접 접근이 불가능할 수 있다.

이터러블 객체 중 `Map`을 한번 사용해서 위 특징들이 모두 일치하는지 확인해 보겠습니다.

```js
const ito = new Map([
  ["name", "ito"],
  ["age", "secret"]
]);


// ✅ for ... of 반복문 사용 가능
for (const [key, value] of ito) {
  console.log(key, value); // "name ito", "age secret"
}

// ✅ 스프레드 연산자 사용 가능
console.log([...ito]); // [["name", "ito"], ["age", "secret"]]

// ✅ Array.from을 사용해 배열로 변환 가능
console.log(Array.isArray(ito)); // false
const arrayIto = Array.from(ito);
console.log(Array.isArray(arrayIto)); // true

// ❌ length 속성이 없음
console.log(typeof ito.length); // undefined

// ❌ 인덱스를 통한 요소 접근 불가
console.log(ito[0]); // undefined
```

살펴본 바와 같이 유사배열과 이터러블 객체는 서로 다른 개념이라는 것을 알 수 있습니다. <br>

조금 눈치가 빠르신 분들은 알아채셨을 텐데요! <br>
두 개념의 특징들을 자세히 살펴보면 **유사배열 객체의 특징**들은 **이터러블 객체에 속할 수** 있습니다. <br>
이터러블 객체는 **<code>length</code> 속성이 존재하고 인덱스를 통한 요소 접근이 가능해도 된다**고 볼 수 있습니다.

즉, **유사배열 객체이면서 이터러블 객체가 될 수 있다는 것**이죠! <br>
제가 두 개념을 혼동한 이유가 여기에 있었습니다. 😅


그런데 이터러블 객체가 되기 위해서는 근본적으로 아주 중요한 특징이 한 가지 더 있는데요! <br>
내부적으로 **이터레이터를 구현해야 한다는 것**입니다.

이터레이터가 무엇인지, 어떻게 구현해야 하는지 바로 알아보겠습니다.


## 이터레이터 (Iterator)

**이터레이터**란 **이터레이션 프로토콜을 구현하기 위한 함수**라고 볼 수 있습니다. <br>
이 이터레이션 프로토콜을 구현해야지만, `for...of` 반복문, 스프레드 연산자 등과 같은 동작을 수행할 수가 있는거죠. <br>
반대로 말하면 이터레이션 프로토콜을 구현하지 않으면 사용할 수 없다는 것입니다. 

어떻게 이터레이터를 사용해 이터레이션 프로토콜을 구현하고, <br>
이터러블 객체를 만들 수 있는지 코드로 살펴보겠습니다.

```js
const iterable = {
  data: ["apple", "banana", "cherry"],
  // 1️⃣ Symbol.iterator를 키로 가지는 함수를 구현합니다.
  [Symbol.iterator]() {
    let index = 0;
    // 2️⃣ Symbol.iterator 함수는 next 함수 속성을 가지는 객체를 반환합니다.
    return {
      // 3️⃣ next 함수는 value, done 속성을 가지는 객체를 반환합니다.
      next: () => ({
        value: this.data[index],
        done: index++ >= this.data.length,
      }),
    };
  }
};

// ✅ for...of 사용 가능
for (const item of iterable) {
  console.log(item); // "apple", "banana", "cherry"
}

// ✅ 스프레드 연산자 사용 가능
console.log([...iterable]); // ["apple", "banana", "cherry"]
```

`for...of` 반복문을 실행하면

1. `next` 함수가 반복적으로 실행되고
2. `done`이 `true`가 될 때까지
3. `next` 함수에서 반환된 객체의 `value`를 반환한다.

고 볼 수 있겠습니다.

<br>

그럼 앞선 설명에서 얘기했던 것 처럼 <br>
**유사배열 객체이면서 동시에 이터러블 객체**를 한번 구현해 보겠습니다.

```js
const arrayLikeIterable = {
  0: "apple",
  1: "banana",
  2: "cherry",
  length: 3,
  [Symbol.iterator]() {
    let index = 0;
    return {
      next: () => ({
        value: this[index],
        done: index++ >= this.length,
      })
    }
  }
};

// ✅ 유사배열이었었지만 이터레이터를 구현해 이터러블 객체로 만들어 for...of 반복문이 사용 가능해 졌다!
for(const fruit of arrayLikeIterable) {
  console.log(fruit); // "apple", "banana", "cherry"
}
```

이렇게 유사배열과 이터러블 객체, 더 나아가 이터레이터까지 살펴보았습니다. <br>
이제는 `for...of` 반복문 동작을 수행하는 객체를 보고도 유사배열일 것이라는 착각을 하지 않을 것 같습니다 하하. 😄

그런데 저의 궁금증은 여기서 멈추지 않았죠. <br>
자바스크립트 코드를 작성하면서 리터럴 방식(`[ ... ]`)으로 편리하게 생성하고, <br>
배열(`Array`)의 프로토타입 메서드(`map`, `filter` 등)까지 잘 활용하는 그 배열! <br>
'이 배열을 직접 객체로 만들어볼 수는 없을까?' 라는 생각을 합니다.

그래서 이 글의 제목처럼 객체로 배열을 만들어 보려고 해봤습니다. 😆

## 객체로 배열(Array) 만들기

우선 배열의 특징에 대해서 살펴보겠습니다.

1. 인덱스를 통한 요소의 접근이 가능하다.
2. 이터러블 객체이다.
3. `length` 속성을 가진다. (이 `length` 속성은 요소가 추가되거나 삭제될 때마다 자동으로 갱신되어야 한다.)
4. `map`, `forEach`와 같은 배열 메서드 사용이 가능하다.
5. `Array.isArray`의 결과값으로 `true`를 반환한다.

`Array` 내장 객체나 리터럴 배열(`[]`)을 전혀 쓰지 않고! <br>
리터럴 객체에 이 배열의 특징들을 적용해보겠습니다.

우선 앞선 예제들로 1, 2번 특징은 적용이 되었으니 3, 4번 특징부터 살펴보겠습니다. <br>
3번 특징인 `length`의 자동 갱신은 <br>
4번 특징인 배열 메서드가 실행(`예: push`)될 때 반영하면 될 것 같습니다.

```js
const arrayLikeIterable = {
  0: "apple",
  1: "banana",
  2: "cherry",
  length: 3,
  // ✅ push 메서드 구현
  push(el) {
    const nextIndex = this.length;
    this[nextIndex] = el;
    this.length++;
    return this;
  },
  [Symbol.iterator]() {
    // ...
  }
};

arrayLikeIterable.push('lemon');
// ✅ {0: 'apple', 1: 'banana', 2: 'cherry', 3: 'lemon', length: 4, push: ƒ, Symbol(Symbol.iterator): ƒ}
```

`push` 메서드를 실행하여 `"lemon"`이 3번 인덱스에 잘 추가되었고 (4번 특징) <br>
`length` 속성도 1만큼 잘 늘어났네요! (3번 특징)

4번 특징을 위해 모든 배열 메서드를 구현하기는 글이 너무 길어지기 때문에 <br>
`forEach` 메서드까지만 한 번 구현해보죠!

```js
const arrayLikeIterable = {
  0: "apple",
  1: "banana",
  2: "cherry",
  length: 3,
  push(el) {
    // ...
  },
  // ✅ forEach 메서드 구현
  forEach(callback) {
    for(const el of this) {
      callback(el);
    }
  },
  [Symbol.iterator]() {
    // ...
  }
};

arrayLikeIterable.forEach((el) => {
    console.log(el); // ✅ "apple", "banana", "cherry"
})
```

`forEach` 메서드가 의도한대로 잘 작동하는 것을 알 수 있습니다! <br>
이제 거의 다 되었네요!! 😆

 <br>

자, 이제 마지막 5번 특징(5. `Array.isArray`의 결과값으로 `true`를 반환한다.)을 살펴보겠습니다. <br>
당연히 현재의 객체를 전달하면 `false`를 반환합니다. <br>
**`Array.isArray`는 어떤 값을 기준으로 배열인지 아닌지를 판별하는 걸까요?**

당장 생각했을 때, 자바스크립트는 프로토타입 기반 언어이니 '프로토타입을 참조하지 않을까' 생각해봤습니다. <br>
`Array` 내장 객체를 쓰지 않기로 했지만, 다른 방법이 없어 적용해 보겠습니다.

```js
Object.setPrototypeOf(arrayLikeIterable, Array.prototype);
console.log(Array.isArray(arrayLikeIterable)); // ?
```

결과는...!? <br>
아쉽게도 `false`가 반환됩니다. 객체의 프로토타입으로 판별하지는 않는군요. <br>

그럼 어떻게 해결할 수 있을까요? <br>

<br>

...답은 **"방법이 없다"** 입니다. 😅

기대하신 분들에게는 실망을 드려 죄송하지만 <br>
확인한 바로는 객체를 완전히 배열로 변경할 수는 없습니다.

객체를 배열로 변경할 수 있는 가장 확실하고 빠른 방법은 <br>
**`Array.from` 메서드를 사용**하거나

```js
const array = Array.from(arrayLikeIterable);
console.log(Array.isArray(array)); // true
```

**스프레드 연산자를 사용해 배열에 재 할당**을 하거나

```js
const array = [...arrayLikeIterable];
console.log(Array.isArray(array)); // true
```

정도 밖에 없습니다.

`Array.isArray` 메서드는 전달받은 객체를 <br>
**"Array exotic object"인지 여부를 기준으로 판별**합니다. <br>

"Array exotic object"는 <br>
`length` 속성을 자동으로 관리하고, 인덱스 속성과 `length` 속성이 연동되는 등 <br>
`[[DefineOwnProperty]]` 내부 슬롯의 메서드 동작이 일반 객체와는 다르기 때문에 판별될 수 있습니다.

사실 앞서 말씀드린 특징들 말고도 배열에는 더 다양한 특징 들이 있습니다. <br>
인덱스로 접근하여 요소를 할당하여도 `length` 속성의 값이 증가하거나 <br>
`delete`로 배열의 요소를 삭제하면 사리지는 것이 아니라 `empty` 상태로 남게 됩니다.


## 정리

오늘은 유사배열과 이터러블, 이터레이터가 무엇인지 알아보았습니다. <br>
덕분에 스스로 혼동하고 있던 개념을 확실히 잡고 갈 수 있는 기회가 되었습니다.

그리고 직접 이터러블 객체로 만들어 봄으로써 <br>
`for...of`나 스프레드 연산자 등이 이터레이션 프로토콜을 구현된 객체들을 대상으로 동작한다는 것을 알게 되었습니다.

또 객체를 이용해 배열을 만드는 시도를 해봤습니다. <br>
완전히 배열로 탈바꿈시키는 동작은 구현할 수 없었지만 <br>
배열이 되기 위한 조건들에 대해서 좀 더 구체적으로 확인해 보는 시간이었습니다.

이상 긴 글 봐주셔서 감사합니다!


## Reference
- [MDN - Iteration 프로토콜](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Iteration_protocols#iterable)
- [MDN - `Array.isArray()`](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray)
- [Witch-Work - JS 탐구생활 - exotic object](https://witch.work/ko/posts/javascript-exotic-object?utm_source=chatgpt.com)