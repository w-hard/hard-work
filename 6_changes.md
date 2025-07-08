В системе есть несколько критических операций, после выполнения которых надо посылать сообщение в стороннюю систему (аудит).
Мне нужно было реализовать новый тип сообщения аудита и его отправку после нужной операции.
Оказалось, что уже есть 5 типов сообщений аудита, которые наследуются от общего предка AuditMessage.
И есть 5 сервисов, у каждого публичный метод, выполняющий некую логику, а в середине этого метода находится операция, на основе результата которой надо создать сообщение аудита и отправить его.
В примере ниже после операции sendDocumentForSigning отсылается сообщение аудита.
```
@Service
public class DocumentService {
    private DocumentSigningSystem documentSigningSystem;
    private AuditService auditService;
    
    public void signDocument(long documentId) {
        ...
        SigningResultDto result = documentSigningSystem.sendDocumentForSigning(document);
        auditService.sendAuditMessage(new DocumentSignedAuditMessage(
            document.getDocumentId(),
            result.getSinger())
        );
        ...
    }

}
```
Изначально мне показалось не очень правильно перемешивать бизнес логику и функционал аудита, который выглядит как cross cutting concern.
Подумала, может быть сделать аудит с помощью аннотаций?
Для этого нужно было в каждом случае декомпозировать исходный метод, вынести операцию, подлежащую аудиту в отдельный метод, и уже его пометить аннотацией.
А параметры для сообщения аудита бы брались из возвращаемого значения, и из параметров, передаваемых в метод. Это можно сделать с помощью библиотеки aspectJ.

Я представляла это как-то так:
```
public void signDocument (long document Id) {
    ...
    sendDocumentToSign(document);
    ...
}
@Audit(message = DocumentSignedAuditMessage.class)
SigningResultDto sendDocumentToSign(Document document) {
    return documentSigningSystem.sendDocumentFonSigning(document);
}
```
Но возникла проблема при декомпозиции. Если просто выделить метод sendDocumentToSign, то он напрашивался быть приватным.
А в этом случае в Spring не работают аннотации.

Либо же надо было декомпозировать методы так, чтобы нужная операция представляла собой полноценный публичный метод, что скорее всего потребовало бы вынести ее в отдельный сервис, чтобы не нарушался single responsibility principle.
Но т.к. исходные методы были довольно громоздкими и не очень хорошо покрыты тестами, показалось, что такой рефакторинг будет слишком трудным и чреватым ошибками.
Поэтому решила просто создать отдельный сервис, в который вынести все операции аудита. Там будет происходить инициализация сообщений разного типа.
В вызывающих классах останется только вызов соответствующего метода из сервиса AuditMessageService.
```
public class AuditMessageService {
    private AuditService auditService;
    
    public void sendDocumentSignedAuditMessage(Document document, SigningResultDto result) {
        auditService.sendAuditMessage(new DocumentSignedAuditMessage (
            document.getDocumentId()
            result.getSigner()
    );
    
    public void sendAssetCreatedAuditEvent(AssetEntity asset, List<Params> params) {
        auditservice.sendAuditMessage(new AssetCreatedAuditMessage(
            asset.getCode()
            params.stream().mар(Pаram::toString).join(",")
        )
    );
    
    //другие методы, инициализирующие разные типы сообщений
}
```
Мне кажется, в результате такого рефакторинга код получился более читабельным. Не нужно выискивать по всей кодовой базе, где создается то или иное сообщение. 
Вызывающие методы стали более читабельны за счет того, что удалены несколько строк кода, не относящегося непосредственно к бизнес-логике. 
