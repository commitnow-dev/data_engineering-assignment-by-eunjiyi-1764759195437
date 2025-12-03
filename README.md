
# Apache Airflow 로컬 실행 & DAG 구현 과제 - 초급자용

## ✏️ 과제의 개요

| 카테고리 | 난이도 | 제한시간 |
|----------|--------|----------|
| data_engineering | easy | 14d |

### 💻 과제에서 요구하는 개발언어

- sql
- airflow
- python


## 📜 과제의 내용

> 과제 설명과 요구사항을 참고하여, 구현해 주세요.

### 👀 과제의 설명

## 1. 과제 목적

이 과제는 **Apache Airflow 환경을 직접 구성하고, 간단한 ETL 파이프라인 DAG를 설계·구현하는 능력**을 확인하기 위한 과제입니다.

특히 다음을 보고자 합니다.

* 로컬 개발 환경에서 Airflow를 설치/실행하는 능력&#x20;
* DAG 구조 설계 및 Task 의존성 설계 능력&#x20;
* PythonOperator를 활용한 간단한 데이터 처리 로직 구현 능력&#x20;
* 코드를 정리하고 README로 설명하는 문서화 능력&#x20;

Airflow 경험이 많지 않더라도, **공식 문서나 검색을 스스로 참고하면서 끝까지 해결하려는 태도**를 중요하게 봅니다.

***

## 2. 과제 시나리오

회사 내부의 간단한 주문 데이터(`orders_raw.csv`)를 매일 집계하는 **일일 주문 요약 파이프라인**을 만든다고 가정합니다.

동봉된 CSV 파일 `orders_raw.csv`에는 다음과 같은 컬럼이 있다고 가정합니다.

* `order_id` (문자열): 주문 ID&#x20;
* `order_date` (YYYY-MM-DD 형식 문자열): 주문일&#x20;
* `customer_id` (문자열): 고객 ID&#x20;
* `country` (문자열): 국가 코드 (예: KR, US 등)&#x20;
* `product_category` (문자열): 상품 카테고리&#x20;
* `quantity` (정수): 수량&#x20;
* `price` (실수): 단가&#x20;

이 데이터를 이용해, **일자별 매출 요약 결과를 만드는 Airflow DAG**를 구현해 주세요.

​

## `orders_raw.csv` (예시/샘플)

```
order_id,order_date,customer_id,country,product_category,quantity,price
O001,2024-01-01,C001,KR,TV,1,1000.0
O002,2024-01-01,C002,US,REFRIGERATOR,2,800.0
O003,2024-01-02,C001,KR,TV,1,1200.0
O004,2023-12-31,C003,KR,WASHER,1,700.0
O005,,C004,KR,TV,1,900.0
```

​

***

## 3. 환경 관련 안내 (Windows 기준)

* 운영체제: Windows 10 / 11&#x20;
* 과제는 **Windows 로컬 환경에서 작업**해 주세요.&#x20;
* Airflow 실행 방법은 자유입니다. 예를 들어,&#x20;

  * Docker Desktop + 공식 Airflow docker-compose 예제&#x20;
  * WSL2(Ubuntu) 안에서 Airflow 설치 후 실행&#x20;
  * Astronomer CLI 등&#x20;
* 어떤 방식을 선택했는지, 설치/실행 방법은 **README에 정리**해 주세요.&#x20;

> ⚠️ Airflow는 리눅스/맥 기반 가이드가 많기 때문에, Windows 사용 시 Docker 또는 WSL2를 활용하는 경우가 많습니다.
> &#x20;본인이 편한 방식을 선택해도 괜찮습니다. 단, **최종적으로 Airflow Web UI에 접속 가능한 상태**가 되어야 합니다.

***

## 4. 구현해야 할 Airflow DAG 요구사항

​

### 4-0. 전체 구조 (예시)

**대략 아래와 비슷한 구조**면 좋아요.

```
project_root/
├─ dags/
│  └─ orders_daily_summary.py
├─ data/
│  ├─ orders_raw.csv          # 원천 데이터 (동봉 파일)
│  └─ output/
│     └─ orders_summary.csv   # 파이프라인 결과
├─ README.md
└─ screenshots/
   ├─ dag_list.png
   └─ dag_run_success.png
```

* 경로/폴더명은 달라도 괜찮지만,&#x20;

  * DAG 파일이 `dags/` 아래 있고&#x20;
  * raw / output 파일 구조가 논리적으로 나눠져 있고&#x20;
  * README / 스크린샷이 포함되어 있는 구조라면 좋겠어요.

​

### 4-1. DAG 기본 설정

아래 요구사항을 만족하는 하나의 DAG를 만들어 주세요.

* `dag_id`: `orders_daily_summary`&#x20;
* DAG 설명(`description`): “일일 주문 요약 집계 파이프라인” 정도로 간단히 작성&#x20;
* `start_date`: 과거 날짜(예: `2024-01-01`)로 고정&#x20;
* `schedule_interval`: 매일 1회 실행(예: `"0 9 * * *"` 또는 `"@daily"` 등)&#x20;
* `catchup=False`&#x20;
* `default_args`에 최소한 다음 항목 포함&#x20;

  * `retries` (예: 1\~2회)&#x20;
  * `retry_delay` (예: 5분 이상)&#x20;

DAG와 Task 정의는 \*\*Python 코드(예: `dags/orders_daily_summary.py`)\*\*로 작성합니다.

***

### 4-2. Task 구성 요구사항

최소 **4개 이상의 Task**를 포함해 주세요.
&#x20;아래는 권장 구조이며, 이름과 세부 구현은 약간 달라도 괜찮습니다.

#### (1) `check_raw_file` – 원천 데이터 존재 여부 확인

* 역할:&#x20;

  * `orders_raw.csv` 파일이 특정 디렉토리에 존재하는지 확인합니다.&#x20;
  * 파일이 없을 경우 에러를 발생시켜 DAG가 실패하도록 합니다.&#x20;
* 구현 예:&#x20;

  * `PythonOperator` 또는 `BashOperator` 사용 (택 1)&#x20;
  * 경로 예시: `/opt/airflow/data/orders_raw.csv` 또는 Windows 환경에 맞는 경로&#x20;

#### (2) `transform_orders` – 데이터 필터링 및 집계

* 역할:&#x20;

  * `orders_raw.csv`를 읽어서 다음 처리를 수행합니다.&#x20;

    * `order_date`가 유효한 행만 사용 (비어 있거나 잘못된 형식은 제외)&#x20;
    * 필요하다면 특정 기간 이후(예: 2024-01-01 이후) 데이터만 사용&#x20;
    * 일자별(`order_date`)로 아래 항목을 집계&#x20;

      * `total_sales`: `quantity * price` 합계&#x20;
      * `order_count`: 주문 건수&#x20;
  * 집계 결과를 pandas DataFrame(또는 파이썬 리스트/딕셔너리) 형태로 메모리에 저장&#x20;
* 구현:&#x20;

  * `PythonOperator` 사용&#x20;
  * `pandas` 사용 여부는 자유지만, 사용하면 가독성이 좋습니다.&#x20;
* 로그:&#x20;

  * Airflow 로그에&#x20;

    * 전체 원천 행 수&#x20;
    * 필터링 후 남은 행 수&#x20;
    * 집계된 날짜 수
      &#x20;를 출력해 주세요.&#x20;

#### (3) `save_summary` – 결과 저장

* 역할:&#x20;

  * `transform_orders`에서 만든 집계 결과를 **새 CSV 파일**로 저장&#x20;
  * 예: `orders_summary.csv`라는 이름으로 `data/output` 디렉토리에 저장&#x20;
* 구현:&#x20;

  * `PythonOperator` 사용&#x20;
  * 디렉토리가 없으면 생성하도록 처리해도 좋습니다.&#x20;
* 출력 형식:&#x20;

  * 최소 컬럼: `order_date`, `total_sales`, `order_count`&#x20;

#### (4) `notify` – 완료 알림 (간단 로그)

* 역할:&#x20;

  * 파이프라인이 성공적으로 끝났다는 메시지를 로그에 남깁니다.&#x20;
  * 실제 이메일/슬랙 연동까지는 필요 없고, `print` 또는 `logging`으로 메시지를 남기면 됩니다.&#x20;
* 구현:&#x20;

  * `PythonOperator` 또는 `DummyOperator`(Airflow 2.7 이전) / `EmptyOperator` 사용&#x20;

#### (5) Task 의존성

* 아래와 같이 순서가 명확하게 보이도록 의존성을 설정해 주세요.&#x20;

```
check_raw_file → transform_orders → save_summary → notify
```

***

### 4-3. 코드 스타일 및 구조

* Airflow 2.x 기준 모듈 사용&#x20;

  * (예시) `from airflow.operators.python import PythonOperator`&#x20;
* 함수/Task 이름은 역할을 잘 드러내도록 작성&#x20;

  * 예: `def _transform_orders(**context):`&#x20;
* 경로/파일명 등은 코드 상단에 상수로 분리해 두면 가독성이 좋습니다.&#x20;
* 가능한 한 **하드코딩을 줄이고**, 최소한 디렉토리 경로나 파일명 정도는 변수로 관리해 주세요.&#x20;

***

## 5. 제출물

과제 제출 시 아래 항목을 하나의 압축파일(zip)로 제출해 주세요.
&#x20;(또는 GitHub Repository 링크를 제출해도 됩니다.)

1. **DAG 코드 파일**&#x20;

   * 예: `dags/orders_daily_summary.py`&#x20;
2. **데이터 파일**&#x20;

   * `orders_raw.csv` (동봉된 원본 파일 그대로 사용)&#x20;
   * `orders_summary.csv` (파이프라인 실행 결과)&#x20;
3. **README 파일 (필수)**&#x20;

   * `README.md` 권장&#x20;
   * 다음 내용을 포함해 주세요.&#x20;

     * 사용한 Airflow 버전(대략이면 충분)&#x20;
     * 로컬 환경 구성 방법 (Docker/WSL 등 어떤 방식을 택했는지)&#x20;
     * Airflow를 실행하는 방법&#x20;

       * 예: `docker-compose up` 또는 `airflow webserver`, `airflow scheduler` 등&#x20;
     * DAG를 확인/실행하는 방법&#x20;

       * Airflow Web UI 접속 경로 (예: `http://localhost:8080`)&#x20;
       * DAG를 수동 트리거하는 방법 안내&#x20;
     * 파이프라인 전체 흐름 설명 (간단한 다이어그램 또는 글)&#x20;
4. **스크린샷**&#x20;

   * Airflow Web UI에서 아래 화면을 캡처하여 포함해 주세요.&#x20;

     * `orders_daily_summary` DAG가 등록된 화면&#x20;
     * DAG를 한 번 실행한 뒤, **모든 Task가 성공(Success)** 상태인 Tree View 또는 Graph View 화면&#x20;

> 코드와 문서는 **한국어/영어 모두 상관없습니다.**
> &#x20;다만, README는 읽는 사람이 쉽게 이해할 수 있도록 목차, 제목 등을 활용해 정리해 주세요.

***

## 6. 평가 기준

아래 항목을 중심으로 종합적으로 평가합니다.

1. **Airflow 이해도**&#x20;

   * DAG / Task / Operator / 의존성 개념을 제대로 사용했는지&#x20;
   * 스케줄, start\_date, catchup 등의 기본 설정이 적절한지&#x20;
2. **코드 품질**&#x20;

   * 함수 및 변수 이름이 역할을 잘 설명하는지&#x20;
   * 파일 구조와 코드가 지나치게 복잡하지 않고 읽기 쉬운지&#x20;
   * 불필요한 하드코딩을 피하려고 노력했는지&#x20;
3. **데이터 처리 로직**&#x20;

   * 요구한 집계 로직(일자별 total\_sales, order\_count)을 정확히 구현했는지&#x20;
   * 에러 상황(파일 없음 등)에 대한 최소한의 방어 로직이 있는지&#x20;
4. **환경 구성 & 실행 재현성**&#x20;

   * README만 보고도 평가자가 로컬에서 Airflow를 띄우고 DAG를 실행할 수 있는지&#x20;
   * Windows 환경 제약 속에서 적절한 방법(Docker/WSL 등)을 선택하고 정리했는지&#x20;
5. **커뮤니케이션 & 문서화**&#x20;

   * README와 코드 주석이 과제 의도와 구현 내용을 잘 전달하는지&#x20;
   * 스크린샷, 폴더 구조 등이 과제를 이해하는 데 도움이 되는지&#x20;

***

## 7. 기타 안내

* 과제 수행 중 공식 문서, 블로그, Stack Overflow 등 다양한 자료를 참고해도 됩니다.&#x20;
* 다만, 코드/설명을 외부 예제를 그대로 복사하기보다는, **본인이 이해한 방식으로 재구성해 주시기** 바랍니다.&#x20;
* 과제 수행 예상 시간은 대략 **3\~5시간** 내외를 예상하고 있습니다. 본인의 속도에 따라 달라질 수 있습니다.


### 🎯 과제의 요구사항

- 과제에 함께 기재함.
