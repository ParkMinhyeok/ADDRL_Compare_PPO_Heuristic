# 에지 컴퓨팅 작업 오프로딩 최적화: PPO vs 휴리스틱

## 개요 및 실험 목표

본 연구의 핵심 목표는 **에지 컴퓨팅 환경의 부하를 동적으로 고려하여, 클라이언트와 서버 간의 작업 분배를 최적화하는 지능형 오프로딩 정책을 개발**하는 것입니다.

사용자의 요청이 계속해서 발생하는 동적인 환경에서 한정된 자원을 가진 에지 서버의 과부하를 방지하고 전체 시스템의 처리량을 극대화하기 위해, **강화학습(PPO) 에이전트**를 학습시킵니다. 그리고 이 에이전트의 성능을 규칙 기반 **휴리스틱 모델**과 비교하여, 강화학습 기반 정책의 효율성과 우수성을 검증하고자 합니다.

---

## 강화학습 환경 설계

에이전트가 최적의 정책 $\pi(a|s)$을 학습할 수 있도록, 시뮬레이션 환경의 **상태(State)**, **행동(Action)**, **보상(Reward)** 을 다음과 같이 설계했습니다.

### 1. 상태 (State)
상태 `S_t`는 에이전트가 다음 행동을 결정하는 데 필요한 모든 핵심 정보를 포함합니다. 현재의 단순한 상태뿐만 아니라, 행동의 미래 가치를 예측하는 데 도움이 되는 정보를 추가하여 상태 공간을 구성했습니다.

> **`상태 벡터 = [서버 대기 시간, 작업 큐 길이]`**

- **`서버 대기 시간`**: 에지 서버가 유휴 상태가 될 때까지 남은 시간. 오프로딩의 '비용'을 판단하는 핵심 지표입니다.
- **`작업 큐 길이`**: 서버에서 처리를 기다리는 작업의 수. 시스템의 전반적인 부하를 나타냅니다.

### 2. 행동 (Action)
행동 `A_t`는 대기 큐의 맨 앞 작업에 대한 **분할 지점(Split Point)** 을 결정하는 것입니다. 액션 공간은 **11개의 이산적인 행동**으로 정의됩니다.

> **`a ∈ {0, 1, ..., 10}`**

각 정수는 작업의 분할 지점을 의미합니다.
예를 들어, 행동 `0`은 `0 ~ 10`, `3`은 `3 ~ 39`, `10`은 `100 ~ 106`(Darknet의 마지막)레이어까지 클라이언트에서 처리하고 나머지를 서버로 오프로딩하라는 의미입니다.

### 3. 보상 (Reward)
보상 `R_{t+1}`은 에이전트의 행동이 얼마나 효율적이었는지를 나타내는 직접적인 피드백입니다. 시스템의 처리량을 높이고 지연 시간을 줄이는 방향으로 학습을 유도하기 위해, **작업의 반환 시간(Turnaround Time)에 반비례**하도록 보상을 설계했습니다.

> **보상 함수**: $R = \frac{c}{\text{Turnaround Time} + \epsilon}$
> - **`Turnaround Time`**: `(작업 완료 시간) - (작업 도착 시간)`

이 보상 함수를 통해 에이전트는 단순히 작업을 완료하는 것을 넘어, **최대한 빨리** 완료시키는 행동을 하도록 학습합니다.

---

## 비교 모델

- **PPO 에이전트** : 위에서 설계된 강화학습 환경과 상호작용하며 장기적인 보상 합을 극대화하는 정책을 학습합니다.
- **휴리스틱 모델** : "서버 대기 시간"이라는 단기적 정보만을 사용하여 탐욕적(Greedy)으로 분할 지점을 결정하는 규칙 기반 모델입니다.

---

## 프로젝트 구조
```bash
├── dataset/
│   ├── client_times.txt
│   └── server_times(x10).txt  #서버 처리 시간을 10배 늘림 
│
├── main_comparison.py
├── environment.py
├── ppo_agent.py
├── config.py
├── time_models.py
├── data_loader.py
└── README.md
```

---

## 설치 및 실행 방법

### 1. 요구사항 설치
```bash
pip install torch numpy matplotlib
```

### 2. 실험 실행
```bash
python main_comparison.py
```
PPO 에이전트 학습 후 휴리스틱 모델이 실행되며, 최종적으로 두 모델의 성능 비교 그래프가 출력됩니다.

### 결과 분석
실험 결과, PPO 에이전트는 휴리스틱 모델보다 **더 높은 처리량(완료된 작업 수)** 을 달성했습니다.
<img width="1400" height="600" alt="Figure_1" src="https://github.com/user-attachments/assets/b30648fe-0bf5-4f14-9c47-de75994b75c9" />


## 최종 실험 결과 비교
| 지표                 | PPO 에이전트 | 휴리스틱 모델 |
| :------------------- | :-----------: | :-----------: |
| 완료된 총 작업 수   | **138 개** | 129 개        |
| 평균 응답 시간       | **131.5979 초** | 131.69 초     |
- 이는 PPO 에이전트가 상태 정보를 바탕으로 단기적인 상황뿐만 아니라 장기적인 시스템 효율성까지 고려하는, 우수한 오프로딩 정책을 학습했음을 의미합니다.

결론적으로, 강화학습은 복잡하고 동적인 에지 컴퓨팅 환경에서 사전에 정의된 규칙보다 더 효과적인 최적화 해법을 제공할 수 있음을 보여줍니다.

