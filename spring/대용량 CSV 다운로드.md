### 대용량 CSV 다운로드

## Mybatis
mybatis에서 지원하는 resultHandler사용하여 하나씩 stream으로 출력 공통화를 위해 resultHandler를 받는 Consumer객체를 외부에서 받음

CsvStreamPrinter.java의 전체 코드는 최하단에 있음

** CsvStreamPrinter.java / resultHandler 구현 **
```java
/**
* Mybatis로 Data 조회시 사용
* @param os HttpServletResponse outputstream
* @param header csv header
* @param dataSelectFunction ResultHandler를 갖는 data select 메소드 * ex) resultHandler -> sampleMapper.getLogList(searchObject, resultHandler)
*/
 @Transactional(readOnly = true)
 public void printCsvByResultHandler(OutputStream os, CsvPrinterHeader[] header, Consumer<ResultHandler> dataSelectFunction) {

     try (CSVWriter cw = new CSVWriter(new OutputStreamWriter(os, "utf-8"), ',', '"')) {
         writeHeader(os, cw, header);

         dataSelectFunction.accept(resultContext -> {
             try {
                 writeData(resultContext.getResultObject(), cw, header);
             } catch (IOException e) {
                 log.error("get ResultHandler error : {}" , e);
             }
         });

     } catch (Exception e) {
         log.error("getCsvWriter error : {}" , e);
     }
 }
```
** Mapper Interface Method ** <br>
resultHandler를 사용하기위해선 return type을 void로 해야함
```java
void getCsvDataWithResultHandler(@Param("param") String param, ResultHandler resultHandler);
```

** Mapper Xml ** <br>
mysql경우 fetchSize를 Integer.MIN_VALUE 인 -2147483648 로 해주어야함
```xml
<select id="getListWithResultHandler" fetchSize="-2147483648" resultMap="csvDataMap">
  select * from table
</select>
```

** printCsvByResultHandler 호출 ** <br>
```java
Consumer<ResultHandler> dataSelectFunction =
           resultHandler -> csvMapper.getCsvDataWithResultHandler(log, resultHandler);
   csvStreamPrinter.printCsvByResultHandler(os, headers, dataSelectFunction);
```

## JPA
ScrollableResults 객체를 이용하여 하나씩 stream으로 출력

** CsvStreamPrinter.java **
```java
/**
* JPA로 Data 조회시 사용
* @param os HttpServletResponse outputstream
* @param header csv header
* @param results JPA 조회 결과
*/
    @Transactional(readOnly = true)
    public void printCsvByScrollableResults(OutputStream os, CsvPrinterHeader[] header, ScrollableResults results) {

        try (CSVWriter cw = new CSVWriter(new OutputStreamWriter(os, "utf-8"), ',', '"')) {
            writeHeader(os, cw, header);

            while (results.next()) {
                writeData(results.get(0), cw, header);
            }
            results.close();
        } catch (Exception e) {
            log.error("getCsvWriter error : {}", e);
        }
    }
```

ScrollableResults 객체를 생성하기 위해 StatelessSession 객체로부터 생성한 Query를 이용해 JPQL사용 mybatis와 마찬가지로 fetchSize를 Integer.MIN_VALUE로 설정해주어야 stream방식으로 동작함. 만약 return type 객체에 join이 걸려있는 필드가 있다면, fetch type 을 LAZY로 적용해야함. join된 필드의 값에 접근하고 싶으면, jpql query문에 join fetch를 사용하여 명시해줘야함. 공통으로 사용하기 위해 CsvDataRepository 클래스를 정의

** printCsvByScrollableResults 호출 **
```java
try(StatelessSession session = ((Session) entityManager.getDelegate()).getSessionFactory().openStatelessSession()) {
          List<String> joinField = new ArrayList<>();
          joinField.add("event");

          CsvDataRepository csvDataRepository = new CsvDataRepository(session, searchData, returnType);
          csvDataRepository.setJoinField(joinField);

          csvStreamPrinter.printCsvByScrollableResults(os, headers, csvDataRepository.getScrollableResults());
        }catch (Exception e) {
          log.debug("csv download error {}", e);
        }
```

** CsvDataRepository.java **
```java
public class CsvDataRepository {

    private StatelessSession session;
    private Class returnClass;
    private List<String> whereCause;
    private Map<String, Object> conditions;
    private Collection<String> joinField;

    /**
    * CsvDataRepository 인스턴스 생성
    * @param session StatelessSession
    * @param searchData 검색조건
    * @param returnClass 결과객체
    * @return CsvDataRepository 인스턴스
    */
    public CsvDataRepository(StatelessSession session, Object searchData, Class returnClass) {
        this.session = session;
        this.returnClass = returnClass;
        this.conditions = getConditions(searchData);

        // 기본검색 조건식 생성
        this.whereCause = Optional.of(this.conditions)
            .map(this::makeWhereCause)
            .orElse(new ArrayList<>());
    }

    /**
    * 값 가져올 조인 필드
    * StatelessSession은 FetchType.LAZY 일때만 동작해서 query문에 join fetch를 명시 해줘야함
    * @param joinField 조인 필드명 list
    */
    public void setJoinField(Collection<String> joinField) {
        this.joinField = joinField;
    }

    /**
    * 임의 조건식 
    * @param whereCause 조건식
    */
    public void setWhereCause(List<String> whereCause) {
        this.whereCause = whereCause;
    }

    /**
    * ScrollableResults 객체 생성
    * @return ScrollableResults
    */
    @Transactional(readOnly = true)
    public ScrollableResults getScrollableResults() {
        StringBuilder queryBuilder = new StringBuilder();
        queryBuilder.append("select e from ").append(this.returnClass.getSimpleName()).append(" e ");

        if(!CollectionUtils.isEmpty(this.joinField))
            queryBuilder.append(" join fetch ")
                .append(StringUtils.join(this.joinField.stream().map(f -> String.format(" e.%s ", f)).toArray(), " join fetch "));

        if(!CollectionUtils.isEmpty(this.whereCause))
            queryBuilder.append(" where ")
                .append(StringUtils.join(this.whereCause, " and "));

        Query query = this.session
            .createQuery(queryBuilder.toString(), this.returnClass);
        query.setFetchSize(Integer.MIN_VALUE);
        query.setCacheable(false);

        convertValue(this.conditions, this.returnClass);
        this.conditions.keySet().forEach(s -> query.setParameter(s, this.conditions.get(s)));

        return query.scroll(ScrollMode.FORWARD_ONLY);
    }

    /**
    * 검색조건 Map 생성
    * @param searchData 검색조건
    * @return 검색조건 Map
    */
    private Map<String, Object> getConditions(Object searchData) {
        ObjectMapper objectMapper = new ObjectMapper();
        Map<String, Object> conditions = objectMapper.convertValue(searchData, Map.class);
        removeNullSearchData(conditions);

        return conditions;
    }

    /**
    * value 없는 검색조건 제거
    * @param conditions 검색조건
    */
    private void removeNullSearchData(Map<String, Object> conditions) {
        List<String> removeKey = new ArrayList<>();

        conditions.keySet().forEach(k -> {
            if (Objects.isNull(conditions.get(k)) || StringUtils.isEmpty(String.valueOf(conditions.get(k))))
                removeKey.add(k);
        });

        removeKey.forEach(conditions::remove);
    }

    /**
    * JPA에서 Integer, Long 구분 하기때문에 각 field에 맞춰 컨버팅
    * @param conditions 검색조건
    * @param tClass 결과객체 class
    */
    private void convertValue(Map<String, Object> conditions, Class tClass) {
        List<Field> fields = Arrays.asList(tClass.getDeclaredFields());

        conditions.keySet().forEach(k -> {
            Field field = fields.stream()
                .filter(f -> f.getName().equalsIgnoreCase(k))
                .findFirst()
                .orElse(null);

            if(field != null) {
                if(field.getType().getSimpleName().equalsIgnoreCase("integer"))
                    conditions.replace(k, Integer.parseInt(String.valueOf(conditions.get(k))));
                if(field.getType().getSimpleName().equalsIgnoreCase("long"))
                    conditions.replace(k, Long.parseLong(String.valueOf(conditions.get(k))));
            }
        });
    }

    /**
    * Where 조건 생성
    * @param conditions 검색조건
    * @return 쿼리문 where 조건
    */
    private List<String> makeWhereCause(Map<String, Object> conditions) {
        List<String> whereCause = new ArrayList<>();
        conditions.keySet().forEach(k -> {
            DateTimeFormatter df = DateTimeFormatter.ofPattern("yyyyMMdd");
            if ("startDate".equals(k)) {
                whereCause.add(String.format("e.%s >= :%s", "createdAt", k));
                conditions.replace(k, LocalDate.parse(conditions.get(k).toString(), df).atTime(00, 00, 00));
            } else if ("endDate".equals(k)) {
                whereCause.add(String.format("e.%s <= :%s", "createdAt", k));
                conditions.replace(k, LocalDate.parse(conditions.get(k).toString(), df).atTime(23, 59, 59));
            } else {
                whereCause.add(String.format("e.%s = :%s", k, k));
            }
        });

        return whereCause;
    }
}
```

** CsvStreamPrinter.java **
```java
@Slf4j
@Component
public class CsvStreamPrinter {

    @AllArgsConstructor
    public static class CsvPrinterHeader {
        private String name;
        private String key; // 내부 객체 접근은 . 으로 구분
    }

    /**
    * Mybatis로 Data 조회시 사용
    * @param os HttpServletResponse outputstream
    * @param header csv header
    * @param dataSelectFunction ResultHandler를 갖는 data select 메소드
    * ex) resultHandler -> sampleMapper.getLogList(searchObject, resultHandler)
    */
    @Transactional(readOnly = true)
    public void printCsvByResultHandler(OutputStream os, CsvPrinterHeader[] header, Consumer<ResultHandler> dataSelectFunction) {

        try (CSVWriter cw = new CSVWriter(new OutputStreamWriter(os, "utf-8"), ',', '"')) {
            writeHeader(os, cw, header);

            dataSelectFunction.accept(resultContext -> {
                try {
                    writeData(resultContext.getResultObject(), cw, header);
                } catch (IOException e) {
                    log.error("get ResultHandler error : {}" , e);
                }
            });

        } catch (Exception e) {
            log.error("getCsvWriter error : {}" , e);
        }
    }

    /**
    * JPA로 Data 조회시 사용
    * @param os HttpServletResponse outputstream
    * @param header csv header
    * @param results JPA 조회 결과
    */
    @Transactional(readOnly = true)
    public void printCsvByScrollableResults(OutputStream os, CsvPrinterHeader[] header, ScrollableResults results) {

        try (CSVWriter cw = new CSVWriter(new OutputStreamWriter(os, "utf-8"), ',', '"')) {
            writeHeader(os, cw, header);

            while (results.next()) {
                writeData(results.get(0), cw, header);
            }
            results.close();
        } catch (Exception e) {
            log.error("getCsvWriter error : {}", e);
        }
    }

    /**
    * csv 헤더 작성
    * @param os HttpServletResponse outputstream
    * @param cw CSVWriter
    * @param header csv 헤더
    * @throws IOException
    */
    private void writeHeader(OutputStream os, CSVWriter cw, CsvPrinterHeader[] header) throws IOException {
        os.write(0xef);
        os.write(0xbb);
        os.write(0xbf);
        cw.writeNext(Arrays.stream(header).map(h -> h.name).toArray(String[]::new));
        cw.flush();
    }

    /**
    * csv 데이터 작성
    * @param items 데이터 row
    * @param cw CSVWriter
    * @param header csv 헤더
    * @throws IOException
    */
    private void writeData(Object items, CSVWriter cw, CsvPrinterHeader[] header) throws IOException {
        Object[] objects = Arrays.stream(Arrays.stream(header).map(h -> h.key).toArray(String[]::new))
            .map(s -> getFieldValue(items.getClass(), items, s))
            .collect(Collectors.toList())
            .toArray();

        cw.writeNext(Arrays.copyOf(objects, objects.length, String[].class));
        cw.flush();
    }

    /**
    * reflection을 이용하여 value 얻기
    * @param clazz reflection 클래스 정보
    * @param obj 데이터 Object
    * @param name 필드명
    * @return
    */
    private String getFieldValue(Class clazz, Object obj, String name) {
        try {
            Field field;
            if(name.contains(".")){
                String[] str = name.split("\\.");
                Class lowestClass = null;

                for(int i=0; i < str.length - 1; i++) {
                    Field tempField = clazz.getDeclaredField(str[i]);
                    tempField.setAccessible(true);
                    obj = tempField.get(obj);
                    lowestClass = obj.getClass();
                }

                field = lowestClass.getDeclaredField(str[str.length - 1]);
            } else {
                field = clazz.getDeclaredField(name);
            }
            field.setAccessible(true);

            //날짜타입일경우
            if ("LocalDateTime".equals(field.getType().getSimpleName())) {
                LocalDateTime localDateTime = (LocalDateTime) field.get(obj);
                return localDateTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")).toString();
            }

            return field.get(obj).toString();
        } catch (Exception e) {
            return "";
        }
    }
}
```
