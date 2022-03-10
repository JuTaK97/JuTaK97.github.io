---
title : "Chapter 14. The Preprocessor"
categories : 
  - C
tags :
  - C
last_modified_at: 2020-03-10T12:33:00-05:00
---
# K.N.KING - C PROGRAMMING *A Modern Approach*
## Chapter 14. The Preprocessor
### 14.1 How the Preprocessor Works
#### Macro definition<br />
```#define``` <br />
marco를 정의하기 위한 directive이다. Preprocessor는 \#define을 통해 정의된 macro의 이름(=syntax)과 정의(=semantic)를 저장하고, 프로그램에서 매크로가 사용될 때 정의된 값으로 대체한다. <br />
#### File inclusion<br />
```#include``` <br />
특정 파일의 content를 해당 프로그램에 포함시킨다. <br/>
- Conditional compilation<br />
```#if```, ```#ifdef```, ```ifndef```, ```#elif```, ```#else```<br />
특정 조건에 따라 코드의 각 부분들이 프로그램에서 포함될지 제외될지를 결정한다. preprocessing 단계에서 이루어진다.

### 14.2 Preprocessing Directives
#### Directive 작성의 몇 가지 rule들
1. Directive는 반드시 ```#```로 시작해야 한다. ```#``` 앞에 white space만 있다면 line의 시작이 꼭 아니어도 되지만, 
```#``` 뒤에 나오는 부분은 반드시 그 directive의 정보에 대한 것이어야 한다.
2. ```# define N 100```과 같이, 사이사이에 space나 tab으로 분리되어도 괜찮다.
3. Directive는 ```\n```이 나올 때 끝난다. 여러 줄에 걸쳐서 쓰고 싶으면. ```\```로 줄바꿈을 해서 쓴다.
4. Directive는 프로그램의 어느 부분에서든 나와도 된다. 꼭 맨 앞일 필요는 없다.
5. Directive 오른쪽에 주석은 써도 된다. 설명을 쓰는 것이 가독성에 좋다.


### 14.3 Macro Definition
#### Simple Macros
아무 파라미터 없이, 단순한 replacement를 위한 macro이다.<br />
``` C
#define identifier replacement-list 
```
특정 상수를 정의하거나, 자주 쓰이는 문구를 축약하는 데 좋다. 프로그램의 가독성을 높이는 데 도움이 되고, 프로그램 수정을 쉽게 만들어 주는 장점이 있다.
#### Parameterized Macros
``` C
#define identifier(x1, x2, ..., xn) replacement-list
```
마치 함수 같은 형태를 가지고 있다.<br />
Preprocessor는 프로그램에서 저 형태의 코드를 발견했을 때, 예를 들어 ```#define identifier(y1, y2, ..., yn)```이 있으면 각 y1 ~ yn을 replacement-list에 정의된
것들로 치환하게 된다.<br /><br />
(예시)
> ``` C
> #define MAX(x, y) ((x)>(y) ? (x) : (y))
> i = MAX(j+k, m-n);
> /* i = ((j+k)>(m-n) ? (j+k) : (m-n)) 과 동일 */
> ```

간편한 함수를 하나 만드는 것과 사실상 동일하다. <br /> <br />
parameter가 없는 parameterized macro도 만들 수 있다.
> ``` C
> #define getchar() getc(stdin)
> ```


#### parameterized macro의 장점
1. function에 비해 빠르다. Run-time overhead가 없기 때문이다.
2. Generic하다. 함수의 경우 argument의 타입이 정해져 있지만, macro는 그렇지 않다.
#### parameterized macro의 단점
1. 컴파일 후 source code가 아주 길어질 수 있다. 특히 nested되었을 경우 함수를 쓰는 것보다 훨씬 코드가 길어진다.
2. Argument의 type-check가 이루어지지 않는다. preprocessor가 잡아내지도 못하고, type conversion도 받지 못한다.
3. 함수 포인터와 다르게 macro는 포인터를 가질 수 없다.
4. 특정 상황에서 unexpected behavior를 보일 수 있는데 :
> ``` C
> #define MAX(x, y) ((x)>(y) ? (x) : (y))
> int n = MAX(i++, j); 
> ```
이런 코드의 경우, 함수로 구현되었을 때는 i++가 당연히 한 번만 수행되지만, 매크로로 정의하면 (x)가 등장하는 두 군데에서 모두 ++ 연산을 하게 된다.<br /> 

#### The # Operator



