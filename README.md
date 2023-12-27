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

## 요구사항 부분
### 1. 로그인 폼에 다국어 처리
```java
 public void initialize(URL url, ResourceBundle resourceBundle){

   // 생략

   // 현재 시간대 가져오기
   String zoneId = CommonUtil.getZoneId();

   // 기본 영문
   String messageProperties = "message_en" ;

   // 시간대가 Europe/Paris 라면 message_fr을 읽어 프랑스어로 라벨 및 메세지 표시
   if( zoneId.equals("Europe/Paris") ){
      messageProperties = "message_fr" ;
   }
   ResourceBundle resourceBdl = ResourceBundle.getBundle(messageProperties);

   systemName.setText(resourceBdl.getString("SYSTEMNAME"));
   idLbl.setText(resourceBdl.getString("LABEL.ID"));
   passwordLbl.setText(resourceBdl.getString("LABEL.PASSWORD"));
   loginBtn.setText(resourceBdl.getString("BUTTON.LOGIN"));
   closeBtn.setText(resourceBdl.getString("BUTTON.CLOSE"));

   MSG_INPUT_ID = resourceBdl.getString("LOG00001");
   MSG_INPUT_PASSWORD = resourceBdl.getString("LOG00002");
   MSG_CHECK_ID = resourceBdl.getString("LOG00003");
   MSG_CHECK_PASSWORD = resourceBdl.getString("LOG00004");
}
```

### 2. Customer 생성,수정,삭제
   - 조회 : CustomerController > selectCustomer
   - 생성 : CustomerController > createCustomerClicked > CustomerFormController > saveBtnClicked
   - 수정 : CustomerController > modifyCustomerClicked > CustomerUpdateFormController > modifyBtnClicked
   - 삭제 : CustomerController > deleteCustomerClicked

### 3. Customer를 삭제하기 전 Appointment가 있는지 확인
```java
CustomerController.java

public void deleteCustomerClicked() throws Exception {

   // 생략

   // 삭제 전 appointment가 있는지 체크
   int appointmentCount = commonDao.customersDeleteCheck(customersVo);
   if( appointmentCount > 0 ){
      alert("Before deleting a customer, you must first delete the customer's appointment.") ;
      return ;
   }else{
      // 생략
   }

```

### 4. combobox에 countries, divisions, contact 등 자동로딩
```java
   CustomerFormController.java

   // contries 로딩
   countryIdCmb.setItems(commonDao.selectCountries());

   // countries 가 변경 될 경우 Country_ID에 맞는 divisions 로딩
   countryIdCmb.getSelectionModel().selectedItemProperty().addListener(new ChangeListener<CountriesVo>() {
       @Override
       public void changed(ObservableValue<? extends CountriesVo> observable, CountriesVo oldValue, CountriesVo newValue) {
           if (newValue != null) {
               // 선택된 아이템이 변경되었을 때의 작업 수행
               System.out.println("선택된 아이템의 키: " + newValue.getCountryId());
               try {
                   divisionIdCmb.setItems(commonDao.selectFirstLevelDivisions(newValue.getCountryId()));
               } catch (Exception e) {
                   throw new RuntimeException(e);
               }
           }
       }
   });
```

```java
   AppointmentFormController.java
   // contact 로딩
   contactCmb.setItems(commonDao.selectContacts());
```

### 5. 시간 대 관련(중요함)
1. 기본로직
   - display : 현재시간대
   - db : UTC
   - 시간체크 : ET

2. db 저장
   ```java
      AppointmentFormController.java
   
      // 현재시간 대를 UTC로 변환하여 VO에 담는다
      LocalDateTime startUtcDateTime = CommonUtil.localToUtc(startLocalDateTime);
      LocalDateTime endUtcDateTime = CommonUtil.localToUtc(endLocalDateTime);
      
      appointmentsVo.setStart(startUtcDateTime);
      appointmentsVo.setEnd(endUtcDateTime);

   ```
3. 비지니스 시간 체크(08~22) ET기준
   ```java
      AppointmentFormController.java

      // form에서 날짜,시간,분을 문자열 하나로 만든 후 LocalDateTime으로 변환
      LocalDateTime startLocalDateTime = CommonUtil.stringToDateTime(startDateFormat+startHour+startMin);
      LocalDateTime endLocalDateTime = CommonUtil.stringToDateTime(endDateFormat+endHour+endMin);

      // 변환한 시간을 ET로 다시 변환하여 시간체크
      if( !CommonUtil.checkAgainstET(startLocalDateTime, endLocalDateTime) ){
         StringBuilder msg = new StringBuilder();
         msg.append("Business hours are 8:00 AM to 10:00 PM Eastern Time (ET).")
         .append("\n").append("\n")
         .append("▶ set business hours")
         .append("\n")
         .append(" - Start : "+CommonUtil.getDateDbFormat(startLocalDateTime,"yyyy-MM-dd HH:mm"))
         .append("\n")
         .append(" - End : "+CommonUtil.getDateDbFormat(endLocalDateTime,"yyyy-MM-dd HH:mm"));
         alert(msg.toString());
         return ;
      }

      CommonUtil.java
      public static boolean checkAgainstET(LocalDateTime startLocalDateTime , LocalDateTime endLocalDateTime){
         LocalDateTime startETDateTime = localToEt(startLocalDateTime) ;
         LocalDateTime endETDateTime = localToEt(endLocalDateTime) ;
         
         int etStartHour = startETDateTime.getHour();
         int etEndHour = endETDateTime.getHour();

         // 시작과 종료 시간이 설정한 8~22 안에 들어있는지 체크 , BUSINESS_START_HOUR : 8 , BUSINESS_END_HOUR : 22
         return etStartHour >= BUSINESS_START_HOUR && etEndHour <= BUSINESS_END_HOUR ;
      }

   ```

   4. 조회 시
      - 저장은 UTC이지만 데이터베이스 connectionTimeZone=LOCAL 로컬설정이여서 자동으로 현재시간대로 변환되어 보여줌.
      - connectionTimeZone=LOCAL <- 이 설정을 변경하면 시간변환관련 로직 전체를 수정하여야 함.
     
### 6. 로그파일 생성관련
- 로그인 실패, 성공 시 프로젝트 루트 디렉토리에 로그파일 기록
  * 파일명 : login_activity.txt
  ``` java
      CommonUtil.java
     public static void writerLoginLog(String logMessage){
           // 프로젝트 루트 디렉토리 경로
           String projectRoot = System.getProperty("user.dir");
   
           // 로그 파일 경로
           String logFilePath = projectRoot + "/login_activity.txt";
   
           // 날짜 및 시간 포맷 지정
           SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
           String formattedDate = dateFormat.format(new Date());
   
           // 로그 메시지 생성
           String logEntry = formattedDate + " - " + logMessage + "\n";
   
           try (BufferedWriter writer = new BufferedWriter(new FileWriter(logFilePath, true))) {
               // 로그 파일에 메시지 추가
               writer.write(logEntry);
           } catch (IOException e) {
               System.err.println("로그를 기록하는 중에 오류가 발생했습니다: " + e.getMessage());
           }
       }
  ```
  - 로그파일 내용
    ```txt
   2023-12-10 10:54:43 - [FAIL] There is no existing ID
   2023-12-10 10:55:12 - [FAIL] There is no existing ID
   2023-12-10 18:24:32 - [FAIL] There is no existing ID
   2023-12-10 18:24:58 - [SUCCESS] LOGIN
   2023-12-10 18:26:44 - [FAIL] Passwords do not match
   2023-12-10 18:26:47 - [SUCCESS] LOGIN
   2023-12-10 18:52:58 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 22:45:03 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 22:46:45 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 22:48:49 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 22:49:36 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 22:50:58 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 22:52:42 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 22:54:00 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 22:55:12 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 22:57:24 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 22:59:25 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 23:03:31 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 23:38:54 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 23:40:25 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 23:41:59 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 23:44:52 - [SUCCESS] LOGIN - User_ID[1]
   2023-12-10 23:45:30 - [SUCCESS] LOGIN - User_ID[1]
   // 생략
    ```

### 7. lamda 식 두개이상 적용
```java
   AppointmentController.java

   // 일자, 월, 주를 선택하면 tableview의 컬럼명이 동적으로 변경가능하도록 수정
   List<RadioButton> radioButtons = Arrays.asList(radioNormal, radioMonth, radioWeek);
   
   List<String> labels = Arrays.asList("DATE", "MONTH", "WEEK");
   
   IntStream.range(0, radioButtons.size()).forEach(i -> {
      boolean isSelected = (viewIndex == i);
      radioButtons.get(i).setSelected(isSelected);
      Start.setText("START(" + labels.get(viewIndex) + ")");
      End.setText("END(" + labels.get(viewIndex) + ")");
   });
```

```java

   AppointmentFormController.java

   // 시,분 combobox 에 각각 0~23, 0~59 값이 할당되도록 적용
   startHourCmb.setItems(FXCollections.observableArrayList(
           IntStream.rangeClosed(0, 23)
                   .mapToObj(i -> (i < 10) ? "0" + i : String.valueOf(i))
                   .collect(Collectors.toList())
   ));

   startMinCmb.setItems(FXCollections.observableArrayList(
           IntStream.rangeClosed(0, 59)
                   .mapToObj(i -> (i < 10) ? "0" + i : String.valueOf(i))
                   .collect(Collectors.toList())
   ));

```

### 8. 로그인 후 15분 이내 Appointment가 있을 경우 알림
```java

   LoginController.java

   Platform.runLater(() -> {
         List<AppointmentsVo> checkList = null;
         try {
             // 로그인 한 user id가 등록한 appointment list 가져오기
             checkList = commonDao.userAppointmentCheck(SqlParser.userMap.get("usersVo").getUserId());
         } catch (Exception ex) {
             throw new RuntimeException(ex);
         }
         ArrayList<AppointmentsVo> appointmentsVos = new ArrayList<>();
         NoticeVo noticeVo = new NoticeVo();
         boolean isAppointment = false ;

         // 현재시간을 ET로 변환
         LocalDateTime etCurrent = CommonUtil.localToEt(LocalDateTime.now());
   
         for( AppointmentsVo appointmentsVo : checkList ){
   
             LocalDateTime originStart = appointmentsVo.getStart() ;
             LocalDateTime originEnd = appointmentsVo.getEnd() ;

             // db저장값(UTC)을 ET로 변환
             LocalDateTime etStart  =  CommonUtil.localToEt(CommonUtil.utcToLocal(originStart));
             LocalDateTime etEnd  = CommonUtil.localToEt(CommonUtil.utcToLocal(originEnd));
   
             String startView = CommonUtil.getDateDbFormat(etStart,"yyyy-MM-dd HH:mm");
             String endView =  CommonUtil.getDateDbFormat(etEnd,"yyyy-MM-dd HH:mm");

             // 현재시간과 시작일시의 차이를 구함
             long minutesDifference = ChronoUnit.MINUTES.between(etCurrent, etStart);
   
             // 15분 이내인지 확인
             if (minutesDifference >= -15 && minutesDifference <= 15) {
                 appointmentsVo.setStart(etStart);
                 appointmentsVo.setEnd(etEnd);
                 appointmentsVo.setStartView(startView);
                 appointmentsVo.setEndView(endView);
                 appointmentsVos.add(appointmentsVo);
                 isAppointment = true ;
             } else {
                 continue;
             }
         }
   
         noticeVo.setAppointmentExist(isAppointment);

         // 알림메세지 설정
         if( isAppointment ){
             noticeVo.setEtLocalDateTime(etCurrent);
             noticeVo.setAppointmentsVos(appointmentsVos);
             noticeVo.setNoticeMessage("Notice\n▶ There is an appointment scheduled within 15 minutes.");
         }else{
             noticeVo.setNoticeMessage("Notice\n▶ There are no appointments scheduled within 15 minutes.");
         }
   
         FXMLLoader loaderNotice = new FXMLLoader();
         loaderNotice.setLocation(MainApplication.class.getResource("notice-view.fxml"));
         Parent rootNotice = null;
         try {
             rootNotice = (Parent) loaderNotice.load();
         } catch (IOException ex) {
             throw new RuntimeException(ex);
         }
         Scene sceneNotice = new Scene(rootNotice);
   
         NoticeController noticeController = loaderNotice.getController();
         noticeController.makeNotice(noticeVo);
         try{
             // 1초 지연 후 알림윈도우 표시
             Thread.sleep(500);
         }catch(Exception ee){
   
         }
         Stage stageNotice = new Stage();
         stageNotice.setTitle("Notice");
         stageNotice.initModality(Modality.APPLICATION_MODAL);
         stageNotice.setScene(sceneNotice);
         stageNotice.show();
     });
```

### 9. 기타
   - 그 외 사항은 직접 UI실행 하면서 확인

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
   
