지난 1편에서 만들어진 문자열을 다시 자바스크립트 객체로 변환하는 작업을 해야합니다.

너무 중요해서 다시한번 이야기합니다. 파싱의 과정은 다음의 순서를 따릅니다.

1. 어휘 분석(Lexical Analyze): 텍스트를 분석해서 토큰으로 만들어냅니다.
2. 구문 분석(Syntax Analyze) : 생성된 토큰의 구조를 분석하여 컴퓨터가 이해할 수 있는 자료구조인 AST(Abstract Syntax Tree)를 만듭니다.
3. 작성된 AST를 기반으로 원하는 작업을 만듭니다.

파싱의 꽃, 컴파일러의 꽃, 어휘분석부터 진행해보겠습니다.

### 어휘 분석(Lexical Analyze)

JSON을 분석하기전에 가장 기본적인 JSON 타입을 몇개 살펴보겠습니다.

- NULL - `null`
- 불리언 - `true`
- 문자열 - `"abc"`
- 숫자 - `3.1415`
- 배열 - `["foo"]`
- 객체 - `{"foo":null}`

우리는 이 각각의 상태를 무엇을 기준으로 구분할까요. 불리언과 NULL은 쉽습니다. 그냥 텍스트로 `true`, `false`, `null`이 나오면 됩니다. 문자열은 처음 시작이 `"`로 됩니다. 배열은 `[`, 객체는 `{`으로 시작합니다. `true`, `false` 와 같은 텍스트와 `"`, `[`, `]` 등과 같은 문자 등 이렇게 언어를 구조화 할 수 있는 최소한의 단위를 토큰이라고 합니다.

지난 1편에 만들어진 JSON 구문을 봅시다.

```json
{
  "string": "foo",
  "number": 30,
  "number2": 3.14156,
  "true": true,
  "false": false,
  "null": null,
  "infinity": Infinity,
  "ninfinity": -Infinity,
  "nan": NaN,
  "re": /hello/gi,
  "array": [1,2,3],
  "object": {
    "foo": "hello"
  }
}
```

여기서 만들어낼 수 있는 토큰은 아마 다음과 같습니다.

```
T_STRING = /"(?:[^"\\]*|\\")*"/
T_NUMBER = /(-?[0-9]+(?:\.[0-9]*)?)/
T_TRUE = /true/i
T_FALSE = /false/i
T_NULL = /null/i
T_INFINITY = /Infinity/
T_NINFINITY = /-Infinity/
T_NAN = /NaN/
T_REGEXP = /\/((?:[^\/\\]*|\\\/)+)\/(igm|img|gim|gmi|mig|mgi|ig|im|gi|gm|mi|mg|i|g|m)?/
```

### 구문 분석(Syntax Analyze)

배열은 어떻게 생겼을까요? 배열은 3개의 특수문자(`[`, `,`, `]`)와 내부에는 다른 자료형들을 다시 열거합니다.
이러한 방식을 쉽게 표현하는 방법이 있습니다. LL파서 문법, LR 파서 문법, LALR 파서 문법 등 여러가지가 있습니다만, 어려운 내용이라 자세한 내용은 생략하겠습니다.

배열(`array`)은 빈 배열과, 비어있지 않은 배열 두가지 방식으로 표현됩니다. 그리고 비어있지 않은 배열에는 다른 값들(`values`)이 들어갑니다. 그 정의를 표현한게 다음 식입니다. `|` 문자는 "또는"을 의미합니다.

```
array: '[' ']' // 빈 배열
  | '[' values ']' // 배열 내에 다른 값들이 열거됨
```

이제 값들(`values`)의 형태를 정의합니다. 이 표현식에서 재귀를 통해 반복문을 표현할 수 있으며, 재귀를 통해 1개부터 n개의 값(`value`)으로 표현할 수 있으며, 다음과 같이 정의합니다. (뭔가 점화식 같기도 합니다.)

```
values: value
  | value ',' values // values를 재귀로 사용합니다.
```

값(`value`)은 위의 열거한 토큰들과 배열, 객체(`object`)입니다. 그리고 그 표현식은 다음과 같습니다. 정규표현식을 그대로 넣어도 됩니다.

```
value: T_STRING
  | T_NUMBER
  | T_TRUE
  | T_FALSE
  | T_NULL
  | T_INFINITY
  | T_NINFINITY
  | T_NAN
  | T_REGEXP
  | array
  | object
```

아직 객체(`object`)를 정의하지 않았습니다. 객체는 Key-Value로 구성되어있고, 키(Key)값은 `string`과 규칙을 같이합니다.

```
object: '{' '}'
  | '{' objectpairs '}'

objectpairs: objectpair
  | objectpairs ',' objectpair

objectpair: objectkey ':' value
objectkey: T_STRING
```

이제 이 내용을 그대로 코딩하면 됩니다. 보통 구문분석을 기반으로 파서를 만들어주는 라이브러리들이 있습니다. 보통  `lexer`, `yacc`으로 검색하면 관련 라이브러리가 나옵니다. 언어마다 구현체가 다양하게 존재합니다. 자바스크립트계열에는 다음 3개가 가장 유명합니다.

- Jison
- PEG.js
- Chevrotain

그리고 이 라이브러리들의 비교를 위한 벤치마크(Benchmark) 페이지는 [여기](https://sap.github.io/chevrotain/performance/)에서 볼 수 있습니다. 이런 손쉬운 라이브러리들이 있음에도 불구하고, 파싱과정을 자세히 이해하기 위해 바닐라(!?)로 작업하겠습니다.

처음에는 이렇게 작업하면서 파싱과정을 이해하고, 이런 과정이 또 필요하다면 위에 열거한 라이브러리를 이용해보는 것도 좋은 방법입니다.

기본적으로 파서는 주어진 문자열을 앞에서부터 읽으며 처리합니다. 그리고 문자열을 읽을 때, 구문분석을 진행해서 읽어들입니다. 그래서 토큰을 정의하는 일이 가장 중요합니다. 토큰은 숫자의 경우 `/(-?[0-9]+(?:\.[0-9]*)?)/`과 같이 정의했지만, 앞에서 부터 읽어들여야하기 때문에 이렇게 `/^(-?[0-9]+(?:\.[0-9]*)?)/` 변경합니다. 모든 토큰에 동일하게 적용됩니다. `^`를 집어넣습니다.

```javascript
const T_WHITESPACE = /^\s+/
const T_STRING = /^"(?:[^"\\]*|\\")*"/
const T_NUMBER = /^(-?[0-9]+(?:\.[0-9]*)?)/
const T_TRUE = /^true/i
const T_FALSE = /^false/i
const T_NULL = /^null/i
const T_INFINITY = /^Infinity/
const T_NINFINITY = /^-Infinity/
const T_NAN = /^NaN/
const T_REGEXP = /^\/((?:[^\/\\]*|\\\/)+)\/(igm|img|gim|gmi|mig|mgi|ig|im|gi|gm|mi|mg|i|g|m)?/
```

다음은 차례로 문자열을 읽어들일 전체 구성을 만들어 봅시다. 주석에 설명을 첨부하였습니다. :-)

```javascript
// buf는 처리하고 남은 구문 문자열입니다. 파서는 buf가 빈문자열("")이 될 때까지 반복합니다.
let buf = null
let match = null

// 구문분석 중, 공백 문자를 처리합니다.
const T_WHITESPACE = /^\s+/ // 공백 문자열에 대해 정의합니다.
function white() {
  while (1) {
    match = buf.match(T_WHITESPACE)
    if (!match) {
      return
    } else {
      buf = buf.slice(match[0].length)
    }
  }
}

// next함수는 buf를 읽어들이고 소비하는 함수입니다. slice를 이용합니다.
function next(cnt = 1) {
  buf = buf.slice(cnt)
  white()
}

// 그냥 에러 메시지 통일을 위한 에러를 생성하는 함수입니다. throw error() 처럼 사용합니다.
function error(expected) {
  throw new Error(`expected ${expected}, unexpected ${buf[0]}`)
}

// ctx는 해석해야하는 구문, 즉 stirng 입니다.
function parse(ctx) {
  buf = ctx // 함수가 실행되면 buf에 해석해야하는 구문 전체를 집어넣습니다.

  const result = // 뭔가 처리한 결과물을 result 변수에 넣습니다.

  if (buf.length) { // 모든 파싱이 종료된 후에 buf가 빈문자열이 아니라면 에러를 보여줍니다.
    throw error("EOF")
  }
  return result // 이상이 없으면 결과를 반환합니다.
}
```

위의 구조에서 result를 만들어야 합니다. 즉, 시작하는 지점이 있어야 합니다. 파서는 어디에서 시작할까요?

1. T_로 시작하는 토큰들
2. 가장 처음으로 정의한 array
3. 뭔가 많이 딸려있는 value
4. 가장 복잡한 object


두구두구두구... 정답은 3번 `value`입니다. 이유는 간단합니다. `"string"`, `30`과 같은 JSON도 존재합니다. 이런 다양한 타입을 대응하려면 `value`를 시작점으로 잡아야 합니다.

`value`부터 차근차근 코딩해나가봅시다.

```javascript
function parse(ctx) {
  buf = ctx

  const result = value() // 이렇게 넣어줍니다.

  if (buf.length) {
    throw error("EOF")
  }
  return result
}
```

`value` 함수는 다음과 같이 작성할 수 있습니다. 위에 파서문법과 매칭해서 보면 코드가 더 쉽게 이해됩니다.

```javascript
/*
value: T_STRING
  | T_NUMBER
  | T_TRUE
  | T_FALSE
  | T_NULL
  | T_INFINITY
  | T_NINFINITY
  | T_NAN
  | T_REGEXP
  | array
  | object
 */
function value() {
  white()
  match = buf.match(T_STRING)
  if (match) {
    const t = match[0] // t는 토큰의 약자입니다. :-)
    next(t.length) // 토큰을 소비합니다.
    return t.replace(/^"|"$/g, "").replace(/\\\"/g, "\"") // 토큰에서 실제 문자열로 정재합니다.
  }
  match = buf.match(T_NUMBER)
  if (match) {
    const t = match[0]
    next(t.length)
    return +t
  }
  match = buf.match(T_TRUE)
  if (match) {
    const t = match[0]
    next(t.length)
    return true
  }
  match = buf.match(T_FALSE)
  if (match) {
    const t = match[0]
    next(t.length)
    return false
  }
  match = buf.match(T_NULL)
  if (match) {
    const t = match[0]
    next(t.length)
    return null
  }
  match = buf.match(T_INFINITY)
  if (match) {
    const t = match[0]
    next(t.length)
    return Infinity
  }
  match = buf.match(T_NINFINITY)
  if (match) {
    const t = match[0]
    next(t.length)
    return -Infinity
  }
  match = buf.match(T_NAN)
  if (match) {
    const t = match[0]
    next(t.length)
    return NaN
  }
  match = buf.match(T_REGEXP)
  if (match) {
    const t = match[0]
    const r = match[1]
    const m = match[2]
    next(t.length)
    return new RegExp(r.replace(/\\\//g, "/"), m || undefined)
  }
  switch (buf[0]) { // 첫번째 문자열에 따라서 다음에 실행할 상태가 달라집니다.
    case "[":
     return array()
    case "{":
     return object()
  }
  error("value")
}
```

array는 다음과 같이 정의할 수 있습니다. `values` 구문은 `while`을 사용해서 생략되었습니다.

```javascript
/*
array: '[' ']'
  | '[' values ']'
values: value
  | value ',' values
*/
function array() {
  next() // resolve [
  if (buf[0] === "]") { // 빈 배열 처리
    next()
    return []
  }
  const result = [value()]
  white()
  while (buf[0] === ",") {
    next()
    result.push(value()) // value() 를 다시 재귀로 호출
    white()
  }
  if (buf[0] === "]") {
    next()
    return result
  }
  throw error("]") // 배열을 닫아야 하는데 닫는 문자열이 없는 경우 에러
}
```

마지막으로 object 입니다. 마찬가지로 objectpairs 부분은 while로 처리되었습니다.

```javascript
/*
object: '{' '}'
  | '{' objectpairs '}'

objectpairs: objectpair
  | objectpairs ',' objectpair

objectpair: objectkey ':' value
objectkey: T_STRING
*/
function object() {
  next()
  if (buf[0] === "}") { // 빈 객체
    next()
    return {}
  }
  const result = objectpair()
  white()
  while (buf[0] === ",") {
    next()
    Object.assign(result, objectpair())
    white()
  }
  if (buf[0] === "}") {
    next()
    return result
  }
  throw error("}")
}

function objectpair() {
  match = buf.match(T_STRING) // key 처리
  if (match) {
    const t = match[0]
    next(t.length)
    const key = t.replace(/^"|"$/g, "").replace(/\\\"/g, "\"")
    if (buf[0] === ":") {
      next()
      return {[key]: value()}
    }
    throw error(":")
  }
  throw error("object key")
}
```
