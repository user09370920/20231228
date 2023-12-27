# 어플리케이션 정리
## 실행 전 필수체크사항
1. MySQL 설치 : com.client_schedule.database.ConnectionMySQL 참고
2. databaseName : client_schedule
3. 파라미터 connectionTimeZone=LOCAL
4. username : sqlUser
5. password : Passw0rd!

## 테이블 생성 및 초기데이터 세팅
1. 제공 된 3. Using JDBC to manipulate an external SQL Database_2023_11_17_08_57_49.zip 압축 해제 후
2. 테이블생성 : C195_db_ddl.txt
3. 데이터생성 : C195_db_dml.txt

## 실행방법
1. MainApplication.java 파일 오픈 후 Run
2. 아이디와 패스워드는 제공 된 아이디(1) , 패스워드(test) 로 로그인

## 다른 어플리케이션에 없는 기능(추가점 가능한 부분)
### Sql메모리 관리
1. 자바 및 자바FX라이브러리 외 다른 라이브러리를 사용하지 못하여 MyBatis에서 xml에 SQL을 선언한것과 유사하게 구현
2. sql.xml 에 어플리케이션에서 사용할 sql을 넣고 초기 로드 시 public static 으로 선언 된 hashmap에 xml id="" 값으로 쿼리문 할당
   
```java
SqlParser.java

// static으로 선언하여 객체생성 없이 CommonDao에서 사용가능하도록 함.
public static HashMap<String,String> sqlMap = new HashMap<String,String>();

Document document = builder.parse(CommonUtil.getResource("sql.xml"));
Element root = document.getDocumentElement();
NodeList stmtList = root.getElementsByTagName("STMT");
sqlMap.clear();
for( int i = 0 ; i < stmtList.getLength() ; i++ ){
  Element element = (Element) stmtList.item(i);
  // xml의  STMT태그의 id속성을 키로하여 쿼리문을 가져와 hashmap에 넣는다
  String sqlId = element.getAttribute("id");
  String sqlStmt = element.getTextContent();
  sqlMap.put(sqlId, sqlStmt);
}
```
3. CommonDao 에서 hashmap.get(xml id값)으로 쿼리문을 읽어와서 사용함

```java
CommonDao.java

ps = ConnectionMySQL.connection.prepareStatement(SqlParser.sqlMap.get("selectCustomerById"));

```

### Appointment Window에 3개의 시간대표시하여 스케쥴 관리 용이하도록 설정
```java
AppointmentController.java

@Override
public void initialize(URL url, ResourceBundle resourceBundle){
  // 생략
  // 1초마다 3개의 시간대 업데이트
  Timeline timeline = new Timeline(new KeyFrame(Duration.seconds(1), event -> updateTime()));
  timeline.setCycleCount(Animation.INDEFINITE);
  timeline.play();

  // 생략

}

// 3개의 시간대(로컬, UTC, ET) 표시함
public void updateTime(){
  LocalDateTime currentNow = LocalDateTime.now();
  LocalDateTime utcNow = CommonUtil.localToUtc(currentNow);
  LocalDateTime etNow = CommonUtil.localToEt(currentNow);

  DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

  currentZoneLbl.setText(currentNow.format(formatter));
  utcZoneLbl.setText(utcNow.format(formatter));
  etZoneLbl.setText(etNow.format(formatter));
}

```
   
