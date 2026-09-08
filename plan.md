가능해. 다만 아까 내가 제안한 전체 구조를 1주 안에 다 구현하는 건, 입문자 5명에게는 무리라고 보는 게 맞아. 프로젝트 아이디어가 어려운 게 아니라, Trichimera + 위험판단 + Wayland + 멀티프로세스 + SHM + Socket + epoll + timerfd + GPIO를 한꺼번에 하려는 범위가 어렵다.

대신 5명이 병렬로 움직이고, Linux 요소를 2~3개만 제대로 구현하면 충분히 완성 가능해.

내가 권하는 현실적인 목표

최종 시스템을 이 정도로 줄이자.

Camera
   ↓
Trichimera
   ↓
Detection + Freespace + Lane
   ↓
┌─────────────────────────┐
│    Risk Decision        │
│                         │
│  객체가 주행영역인가?   │
│  내 차선에 있는가?      │
│  위험구역에 들어왔나?   │
└───────────┬─────────────┘
            ↓
   SAFE / WARNING / DANGER
            ↓
        Wayland 표시
            +
        LED/Buzzer

그리고 Linux 시스템 프로그래밍은 욕심내지 말고 이 세 가지에 집중하는 걸 추천해.

① pthread → ② IPC 하나 → ③ GPIO 또는 timerfd

epoll, signalfd, syslog, 여러 프로세스 구조까지 전부 넣을 필요 없다.

특히 멀티프로세스는 처음부터 넣지 마. 먼저 하나의 프로그램으로 완성한 뒤 시간이 남으면 분리해야 한다.

5명 역할은 이렇게 나누는 게 좋다
인원	담당	난이도
1	Trichimera/NPU 담당 — SDK 샘플 실행, Detection/Freespace/Lane 출력 구조 파악	★★★★
2	위험판단 담당 — bbox/freespace/lane을 받아 SAFE/WARNING/DANGER 계산	★★★
3	Wayland/UI 담당 — bbox, lane, freespace, 위험 상태 표시	★★★
4	Linux 담당 — pthread, IPC 등 시스템 프로그래밍 구조	★★★★
5	GPIO/통합 담당 — LED/Buzzer + 빌드/테스트/통합 지원	★★

여기서 중요한 게 있어.

1번을 혼자 내버려두면 안 돼.

프로젝트 초반 최대 리스크는 위험 판단 알고리즘이 아니라 “Trichimera 결과를 우리가 원하는 형태로 뽑아낼 수 있는가?”이기 때문이다. Day 1~2에는 1번과 3번 또는 5번이 같이 SDK 샘플을 분석하는 편이 좋다.

입문자라면 위험 판단도 더 단순하게 시작하자

처음부터 lane geometry를 계산해서 완벽한 ego lane을 구하려고 하지 마.

Level 1 — 가장 먼저 구현

Detection bbox의 하단 중앙 좌표만 사용한다.

         ┌────────┐
         │  CAR   │
         └───●────┘
             ↑
        판단 기준점


┌──────────────────────────┐
│                          │
│          SAFE            │
│                          │
│       ┌────────┐         │
│       │WARNING │         │
│       │        │         │
│       │ DANGER │         │
│       └────────┘         │
│                          │
└──────────────────────────┘

객체 좌표가 중앙 위험 ROI 안으로 들어오면:

if (inside_danger_roi(object))
    state = DANGER;
else if (inside_warning_roi(object))
    state = WARNING;
else
    state = SAFE;

이것부터 성공시킨다.

Level 2 — Freespace 추가
if (inside_freespace(object)) {
    // 실제 주행 가능 영역에 존재
}

그러면

Detection + Freespace 융합

이 된다.

Level 3 — Lane 추가

마지막으로 가능할 때:

if (inside_freespace &&
    inside_ego_lane &&
    inside_warning_roi)
{
    state = WARNING;
}

을 넣는다.

즉,

Detection
    ↓
Detection + ROI
    ↓
Detection + Freespace + ROI
    ↓
Detection + Freespace + Lane + ROI

순서로 개발해야 한다.

처음부터 세 출력을 모두 융합하려고 하면 디버깅이 굉장히 어려워진다.

1주 일정도 5명 입문자 기준으로 다시 잡자

Day 1 — 전원이 SDK와 보드 파악

Trichimera 샘플을 빌드하고 실행한다. 이날은 역할에 너무 얽매이지 말고 다 같이 보는 게 좋다.

최소 성공:

Camera → Trichimera → Wayland

Day 2 — 각자 기능 분리

1번:

Trichimera 출력 분석

2번:

가짜 bbox 좌표로 Risk Engine 개발

3번:

Wayland overlay 분석

4번:

pthread / IPC 작은 테스트 프로그램

5번:

LED/Buzzer GPIO 테스트

이게 중요하다. 서로 기다리면 안 된다.

AI 담당이 아직 결과를 못 뽑았더라도 위험판단 담당은

Detection test_object = {
    .x1 = 200,
    .y1 = 150,
    .x2 = 300,
    .y2 = 350
};

같은 dummy data로 개발하면 된다.

Day 3 — Detection 기반 MVP 통합

Camera
 ↓
Trichimera
 ↓
Detection
 ↓
Risk ROI
 ↓
SAFE/WARNING/DANGER
 ↓
Wayland

여기까지 성공하면 프로젝트는 일단 살아 있다.

이 시점에서 동작 영상을 반드시 찍어둬.

Day 4 — Freespace 추가

Detection
     +
Freespace
     ↓
Risk Engine

객체의 bbox bottom-center가 freespace 안에 있는지 검사한다.

성공하면:

단순 객체 검출이 아니라 주행 가능 영역과 객체 위치를 융합해 위험도를 판단

이라고 설명할 수 있다.

Day 5 — Lane 추가

가능하면:

Detection
     +
Freespace
     +
Lane
     ↓
Risk Engine

까지 간다.

Lane 때문에 하루 종일 막히면 과감히 포기하고 Detection + Freespace 버전으로 돌아간다.

Trichimera에서 Lane Detection 자체가 화면에 표시되는 것과, 그 출력을 우리 C/C++ 코드에서 좌표 형태로 활용하는 건 별개의 문제다.

Day 6 — Linux 요소 + GPIO

기본 시스템이 정상이라면 그때:

pthread
+
IPC
+
LED/Buzzer

를 넣는다.

Linux 담당이 준비해둔 코드를 통합한다.

여유가 있으면 timerfd 정도를 추가한다.

epoll, signalfd는 여기서 시간이 정말 남을 때만 한다.

Day 7 — 기능 추가 금지

테스트
→ 버그 수정
→ 시연 영상
→ 구조도
→ 발표자료

만 한다.
