# ESS 배터리 수명 예측

초기 100사이클의 충·방전 데이터로 리튬이온 배터리의 최종 사이클 수명(`cycle_life`)을 예측하는 회귀 프로젝트입니다.

## 프로젝트 개요

- 데이터셋: MIT-Stanford Battery Dataset (Severson et al., 2019)
- 학습 데이터: Batch 1 (`2017-05-12`), 46개 셀
- 최종 평가: Batch 2 (`2018-02-20`), 정답이 있는 39개 셀
- 태스크: Regression
- Target: `cycle_life`
- 주요 평가 지표: MAPE
- 원논문 비교 기준: MAPE 9.1%

Batch 2 전체 47개 중 `cycle_life`가 없는 VarCharge·SlowCycle 8개 셀은 오차를 계산할 수 없어 성능 평가에서 제외했습니다.

## 빠른 실행 순서

### 1. 원본 데이터 배치

Git에는 원본 데이터가 포함되지 않습니다. 다음 MAT 파일을 `data/archive/`에 배치합니다.

```text
data/archive/2017-05-12_batchdata_updated_struct_errorcorrect.mat
data/archive/2018-02-20_batchdata_updated_struct_errorcorrect.mat
```

### 2. 환경 설정

```bash
git clone https://github.com/BigMacHun-del/mini_project.git
cd mini_project
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook
```

### 3. 노트북 순차 실행

| 순서 | 노트북 | 역할 | 주요 생성 결과 |
|---:|---|---|---|
| 1 | `00_data_preparation.ipynb` | Batch 1 원본 파싱과 초기 피처 생성 | `data/processed/batch1_*` |
| 2 | `01_day1_eda_model_strategy.ipynb` | EDA, 가설 검정, 피처·모델 전략 | `results/feature_selection.csv` |
| 3 | `02_batch2_preparation.ipynb` | Batch 2에 동일한 전처리 적용 | `data/processed/batch2_*`, `results/batch_profile.csv` |
| 4 | `03_day2_modeling_evaluation.ipynb` | Ridge 학습, Valid, Batch 2 최종 평가 | 성능·예측 CSV, 저장 모델 |

노트북은 반드시 1→4 순서로 실행합니다. `*.executed.ipynb`에서는 데이터를 다시 실행하지 않고도 저장된 결과를 확인할 수 있습니다.

## 파일 구조

```text
mini_project/
├── data/                                  # Git 제외
│   ├── archive/                           # 원본 MAT 파일
│   └── processed/                         # 생성된 parquet/npz
├── notebooks/
│   ├── 00_data_preparation.ipynb
│   ├── 01_day1_eda_model_strategy.ipynb
│   ├── 02_batch2_preparation.ipynb
│   └── 03_day2_modeling_evaluation.ipynb
├── results/
│   ├── feature_selection.csv
│   ├── model_performance.csv
│   ├── batch2_predictions.csv
│   ├── batch2_subgroup_performance.csv
│   ├── ridge_coefficients.csv
│   └── models/
├── output/pdf/
│   └── DS-MINI-Design-울산캠퍼스_3반-김대훈.pdf
├── requirements.txt
└── README.md
```

`*.executed.ipynb`는 실행 결과가 저장된 노트북입니다.

## EDA 핵심 결과

### Cycle Life 분포

- Batch 1 수명 범위: 534~1,227 cycles
- 550 cycle 기준 단수명:장수명이 1:45로 심각하게 불균형
- 이진 분류보다 연속형 수명을 모두 사용하는 회귀를 선택

### 초기 열화와 ΔQ(V)

- `dQ_log_var`가 수명과 가장 강한 단조 관계: Spearman ρ=-0.871
- `QD_slope`: Spearman ρ=0.613
- ΔQ 파생 피처는 |ρ|=0.97~1.00의 높은 중복을 보여 `dQ_log_var`로 통합
- Knee point는 수명 후반 정보를 포함하므로 모델 입력에서 제외

### 충전 조건

- `c_rate_1`과 수명: Spearman ρ=-0.483
- 충전 정책별 수명 차이: Kruskal-Wallis H=40.15, p=0.010
- 정책별 표본이 1~3개로 작아 인과가 아닌 탐색적 관련성으로 해석

## 피처 엔지니어링

| 피처 | 유형 | 정의 | 선정 근거 |
|---|---|---|---|
| `dQ_log_var` | 파생 | `log10(Var(Qdlin_100-Qdlin_10))` | 수명과 가장 강한 관계, 곡선 차원 축소 |
| `QD_slope` | 파생 | `(QD_100-QD_10)/90` | 초기 방전 용량 변화율 |
| `chargetime_10` | 원본 추출 | Cycle 10 충전시간 | 초기 충전 거동 |
| `c_rate_1` | 원본 추출 | 충전 정책의 1차 C-rate | 초기 충전 스트레스 |

## Modeling

### 검증 설계

- Batch 1 Train·Valid를 `charging_policy` 단위로 분리
- Train 성능은 중첩 GroupKFold로 측정
- 각 fold의 전처리와 표준화는 학습 부분에서만 fit
- Batch 2는 모델·피처·`alpha`를 고정한 뒤 최종 평가

### 모델 선택

- 기준선: Linear Regression
- 우선 모델: Ridge Regression
- 최종 `alpha`: 3.1623
- 선택 이유:
  - ΔQ 피처 간 다중공선성에 대한 L2 규제
  - 46개 소표본에서 계수 분산과 과적합 완화
  - Linear Regression 대비 Group CV MAPE가 9.48%에서 9.03%로 개선

## 성능 결과

| 구분 | MAPE (%) | 비고 |
|---|---:|---|
| Train (Batch 1 CV) | 9.03 | Nested GroupKFold, fold MAPE 8.96 ± 2.21 |
| Valid (Batch 1 Hold-out) | 11.95 | 5개 충전 정책 Hold-out |
| Test (Batch 2) | 46.78 | 정답이 있는 39개 셀, 최종 평가 |
| Gap (Train-Valid) | +2.92 | Batch 1 내 과적합 가능성 |
| Gap (Valid-Test) | +34.83 | 큰 배치 일반화 저하 |
| Gap (Target-Test) | +37.68 | 원논문 Target 9.1% 대비 |

Batch 1에서는 Ridge가 기준선보다 안정적이었지만 Batch 2에서 크게 악화됐습니다. 원논문 9.1%와 데이터 분할, 피처, 검증 설계가 다르므로 단순한 재현 실패로 해석하지 않습니다.

## 오류 분석

- Batch 1 `cycle_life` 중앙값: 858.5
- Batch 2 평가 셀 중앙값: 472.0
- Batch 2 예측 중앙값: 약 744.4
- Batch 2 일반 구조 30개 셀: MAPE 56.44%
- Batch 2 `newstructure` 9개 셀: MAPE 14.57%
- Batch 1 피처 범위를 벗어난 Batch 2 셀: 23/39, MAPE 54.16%

가장 큰 오차는 수명이 392~474사이클인 Batch 2 일반 구조 셀을 650~815사이클로 과대 예측한 경우에서 발생했습니다. Batch 1에서 학습한 피처-수명 보정 관계가 Batch 2 일반 구조 셀에 그대로 적용되지 않은 것이 주요 원인으로 보입니다.

## ESS 도메인 해석

### 활용 가능성

- 초기 운전 데이터를 활용한 수명 위험 셀 조기 선별
- 배터리 팩 구성 전 품질 관리
- 예상 교체 시점과 유지보수 우선순위 설정

### 한계와 개선 방향

- 실험실 단일 셀 데이터로 실제 BESS의 온도·부하·셀 불균형을 미반영
- 충전 정책과 배치 구조가 달라질 때 예측 보정이 크게 변함
- 실제 배포 전 다양한 셀 구조와 운전 조건을 포함한 외부 검증 필요
- 배치 표시자, 구조 정보, 온도·내저항 변화 등을 포함한 추가 모델 검토 필요

## 실행 순서

1. `00_data_preparation.ipynb`
2. `01_day1_eda_model_strategy.ipynb`
3. `02_batch2_preparation.ipynb`
4. `03_day2_modeling_evaluation.ipynb`

## 참고문헌

- Severson, K. A. et al. (2019). Data-driven prediction of battery cycle life before capacity degradation. *Nature Energy*, 4, 383‑391.

## 팀 구성

- 울산캠퍼스 3반 김대훈: EDA, 피처 엔지니어링, 모델 개발, Batch 2 성능 평가
