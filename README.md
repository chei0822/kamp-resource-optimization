# KAMP 제조 전력 사용량 예측 및 피크 관리

제조 공장의 생산·환경·인력 데이터로 **15분 단위 전력 사용량을 예측**하고, **피크 전력 위험 구간을 탐지**한 뒤 부하 이동 시뮬레이션으로 자원 최적화 가능성을 검토했다. 

## 프로젝트 개요

| 항목 | 내용 |
|---|---|
| 목표 | 15분 단위 전력 사용량(kW) 예측 및 피크 구간 식별 |
| 데이터 | `okm_augumented_2021.csv` (2021.01.01 ~ 2021.09.14, 시간 단위 6,168행 × 18열) |
| 최종 모델 | HistGradientBoostingRegressor + Feature Set B |
| 검증 방식 | 월별 Expanding Window (2021년 4~8월) |

## 저장소 구조

```
kamp-resource-optimization/
├── EDA.ipynb           # 데이터 탐색, 품질 점검, 전처리 기준 정리
├── Final_Model.ipynb   # 피처 엔지니어링, 모델 비교, 검증, 피크·최적화 분석
├── .gitignore
└── README.md
```


## 데이터

- **전력**: 시간당 15분 간격 측정값 4개(`15분`, `30분`, `45분`, `60분`)와 평균
- **생산**: `생산량`
- **기상**: `기온`, `풍속`, `습도`, `강수량`
- **운영**: `공장인원`, `인건비`, 계절별 전기요금
- **시간**: `날짜`, `시간`, 요일, 일, 월

## EDA 주요 결과

- 전체 257일, 날짜별 24개 레코드로 구성
- 결측치: 풍속 3건(0.049%), 강수량 1건(0.016%), 공장인원 17건(0.276%)
- 7월 13일, 15일에 시간 값 오류 존재(최대 188)
- 생산량의 43.1%가 0(비가동 시간으로 추정), 전력은 0이 0.3%, 범위 0~222

## 전처리

1. 7월 13~15일 타임스탬프를 원본 행 순서 기준으로 복원
2. 풍속 결측치는 시간 기반 보간
3. 강수량·공장인원 결측치는 0으로 대체
4. 4개 전력 컬럼을 펼쳐 시간 단위 → 15분 단위로 변환 (6,168행 → 24,672행)
5. 1주일치(672 시점) lag 확보를 위해 앞부분 672행 제거

전력·생산량 값 자체는 수정 x

## 피처 엔지니어링

| Set | 피처 수 | 구성 |
|---|---|---|
| A | 22 | 전력 lag(1/2/4/8/96/672), rolling 평균·최대·표준편차(4/24/96), 전력 변화량, 생산량·생산량 변화·로그 생산량 |
| B | 26 | A + 캘린더 피처(시, 15분 위치, 요일, 주말 여부) |
| C | 31 | B + 요일×시간 상호작용, 일·월 주기 sin/cos 인코딩 |

## 모델링

**비교 모델**
- Baseline: Persistence(직전 시점 전력), Weekly Naive(1주 전 동시각)
- Ridge Regression (alpha=10.0, 스케일링 + 원-핫 인코딩)
- HistGradientBoostingRegressor (Set A / B / C)

**최종 모델: HGB_B**

```python
HistGradientBoostingRegressor(
    max_iter=160,
    learning_rate=0.06,
    max_leaf_nodes=15,
    min_samples_leaf=40,
    l2_regularization=1.0,
    random_state=42,
)
```

**검증 방식**
- 2021년 4~8월 각 월을 테스트로 두고, 그 이전 전체 데이터로 학습하는 Expanding Window
- 피크 기준: 학습 구간 전력의 95번째 백분위수
- 지표는 월별로 계산 후 평균

## 결과 (4~8월 평균)

| 지표 | 값 |
|---|---|
| MAE | 5.31 kW |
| RMSE | 7.96 kW |
| 요일별 macro MAE | 5.32 kW |
| 피크 구간 MAE | 9.52 kW |
| 피크 과소예측 | 8.89 kW |
| 피크 Recall | 0.463 |

전체 오차는 안정적이지만 피크 구간에서는 과소예측 경향이 있고, 피크 Recall이 0.463으로 개선 여지가 있다. 

## 추가 분석

`Final_Model.ipynb` 실행 시 아래 폴더에 결과가 생성

| 폴더 | 내용 |
|---|---|
| `kai_models/` | 월별 검증 결과, 모델 파일 |
| `kai_peak/` | 피크 전력 분석 |
| `kai_hour/` | 1시간 단위 예측 결과 |
| `kai_optimization/` | 가상 부하 이동(load shifting) 시뮬레이션 |
| `kai_followup/` | 오차 분석 |
| `kai_closeout/` | 요약 리포트 |

## 실행 환경

- Python 3.12
- pandas 2.2.3, scikit-learn 1.8.0, SciPy 1.17.0
- numpy, joblib, matplotlib

```bash
pip install -r requirements.txt
jupyter notebook
```

`EDA.ipynb` → `Final_Model.ipynb` 순서로 실행