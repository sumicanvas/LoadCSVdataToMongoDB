# LoadCSVdataToMongoDB
# MongoDB Atlas 테스트 데이터 로딩 가이드

이 문서는 `kbtestdata/generated_utf8` 폴더에 생성한 UTF-8 탭 구분 파일을 MongoDB Atlas Cluster로 로딩하는 방법을 정리한 가이드입니다.

## 생성된 테스트 데이터

파일 위치:

```text
kbtestdata/generated_utf8/
```

파일 목록:

```text
news_mast_utf8_tab.csv
news_jmcode_utf8_tab.csv
news_cont_i_utf8_tab.csv
```

파일 형식:

```text
Encoding: UTF-8
Delimiter: Tab (\t)
Header: 있음
```

생성 건수:

```text
news_mast   : 24 rows
news_jmcode : 38 rows
news_cont_i : 63 rows
```

## 전제 조건

로컬에 MongoDB Database Tools가 설치되어 있어야 합니다.

설치 확인:

```sh
mongoimport --version
```

MongoDB Shell도 있으면 로딩 후 검증에 사용할 수 있습니다.

```sh
mongosh --version
```

## Atlas 접속 문자열 준비

MongoDB Atlas UI에서 다음 경로로 접속 문자열을 가져옵니다.

```text
Atlas Cluster > Connect > Drivers 또는 Compass
```

접속 문자열 형식:

```text
mongodb+srv://<user>:<password>@<cluster-url>/<database>?retryWrites=true&w=majority
```

환경변수로 설정합니다.

```sh
export MONGODB_URI='mongodb+srv://<user>:<password>@<cluster-url>/kbpoc?retryWrites=true&w=majority'
```

주의: 실제 사용자명과 비밀번호가 포함된 URI는 GitHub에 커밋하지 마세요.

## TSV 파일 로딩

생성한 파일은 탭 구분 파일이므로 `mongoimport`에서 `--type tsv`와 `--headerline`을 사용합니다.

### 1. NEWS_MAST 로딩

```sh
mongoimport \
  --uri "$MONGODB_URI" \
  --collection news_mast \
  --type tsv \
  --headerline \
  --file /Users/sumi.ryu/Documents/opencode/kbtestdata/generated_utf8/news_mast_utf8_tab.csv
```

### 2. NEWS_JMCODE 로딩

```sh
mongoimport \
  --uri "$MONGODB_URI" \
  --collection news_jmcode \
  --type tsv \
  --headerline \
  --file /Users/sumi.ryu/Documents/opencode/kbtestdata/generated_utf8/news_jmcode_utf8_tab.csv
```

### 3. NEWS_CONT_I 로딩

```sh
mongoimport \
  --uri "$MONGODB_URI" \
  --collection news_cont_i \
  --type tsv \
  --headerline \
  --file /Users/sumi.ryu/Documents/opencode/kbtestdata/generated_utf8/news_cont_i_utf8_tab.csv
```

## 기존 데이터를 지우고 다시 로딩하는 경우

동일 컬렉션을 비우고 다시 넣으려면 `--drop` 옵션을 추가합니다.

```sh
mongoimport \
  --uri "$MONGODB_URI" \
  --collection news_mast \
  --type tsv \
  --headerline \
  --drop \
  --file /Users/sumi.ryu/Documents/opencode/kbtestdata/generated_utf8/news_mast_utf8_tab.csv
```

`news_jmcode`, `news_cont_i`도 동일하게 `--drop`을 추가하면 됩니다.

## 로딩 검증

Atlas에 접속합니다.

```sh
mongosh "$MONGODB_URI"
```

건수를 확인합니다.

```js
db.news_mast.countDocuments()
db.news_jmcode.countDocuments()
db.news_cont_i.countDocuments()
```

예상 결과:

```text
news_mast   : 24
news_jmcode : 38
news_cont_i : 63
```

샘플 데이터를 확인합니다.

```js
db.news_mast.findOne()
db.news_jmcode.findOne()
db.news_cont_i.findOne()
```

최신 뉴스 기준으로 확인합니다.

```js
db.news_mast.find({}, { _id: 0 }).sort({ NEWSCODE: -1 }).limit(5)
```

## 참고: 최종 news 컬렉션 모델링

원천 파일은 Oracle 테이블 구조에 맞춰 3개 컬렉션으로 로딩합니다.

```text
news_mast
news_jmcode
news_cont_i
```

최종 검색용 모델은 하나의 `news` 컬렉션으로 비정규화하는 것을 전제로 합니다.

예상 구조:

```js
{
  _id: ISODate("2026-06-25T15:22:57.000Z"),
  ymd: "20260625",
  seqno: "15225701",
  newscode: "2026062515225701",
  dgubun: "4",
  title: "HMM, 벌크선 비중 확대에 선대 다변화 기대감 부각",
  contents: "HMM은 벌크선과 가스선 발주를 확대하며...",
  shcode: ["011200"]
}
```

이후 Atlas Search를 적용할 컬렉션은 `news`입니다.

## 참고: 일반 조회용 인덱스

검색어가 없는 시나리오에서는 Atlas Search보다 일반 인덱스 기반 `find()`가 적합합니다.

```js
db.news.createIndex({ _id: -1 })
db.news.createIndex({ dgubun: 1, _id: -1 })
db.news.createIndex({ shcode: 1, _id: -1 })
db.news.createIndex({ shcode: 1, dgubun: 1, _id: -1 })
```

`shcode`가 배열이어도 다음 조건으로 배열 포함 검색이 가능합니다.

```js
db.news.find({ shcode: "005930" }).sort({ _id: -1 }).limit(100)
```



