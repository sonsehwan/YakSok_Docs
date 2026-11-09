# 화면 인벤토리

YakSok_Frontend의 모든 Activity/Fragment(바텀시트 포함)를 화면 단위로 정리한 목록이다. 화면 위에 뜨는
다이얼로그는 맨 아래 "다이얼로그" 절에 따로 모았다. `ui/base/BaseActivity`는 공통 베이스 클래스(뒤로가기
강제 로그아웃 처리 등)일 뿐 자체 화면이 아니므로 제외했다.

`design/yaksok_design.pen`에서 x ≥ 2400의 Flow 01~06은 개선 디자인(와이어프레임)이고, 그 왼쪽의 AS-IS 열은
지금 `res/layout` 그대로 옮긴 현재 앱 화면이다.

## 화면 목록

### login

## 로그인 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/login/Login.java)
- 목적: 아이디/비밀번호 또는 카카오로 로그인하고 FCM 토큰을 등록하는 앱 진입 화면.
- 컴포넌트:
  - 버튼: btn_login(로그인 실행), btn_kakao_login(카카오 로그인 — 처음이면 SignUp_Kakao로)
  - 입력창: input_login_id(아이디), input_pw(비밀번호)
- 화면 이동: 로그인 성공 시 MainActivity로 이동(기존 스택 제거). tv_go_signup 클릭 시 SignUpTypeBottomSheet(회원가입 유형 선택)를 띄움. tv_find_id 클릭 시 FindId, tv_find_pw 클릭 시 FindPassword로 이동. 이미 로그인돼 있으면 화면을 그리지 않고 바로 MainActivity로 보낸다.

## 아이디 찾기 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/login/FindId.java)
- 목적: 이메일 인증코드로 본인 확인 후 가입한 아이디를 보여주는 화면.
- 컴포넌트:
  - 버튼: btn_send_code(인증코드 발송), btn_go_login(로그인 화면으로 이동, 인증 완료 후 노출)
  - 입력창: input_email(이메일), verification_code(인증코드 입력/재발송/검증 커스텀 뷰)
- 화면 이동: 인증 성공 시 아이디를 화면에 표시. btn_go_login 클릭 시 Login으로 이동(스택 초기화).

## 비밀번호 찾기 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/login/FindPassword.java)
- 목적: 아이디+이메일 인증코드로 본인 확인 후 비밀번호 재설정 화면으로 넘어가는 진입 화면.
- 컴포넌트:
  - 버튼: btn_send_code(인증코드 발송)
  - 입력창: input_login_id(아이디), input_email(이메일), verification_code(인증코드)
- 화면 이동: 인증 성공 시 재설정 토큰(resetToken)을 담아 ResetPassword로 이동.

## 새 비밀번호 설정 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/login/ResetPassword.java)
- 목적: FindPassword에서 받은 재설정 토큰으로 새 비밀번호를 설정하는 화면.
- 컴포넌트:
  - 버튼: btn_confirm(비밀번호 변경)
  - 입력창: input_new_pw(새 비밀번호), input_check_pw(새 비밀번호 확인)
- 화면 이동: 변경 성공/실패 모두 Login으로 이동(스택 초기화). resetToken이 없이 잘못 진입하면 즉시 Login으로 되돌아간다.

### signup

## 회원가입 유형 선택 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/signup/SignUpTypeBottomSheet.java)
- 목적: 일반 유저/약사(약국) 중 회원가입 유형을 고르는 바텀시트.
- 컴포넌트:
  - 버튼: user_type(일반 유저 선택), drugStore_type(약사 선택)
- 화면 이동: 선택에 따라 SignUp_Normal 또는 SignUp_DrugStore로 이동.

## 일반 회원가입 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/signup/SignUp_Normal.java)
- 목적: 일반 사용자 계정 정보를 입력받아 가입시키는 화면. 이메일은 인증코드(`verification_code`)로 소유를 확인해야 가입된다(46장).
- 컴포넌트:
  - 버튼: btn_finish(회원가입 완료)
  - 입력창: input_login_id(아이디), input_email(이메일), input_pw(비밀번호), input_check_pw(비밀번호 확인), input_nickname(닉네임)
- 화면 이동: 가입 성공 시 Login으로 이동(스택 초기화).

## 약국 회원가입 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/signup/SignUp_DrugStore.java)
- 목적: 약사(약국) 계정 정보와 소속 약국을 입력받아 가입시키는 화면.
- 컴포넌트:
  - 버튼: btn_find_drugstore(약국 검색), btn_sign_up(회원가입)
  - 입력창: input_login_id, input_email, input_pw, input_check_pw, input_nickname
- 화면 이동: btn_find_drugstore 클릭 시 FindDrugStore를 sign_up 모드로 결과 대기 실행해 약국을 선택받는다. 가입 성공 시 Login으로 이동(스택 초기화).

## 카카오 회원가입 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/signup/SignUp_Kakao.java)
- 목적: 처음 카카오로 로그인한 사용자가 역할(일반/약국)과 닉네임을 정해 가입을 마치는 화면(33.13). 카카오 토큰·닉네임·이메일은 인텐트로 받는다.
- 컴포넌트:
  - 버튼: tv_role_normal/tv_role_drugstore(역할 선택), btn_find_drugstore(약국 회원일 때 소속 약국 검색), btn_sign_up(가입 완료)
  - 입력창: input_nickname(닉네임), input_email(이메일)
- 화면 이동: btn_find_drugstore 클릭 시 FindDrugStore를 결과 대기로 실행. 가입 성공 시 메인으로 이동. 이메일이 겹치면 서버가 가입을 막고 연동을 안내한다.

## 주소 검색 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/signup/AddressSearch.java)
- 목적: 카카오 주소 검색 웹페이지를 WebView로 띄우고 자바스크립트 브릿지로 선택한 주소를 돌려받는 화면.
- 컴포넌트: (WebView 전체 화면뿐이며, 상호작용은 웹페이지 내부에서 일어나 네이티브 버튼/리스트/입력창이 없음)
- 화면 이동: 주소 선택 시 결과(address_result, selectedZonecode)를 RESULT_OK로 담아 호출한 화면으로 종료.
- 참고: 코드베이스 어디에서도 `AddressSearch.class`로 이 화면을 시작하는 호출부를 찾지 못했다. 현재는 진입 경로가 없는 죽은 코드로 보인다.

### main

## 메인(탭 컨테이너) (YakSok_Frontend/app/src/main/java/com/example/medication/ui/main/MainActivity.java)
- 목적: 하단 네비게이션(홈/약속목록/채팅/친구/설정) 탭과 좌측 드로어를 갖는 앱의 메인 컨테이너 화면.
- 컴포넌트: (자체 리스트/입력창 없음. bottom_navigation은 탭 전환용, side_nav_view는 드로어 메뉴)
- 화면 이동: 드로어의 "약국" 메뉴 클릭 시 DrugStoreList로 이동. 알림 등으로 EXTRA_TARGET_TAB이 담긴 인텐트로 재진입하면 해당 탭으로 전환.

## 홈 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/main/HomeFragment.java) — MainActivity의 nav_home 탭
- 목적: 선택한 날짜의 복약 알림 목록과 진행률(%)을 보여주는 첫 화면.
- 컴포넌트:
  - 버튼: iv_menu(좌측 드로어 열기), iv_prev_date/iv_next_date(하루 이동), iv_calendar(날짜 선택 다이얼로그)
  - 리스트: rv_medication — 선택한 날짜의 복약 알림 목록(아침/점심/저녁 구간 헤더 + NotificationMultiViewAdapter)
- 화면 이동: 자체 화면 전환은 없고, iv_menu로 드로어를 열어 다른 메뉴(약국 목록)로 이동한다.

### yaksok

## 약속 목록 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/yaksok/YaksokListFragment.java) — MainActivity의 nav_history 탭
- 목적: 내 약속 목록과 남이 공유한 약속 목록을 탭으로 전환하며 보여주는 화면.
- 컴포넌트:
  - 버튼: iv_menu(드로어 열기), tv_my_yaksok/tv_shared_yaksok(내 약속 ↔ 공유 약속 탭 전환), fab_scan(약속 등록 방법 선택 바텀시트 열기)
  - 리스트: rv_share_user_list(나에게 약속을 공유해준 친구 목록, 가로 스크롤), rv_yaksok_list(내 약속 목록 또는 선택한 친구의 공유 약속 목록)
- 화면 이동: 내 약속 항목 클릭 시 결과 대기로 YaksokDetail 실행. 공유 약속 항목 클릭 시 ShareYaksokDetail(이미 저장됨 모드)로 이동. fab_scan은 BottomSheetAddYaksok 바텀시트를 띄운다.

## 약속 상세(내 약속) (YaksokDetail.java, 경로: YakSok_Frontend/app/src/main/java/com/example/medication/ui/yaksok/YaksokDetail.java)
- 목적: 내가 등록한 약속의 상세 정보를 읽기 전용으로 보여주고, 메뉴로 공유/수정/삭제를 제공하는 화면.
- 컴포넌트:
  - 버튼: iv_back(뒤로가기), iv_menu(공유/수정/삭제 팝업 메뉴)
  - 리스트: rv_selected_pills(등록된 약 목록)
  - 입력창: input_card_title, input_start_date, input_prescriptionDays, cb_morning/cb_lunch/cb_dinner, rg_dosage_time — 전부 읽기 전용으로 잠겨 있음
- 화면 이동: 메뉴의 "수정"은 결과 대기로 ModifyYaksok 실행. "공유"는 친구 선택 다이얼로그 → 선택한 친구와의 채팅방(ChattingRoom)을 열며 공유 메시지를 함께 전송. "삭제"는 확인 다이얼로그 후 삭제하고 화면 종료.

## 약속 등록 방법 선택 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/yaksok/BottomSheetAddYaksok.java)
- 목적: 처방전 스캔/약봉투 스캔/직접 입력 중 약속 등록 방식을 고르는 바텀시트.
- 컴포넌트:
  - 버튼: ll_create_prescription(처방전 등록), ll_create_medicine_envelope(약봉투 등록), ll_create_direct(직접 입력)
- 화면 이동: 선택에 따라 CreatePrescription / CreatePaperEnvelope / CreateDirectSchedule로 이동. (옛 이름 `ShowAddMedicationList`)

## 사진 가져오는 방법 선택 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/yaksok/ImageSourceBottomSheet.java)
- 목적: 처방전·약봉투 화면에 들어가면 먼저 뜨는 "카메라 / 앨범" 선택 바텀시트. 갤러리 선택, 카메라 촬영(FileProvider 임시 파일), `CAMERA` 권한 요청을 모두 이 클래스가 맡고 결과는 Fragment Result로 Uri만 넘긴다(43장).
- 컴포넌트: (카메라 / 앨범 두 선택지. 권한을 거부하면 촬영은 진행하지 않는다)
- 화면 이동: 선택한 사진의 Uri를 호출한 화면(CreatePrescription / CreatePaperEnvelope)에 돌려주고 닫는다.

## 처방전으로 약속 등록 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/yaksok/CreatePrescription.java)
- 목적: 갤러리에서 고른 처방전 사진을 ML Kit OCR + Gemini AI로 분석해 약 정보를 자동 입력하고 약속을 등록하는 화면.
- 컴포넌트:
  - 버튼: iv_back(뒤로가기), btn_add_pill(약 직접 추가), btn_register(등록)
  - 리스트: rv_selected_pills(인식되었거나 직접 추가한 약 목록)
  - 입력창: input_card_title(약속 별명), input_start_date(투약 시작일, 날짜 선택), input_prescriptionDays(투약일 수), cb_morning/cb_lunch/cb_dinner(투약 시간대 체크), input_set_morning/lunch/dinner_time(각 시간 선택), rg_dosage_time(식전 30분/식후 30분/직후 라디오)
- 화면 이동: 화면 진입과 동시에 갤러리를 열어 사진을 선택(취소 시 화면 종료). btn_add_pill 클릭 시 결과 대기로 MedicineSearchActivity 실행. 등록 성공 시 MainActivity로 이동.

## 약봉투로 약속 등록 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/yaksok/CreatePaperEnvelope.java)
- 목적: 약봉투 사진을 Gemini AI로 분석해 약 정보와 투약 시간대까지 자동 입력하고 약속을 등록하는 화면. 레이아웃은 CreatePrescription·CreateDirectSchedule과 같은 `activity_create_prescription.xml`을 공유한다(`activity_paper_envelope.xml`은 같은 내용이지만 어느 화면도 쓰지 않는다).
- 컴포넌트: CreatePrescription과 동일한 구성 — 버튼: iv_back, btn_add_pill, btn_register / 리스트: rv_selected_pills / 입력창: input_card_title, input_start_date, input_prescriptionDays, cb_morning/lunch/dinner, input_set_morning/lunch/dinner_time, rg_dosage_time
- 화면 이동: 진입과 동시에 갤러리를 열어 사진을 선택(취소 시 종료). btn_add_pill로 결과 대기 MedicineSearchActivity 실행. 등록 성공 시 MainActivity로 이동.

## 직접 입력으로 약속 등록 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/yaksok/CreateDirectSchedule.java)
- 목적: AI 스캔 없이 사용자가 모든 항목을 직접 입력해 약속을 등록하는 화면.
- 컴포넌트:
  - 버튼: iv_back(뒤로가기), btn_add_pill(약 검색 추가), btn_register(등록)
  - 리스트: rv_selected_pills(추가된 약 목록)
  - 입력창: input_card_title, input_start_date, input_prescriptionDays, cb_morning/lunch/dinner, input_set_morning/lunch/dinner_time, rg_dosage_time
- 화면 이동: btn_add_pill 클릭 시 결과 대기로 MedicineSearchActivity 실행. 등록 성공 시 MainActivity로 이동.

## 약속 수정 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/yaksok/ModifyYaksok.java)
- 목적: YaksokDetail에서 넘어온 기존 약속 데이터를 채워 넣고 수정할 수 있게 하는 화면.
- 컴포넌트:
  - 버튼: iv_back(뒤로가기), btn_add_pill(약 검색 추가), btn_register(수정 저장)
  - 리스트: rv_selected_pills(약 목록)
  - 입력창: input_card_title, input_start_date, input_prescriptionDays, cb_morning/lunch/dinner, input_set_morning/lunch/dinner_time, rg_dosage_time
- 화면 이동: btn_add_pill로 결과 대기 MedicineSearchActivity 실행. 저장 성공 시 수정된 약속 데이터를 RESULT_OK로 담아 호출한 YaksokDetail로 복귀.

### sharedyaksok

## 공유 약속 상세 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/sharedyaksok/ShareYaksokDetail.java)
- 목적: 친구가 공유한 약속의 상세 정보를 읽기 전용으로 보여주고, 원하면 내 목록에 저장하게 하는 화면.
- 컴포넌트:
  - 버튼: iv_back(뒤로가기), btn_save(내 목록에 저장 — 이미 저장된 경우 숨김)
  - 리스트: rv_selected_pills(약 목록)
  - 입력창: input_card_title, input_start_date, input_prescriptionDays, cb_morning/lunch/dinner, rg_dosage_time — 전부 읽기 전용
- 화면 이동: btn_save 저장 성공 시 화면 종료. (YaksokDetail의 iv_menu 공유하기, ChattingRoom의 공유 메시지 클릭, YaksokListFragment의 공유 약속 항목 클릭에서 이 화면으로 진입한다.)

## 공유받은 약속 목록 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/sharedyaksok/ShareYaksokList.java)
- 목적: 특정 발신자가 공유한 약속들을 리스트로 보여주는 화면.
- 컴포넌트:
  - 버튼: iv_back(뒤로가기)
  - 리스트: rv_share_yaksok(공유 약속 목록, 길게 눌러 목록에서 빼기)
- 화면 이동: 항목 클릭 시 ShareYaksokDetail(이미 저장됨 모드)로 이동.
- 참고: 코드베이스 어디에서도 `ShareYaksokList.class`로 이 액티비티를 시작하는 호출부를 찾지 못했다. YaksokListFragment는 대신 ShareYaksokDetail을 바로 호출하므로, 현재는 진입 경로가 없는 죽은 코드로 보인다.

### chattingroom

## 채팅방 목록 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/chattingroom/ChatRoomListFragment.java) — MainActivity의 nav_chat 탭
- 목적: 내가 참여 중인 채팅방 목록을 보여주는 화면(약국 계정은 친구/상담 채팅을 탭으로 구분).
- 컴포넌트:
  - 리스트: rv_chat_rooms(채팅방 목록, 길게 눌러 나가기/삭제)
  - 입력창: rg_chat_type(친구/상담 라디오 — 약국 계정에만 노출)
- 화면 이동: 채팅방 항목 클릭 시 ChattingRoom으로 이동.

## 채팅방 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/chattingroom/ChattingRoom.java)
- 목적: STOMP 웹소켓으로 실시간 채팅을 주고받는 화면. 약속 공유 메시지를 표시하고 클릭할 수 있다.
- 컴포넌트:
  - 버튼: btn_send(메시지 전송)
  - 리스트: rv_chat_messages(대화 내역 — 약속 공유 메시지 항목은 클릭 가능)
  - 입력창: et_message(메시지 입력)
- 화면 이동: 대화 중 약속 공유 메시지를 클릭하면 ShareYaksokDetail로 이동.

### drugstore

## 약국 검색 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/drugstore/FindDrugStore.java)
- 목적: 지역(시/도·시군구)과 이름으로 약국을 검색해 선택하는 화면. 회원가입 중 약국 등록, 또는 내 정보에서 약국 변경 시 재사용된다.
- 컴포넌트:
  - 버튼: iv_back(뒤로가기), btn_find_drugstore(조회)
  - 리스트: rv_drugstore_list(검색 결과 목록, 클릭 시 선택 확인 다이얼로그)
  - 입력창: sp_first_address(시/도 스피너), sp_second_address(시군구 스피너, 시/도 선택에 따라 갱신), et_drugstore_name(약국 이름, 선택 입력)
- 화면 이동: 회원가입 중(sign_up 모드)이면 선택 결과를 RESULT_OK로 돌려주며 종료. 그 외(modify 모드)에는 선택한 약국으로 내 정보를 서버에 갱신한 뒤 MyInfo로 이동.

## 인근 약국 목록 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/drugstore/DrugStoreList.java)
- 목적: 현재 위치 기준으로 가까운 약국을 거리순으로 무한 스크롤 보여주는 화면(MainActivity 좌측 드로어에서 진입).
- 컴포넌트:
  - 버튼: iv_back(뒤로가기)
  - 리스트: rv_drugstore_list(거리순 약국 목록, 스크롤이 바닥에 가까워지면 다음 페이지 자동 로드)
- 화면 이동: 항목 클릭 시 DrugStoreDetail로 이동.

## 약국 상세 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/drugstore/DrugStoreDetail.java)
- 목적: 선택한 약국의 정보(이름/주소/전화/영업시간)와 카카오맵 위치를 보여주고, 해당 약국과 채팅을 시작하는 화면.
- 컴포넌트:
  - 버튼: btn_start_chatting(채팅 시작)
- 화면 이동: btn_start_chatting 클릭 시(자신의 약국이 아닐 때) ChattingRoom으로 이동.
- 참고: 레이아웃의 iv_detail_back(뒤로가기 아이콘)은 클릭 리스너가 연결돼 있지 않고 clickable 속성도 없어 현재 동작하지 않는 장식용 아이콘으로 보인다.

### medicine

## 제품 검색 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/medicine/MedicineSearchActivity.java)
- 목적: 약 이름으로 공공데이터 의약품 정보를 검색해 하나를 선택하는 화면. 약속 등록/수정 화면들에서 결과 대기로 호출된다.
- 컴포넌트:
  - 버튼: iv_back(뒤로가기), btn_search(검색)
  - 리스트: rv_medicine(검색 결과, 무한 스크롤)
  - 입력창: et_search(제품명 검색어)
- 화면 이동: 항목 선택 시 선택한 약 이름/이미지를 RESULT_OK로 담아 호출한 화면(CreatePrescription/CreatePaperEnvelope/CreateDirectSchedule/ModifyYaksok)으로 복귀.

### friend

## 친구 목록 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/friend/FriendListFragment.java) — MainActivity의 nav_friend 탭
- 목적: 내 친구 목록을 보여주고, 친구 추가/받은 친구 요청 처리를 다이얼로그로 제공하는 화면.
- 컴포넌트:
  - 버튼: iv_qr(QR 팝업 메뉴 — "내 프로필"/"코드 스캔"), iv_add_friend(친구 추가 다이얼로그 열기),
    layout_friend_request(받은 요청 다이얼로그 열기)
  - 리스트: rv_yaksok_list(친구 목록, 길게 눌러 삭제)
- 화면 이동: 친구 항목 클릭 시 프로필 다이얼로그(FRIEND 상태, "채팅하기")를 띄운다(31장). `WipActivity`로 가던 막다른 길은 없어졌다.
- 참고: 친구 추가/받은 친구 요청/친구 길게 누르기(삭제)/내 QR 코드/QR로 알아낸 상대 프로필 —
  전부 별도 Activity/Fragment가 아닌 일반 Dialog라 이 목록에는 별도 항목으로 넣지 않았다.
- **QR 진입점(구현됨).** 헤더 오른쪽에 `[iv_qr][iv_add_friend]` 순서로 아이콘 두 개. `iv_qr` 클릭 시
  `PopupMenu`로 "내 프로필"/"코드 스캔" 표시:
  - **내 프로필** — `issueQrCode()` 호출 후 Glide로 QR 이미지를 그린 다이얼로그(`dialog_my_qr.xml`)를 띄운다.
  - **코드 스캔** — `CAMERA` 런타임 권한 확인 후 ZXing(`zxing-android-embedded`)의 `ScanContract`로
    스캔 화면(`PortraitCaptureActivity`)을 열고, 인식된 문자열로 상대 프로필 다이얼로그(`dialog_profile.xml`)를 띄운다. 관계
    상태별 액션 버튼: SELF 숨김, FRIEND는 "채팅하기"(그 친구와의 채팅방을 엶), SENT는 비활성
    "요청 보냄", RECEIVED는 "친구 요청 수락하기", NONE은 "친구 요청 보내기". 액션 성공 시 같은
    다이얼로그를 다시 조회해 버튼만 갱신, 닫기는 `btn_close`.
- 참고: QR 프로필 다이얼로그는 원래 별도 Activity(`Profile`, 구 `FriendProfile`)였다가 이 화면 위
  다이얼로그로 흡수됐다 — 설계 배경은 `docs/active/17-qr-friend-add.md` 참고.

### myinfo

## 내 정보 관리 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/myinfo/MyInfo.java)
- 목적: 닉네임 수정, 비밀번호 변경 화면 진입, 약국 계정의 소속 약국 재등록, 회원 탈퇴를 처리하는 화면.
- 컴포넌트:
  - 버튼: iv_back(뒤로가기), btn_change_pw(비밀번호 변경 화면으로 이동), btn_find_drugstore(약국 재검색 — 약국 계정에만 노출), btn_withdraw(회원 탈퇴), btn_save_info(수정 저장)
  - 입력창: input_nickname(닉네임)
- 화면 이동: btn_change_pw 클릭 시 ModifyPassword로 이동. btn_find_drugstore 클릭 시 FindDrugStore(modify 모드)로 이동. 탈퇴 성공 시 Login으로 이동(스택 초기화). 닉네임 수정 성공 시 MainActivity의 설정 탭으로 이동.

## 비밀번호 변경 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/myinfo/ModifyPassword.java)
- 목적: 현재 비밀번호를 확인한 뒤 새 비밀번호로 바꾸는 화면.
- 컴포넌트:
  - 버튼: btn_back(뒤로가기), btn_modify_pw(변경하기)
  - 입력창: input_current_pw(현재 비밀번호), input_new_pw(새 비밀번호), input_check_pw(새 비밀번호 확인)
- 화면 이동: 변경 성공 시 로그인 정보를 지우고 Login으로 이동(스택 초기화).

### setting

## 설정 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/setting/SettingsFragment.java) — MainActivity의 nav_settings 탭
- 목적: 내 정보 관리 진입, 알림 수신 여부 토글, 로그아웃을 제공하는 화면.
- 컴포넌트:
  - 버튼: iv_logout(로그아웃 확인 다이얼로그), ll_my_info(내 정보 관리로 이동)
  - 입력창: sw_notification(알림 수신 스위치)
- 화면 이동: ll_my_info 클릭 시 MyInfo로 이동. 로그아웃 확인 시 FCM 토큰을 서버에서 삭제하고 모든 액티비티를 종료한 뒤 Login으로 이동.

### common

## 준비 중 화면 (YakSok_Frontend/app/src/main/java/com/example/medication/ui/common/WipActivity.java)
- 목적: 아직 구현되지 않은 기능으로 연결됐을 때 보여주는 공용 임시 화면. **현재 호출부가 없는 죽은 코드다**(31장에서 친구 목록의 연결을 걷어냈다).
- 컴포넌트:
  - 버튼: button(돌아가기)
- 화면 이동: button 클릭 시 이전 화면으로 복귀(finish).

### drugstore 보조 클래스

## BelowSpinner (YakSok_Frontend/app/src/main/java/com/example/medication/ui/drugstore/BelowSpinner.java)
- 화면이 아니라 약국 검색의 시/도·시군구 선택에 쓰는 커스텀 Spinner. 기본 Spinner는 드롭다운이 앵커 위에서부터 겹쳐 뜨고 높이 제한을 못 걸어서, 앵커 바로 아래에 최대 10개 항목까지만 보이는 `ListPopupWindow`를 직접 띄운다.

### dialog

화면 위에 뜨는 다이얼로그. 호출하는 메서드와 레이아웃을 함께 적었다. 개선 디자인은 `design/yaksok_design.pen`
Flow 06(프로필·내 QR은 Flow 05)이다.

**디자인이 코드와 다른 점:**
- 커스텀 다이얼로그(친구 추가/받은 친구 요청/약속 공유)에 `btn_close`(닫기)를 넣었다. 현재 코드는 바깥 영역을 눌러야만 닫힌다.
- AlertDialog들은 시스템 기본 모양 대신 같은 카드형 확인 다이얼로그(`Dialog/Confirm` 컴포넌트)로 통일했다. 되돌릴 수 없는 동작(삭제/빼기/나가기/탈퇴)은 확인 버튼을 빨간색으로 둔다.
- 버튼 순서는 안드로이드 기본과 같게 **취소가 왼쪽, 확인 동작이 오른쪽**이다.

## 친구 추가 (FriendListFragment.showAddFriendDialog, `dialog_add_friend.xml`)
- 목적: 이메일로 사용자를 검색해 친구 요청을 보내는 다이얼로그. 친구 목록의 iv_add_friend로 연다.
- 컴포넌트:
  - 입력창: et_email(이메일 입력)
  - 버튼: btn_search(검색), btn_send_request(친구 요청 보내기 — 검색 결과 영역 layout_result 안, 결과가 있을 때만 노출)
- 화면 이동: 요청 성공 시 다이얼로그를 닫는다. 검색 실패는 토스트만 띄운다.

## 받은 친구 요청 (FriendListFragment.showReceivedRequestDialog, `dialog_received_request.xml` + `item_received_request.xml`)
- 목적: 나에게 온 친구 요청 목록을 보여주고 수락/거절하게 하는 다이얼로그. 친구 목록의 layout_friend_request로 연다.
- 컴포넌트:
  - 리스트: rv_received_request(요청 목록 — 항목마다 닉네임/이메일과 btn_accept(수락)/btn_reject(거절)), tv_empty(요청이 없을 때 "받은 친구 요청이 없어요.")
- 화면 이동: 수락/거절 성공 시 다이얼로그를 닫고 친구 목록과 요청 개수를 다시 불러온다.

## 약속 공유 · 친구 선택 (YaksokDetail.showFriendPickerDialog, `dialog_select_friend.xml` + `item_friend.xml`)
- 목적: 약속을 공유할 친구를 고르는 다이얼로그. 약속 상세의 메뉴 "공유"로 연다.
- 컴포넌트:
  - 리스트: rv_select_friend(친구 목록), tv_empty(친구가 없을 때 "아직 친구가 없어요.")
- 화면 이동: 친구를 누르면 다이얼로그를 닫고 그 친구와의 채팅방(ChattingRoom)을 열며 공유 메시지를 함께 보낸다.

## 로딩 (LoadingDialog, `activity_loading_dialog.xml`)
- 목적: 서버/AI 응답을 기다리는 동안 띄우는 진행 표시. CreatePrescription·CreatePaperEnvelope(AI 분석), FindDrugStore(약국 조회·선택), DrugStoreList(위치 조회)에서 쓴다.
- 컴포넌트: (버튼/입력 없음. ProgressBar와 "분석 중입니다... 잠시만 기다려주세요." 문구)
- 참고: 문구가 레이아웃에 고정돼 있어 약국 조회처럼 AI 분석이 아닌 곳에서도 "분석 중입니다"가 보인다.

## 확인·액션 다이얼로그 (`AppDialog`)
제목·메시지·취소/확인 버튼만 있는 카드형 확인 다이얼로그. 시스템 `AlertDialog`를 공통 헬퍼 `ui/common/AppDialog`
(`confirm` / `deleteAction` / `popup` / `showBelow`)로 교체했다(26장). 목록의 `AlertDialog` 표기는 옛 이름이다. 확인을 누르면 괄호 안 동작을 실행하고, 취소는 닫기만 한다.
- 복용 완료/취소 확인 — `NotificationMultiViewAdapter.showConfirmDialog` (홈의 cb_done). 제목은 상태에 따라 "복용 완료 처리하시겠습니까?"/"복용 취소 처리하시겠습니까?" (확인: 상태 반전 후 서버 반영, 실패 시 롤백)
- 약속 삭제 — `YaksokDetail.showDeleteConfirmDialog` (메뉴 "삭제"). (삭제: 삭제 후 화면 종료)
- 공유 목록에서 빼기 — `YaksokListFragment.showRemoveSharedConfirmDialog` (공유 약속 길게 누르기). (빼기: 원본은 두고 내 목록에서만 제거) — 죽은 코드인 `ShareYaksokList`에도 같은 다이얼로그가 있다.
- 친구 액션 → 친구 삭제 — `FriendListFragment.showFriendActionsDialog` → `showDeleteConfirmDialog` (친구 길게 누르기). 액션 목록은 "삭제" 하나뿐이고, 고르면 삭제 확인이 이어진다. (삭제: 내 목록에서만 삭제)
- 채팅방 액션 → 채팅방 나가기 — `ChatRoomListFragment.showRoomActionsDialog` → `showDeleteConfirmDialog` (채팅방 길게 누르기). 액션 목록은 "삭제" 하나뿐이다. (나가기: 목록에서 제거)
- 약국 선택 확인 — `FindDrugStore.showDialog` (검색 결과 항목 클릭). (선택: sign_up 모드는 결과 반환, modify 모드는 서버에 약국 변경)
- 로그아웃 — `SettingsFragment.showLogOutDialog` (iv_logout). (로그아웃: FCM 토큰 삭제 후 Login으로)
- 회원 탈퇴 — `MyInfo.showWithdrawConfirmDialog` (btn_withdraw). (탈퇴: 탈퇴 후 Login으로)

## 시스템 날짜/시간 선택
- DatePickerDialog — 홈 iv_calendar, 약속 등록/수정 화면의 input_start_date
- TimePickerDialog — 약속 등록/수정 화면의 input_set_morning/lunch/dinner_time
- 안드로이드 기본 피커라 디자인 파일에는 그리지 않았다.

## 팝업 메뉴 (PopupMenu)
- 약속 상세 iv_menu — 공유/수정/삭제(`res/menu/yaksok_menu.xml`)
- 친구 목록 QR 아이콘 — 내 프로필/코드 스캔 (구현됨, 17장. 위 "친구 목록" 참고)
