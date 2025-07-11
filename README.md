# US Accidents Partitioning Performance Comparison

대용량 데이터셋으로 **파티셔닝 후 성능 비교**를 진행한 프로젝트이다.

## 파티션 프로젝트 목적

- 대용량 데이터 파티셔닝을 통해 **쿼리 속도, 스캔량, 처리 비용 비교**
- `LIST`, `RANGE`, 파티셔닝의 실제 성능 차이 실험
- 대용량 데이터 핸들링 및 파티셔닝 실습

---

## 팀원

| <img src="https://avatars.githubusercontent.com/yunkihong-dev" width="140"/> | <img src="https://avatars.githubusercontent.com/shinjunsuuu" width="140"/> | <img src="https://avatars.githubusercontent.com/LeeJoEun-01" width="140"/> |
| :--------------------------------------------------------------------------: | :------------------------------------------------------------------------: | :------------------------------------------------------------------------: |
|                                    홍윤기                                    |                                   신준수                                   |                                   이조은                                   |

---

## 데이터셋

🚑 **[US Accidents (2016 - 2023) - Kaggle](https://www.kaggle.com/datasets/sobhanmoosavi/us-accidents/data)**

- **설명:**
  - 2016년 2월 ~ 2023년 3월, 미국 49개 주 **약 770만 건의 사고 데이터**
  - 교통 센서, 카메라, 도로교통부 API 기반 수집
  - **사고 예측, 지역·날씨·시간별 분석 가능**
- **파이썬을 사용한 전처리:**
  - `TRUE/FALSE` → SQL용 `1/0` 변환
  - 분석 목적에 맞지 않는 컬럼 15개를 제거하여 데이터 크기를 줄이고 처리 효율성을 향상시켰으며, 이를 통해 본격 분석 전 테스트 및 쿼리 성능 검증을 효과적으로 수행할 수 있도록 전처리를 진행

## 데이터 속성명

| 컬럼명            | 설명                                | 컬럼명            | 설명                         |
| ----------------- | ----------------------------------- | ----------------- | ---------------------------- |
| ID                | 사고 고유 식별자                    | Source            | 데이터 수집 소스             |
| Severity          | 사고 심각도 (1: 낮음, 4: 매우 높음) | Start_Time        | 사고 발생 시각               |
| End_Time          | 사고 종료 시각                      | City              | 사고 발생 도시               |
| County            | 사고 발생 카운티                    | State             | 사고 발생 주(State)          |
| Country           | 사고 발생 국가                      | Timezone          | 사고 발생 지역의 표준 시간대 |
| Airport_Code      | 인근 공항 코드                      | Weather_Timestamp | 날씨 데이터 기록 시각        |
| Temperature(F)    | 기온(화씨)                          | Visibility(mi)    | 가시거리(마일)               |
| Wind_Direction    | 바람 방향                           | Wind_Speed(mph)   | 바람 속도(마일/시간)         |
| Precipitation(in) | 강수량(인치)                        | Weather_Condition | 날씨 상태 (Rain, Snow 등)    |
| Amenity           | 사고 인근 편의시설 여부 (0/1)       | Bump              | 과속 방지턱 여부 (0/1)       |
| Crossing          | 횡단보도 여부 (0/1)                 | Give_Way          | 양보 표지 여부 (0/1)         |
| Junction          | 교차로 여부 (0/1)                   | No_Exit           | 출구 없음 구역 여부 (0/1)    |
| Railway           | 철도 교차 여부 (0/1)                | Roundabout        | 로터리 여부 (0/1)            |
| Station           | 역 근처 여부 (0/1)                  | Stop              | 정지 표지 여부 (0/1)         |
| Traffic_Calming   | 교통 진정 시설 여부 (0/1)           | Traffic_Signal    | 신호등 여부 (0/1)            |
| Turning_Loop      | 회전 구간 여부 (0/1)                |                   |                              |

---

<details>
<summary><span style="font-size: 24px; font-weight: bold;">DBeaver에 CSV 파일 Import</span></summary>

1. database 생성
2. database 우클릭 → `Import Data`
3. `Input File` > `Browse` > `.csv` 파일 선택
4. `Tables Mapping` > `Configure`
   - `REAL`-> `FLOAT`
   - 자료 타입이 다르면 여기서 직접 매핑 필요

예시:  
<img width="700" alt="image" src="https://github.com/user-attachments/assets/83d82405-b80b-46dd-ad91-ea2bb5713aae" />

</details>

---

## 📊 파티션 방식

| 기준 | 방식 | 설명 |
| --- | --- | --- |
| 📆 **년도 (Start_Time - YEAR)** | RANGE | 2016~2023 연도별 분리 |
| 🍂 **계절 (Start_Time - MONTH)** | LIST | 월 기준 계절별 분리 (겨울, 봄, 여름, 가을) |
| 🚦 **사고 수준 (Severity)** | LIST | 심각도 기준 분리 (1~4) |

> ### 년도(start_time-**year**) 파티셔닝 SQL

```sql
CREATE TABLE US_Accidents_patitioned_year (
    -- 기존 컬럼들...
    Accident_Year INT GENERATED ALWAYS AS (YEAR(Start_Time)) STORED,
    PRIMARY KEY (ID, Accident_Year)
)
PARTITION BY RANGE (Accident_Year) (
    PARTITION p2016 VALUES LESS THAN (2017),
    PARTITION p2017 VALUES LESS THAN (2018),
    PARTITION p2018 VALUES LESS THAN (2019),
    PARTITION p2019 VALUES LESS THAN (2020),
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION pMax   VALUES LESS THAN MAXVALUE
);
```

> ### 계절(start_time-**month**) 파티셔닝 SQL

```sql
CREATE TABLE US_Accidents2_season_partitioned (
    -- 기존 컬럼들...
    Start_Month INT GENERATED ALWAYS AS (MONTH(Start_Time)) STORED,
        PRIMARY KEY (ID, Start_Month)

)
PARTITION BY LIST (Start_Month)(
PARTITION p_winter VALUES IN (12, 1, 2),
PARTITION p_spring VALUES IN (3, 4, 5),
PARTITION p_summer VALUES IN (6, 7, 8),
PARTITION p_fall VALUES IN (9, 10, 11)
);
```

> ### 사고 수준(**severity**) 파티셔닝 SQL

```sql
ALTER TABLE severity_partitioned_data
PARTITION BY RANGE (Severity) (
    PARTITION p01 VALUES LESS THAN (2),
    PARTITION p02 VALUES LESS THAN (3),
    PARTITION p03 VALUES LESS THAN (4),
    PARTITION p04 VALUES LESS THAN (5),
    PARTITION p_max VALUES LESS THAN MAXVALUE
);
```

---

# 쿼리별 다양한 파티션 성능 비교

<details>
<summary>DBeaver 사용해서 쿼리별 데이터 성능 비교하는 방법</summary>
 - Window > Show View > Query Manager
<img width="658" height="462" alt="image" src="https://github.com/user-attachments/assets/06923af3-d336-47ea-addf-721d6c47ceeb" />

</details>

## 날씨별 심각한 사고(Severtiry>=3) 건수 쿼리 비교

```sql
SELECT 
    Weather_Condition,
    COUNT(*) AS accident_count
FROM
    severity_partitioned_data
WHERE
    Severity >= 3
GROUP BY
    Weather_Condition
ORDER BY
    accident_count DESC;
``` 
<img width="441" height="235" alt="image" src="https://github.com/user-attachments/assets/24e50f60-de5f-425c-bac8-8d27e56a28bf" />

- 원본 데이터 (파티셔닝 진행 X)
  - <img width="1520" height="45" alt="image" src="https://github.com/user-attachments/assets/dc4bc19b-a675-4388-bbbb-7e72b143e997" />

- `Severity` 기준으로 파티셔닝
  - <img width="1517" height="45" alt="image" src="https://github.com/user-attachments/assets/4e4afe76-b158-41cb-b3a1-1bd011af737d" />

- `년도` 기준으로 파티셔닝
  - <img width="1523" height="48" alt="image" src="https://github.com/user-attachments/assets/62d3f50c-0157-4731-9149-dee7b1c66a14" />

- `계절` 기준으로 파티셔닝
  - <img width="1461" height="61" alt="image" src="https://github.com/user-attachments/assets/82b6e806-d54a-4bdf-945e-01131639cd19" />
  
#### 결과 고찰 🔍
> 맑음, 구름 많음 등에서 사고 건수가 높았으며 이는 해당 날씨 조건에서 교통량이 많아 사고가 자주 발생했을 가능성을 시사한다. 비, 눈, 안개에서도 심각한 사고가 다수 발생해 도로 미끄러움과 시야 제한이 사고에 영향을 주었음을 보여주고 있어 악천후 시 사고 건수는 적어도 심각도는 높을 수 있어 안전 운전이 중요하다. 이를 통해 날씨별 사고 대응 정책 수립 및 사고 예방 방안을 마련하는 기반 자료로 활용이 가능하다.

## 코로나 이전, 이후 도시별 사고 건수 쿼리 비교
```sql
SELECT 
  City,
  CASE
    WHEN YEAR(Start_Time) BETWEEN 2016 AND 2019 THEN '2016~2019'
    WHEN YEAR(Start_Time) BETWEEN 2020 AND 2023 THEN '2020~2023'
  END AS 기간구분,
  COUNT(*) AS 사고건수
FROM US_Accidents2
WHERE YEAR(Start_Time) BETWEEN 2016 AND 2023
  AND City IS NOT NULL
GROUP BY City, 기간구분
ORDER BY City, 기간구분;
``` 
<img width="515" height="390" alt="image" src="https://github.com/user-attachments/assets/f23a2faa-ca14-432f-adda-df535c773114" />

- 원본 데이터 (파티셔닝 진행 X)
  - <img width="1523" height="50" alt="image" src="https://github.com/user-attachments/assets/f2761c0c-6e1d-475b-a9cf-a101b023cb75" />

- `년도` 기준으로 파티셔닝
  - <img width="1610" height="45" alt="image" src="https://github.com/user-attachments/assets/8154b456-8e71-4471-b606-0718a3bb08b4" />

#### 결과 고찰 🔍
> 코로나 이전(2016-2019)과 이후(2020-2023) 사고 건수를 도시별로 비교한 결과, 일부 지역은 감소했지만 전반적으로는 증가세를 보였다. 특히 2021년 이후 사회적 거리두기 완화와 이동량 증가로 사고가 다시 급증했으며, 대도시를 중심으로 증가 폭이 커졌다. 이는 팬데믹 이후 배달 수요 증가와 자가용 이동 증가 등의 생활 변화가 주요 원인으로 해석된다. 이러한 결과는 교통안전 정책 및 인프라 개선 방향 설정에 활용될 수 있다.


## 계절별 사고 건수와 평균 기온 쿼리 비교
```sql
SELECT
    CASE
        WHEN Start_Month IN (12, 1, 2) THEN 'Winter'
        WHEN Start_Month IN (3, 4, 5) THEN 'Spring'
        WHEN Start_Month IN (6, 7, 8) THEN 'Summer'
        WHEN Start_Month IN (9, 10, 11) THEN 'Fall'
    END AS Season,
    COUNT(*) AS Accident_Count,
    ROUND(AVG(Temperature_F), 2) AS Avg_Temperature_F
FROM US_Accidents2_season_partitioned
GROUP BY Season
ORDER BY FIELD(Season, 'Winter', 'Spring', 'Summer', 'Fall');
```

<img width="578" height="142" alt="image" src="https://github.com/user-attachments/assets/2990ad18-dfb4-4512-ae4f-1a8837554623" /> 

- 원본 데이터 (파티셔닝 진행 X)
  - <img width="708" height="23" alt="image" src="https://github.com/user-attachments/assets/986b8d91-2d69-46f3-8635-31795dff4b0b" />
- `계절` 기준으로 파티셔닝
  - <img width="709" height="22" alt="image" src="https://github.com/user-attachments/assets/9081b148-6bb5-4c7c-8ad8-8c9067f0c085" />
#### 결과 고찰 🔍
> 기온이 낮은 겨울과 가을에 사고 건수가 높아져 저온 및 기상 악화가 사고 위험도를 증가시키는 경향이 나타났다. 여름철은 기온이 높고 도로 상태가 안정되어 사고 건수가 가장 낮게 나타났다. 봄철은 기온이 낮음에도 여름보다 사고가 많아 계절별 사고 특성이 뚜렷하게 구분되었다. 파티셔닝과 전과 후 결과가 매우 뚜렷해 조회 기준에 따른 파티셔닝이 필요해 보인다는 생각이 들었다.
