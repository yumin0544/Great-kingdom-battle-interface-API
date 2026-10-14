# 첫 규칙 프로파일: gk-alpha-default v1

사용자가 2026-10-14에 첫 공통 판정 규칙으로 Alpha-Yoo 기본 규칙을 참조한 프로파일을 선택했습니다. 이 문서는 **구현 기준으로 선정된 규칙**이며, CPU/GPU 구현의 완료를 뜻하지 않습니다.

- `ruleset_id`: `gk-alpha-default`
- `ruleset_version`: `1`
- 설정 값: [프로파일 JSON](../../configs/rulesets/gk-alpha-default-v1.json)
- 참조 revision: [Alpha-Yoo 0c6150e](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/commit/0c6150e32293e437df6bf11fd4dcbc4c02f321f2)

원본 저장소의 이후 변경은 자동 반영하지 않습니다. 기본 규칙 외의 중립 돌 이동·제거, 자충수 금지 등 변형은 이 프로파일에 포함하지 않습니다.

## 초기판과 행동

보드는 9×9이며 선공은 흑/파랑, 후공은 백/주황입니다. 양측 재고는 각각 41개이고 중앙 `(row=4,col=4)`에 파괴 불가능한 중립 돌을 둡니다. 나머지는 빈칸, 소유권은 미소유, 차례는 선공, 연속 패스는 0입니다.

좌표는 0기반 행·열이고 착수 행동은 `row*9+col`의 `0..80`, 패스는 `81`입니다. 연결은 상하좌우만 인정합니다. 기권은 행동 공간에 추가하지 않고 대회 운영의 별도 status로 처리합니다.

근거: [Board 정의](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/engine/include/board/Board.h), [GameRules 기본값](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/engine/include/board/State.h#L14-L20).

## 착수·포획·자충수

착수는 빈칸이고 양측 어느 집으로도 확정되지 않은 곳에만 허용합니다. 해당 플레이어의 재고가 남아 있어야 합니다. 거부된 착수는 보드·소유권·재고·패스·차례·종료 상태를 보존합니다.

착수 후 인접한 상대 연결 무리 중 활로가 없는 무리를 먼저 판정합니다. 포획되면 해당 상대 돌을 제거하고 착수자 승리로 즉시 종료합니다. 양쪽 동시 포위도 상대 포획이 우선입니다. 중립 돌은 포획 대상이 아닙니다.

상대 포획 없이 착수자의 연결 무리 활로가 0이면 자충수입니다. 이 착수는 **합법 수로 수락한 뒤 착수자 패배**로 종료하며 착수한 돌은 유지합니다. 합법 마스크에서 자충수라는 이유만으로 제외하지 않습니다.

활로는 보드상의 빈칸을 기준으로 셉니다. 집 소유권 때문에 착수할 수 없는 빈칸도 활로 계산에서는 빈칸이며, 소유권 필터를 별도로 추가하지 않습니다. 이는 다른 구현의 활로 규칙과 구별해야 하는 참조 동작입니다.

수락된 착수는 재고를 1개 사용하고 패스 횟수를 0으로 만듭니다. 차례는 종료 여부와 관계없이 상대에게 바뀝니다. 승자는 종료 결과로 읽으며 차례로 추정하지 않습니다.

근거: [착수·포획·자충수 전이](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/engine/src/State.cpp#L73-L166), [활로 계산](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/engine/src/Board.cpp#L131-L146).

## 집 소유권

양측을 각각 기준으로 자기 돌과 중립 돌을 벽으로 두고, 빈칸과 상대 돌을 함께 탐색합니다. 영역에 상대 돌이 있거나 자기 돌 경계가 없으면 집이 아닙니다. 보드 가장자리 0·1·2·3개 변을 사용하는 집은 허용하고 4개 변을 모두 사용하는 영역은 제외합니다. 집으로 기록하고 점수에 포함하는 것은 빈칸뿐입니다.

소유권은 보드와 별도로 보존합니다. 새 집은 미소유 칸에만 추가하며 기존 소유권을 덮어쓰지 않습니다. 정상 진행 중 착수 후 집을 갱신합니다. 패스와 포획·자충수로 종료된 착수에서는 새 집을 추가하지 않습니다.

따라서 진행 중 보드만으로 상태를 복원하지 않습니다. 소유권·재고·연속 패스·차례·종료 상태를 함께 저장해야 합니다.

근거: [영역 판정](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/engine/src/Board.cpp#L149-L194), [소유권 유지](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/engine/src/State.cpp#L49-L70).

## 패스·종료·점수

진행 중 패스는 재고와 무관하게 허용합니다. 패스 때 연속 횟수를 1 늘리고 차례를 바꾸며, 2회 연속 패스하면 종료합니다. 거부된 착수는 연속 패스를 끊지 않습니다.

점수는 각 플레이어 소유권으로 확정된 현재 빈칸 수입니다. 선공 점수에서 후공 점수를 뺀 값이 3 이상이면 선공 승리, 나머지는 후공 승리이며 무승부는 없습니다. 포획·자충수 종료의 승자는 이 점수 비교로 뒤집지 않습니다.

종료 상태의 합법 마스크는 모두 거짓이며 이후 게임 행동은 거부합니다. 기권·시간초과·수 제한·실행 오류의 처리와 집계는 게임 규칙 외의 대회 설정에 둡니다.

근거: [패스와 종료 입력](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/engine/src/State.cpp#L92-L105), [점수 승자](https://github.com/yumin0544/Alpha-Yoo-for-Great-Kingdom/blob/0c6150e32293e437df6bf11fd4dcbc4c02f321f2/engine/include/board/GameResult.h#L9-L19).

## CPU/GPU 구현의 검증 기준

같은 초기 상태와 행동열에서 보드, 영구 소유권, 차례, 재고, 연속 패스, 합법 마스크, 수락 여부, 종료 사유, 승자, 점수, 포획 수를 비교합니다.

자충수 수락·패배, 상대 포획 우선, 패스 뒤 거부된 착수, 재고 소진 후 패스, 종료 뒤 입력, 소유권 칸의 활로, 포획 종료 때 새 집을 추가하지 않는 동작을 고정 사례로 포함합니다.

첫 규칙 선택은 완료됐습니다. 다음 작업은 [구현 단계 설계](../implementation-plan.md)의 나머지 P0 계약·환경 검증과 P1 CPU 참조 구현입니다.
