# 저장소별 호환성 검토

2026-10-14에 GitHub 원문을 읽어 확인한 내용입니다. 실행·성능·완성된 어댑터 호환성을 검증한 문서는 아닙니다. 규격 후보는 [GPU 배치 입출력 규격 후보](io-spec-candidates.md)를 참고하세요.

## 조사 기준

| 저장소 | 확인한 `main` 커밋 |
|---|---|
| LSM107/GreatKingdom | [770bd77](https://github.com/LSM107/GreatKingdom/commit/770bd77b94598f98d1536a7fce415a2f586e16bc) |
| dtfgzbsfztfg/Great-Kingdom | [1659012](https://github.com/dtfgzbsfztfg/Great-Kingdom/commit/165901210dac6ba08747587b129423b9e7f9dc83) |
| kikuke/GreatKingdomServer | [9d20628](https://github.com/kikuke/GreatKingdomServer/commit/9d20628b1b70efd7ccfc168b6848309c02ab87e9) |
| yumin0544/Alpha-Yoo-for-Great-Kingdom | [0c6150e](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/commit/0c6150e32293e437df6bf11fd4dcbc4c02f321f2) |

## 역할과 연결 대상

| 저장소 | 확인한 역할 | 실제 출력 또는 상태 | 제안하는 연결 방식 |
|---|---|---|---|
| LSM107 | Python/PyTorch AI와 MCTS | 선택 행동 정수, 모델의 로그 확률·가치 | 전체 로봇 어댑터 또는 정책 모델 어댑터 |
| dtfgzbsfztfg | Discord 인간 대국 봇 | 착수 결과 객체, 상태·기보·보드 이미지 | 좌표·명령·상태·결과 변환 |
| kikuke | C++ TCP 방·상태 중계 서버 | 전체 `GameRoomInfo` 구조체 | TCP 서버 게이트웨이 |
| Alpha-Yoo | C++ 엔진, PyTorch 정책, CUDA 배치 탐색 | 행동·정책·가치·탐색 통계 | GPU 로봇 또는 정책 모델 어댑터 |

아래 연결 방식은 설계 제안입니다. Discord 봇과 서버를 자동으로 수를 선택하는 AI 참가자로 등록하지 않습니다.

## LSM107/GreatKingdom

### 실제 계약

- 보드: NumPy `[9,9]`, `0=빈칸, 5=중립, 6=PLAYER1, 7=PLAYER2`.
- 상태: 보드, 현재 플레이어, 연속 패스, 종료 여부, 승자. 집을 재계산하며 영구 소유권과 돌 재고 필드는 없습니다.
- 행동: `row*9+col`, 0기반 `0..80=착수`, `81=패스`.

근거: [game.py 상태 정의](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/game.py#L8-L59), [game.py 상태 전이](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/game.py#L292-L349).

모델 입력은 `float32[N,5,9,9]`입니다. 채널은 PLAYER1 돌, PLAYER2 돌, 중립 돌, PLAYER1 집, PLAYER2 집 순서의 절대 관점입니다. 출력은 `log_probs[N,82]`, `value[N,1]`입니다. 정책은 `log_softmax` 결과이고 가치는 `tanh` 결과입니다. 전체 로봇의 `MCTS.search(GameState)`는 행동 정수 하나를 반환합니다. [model.py 모델·전처리](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/model.py#L29-L93), [model.py search](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/model.py#L283-L289)

### 설계에 반영할 점

보드 칸 코드와 5채널 순서를 변환해야 합니다. 로그 확률을 이미 정규화된 양수 확률로 읽어서는 안 됩니다. 원래 모델은 현재 플레이어 정보를 입력 채널에 포함하지 않으며, 학습 목표와 MCTS 종료 가치의 관점도 동일한 계약이라고 단정하기 어렵습니다. `value`의 관점을 확인하기 전에는 공통 평가값으로 자동 변환하지 않습니다. [학습 목표](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/train_self_play.py#L90-L102), [MCTS 종료 가치](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/model.py#L266-L279)

모델은 배치를 처리하지만 기존 MCTS·게임 진행은 Python/NumPy이고 추론 결과를 `.cpu().numpy()`로 회수합니다. 전체 로봇을 유지하는 CPU 호환 경로와, GPU 배치 모델에 별도 탐색기를 연결하는 경로는 다른 참가 설정입니다. [MCTS 추론 경계](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/model.py#L172-L191)

자기 집 착수 허용, 집 소유권 재계산, 돌 재고 제약과 자충수 즉시 패배 판정의 부재 때문에 Alpha-Yoo 기본 규칙과 같다고 취급할 수 없습니다. 공통 규칙을 적용할 경우 그 차이와 어댑터 제한을 참가자 정보에 기록해야 합니다. [game.py 착수·집 갱신](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/game.py#L157-L205), [game.py 착수 판정](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/game.py#L292-L349)

종료 때 `get_next_state`와 `place_castle`가 차례를 유지·교대하는 방식도 다르므로 차례만으로 승자를 추정하지 않습니다. native 합법 수 함수는 종료 여부를 받지 않고 패스를 허용하므로 어댑터가 종료 마스크를 별도로 처리해야 합니다. [game.py 차례 처리](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/game.py#L157-L187), [game.py 합법 수·전이](https://github.com/LSM107/GreatKingdom/blob/770bd77b94598f98d1536a7fce415a2f586e16bc/game.py#L292-L349)

## dtfgzbsfztfg/Great-Kingdom

### 실제 계약

Node.js/discord.js로 사람이 대국하는 봇입니다. 입력은 `/king-move`, `/king-pass`, `/king-resign` 등이며 정책 모델이나 수 선택 함수는 확인되지 않았습니다. Discord 상대의 `bot` 속성을 검사해 봇 상대를 거부합니다. [commands.js](https://github.com/dtfgzbsfztfg/Great-Kingdom/blob/165901210dac6ba08747587b129423b9e7f9dc83/src/commands.js#L5-L21), [index.js 상대 검사](https://github.com/dtfgzbsfztfg/Great-Kingdom/blob/165901210dac6ba08747587b129423b9e7f9dc83/index.js#L56-L62)

- 보드: JS 숫자 배열 `[9][9]`, `0=빈칸, 1=BLUE, 2=ORANGE, 3=중립`.
- 별도 상태: `territory[9][9]`, `turn`, `passCount`, `finished`, `winner`, `winReason`, `lastMove`, `moveHistory`.
- 좌표: 열 `A..I`, 행 표기는 아래쪽이 1입니다. `row=9-표기행`, `col=문자-A`.
- 변환 예: `A9=(0,0)→0`, `A1=(8,0)→72`, `E5=(4,4)→40`. 초기 중앙 중립 칸인 E5는 좌표 변환이 가능해도 합법 착수는 아닙니다.
- `place(playerColor,r,c)`는 상태를 갱신하고 `legal`, 오류, 종료·승자 등의 객체를 반환합니다. 패스와 기권은 별도 메서드입니다.

근거: [game.js 좌표·상태](https://github.com/dtfgzbsfztfg/Great-Kingdom/blob/165901210dac6ba08747587b129423b9e7f9dc83/src/game.js#L18-L79), [place](https://github.com/dtfgzbsfztfg/Great-Kingdom/blob/165901210dac6ba08747587b129423b9e7f9dc83/src/game.js#L130-L200), [pass/resign](https://github.com/dtfgzbsfztfg/Great-Kingdom/blob/165901210dac6ba08747587b129423b9e7f9dc83/src/game.js#L272-L299).

### 설계에 반영할 점

입출력 변환의 대상은 명령·상태·기보·대국 결과입니다. GPU 배치의 행동 인덱스를 Discord 좌표로 변환하는 표시 경로는 가능하지만 기존 JavaScript 규칙이 자동으로 GPU에서 실행되지는 않습니다. 이미지와 Discord 메시지 생성은 대량 대국의 매 턴 경로에서 제외하는 방향을 제안합니다. 현재 출력은 보드 이미지를 첨부한 Discord 메시지입니다. [embeds.js](https://github.com/dtfgzbsfztfg/Great-Kingdom/blob/165901210dac6ba08747587b129423b9e7f9dc83/src/embeds.js#L15-L29)

영토는 별도 배열에 보존하며, 영토 탐색에서는 중립 칸도 연결 영역으로 다룹니다. 원래 규칙과 GPU 공통 규칙의 일치 여부를 검증해야 합니다. 원래 `territory`를 지우고 보드에서 추정하는 변환은 피합니다. [game.js 영토 탐색](https://github.com/dtfgzbsfztfg/Great-Kingdom/blob/165901210dac6ba08747587b129423b9e7f9dc83/src/game.js#L203-L264)

## kikuke/GreatKingdomServer

### 실제 계약

C++ TCP 서버이며 GPU 엔진이나 AI 정책 출력은 없습니다. 방 갱신 입력은 단일 착수 대신 전체 `GameRoomInfo`입니다. `panel[81]`, `passNum`, `roomStatus`, `roomStatusBefore`, `isCSP1`, `isCSP2` 등이 C++ `int` 필드로 전달됩니다. 방 ID, 참가자 ID 등 연결 관리 정보도 같은 구조체에 있습니다. [ServerInfo.h](https://github.com/kikuke/GreatKingdomServer/blob/9d20628b1b70efd7ccfc168b6848309c02ab87e9/info/ServerInfo.h#L27-L45), [상태 갱신 처리](https://github.com/kikuke/GreatKingdomServer/blob/9d20628b1b70efd7ccfc168b6848309c02ab87e9/handler/GameRoomHandler.cpp#L58-L99)

`panel` 코드표·좌표 방향, `passNum`의 의미, 상태 코드와 `isCSP`의 의미는 서버 저장소만으로 확정하지 못했습니다. 해당 클라이언트의 계약을 확인하기 전에는 값들을 보드 칸·연속 패스 등으로 임의 해석하지 않습니다.

통신은 의존 라이브러리의 header→payload→trailer이며 구조체 바이트를 직접 복사합니다. 네이티브 `int` 크기, 정렬, byte order를 확인해야 하므로 새 공통 이진 규격으로 그대로 채택하지 않습니다. 의존성 조사 기준은 `CppServerTools` 커밋 `2d18975977b25151badb4b62d95d8c35301b6687`입니다. [패킷 정의](https://github.com/kikuke/CppServerTools/blob/2d18975977b25151badb4b62d95d8c35301b6687/packet/basepacket/BasePacketDefine.h#L8-L24), [바이트 복사](https://github.com/kikuke/CppServerTools/blob/2d18975977b25151badb4b62d95d8c35301b6687/packet/basepacket/BasePacket.c#L4-L21)

### 설계에 반영할 점

이 결과물은 로봇 어댑터 대신 TCP 게이트웨이 대상으로 둡니다. 게이트웨이는 `roomID↔game_id`, 원래 패킷·상태 형식, 종료·방 생명주기를 처리하고 대국 판정은 공통 GPU 엔진이 보유하는 방향을 제안합니다.

현재 `BroadCastRoomInfo`는 방과 참가자 정보만 복사하므로 `panel` 등 게임 상태가 초기화된 값으로 방송됩니다. 이 방송을 완전한 최신 대국 상태로 신뢰할 수 없습니다. 또한 전체 스냅숏에서 행동을 변환해야 한다면 이전 상태와 규칙으로 검증해야 하며, 여러 칸의 차이만 보고 착수 하나를 추측해서는 안 됩니다. [방송 처리](https://github.com/kikuke/GreatKingdomServer/blob/9d20628b1b70efd7ccfc168b6848309c02ab87e9/handler/GameRoomHandler.cpp#L277-L303)

## yumin0544/Alpha-Yoo-for-Great-Kingdom

### 실제 계약

정책 입력은 `float32[N,10,9,9]`, 합법 마스크는 `[N,82]`입니다. 10채널은 내 돌, 상대 돌, 중립 돌, 내 집, 상대 집, 선공 차례, 양측 남은 돌 비율, 연속 패스 비율, 합법 착수점입니다. 출력은 `logits[N,82]`, 현재 차례 관점의 `value[N]`입니다. 행동은 `0..80=row*9+col`, `81=패스`입니다. [encoding.py](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/python/kingdom_ai/encoding.py#L9-L87), [model.py](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/python/kingdom_ai/model.py#L30-L37)

기존 GPU 상태는 `int32[N,170]`이며 배열 순서는 81칸의 보드, 81칸의 소유권, 차례, 양측 재고, 패스, 종료 사유, 승자, 포획 수, 예약 칸입니다. `GpuStateBatch.play(actions)`는 장치에서 행동을 적용하고 수락 여부를 반환합니다. GPU PUCT 결과에는 `int64[N]` 행동, 정책·가치·탐색 통계가 있습니다. [GPU 상태 정의](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/python/kingdom_ai/gpu_rules.py#L16-L35), [상태 적용·인코딩](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/python/kingdom_ai/gpu_rules.py#L418-L459), [GPU PUCT 결과](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/python/kingdom_ai/gpu_puct.py#L526-L543)

### 설계에 반영할 점

GPU에서 게임과 탐색을 연결하는 주 경로의 참조 대상입니다. 다만 native `[N,170]` 레이아웃을 전체 공통 규격으로 고정하지 않고 어댑터에서 변환합니다. 종료 사유의 CPU/GPU enum 순서도 다르므로 숫자 복사 대신 의미 기반 매핑을 사용합니다.

LSM107의 절대 5채널과 이 모델의 현재 차례 기준 10채널은 호환 인코딩이 아닙니다. 참가 모델마다 입력 변환기를 등록해야 합니다. GPU 구현은 기본 규칙을 대상으로 하므로 임의의 ruleset을 즉시 지원한다고 선언하지 않습니다.

로컬 `Alpha-Yoo-for-Great-Kingdom`은 읽기 전용입니다. 필요한 어댑터·설계·향후 실행 결과는 `Great-kingdom-battle-interface-API` 안에서 작성합니다.

## 공통 연결 조건

다음 조건을 충족한 결과물만 해당 실행 모드의 참가자로 등록하는 방향을 제안합니다.

1. 원래 상태·행동·정책·가치의 의미와 좌표 방향을 선언합니다.
2. 코드 revision, 모델 구조와 가중치, 탐색·선택 설정을 식별합니다.
3. 같은 규칙에서 평가 가능한지 확인합니다. 정보가 부족하거나 규칙이 다른 상태를 억지로 동일하게 복원하지 않습니다.
4. 행동 선택 로봇, 정책 모델, 게임·서버 연동을 각각의 역할로 등록합니다.
5. 게임·추론·탐색 중 GPU에서 실행하는 범위를 명시합니다. CPU 프로그램에 형식 어댑터만 붙인 결과를 완전 GPU 대국으로 표시하지 않습니다.
6. 어댑터의 좌표, 칸 코드, 패스, 종료, 합법 마스크 변환과 GPU 규칙을 참조 사례로 검증합니다.

이 단계에서는 연결 구조와 필요한 변환을 제안합니다. 실제 봇 연동, 서버 접속, GPU 커널 구현, 1,000대국 벤치마크는 아직 수행하지 않았습니다.
