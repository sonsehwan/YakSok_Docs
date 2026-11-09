# JWT 인증 도입기 (2) — 필터, 컨트롤러 전환, 그리고 숨어있던 구멍

[지난 글](./JWT-인증-도입기-1-배경과-백엔드-설계.md)에서 토큰 발급 인프라까지 만들었다.
이번 글은 그 토큰을 실제로 "검증"하는 필터, 그리고 그 검증 결과를 실제로 "쓰는" 컨트롤러
전환 작업을 다룬다. 그리고 이 전환 작업 도중에 발견한, 지금 돌아봐도 아찔한 문제 하나도
함께 정리한다.

## 필터는 막지 않는다 — 검증만 하고 흘려보낸다

`JwtAuthenticationFilter`를 처음 설계할 때 가장 중요하게 잡은 원칙이 있다. **이 필터는
"막을지 말지"를 결정하지 않는다.** 토큰을 까보고 유효하면 신원을 심어두고, 무효하거나
없으면 그냥 조용히 다음으로 넘긴다.

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtProvider jwtProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        String header = request.getHeader("Authorization");

        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            try {
                Long userId = jwtProvider.getUserId(token);

                Authentication auth = new UsernamePasswordAuthenticationToken(
                        userId, null, Collections.emptyList());
                SecurityContextHolder.getContext().setAuthentication(auth);

            } catch (JwtException e) {
                SecurityContextHolder.clearContext();
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

"막을지 말지"는 뒤에 오는 `authorizeHttpRequests` 규칙의 몫으로 완전히 분리했다. 이렇게
나눠두면, 이 필터를 먼저 배포해서 "유효한 토큰을 보내면 신원이 제대로 채워지는지"를
`permitAll()` 상태에서 안전하게 확인할 수 있다. 실제로 로그를 찍어서 확인해봤다 — 유효한
토큰을 보내면 로그가 찍히며 통과하고, 토큰이 없거나 위조돼도 로그 없이 그냥 통과했다.
의도한 그대로였다.

## 화이트리스트로 뒤집기

마지막 단계에서 `SecurityConfig`의 `permitAll()`을 걷어내고, 반대로 뒤집었다.

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(
        "/api/users/signup", "/api/users/login",
        "/api/auth/refresh", "/api/auth/logout",
        "/api/medicine/**", "/api/drugstore/searchlist", "/api/drugstore/closelist",
        "/ws-stomp/**", "/error"
    ).permitAll()
    .anyRequest().authenticated()
)
```

로그인 없이 호출돼야 하는 경로만 명시적으로 `permitAll()`로 나열하고, 나머지는 전부
`anyRequest().authenticated()`로 묶었다.

블랙리스트(막을 경로만 나열)가 아니라 화이트리스트로 짠 이유는 하나다. **나중에 컨트롤러가
추가됐는데 깜빡하고 규칙을 안 넣으면 어떻게 실패하는가**를 생각해봤다. 블랙리스트 방식이면
새 컨트롤러는 자동으로 뚫린 채로 남는다. 화이트리스트 방식이면 자동으로 막힌 채로 남는다.
실수했을 때 안전한 쪽으로 실패하는 걸 택했다.

배포 후 EC2에서 실동작을 확인했다. 로그인(화이트리스트)은 토큰 없이 성공하고, 그 외 API는
헤더 없이 401로 막히고, 유효한 토큰으로는 정상 통과했다.

## 컨트롤러 전환: 필드 하나 바꾸는 걸로 안 끝났다

이제 컨트롤러들이 `userId`를 body/query param/path variable 대신
`@AuthenticationPrincipal Long userId`로 받도록 바꿔야 했다.

```java
// Before
@PostMapping
public ApiResponse<Void> saveYaksok(@RequestBody YaksokRequest request) { ... }

// After
@PostMapping
public ApiResponse<Void> saveYaksok(@RequestBody YaksokRequest request,
                                     @AuthenticationPrincipal Long userId) { ... }
```

DTO 9개(`ModifyInfoRequest`, `ModifyPasswordRequest`, `YaksokRequest`, `ShareYaksokRequest`,
`ChatRoomRequest`, `FriendChatRoomRequest`, `DeleteChattingRoomRequest`,
`FriendRequestCreateDto`, `FriendRequestAnswerDto`, `CreateDrugStoreRequest`)에서 "나 자신"을
뜻하던 `userId`/`loginUserId` 필드를 지웠다. 여기서 중요했던 건 **"작업 대상이 되는 남"을
뜻하는 필드(`friendId`, `senderId`, `yaksokId` 등)는 그대로 둬야 한다**는 구분이었다. "나는
누구다"는 이제 토큰이 말해주지만, "누구에게 무엇을 할 건지"는 여전히 요청이 말해줘야 하는
정보이기 때문이다.

컨트롤러 6개, 총 28곳을 전환하면서 처음엔 "필드 이름만 바꿔주면 끝나는 기계적인 작업"이라고
생각했다. 그런데 그렇지 않았다.

## 발견한 것: 소유권 검증이 애초에 없던 메서드 4개

전환 도중 `YaksokService`의 네 메서드 — `getYaksok`, `updateYaksok`, `deleteYaksok`,
`updateNotificationStatus` — 가 **`userId` 파라미터 자체를 받고 있지 않다**는 걸 발견했다.

```java
// 전환 전
public void updateNotificationStatus(Long notificationId, boolean isTaken) {
    Notification notification = notificationRepository.findById(notificationId)...;
    notification.setTaken(isTaken);
    // 이 알림이 "누구 것"인지는 아무도 확인하지 않는다
}
```

`notificationId`만 맞으면 그게 로그인한 사람의 것이든 완전히 남의 것이든 구분 없이
조회·수정·삭제·복약완료 처리가 됐다는 뜻이다. 특히 `updateNotificationStatus`는 알림에서
"복약 완료" 버튼을 누르면 호출되는 메서드다. 즉 **id만 알면 남의 복약 기록을 아무나 임의로
완료/취소 처리할 수 있는 상태**였다 — 이 앱의 핵심 데이터를 다루는 메서드가, 애초에
JWT를 붙이기 전부터 그냥 뚫려 있었던 셈이다.

```java
// 전환 후
public void updateNotificationStatus(Long notificationId, boolean isTaken, Long userId) {
    Notification notification = notificationRepository.findById(notificationId)...;

    if (!notification.getYaksok().getUser().getId().equals(userId)) {
        throw new IllegalArgumentException("본인의 알림만 처리할 수 있습니다.");
    }

    notification.setTaken(isTaken);
}
```

네 메서드 모두 `userId` 파라미터를 추가하고 `!resource.getUser().getId().equals(userId)`
검증을 새로 넣었다. 참고로 "공유받은 사람도 복약 완료 처리를 할 수 있어야 하지 않을까"를
잠깐 검토했었는데, 이건 잘못된 전제였다 — 공유받은 사람이 받는 알림(`TAKEN_ALERT`)은 애초에
"완료" 버튼 자체가 없는, 보기만 하는 알림이다. 그래서 소유자만 허용하면 되는 단순한
검증이었다.

## 있던 곳과 없던 곳이 컨트롤러마다 섞여 있었다

이 발견 이후로 나머지 서비스들도 전부 재점검했다. 결과는 이랬다.

| 서비스 | 결과 |
| --- | --- |
| `YaksokService`(4개 메서드) | 검증 없었음 → **수정** |
| `ShareYaksokService.saveSharedYaksok` | 이미 검증됨 |
| `ChatService`(메시지 조회/방 삭제) | 이미 참여자 검증됨 |
| `FriendService`(요청 수락/거절) | 이미 검증됨 |
| `DrugstoreService.modifyDrugStore` | 이미 검증됨 |
| `ChatService.getOrCreateFriendRoom` | **친구 관계 검증이 없음 — 발견했지만 이번 범위에서는 미수정** |

"userId를 어디서 얻느냐"만 바꾸면 되는 곳과 "검증 로직 자체가 없었던" 곳이 컨트롤러마다
섞여 있었다는 게 이 작업의 진짜 함정이었다. 필드 치환이라는 기계적인 작업이라고 가정하고
시작했다면 저 4개 메서드는 그냥 지나쳤을 거다. 리팩터링 도중에 "이 메서드가 정말 소유자
검증을 하고 있나"를 하나하나 직접 열어봐야 했던 이유다.

---

백엔드는 이걸로 사실상 끝났다. 다음 글에서는 안드로이드 앱이 이 토큰을 자동으로 붙이고,
만료되면 자동으로 재발급받는 부분 — 그리고 이 과정에서 마주친, 설계하면서 꽤 헷갈렸던
재귀 호출 문제를 다룬다.

**다음 글:** [JWT 인증 도입기 (3) — 안드로이드 프론트엔드 연동](./JWT-인증-도입기-3-안드로이드-프론트엔드-연동.md)
