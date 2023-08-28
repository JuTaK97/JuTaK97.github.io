---
title : "코틀린 문법 기초 (과제 0-2)"
categories : 
  - Kotlin
---

### Kotlin 파일 실행하기
안드로이드 스튜디오(이하 안스)를 설치하셨으면 kt 파일을 안스에서 실행할 수 있습니다.
적당히 MainActivity.kt 옆에 Main.kt 파일을 만들어 주고 
```kotlin
fun main() {
  println("Hello World!")
}
```
를 입력해 주시면 라인 번호 옆에 초록색 재생 버튼이 생깁니다. 누르시면 main 함수를 실행할 수 있습니다.

더욱 가볍게 코틀린을 실행해 보고 싶으시면, 웹 사이트 [https://play.kotlinlang.org/](https://play.kotlinlang.org/)에서도 코틀린 파일을 실행할 수 있습니다.(그런데 자동완성이 안 돼서 불편합니다)

## 코틀린 기초 문법
1. [https://kotlinlang.org/docs/kotlin-tour-hello-world.html](https://kotlinlang.org/docs/kotlin-tour-hello-world.html)
위 사이트에서 1번(Hello World) 부터 7번(Null Safety)까지 진행해 주시면 됩니다.

2. [https://play.kotlinlang.org/byExample/01_introduction/01_Hello%20world](https://play.kotlinlang.org/byExample/01_introduction/01_Hello%20world) 에서 아래 부분을 제외하고 읽어 보시는 것을 추천드립니다.

    지금 당장은 안 읽어도 되는 부분 :
    - Introduction 챕터: Generics
    - Special Classes 챕터 : Object Keyword
    - Functional 챕터: 이해가 가지 않기 시작하기 전까지
    - Scope Functions 챕터 : let 외 나머지
    - Delegation 챕터 아래부터 쭉 (Kotlin/JS 챕터까지)


## 과제: 코테 문제 Kotlin으로 다시 풀기

### 1. SNUTT 강의 찾기
#### 풀이 조건
1. data class 사용하기
2. filter 혹은 filterNot 사용하기
4. sortBy 사용하기
5. 1번에서 만든 data class의 extension function 사용하기 (두 강의가 서로 겹치는지 안겹치는지 반환하는 함수)

테스트케이스는 아래 제공드리는 것만 맞으면 됩니다.

<details>
<summary>테케 1</summary>
<div markdown="1">
```
input

3 3
1 1 2 3
3 1 4 6
10 2 3 7
7 3 3 5
6 3 3 6
5 3 3 7

output

7
6
5
```
</div>
</details>


<details>
<summary>테케 2</summary>
<div markdown="1">
```
input

0 3
1 1 3 5
3 1 6 8
5 4 3 9

output

1
3
5
```
</div>
</details>

<details>
<summary>테케 3</summary>
<div markdown="1">

```
input

3 0
1 1 2 3
3 1 4 6
5 2 3 7

output

0
```
</div>
</details>

<details>
<summary>테케 4</summary>
<div markdown="1">
```
input

4 5
10 1 2 3
9 1 4 6
11 2 3 7
13 4 3 10
17 1 2 3
1 1 4 6
7 2 3 7
6 4 3 10
5 5 1 9

output

5
```
</div>
</details>

<details>
<summary>테케 5</summary>
<div markdown="1">
```
input

1 3
5 1 3 5
4 1 6 8
2 1 6 8
1 4 3 9

output

2
4
1
```
</div>
</details>

### 2. 쌍포는 와플도 녹인다 문제
걱정 마세요! 아래 스켈레톤 코드를 바탕으로 TODO 부분만 완성해 주시면 됩니다.
참고로 **장군과 궁성(대각선 경로들)은 전부 고려하지 말고**, 오직 이동 가능 여부만 고려하시면 됩니다. 

#### 풀이 조건
1. sealed class를 사용한다.
2. nullable type을 사용한다. (+ 다양한 [null 관련 문법들](https://play.kotlinlang.org/byExample/01_introduction/04_Null%20Safety)) 

<details>
<summary>스켈레톤 코드</summary>
<div markdown="1">

```kotlin
data class Point(
    val x: Int,
    val y: Int,
)

sealed class Piece(open val pos: Point, open val team: Boolean) { // team이 true이면 우리 편 기물
    data class Pho(override val pos: Point, override val team: Boolean) : Piece(pos, team)
    // TODO
    // data class Cha( ... )
    // data class Jol( ... )
    // data class King( ... )
}

fun canPhoMoveTo(board: Array<Array<Piece?>>, next: Point): Boolean {
    // TODO : board가 주어졌을 때, next 위치로 내 포가 이동할 수 있는지 없는지 반환
    return true
}
```
</div>
</details>