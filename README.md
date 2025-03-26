# GenAI-Review-Summary
### 프로젝트 목적
- **온라인 쇼핑몰 내 생성형 AI-요약리뷰의 실증적 효과**를 측정하기 위해, 고객들의 **구매 의사결정에 미친 영향을 준실험설계를 통해 인과분석**한 논문.

### 연구 질문
- Q1. 생성형 AI-요약리뷰는 **고객들이 구매까지 걸리는 시간**을 감소시켰을까?
- Q2. 단순히 시간만 감소한 것이 아니라, 실제 **구매 의사결정 정확도**도 증가했을까?
- Q3. 이런 효과는 **어떤 고객에게서** 더 크게 나타날까?

### 프로젝트 배경
- 퍼널 분석 결과, 상품 정보 페이지가 병목 구간임을 확인함.
- 구매까지 걸리는 시간이 길어질수록 구매 전환율 감소 혹은 반품율 증가를 확인함.
- 상품 정보 페이지 개선을 통해 이를 개선해보자.


WITH PRODUCTS AS (
    SELECT PRD_CD
    FROM CDP.SF_CDPDW.D_CHN_PRD_MSTR
    WHERE (CHN_CD = '036' OR (CHN_CD = '031' AND CHN_BRND_NM = 'INNISFREE'))
        AND PRD_CD != 'Z'
), ORDER_CUSTS AS(
SELECT *
    FROM CDP.SF_CDPDW.F_LOG_GA_STLM_PRD_HIST 
    WHERE SITE_ID IN ('UA-110770460-3', 'UA-110770460-1')
          AND STND_YMD BETWEEN '2024-02-01' AND '2024-03-31'
          AND PRD_CD IN (SELECT PRD_CD FROM PRODUCTS)
    ORDER BY STND_YMD
-- INCS_NO 빈 값 채워주기
), NEW_INCS_GA AS (
    SELECT  *
          , COALESCE(INCS_NO, MAX(INCS_NO) OVER (PARTITION BY SESS_ID, GOOGLE_CID)) AS NEW_INCS_NO
    FROM CDP.SF_CDPDW.F_LOG_GA_HIST
WHERE SESS_ID IN (SELECT SESS_ID FROM ORDER_CUSTS)
      AND SITE_ID IN ('UA-110770460-3', 'UA-110770460-1')
      AND STND_YMD BETWEEN '2024-02-01' AND '2024-03-31'
)
-- 조회할 로그 정보
, LOGS AS (
SELECT LOG_DTTM, SITE_ID, WEB_APP_CL_CD, GOOGLE_CID, SESS_ID, INCS_NO, EMP_YN, AGE, SEX_CD, CUST_GRD_NM, PG_NM, PG_TP_VL, ACCM_STAY_TIME_NSS, PG_STAY_TIME, SESS_SN_STAY_TIME
       , SESS_SN, EVNT_NM, EVNT_ACTN, EVNT_LABEL, NEW_INCS_NO, ROW_NUMBER() OVER (PARTITION BY SESS_ID, NEW_INCS_NO ORDER BY LOG_DTTM ASC) AS LOG_ORDER
       , SUM(SESS_SN_STAY_TIME) OVER (PARTITION BY SESS_ID, NEW_INCS_NO) AS SESS_STAY_TIME
FROM NEW_INCS_GA
)
SELECT L.*, ORD.PRD_CD, ORD.POS_NET_PRD_SAL_AMT
FROM LOGS L LEFT JOIN ORDER_CUSTS ORD ON (L.SESS_ID = ORD.SESS_ID) AND (L.NEW_INCS_NO = ORD.INCS_NO) AND (L.GOOGLE_CID = ORD.GOOGLE_CID)
WHERE EVNT_NM IN ('Scroll', 'product', 'checkout1', 'checkout2', 'checkot3', 'checkout4', 'addToCart', 'purchase')
ORDER BY LOG_DTTM, SESS_ID, NEW_INCS_NO, LOG_ORDER






import matplotlib.pyplot as plt
import seaborn as sns

# 1. AGE가 999인 행 제거
df = df[df['AGE'] != 999]

# 2. EMP_YN, SEX_CD의 'Z' 제거
df = df[~df['EMP_YN'].isin(['Z']) & ~df['SEX_CD'].isin(['Z'])]

# 3. PRD_CNT 이상치 시각화
plt.figure(figsize=(8, 5))
sns.boxplot(x=df['PRD_CNT'])
plt.title("Boxplot of PRD_CNT")
plt.show()

# 4. IQR 기반 이상치 제거
Q1 = df['PRD_CNT'].quantile(0.25)
Q3 = df['PRD_CNT'].quantile(0.75)
IQR = Q3 - Q1
lower = Q1 - 1.5 * IQR
upper = Q3 + 1.5 * IQR
df = df[(df['PRD_CNT'] >= lower) & (df['PRD_CNT'] <= upper)]
