Цикломатическая сложность (ЦС) для приведенных методов измерялась с помощью встроенной функции intellij "Overly complex method".

1. Метод 1: исходная ЦС = 9. Итоговая ЦС = 4
2. Метод 2: исходная ЦС = 6. Итоговая ЦС = 1
3. Метод 3: исходная ЦС = 5. Итоговая ЦС = 2

### Метод 1
ЦС исходного метода = 9.
```java
private AdapterResponse getOraDBAppMonitStageResponse(DelegateExecution execution,  
                                                      Map<String, String> adapterParamsMap,  
                                                      Map<String, String> processVariablesMap) {  
  
    AdapterResponse adapterResponse;  
    String requestedSystemName = (String) execution.getVariable(SYSTEM_NAME);  
  
    adapterResponse = adapterService.oradbappRequest(requestedSystemName, adapterParamsMap, processVariablesMap);  
    LocalDateTime adapterLastDate = adapterResponse.getParams().getLastDate();  
    String processStatus = adapterResponse.getParams().getProcessStatus();  
    execution.setVariable(PROCESS_STATUS,Optional.ofNullable(adapterResponse.getParams()  
            .getProcessStatus()  
            .toString())  
            .orElse(""));  
    adapterLastDate = adapterLastDate.plus(lastModifiedDatetimePeriod, ChronoUnit.SECONDS);  
  
    if ((Objects.equals(adapterResponse.getCode(), CODE_ADAPTER_SUCCESS)  
            && processStatus.matches("[2,-1]"))  
            || adapterLastDate == null) {  
        execution.setVariable(ERROR_MESSAGE, adapterResponse.getMessage());  
        adapterResponse.setCode(CODE_ERROR_EXTERNAL_SYSTEM);   
    } else if (Objects.equals(adapterResponse.getCode(), CODE_ADAPTER_SUCCESS)  
            && processStatus.matches("[3,5]")  
            && adapterLastDate != null) {  
        execution.setVariable(LAST_DATE, adapterLastDate);  
        adapterResponse.setCode(CODE_ADAPTER_SUCCESS);  
    } else if (Objects.equals(adapterResponse.getCode(), CODE_ADAPTER_SUCCESS)  
            && processStatus.matches("[0]")) {  
        execution.setVariable(LAST_DATE, adapterLastDate);  
        adapterResponse.setCode(CODE_ADAPTER_REPEATE);  
    } else {  
        adapterResponse.setCode(AnalyzeLastDate.checkLagTime(execution, adapterLastDate, processStatus));  
    }  
  
    return adapterResponse;  
}
```

С первого взгляда видно, что для снижения ЦС необходимо поработать с фрагментом кода, где реализовано ветвление (if/else). Согласно правил в коде нельзя использовать else и любые цепочки else if. Для начала нужно избавиться от них. Так же в каждом полученном блоке if нужно добавить return, чтобы завершить выполнение метода.
Такие изменения не снизят ЦС, она все еще будет = 9, но стилистически код выглядит лучше и воспринимается проще.
Углубившись в фрагменты кода if, можно заметить, что присутствует скрытая вложеность if друг в друга.
в трех местах есть код вида.
```java
if (a && B)
```
фактически это
```java
if (a)
    if (b)
```

ЦС обоих фрагментов = 2. Если выделить проверку этих условий в отдельный метод, то ЦС исходного метода снизится на 2. Да, появится новый небольшой метод с ЦС = 2. Но в итоге ЦС исходного метода будет ниже.

Следующий шаг снижения ЦС - выделить в отдельные методы код, который выполняется в каждой ветке.
Т.к ветки 4, то и метода можно выделить 4.
```java
private AdapterResponse handleErrorResponse(DelegateExecution execution, AdapterResponse adapterResponse) {
	execution.setVariable(ERROR_MESSAGE, adapterResponse.getMessage());
	adapterResponse.setCode(CODE_ERROR_EXTERNAL_SYSTEM);
	return adapterResponse;
}

private AdapterResponse handleSuccessResponse(DelegateExecution execution, AdapterResponse adapterResponse, Optional<LocalDateTime> adapterLastDate) {
	adapterLastDate.ifPresent(date -> execution.setVariable(LAST_DATE, date));
	adapterResponse.setCode(CODE_ADAPTER_SUCCESS);
	return adapterResponse;

}

private AdapterResponse handleRepeatResponse(DelegateExecution execution, AdapterResponse adapterResponse, Optional<LocalDateTime> adapterLastDate) {
	adapterLastDate.ifPresent(date -> execution.setVariable(LAST_DATE, date));
	adapterResponse.setCode(CODE_ADAPTER_REPEATE);
	return adapterResponse;
}


private AdapterResponse handleAnalyzeLastDateResponse(DelegateExecution execution, AdapterResponse adapterResponse, Optional<LocalDateTime> adapterLastDate, String processStatus) {
	adapterResponse.setCode(AnalyzeLastDate.checkLagTime(execution, adapterLastDate, processStatus));
	return adapterResponse;
}
```
Несложно заметить, что первые 3 метода по сути выполняют идентичную работу, только с разными параметрами, значит можем объединить их в один. Четвертый метод имеет немного отличающееся содержание и набор параметров. По сути он может быть полиморфен.
После этих изменений ЦС исходного метода = 4. Итоговый вариант кода со всеми добавленными методами:
#### Итоговый код 1
```java
private AdapterResponse getOraDBAppMonitStageResponse(DelegateExecution execution,  
                                                      Map<String, String> adapterParamsMap,  
                                                      Map<String, String> processVariablesMap) {  
  
  
    String requestedSystemName = (String) execution.getVariable(SYSTEM_NAME);  
    AdapterResponse adapterResponse = adapterService.oradbappRequest(requestedSystemName, adapterParamsMap, processVariablesMap);  
    String adapterCode = adapterResponse.getCode();  
    LocalDateTime adapterLastDate = adapterResponse.getParams().getLastDate();  
    String processStatus = adapterResponse.getParams().getProcessStatus();  
    execution.setVariable(PROCESS_STATUS, Optional.ofNullable(adapterResponse.getParams()  
                    .getProcessStatus()  
                    .toString())  
            .orElse(""));  
    int lastModifiedDatetimePeriod  = 40;  
    adapterLastDate = adapterLastDate.plus(lastModifiedDatetimePeriod, ChronoUnit.SECONDS);  
  
    if (isErrorResponse(adapterCode, processStatus, "[2,-1]", Optional.ofNullable(adapterLastDate))) {  
        return handleResponse(execution, adapterResponse, ERROR_MESSAGE, adapterResponse.getMessage());  
    }  
  
    if (isSuccessResponse(adapterCode, processStatus, "[3,5]", Optional.ofNullable(adapterLastDate))) {  
        return handleResponse(execution, adapterResponse, LAST_DATE, adapterLastDate);  
    }  
  
    if (isSuccessCodeAndProcessStatusMatches(adapterCode, processStatus, "[0]")) {  
        return handleResponse(execution, adapterResponse, LAST_DATE, adapterLastDate);  
    }  
  
    return handleResponse(execution, adapterResponse, adapterLastDate, processStatus);  
}  
  
private AdapterResponse handleResponse(DelegateExecution execution, AdapterResponse adapterResponse, String variableName, String variableValue) {  
    execution.setVariable(variableName, variableValue);  
    adapterResponse.setCode(CODE_ERROR_EXTERNAL_SYSTEM);  
    return adapterResponse;  
}  
  
private AdapterResponse handleResponse(DelegateExecution execution, AdapterResponse adapterResponse, LocalDateTime adapterLastDate, String processStatus) {  
    adapterResponse.setCode(AnalyzeLastDate.checkLagTime(execution, adapterLastDate, processStatus));  
    return adapterResponse;  
}  
  
private boolean isSuccessCodeAndProcessStatusMatches(String adapterCode, String processStatus, String regexp) {  
    return isSuccessCode(adapterCode) && processStatus.matches(regexp);  
}  
  
private boolean isSuccessCode(String adapterCode) {  
    return Objects.equals(adapterCode, CODE_ADAPTER_SUCCESS);  
}  
  
private boolean isErrorResponse(String adapterCode, String processStatus, String regexp, Optional<LocalDateTime> date) {  
    return isSuccessCodeAndProcessStatusMatches(adapterCode, processStatus, regexp) || date.isEmpty();  
}  
  
private boolean isSuccessResponse(String adapterCode, String processStatus, String regexp, Optional<LocalDateTime> date) {  
    return isSuccessCodeAndProcessStatusMatches(adapterCode, processStatus, regexp) && date.isPresent();  
}
```

### Метод 2
```java
public void handleEvent(HistoryEvent historyEvent) {  
  
  if (historyEvent instanceof HistoricActivityInstanceEventEntity) {  
    HistoricActivityInstanceEventEntity activityInstanceEventEntity =  
      (HistoricActivityInstanceEventEntity) historyEvent;  
  
    if (historyEvent.getEventType().equals(HistoryEventTypes.ACTIVITY_INSTANCE_START.getEventName()) || historyEvent.getEventType()  
      .equals(HistoryEventTypes.ACTIVITY_INSTANCE_END.getEventName())) {  
  
      List<String> relevantActivityTypes = new ArrayList<>();  
      relevantActivityTypes.add(ActivityTypes.TASK_USER_TASK);  
      relevantActivityTypes.add(ActivityTypes.INTERMEDIATE_EVENT_MESSAGE);  
      relevantActivityTypes.add(ActivityTypes.INTERMEDIATE_EVENT_TIMER);  
  
      if (relevantActivityTypes.contains(activityInstanceEventEntity.getActivityType())) {  
        log.info("Received <" + historyEvent.getEventType() + "> event for <" + activityInstanceEventEntity.getActivityType() + "> with activityId <" + activityInstanceEventEntity  
          .getActivityId() + ">");  
      }  
    }  
  } else if (historyEvent instanceof HistoricExternalTaskLogEntity) {  
    HistoricExternalTaskLogEntity externalTaskLogEvent = (HistoricExternalTaskLogEntity) historyEvent;  
  
    log.info("Received <" + historyEvent.getEventType() + "> event for external task with activityId <" + externalTaskLogEvent  
      .getActivityId() + ">");  
  }  
}
```

Этот небольшой фрагмент кода интересен тем, что условия вычисляются на основании типа полученного методом объекта. Также у метода не очень высокая ЦС, а значит сделать ее еще ниже должно быть труднее.
В данном случае для снижения ЦС, во - первых, можно вынести сложные выражения из if в отдельные методы (2 и 3 уровни вложености). А затем и вовсе объединить эти условия в одно, тем самым избавимся от части if.
Как и в предыдущем случае ветвлений в коде стало меньше, ЦС снизилась, но появились дополнительные небольшие методы для проверки условий.
Если теперь посмотреть на верхний уровень ветвления, то видно, что в зависимости от типа объекта выполняется разный код, хотя и очень похожий. Чтобы избежать if-ов в данном случае можно воспользоваться полиморфизмом через создание специального интерфейса и с одним методом, реализации которого будут как раз содержать инструкции из блоков if. У такого интерфейса будет всего один метод абстрактный метод, следовательно в терминологии Java он является функциональным. Т.к в стандартной библиотеке Java уже есть подходящий функциональный интерфейс - Consumer<T>, то отдельный интерфейс создавать не нужно. 
Далее каждому типу нужно назначить своего консъюмера с помощью мапы, например.

### Итоговый код 2

```java
private static final Map<Class<?>, Consumer<HistoryEvent>> handlers = new HashMap<>();  
  
static {  
    handlers.put(HistoricActivityInstanceEventEntity.class, event -> {  
        HistoricActivityInstanceEventEntity activityInstanceEventEntity = (HistoricActivityInstanceEventEntity) event;  
        if (checkEventAndActivityTypes(activityInstanceEventEntity)) {  
            log.info("Received <" + activityInstanceEventEntity.getEventType() + "> event for <" + activityInstanceEventEntity.getActivityType() + "> with activityId <" + activityInstanceEventEntity.getActivityId() + ">");  
        }  
    });  
  
    handlers.put(HistoricExternalTaskLogEntity.class, event -> {  
        HistoricExternalTaskLogEntity externalTaskLogEvent = (HistoricExternalTaskLogEntity) event;  
        log.info("Received <" + externalTaskLogEvent.getEventType() + "> event for external task with activityId <" + externalTaskLogEvent.getActivityId() + ">");  
    });  
}  
   
public void handleEvent(HistoryEvent historyEvent) {  
    Consumer<HistoryEvent> handler = handlers.get(historyEvent.getClass());  
    handler.accept(historyEvent);  
}
```
Итоговая ЦС кода стала = 1. Есть ощущение, что можно лучше использовать возможности ФП и не создавать мапу, но не получается так сделать.


### Метод 3
```java
private static void generatePollutantwisePollutionReport(Map<String, List<PollutantDetailsWithCity>> pollutantwisePollutantEntriesForReport, LocalDateTime generatedTime) throws Exception {  
    try {  
        List<String> headers = Arrays.asList("Average", "Max", "Min", "City");  
        Document document = new Document();  
        PdfWriter.getInstance(document, new FileOutputStream("PollutantwisePollutionReport.pdf"));  
        document.open();  
        document.add(new Paragraph("India Pollution Report - Pollutant-wise", catFont));  
        document.add(new Paragraph("Generated on: " + generatedTime, smallBold));  
  
        for(String tableHeader : pollutantwisePollutantEntriesForReport.keySet()) {  
            Paragraph paragraph = new Paragraph(tableHeader);  
            PdfPTable table = new PdfPTable(headers.size());  
            for(String rowHeader : headers) {  
                PdfPCell pdfPCell = new PdfPCell(new Phrase(rowHeader));  
                table.addCell(pdfPCell);  
            }  
            for (PollutantDetailsWithCity tableEntries : pollutantwisePollutantEntriesForReport.get(tableHeader)) {  
                table.addCell(String.valueOf(tableEntries.average));  
                table.addCell(String.valueOf(tableEntries.max));  
                table.addCell(String.valueOf(tableEntries.min));  
                table.addCell(String.valueOf(tableEntries.city));  
            }  
            paragraph.add(table);  
            document.add(paragraph);  
        }  
        document.close();  
    } catch(Exception e) {  
        System.err.println(e);  
    }  
}
```
Вложенные циклы увеличивают ЦС метода. Так же нельзя не обратить внимание на дублирование кода во вложенном for (table.addCell(...)). Если приглядеться, то можно увидеть, что в методе смешаны бизнес-задача - сгенерировать отчет и детали создания этого отчета. То есть нарушен SRP.

Во - первых, нужно избавиться от нарушения SRP и вынести все детали создания отчета в отдельный класс DocumentCreator с методами addTitle(String title), addHeader(String header), addTable(String title, List<String> headers, List<String> entries), addTableCell(PdfPTable table, String header). Для компактности опущу код класса.
Тогда исходный метод нужно разделить на 2

### Итоговый код 3
```java
public void generate(String fileName, Map<String, List<String>> entries, List<String> columnHeaders) throws Exception {  
    try (PDFDocument report = new PDFDocument(fileName)) {  
        report.addTitle(heading);  
        report.addHeader("Generated at: " + dateTime);  
        entries.forEach((tableTitle, values) -> {  
            addTableToReport(columnHeaders, report, tableTitle, values);  
        });  
    }  
}  
  
private void addTableToReport(List<String> columnHeaders, PDFDocument report, String tableTitle, List<String> values) {  
    try {  
        report.addTable(tableTitle, columnHeaders, values);  
    } catch (DocumentException e) {  
        e.printStackTrace();  
        System.exit(-1);  
    }  
}
```
Таким образом удалось избавиться от вложенных циклов for и избавиться от нарушения SRP. Из-за создания отдельного класса DocumentCreator (в который нужно перенести детали создания документа) кода стало больше, как и в первых двух случаях

Выводы:
1. Снижение ЦС какого-то фрагмента кода приводит к увеличению объема кода: необходимо создавать дополнительные методы/классы. При этом код все равно воспринимается легче. Т.к четче видно как функционирует каждый отдельный небольшой фрагмент кода.
2. Иногда недостаточно вносить изменения внутри метода или класса, которому метод принадлежит. Если высокая ЦС метода обусловлена плохим проектированием связанных классов/методов, то полностью избавиться от условных конструкций невозможно (либо мне не удалось понять как это сделать).

Очень многословно и не очень формально думаю о коде. Нужно учиться думать лучше.
Есть ощущение, что не смог полностью раскрыть фозможности ФП при работе со вторым методом.
