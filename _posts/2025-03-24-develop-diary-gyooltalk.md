---
title: 귤톡 개발일지 - 채팅방 데이터 설계 변경 이야기
description: >-
  데이터 설계 변경의 무서움
author: author_js
date: 2025-03-24 00:30:00 +0900
categories: [React Native Expo, 귤톡]
tags: [react, native, expo, 귤톡, 데이터베이스, database, mongodb, 채팅, 앱, 모바일, 설계]
pin: true
---
## 귤톡 개발일지 - 채팅방 데이터 설계 변경 이야기

우리 팀은 요즘 귤톡이라는 심플(?)한 채팅 앱을 만들고 있다. 사실 심플하진 않다. 여러가지 생각들이 많지만 일단은 채팅 기능을 모바일로 구현하는 것을 목표로 개발중이다. 처음이라 최대한 간다하게 개발하는 것을 목표로 삼았기 때문에 데이터베이스 구조도 MongoDB를 활용해 아주 단순하게 만들었다.

기존에는 `Chat`이라는 컬렉션을 생성했고, 필드는 다음과 같이 네 개로 구성했다.:
- `id`
- `chatroomName`
- `messages`
- `participants`

특히, `participants`는 처음에는 단순히 사용자들의 아이디를 문자열 리스트로 저장했다. 이 방식은 개발 초반에는 꽤 효율적이었다.

그러나 간단한 구조에도 불고하고 예상치 못한 문제가 발생했다. 바로 채팅방에서 유저가 나갔을 때, 해당 유저의 아이디를 participants 리스트에서만 삭제하도록 구현했더니, 그 유저가 다시 채팅방에 들어왔을 때 이전에 나눴던 모든 메시지를 다시 볼 수 있게 되는 문제였다.

팀원들과 상의를 한 결과, 유저들이 채팅방에 다시 들어오거나 새로 들어왔을 때 이전 메시지들을 볼 수 없게 하는 것이 더욱 자연스럽다는 결론을 내렸다. 즉, 채팅방에 참여하기 이전의 메시지는 볼 수 없도록 하고 싶었다.

그래서 데이터 설계를 다시 고민했고, 결곡 participants 필드를 단순 문자열 리스트에서 클래스로 변경하는 방식으로 해결하기로 했다.

새롭게 만든 `Participant` 클래스는 다음과 같은 구조를 가진다.:
```java
public class Paricipant {
	private String userId;
	private LocalDateTime joinTime;
}
```

이제 새로운 유저가 채팅방에 들어오면, 해당 유저의 참여시점을 함께 저장한다. 이를 통해 채팅 메시지를 불러올 때, 유저가 채팅방에 참여한 시간 이후의 메시지만 보여주도록 구현할 수 있게 되었다.

이번 변경 덕분에 초기 설계에서 발견하지 못했던 문제들을 효과적으로 해결할 수 있었고, 사용자 경험도 개선되지 않을까 생각한다.

교육원에서도 잘 배웠던 일이지만, 데이터베이스 설계는 정말 신중하게 해야하는 것 같다. 단지 participants 필드의 타입을 변경하는 일이지만, 타입을 변경함으로써 이미 진행했던 로직들에서 수정해야 할 것들도 같이 나오기 마련이다.

간단한 채팅 앱이지만 데이터 구조는 조금 더 세밀하게 고민하고 접근하지 않으면 나중에 변경해야할 때 받는 피해는 상상하지 못할 정도로 두려운 결과를 낳을지도 모르겠다는 생각이 든다.