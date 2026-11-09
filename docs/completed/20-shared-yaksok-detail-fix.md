# 20. 채팅방 공유 약속 "확인하기" 막힘 수정

## 증상

채팅방에서 공유받은 약속의 "확인하기"를 누르면 "본인의 약속만 조회할 수 있습니다"로 막힌다.

## 원인

`GET /api/yaksok/{yaksokId}`(`YaksokController` → `YaksokService.getYaksok`)는 이름과 위치상
"내 약속 상세 조회"였지만, 실제로는 프론트 `ShareYaksokDetail`(채팅 "확인하기" + 저장한 공유
목록 상세) 딱 한 곳에서만 호출되고 있었다. JWT 인증 도입 커밋(`4edadee`, 8장)에서 IDOR 방지용
소유자 검증(`yaksok.getUser().getId().equals(userId)`)을 이 메서드에 추가했는데, 이 엔드포인트의
유일한 실사용처가 "남의(공유자의) 약속을 보는 화면"이라는 걸 그때 고려하지 못해 항상 막히게 됐다.
마이그레이션 전에는 이 엔드포인트에 소유자 검증 자체가 없어 우연히 동작했었다.

## 수정

새 엔드포인트를 나란히 추가하는 대신, `GET /api/yaksok/{yaksokId}`를 통째로 `ShareYaksokController`/
`ShareYaksokService`로 옮기고 `getSharedYaksokDetail`로 개명, 검증도 실제 용도에 맞게 교체했다:
- 이미 내 공유 목록에 저장돼 있으면(`ShareYaksokRepository.existsByReceiverIdAndYaksokId`) 통과
- 아니면 `saveSharedYaksok`가 쓰던 기존 `validateSharedToMe`(내가 속한 채팅방에 이 약속이 실제로
  공유된 메시지가 있는지)로 검증 — 저장 전 미리보기 케이스

프론트는 `YaksokApi.getYaksok` → `getSharedYaksokDetail`(`GET /api/shared-yaksok/{yaksokId}`)로
교체, `ShareYaksokDetail.loadYaksok()`의 호출부만 변경.

## 왜 그렇게 판단했나

원래 `getYaksok`/`GET /api/yaksok/{yaksokId}`는 백엔드·프론트 양쪽 다 이 화면 말고는 호출하는
곳이 없었다. 그래서 "새 엔드포인트 추가 + 기존 것 방치"안은 죽은 코드를 남긴다 — 기존 것을
옮기고 이름을 실제 용도에 맞춰 바꾸는 쪽이 더 적은 변경으로 죽은 코드 없이 끝난다.

## 검토했지만 택하지 않은 대안

- **`getYaksok`은 그대로 두고 검증 조건에 "공유 관계면 통과" 추가** — 호출부가 하나뿐이라는 걸
  알기 전에 나온 안. `YaksokService`(내 약속 CRUD 전담)에 `ShareYaksokRepository` 의존이
  섞여 들어가는 문제도 있어 기각
- **새 엔드포인트(`getSharedYaksokDetail`)를 추가하고 `getYaksok`은 남겨둠** — 실제 호출처를
  확인해보니 `getYaksok`이 이 화면 전용이었다는 게 드러나 채택 안 함(죽은 코드가 됨)

## 변경된 파일

- `YakSok_Backend`: `yaksok/service/YaksokService.java`(`getYaksok` 삭제),
  `yaksok/controller/YaksokController.java`(`GET /{yaksokId}` 삭제),
  `yaksok/service/ShareYaksokService.java`(`getSharedYaksokDetail` 추가),
  `yaksok/controller/ShareYaksokController.java`(`GET /{yaksokId}` 추가)
- `YakSok_Frontend`: `network/YaksokApi.java`(`getYaksok` → `getSharedYaksokDetail`),
  `ui/sharedyaksok/ShareYaksokDetail.java`(호출부 변경)
