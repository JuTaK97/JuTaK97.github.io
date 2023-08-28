---
title : "안드로이드 스튜디오 설치 (과제 0-1)"
categories : 
  - Android
---

## 안드로이드 스튜디오 설치하기

현재 공식 버전은 [Giraffe](https://developer.android.com/studio)입니다. 이걸 다운받으셔도 되고, 조금 더 최신 버전을 받고 싶으시다면 [Preview 버전(Hedgehog)](https://developer.android.com/studio/preview)를 다운받아 주세요.
위 링크에서 Beta build (Hedgehog)를 설치하시면 됩니다.
<img src="https://github.com/JuTaK97/JuTaK97.github.io/assets/88367636/208c11f8-a5de-4a98-9fbb-43b1e7d1956b" width=800 />

- Preview 안내 글 : [https://developer.android.com/studio/preview/install-preview?hl=ko](https://developer.android.com/studio/preview/install-preview?hl=ko)

## 첫 프로젝트 만들기
1. New Project를 눌러 줍니다. ![2](https://github.com/JuTaK97/JuTaK97.github.io/assets/88367636/760691aa-ccbe-44a1-8a0e-0d6863bd15e6)

2. 선택지가 참 많은데, Empty Views Activity를 골라 줍니다(지금은 아직 보라색 고르시면 안 됩니다!). Activity가 뭔지는 모르겠지만 우선 넘어갑시다. ![3](https://github.com/JuTaK97/JuTaK97.github.io/assets/88367636/09958509-3ab4-4203-83d0-aa329da901d1)

3. 적당히 프로젝트의 이름을 적어 주고 finish를 눌러 줍니다.![4](https://github.com/JuTaK97/JuTaK97.github.io/assets/88367636/b515ef4d-5114-436d-a6f2-03cfbb24769d)
![5](https://github.com/JuTaK97/JuTaK97.github.io/assets/88367636/2c1bb76f-4736-4ddd-bc71-fd6344b36942)

4. Gradle project sync in progress... 알림이 사라질 때까지 대기합니다(우하단 프로그레스 바가 없어지면 됩니다). 꽤 오래 걸립니다. (제 똥컴 기준으로 약 15분 소요) ![6](https://github.com/JuTaK97/JuTaK97.github.io/assets/88367636/a64e0182-7a6a-4c8d-8e7d-dc098f5b2fe7)

5. 최근에 갑자기 UI를 vscode느낌으로 바꿔서 제공하는데, 여러분과 저의 화면이 동일하면 좋으니 Switch to Classic UI를 눌러 줍시다.![7](https://github.com/JuTaK97/JuTaK97.github.io/assets/88367636/a8317647-f968-4713-8f6c-0cd726898e5d)

6. MainActivity.kt라는 것이 생겨 있지만, 일단 신경쓰지 않고 좌상단의 초록색 재생 버튼을 누릅니다.
7. 윈도우 컴퓨터라면 높은 확률로 팬이 돌아가기 시작하고... 조금 기다리면 오른쪽에 휴대폰 화면이 등장합니다. Hello World! 라는 글씨가 보이면 완료입니다. 이 상태로 첫 번째 세미나에 오시면 됩니다!![9](https://github.com/JuTaK97/JuTaK97.github.io/assets/88367636/a9e47253-f143-49e2-a88f-5949f90c1e52)


## 그 외에 깔면 좋은 프로그램들

- Git 관리 프로그램
    - 어디까지나 본인의 취향입니다. 그냥 전부 터미널로 하셔도 됩니다.
    - Android Studio에서 Git 관련 GUI들을 제공해 줍니다. 이걸 사용하셔도 됩니다.
    - Github Desktop과 같은 프로그램을 쓰셔도 됩니다. 저는 이걸 씁니다.
    - 하지만 git 커맨드는 반드시 알아야 됩니다!
- Flipper
    - 디버깅 프로그램
    - 앱의 레이아웃 구조를 아주 쉽게 볼 수 있다.
    - 안드로이드 스튜디오의 logcat 기능과 더불어서 mocking 기능까지 제공
