# 일일안전당직자모니터링 그리드 Footer 합계 조치율

## 대상

- 화면 XML: `C:\CIP_DEFG_v2.0.0\workspace\cip-defg-saas\src\main\webapp\wqxml\mont\MontInqCompanyPersonInChargeMonitoring.xml`
- 개인 탭 그리드 / DataList: `wgrdIndivActRate` / `wdlIndivActRate`
- 현장 탭 그리드 / DataList: `wgrdProjActRate` / `wdlProjActRate`
- Footer 조치율 컬럼: `GF_EMPHS_ACT_RATE`, `GF_ADD_RISK_ACT_RATE`

## 요구사항

개인별·현장별 상세 결과에 0건 대상자가 포함되면서, 기존 Footer의 조치율 평균식은 전체 조치율을 올바르게 나타내지 못하게 되었다.

또한 조회 결과가 0건이면 WebSquare Footer의 `SUM()` 결과가 숫자 0이 아닌 `NaN`으로 처리될 수 있어, 조치율 칸에 `NaNNaNNaN%`가 표시되는 문제가 있었다.

목표는 다음과 같다.

- 데이터가 있으면 **건수 합계 기준**으로 조치율을 계산한다.
- 조회 결과가 0건이거나 분모 합계가 0이면 `0%`를 표시한다.

## 합계 산식

`AVG("EMPHS_ACT_RATE")`처럼 상세 행 조치율을 평균내지 않는다. 현장·개인별 분모가 달라 평균은 전체 실적을 왜곡할 수 있다.

| 구분 | 분자 | 분모 | Footer 조치율 |
| --- | --- | --- | --- |
| 중점점검 | 실시건수 합계 (`EMPHS_EXEC_CNT`) | 점검건수 합계 (`EMPHS_CHK_CNT`) | 실시건수 합계 / 점검건수 합계 × 100 |
| 추가위험 | 조치건수 합계 (`ADD_RISK_ACT_CNT`) | 지시건수 합계 (`ADD_RISK_DIR_CNT`) | 조치건수 합계 / 지시건수 합계 × 100 |

## Footer expression만으로 처리하지 않은 이유

다음 방식은 사용하지 않는다.

```xml
expression='AVG("EMPHS_ACT_RATE")'
```

상세 조치율의 단순 평균이므로 전체 건수 기준 조치율과 다를 수 있다.

```xml
expression='SUM("EMPHS_EXEC_CNT") / SUM("EMPHS_CHK_CNT") * 100'
```

조회 결과가 0건이면 `SUM()`이 `NaN`이 되어 화면에 `NaNNaNNaN%`가 표시될 수 있다.

`Math.max(...)`, `||` 등의 조건식을 Footer `expression`에 직접 작성하는 방식도 이 화면의 WebSquare Footer expression 파서에서 안정적으로 처리되지 않았다. Footer expression은 공통 함수 호출만 하고, 0건·0분모 처리는 화면 JavaScript에서 수행한다.

## 적용 코드

### 공통 계산 함수

```javascript
/**
 * 그리드 합계 조치율을 계산한다.
 * - 조회 결과가 0건이거나 분모 합계가 0이면 0을 반환하여 화면에 0%로 표시한다.
 * - 중점점검: 합계 실시건수 / 합계 점검건수 * 100
 * - 추가위험: 합계 조치건수 / 합계 지시건수 * 100
 */
scwin.getActRate = function(aoDataList, asNumeratorColId, asDenominatorColId) {
	let nNumerator = 0;
	let nDenominator = 0;

	for(let i = 0; i < aoDataList.getTotalRow(); i++) {
		nNumerator += Number(aoDataList.getCellData(i, asNumeratorColId)) || 0;
		nDenominator += Number(aoDataList.getCellData(i, asDenominatorColId)) || 0;
	}

	if(nDenominator === 0) {
		return 0;
	}

	return nNumerator / nDenominator * 100;
};
```

## 조회 결과 유무에 따른 함수 실행

### 1. 조회 결과가 없을 때

처음 화면을 열었거나 조회 조건에 해당하는 데이터가 없으면 DataList의 행 수는 0이다.

```javascript
aoDataList.getTotalRow(); // 0
```

반복문의 첫 조건은 `0 < 0`이므로 거짓이다. 따라서 반복문 안의 두 누적 코드는 한 번도 실행되지 않는다.

```javascript
for(let i = 0; i < aoDataList.getTotalRow(); i++) {
	// 실행되지 않음
}
```

함수 시작 시 설정한 값이 그대로 유지된다.

```javascript
let nNumerator = 0;
let nDenominator = 0;
```

이후 분모가 0인지 검사하는 조건이 참이므로, 함수는 `0`을 반환한다. Footer의 `displayFormat="###%"`에 의해 화면에는 `0%`가 표시된다.

```javascript
if(nDenominator === 0) {
	return 0;
}
```

이 상황에서는 `Number(...) || 0` 줄 자체가 실행되지 않는다. 처음 조회 전 0% 처리는 `nDenominator`의 초기값과 `return 0` 분기로 이루어진다.

### 2. 조회 결과가 있을 때

행이 하나 이상이면 반복문이 각 행의 분자·분모 건수를 차례로 더한다.

```javascript
nNumerator += Number(aoDataList.getCellData(i, asNumeratorColId)) || 0;
nDenominator += Number(aoDataList.getCellData(i, asDenominatorColId)) || 0;
```

`A || B`는 `A`가 정상적으로 사용할 수 있는 값이면 `A`를 쓰고, 그렇지 않으면 `B`를 쓰는 표현이다. 여기서는 셀 값을 숫자로 바꾼 결과가 이상하면 그 행을 0건으로 계산한다.

| 셀 값 | `Number(셀 값)` 결과 | 실제 누적값 |
| --- | --- | --- |
| `"3"` | `3` | `3` |
| `"0"` | `0` | `0` |
| `undefined` | `NaN` | `0` |
| `"문자"` | `NaN` | `0` |

`NaN`은 “숫자로 계산할 수 없음”이라는 뜻이다. `|| 0`이 `NaN`을 숫자로 변환하는 것은 아니다. `NaN`을 그대로 더하면 전체 합계도 `NaN`이 되므로, 그 대신 0을 선택해 합계 계산을 계속할 수 있게 한다.

```javascript
10 + NaN; // NaN
```

### 탭별 래퍼 함수

```javascript
scwin.getIndivActRate = function(asNumeratorColId, asDenominatorColId) {
	return scwin.getActRate(wdlIndivActRate, asNumeratorColId, asDenominatorColId);
};

scwin.getProjActRate = function(asNumeratorColId, asDenominatorColId) {
	return scwin.getActRate(wdlProjActRate, asNumeratorColId, asDenominatorColId);
};
```

### 개인 탭 Footer

```xml
<w2:column id="GF_EMPHS_ACT_RATE"
    inputType="expression"
    dataType="number"
    displayFormat="###%"
    expression='scwin.getIndivActRate("EMPHS_EXEC_CNT", "EMPHS_CHK_CNT")' />

<w2:column id="GF_ADD_RISK_ACT_RATE"
    inputType="expression"
    dataType="number"
    displayFormat="###%"
    expression='scwin.getIndivActRate("ADD_RISK_ACT_CNT", "ADD_RISK_DIR_CNT")' />
```

### 현장 탭 Footer

```xml
<w2:column id="GF_EMPHS_ACT_RATE"
    inputType="expression"
    dataType="number"
    displayFormat="###%"
    expression='scwin.getProjActRate("EMPHS_EXEC_CNT", "EMPHS_CHK_CNT")' />

<w2:column id="GF_ADD_RISK_ACT_RATE"
    inputType="expression"
    dataType="number"
    displayFormat="###%"
    expression='scwin.getProjActRate("ADD_RISK_ACT_CNT", "ADD_RISK_DIR_CNT")' />
```

## DataList 전달 및 계산 흐름

```text
조회 Submission 응답
  → 개인 탭: wdlIndivActRate / 현장 탭: wdlProjActRate에 행 데이터 적재
  → Grid Footer expression 실행
  → 탭별 래퍼 함수 호출
  → 공통 getActRate()에 해당 DataList 객체 전달
  → getTotalRow(), getCellData()로 분자·분모를 누적
  → 분모가 0이면 0, 아니면 분자 / 분모 * 100 반환
```

`wdlIndivActRate`와 `wdlProjActRate`는 각각 XML의 `<w2:dataList id="...">`로 선언된 WebSquare DataList 객체다.

개인 탭 Footer가 다음 expression을 실행하면:

```javascript
scwin.getIndivActRate("EMPHS_EXEC_CNT", "EMPHS_CHK_CNT")
```

`getIndivActRate()`는 화면에 이미 생성된 `wdlIndivActRate` 객체를 첫 번째 인자로 넣어 공통 함수에 전달한다.

```javascript
scwin.getActRate(wdlIndivActRate, asNumeratorColId, asDenominatorColId)
```

따라서 공통 함수의 `aoDataList`는 개인 탭 DataList와 같은 객체를 가리킨다. 현장 탭은 같은 함수에 `wdlProjActRate`를 전달하므로 계산 로직을 중복하지 않는다.

## 확인 항목

- 조회 0건: 두 조치율 모두 `0%`인지 확인
- 분모 건수 0, 분자 건수 0: `0%`인지 확인
- 정상 데이터: Footer의 중점/추가위험 조치율이 표시된 분자·분모 합계와 일치하는지 확인
- 개인 탭과 현장 탭을 각각 확인
- 소스 반영 후 브라우저 캐시를 무시하고 새로고침하거나 재배포 환경에서 확인
